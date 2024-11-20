---
title: Docker + Snyk 으로 컨테이너 이미지 취약점 맛보기
lang: ko
layout: post
---

## 들어가며

### Disclaimer

- Last Update: 2024/11/20

- 컨테이너 보안의 중요성 및 기본 개념과 Snyk을 이용해 Docker 컨테이너 이미지의 취약점을 스캔해 살펴보는 글입니다.

- `docker scan` 명령어가 폐기된 이후, Docker에서는 `docker scout`을 이용해 컨테이너 취약점 검사를 제공하고 있습니다. 하지만 이번 글에서는 추후 CI/CD 파이프라인과의 통합을 고려하여, Snyk 및 이를 통한 컨테이너 취약점 검사를 살펴보겠습니다.

- 사전 환경 구성
    - Docker Desktop의 snyk extension 혹은 snyk cli에 대한 설치가 필요합니다.
    - Snyk의 Authentication이 필요하므로 가입이 필요합니다. (해당 Extension에서 이동하거나, `snyk auth` 명령어로 진행 가능합니다.)

<br/>

## Docker 컨테이너 보안의 중요성

클라우드 환경으로의 전환이 일어나면서, 전통적인 애플리케이션 보안을 넘어 컨테이너 보안의 중요성은 높아지고 있습니다. 컨테이너 이미지 내의 취약점은 배포 과정에서의 보안 문제를 일으키거나, 서비스에서 문제를 일으키기 때문입니다. 특히, 한 컨테이너 이미지의 취약점이 악용되면, Docker 네트워크에 연결된 다른 컨테이너가 공격받거나 호스트 시스템이 손상될 수도 있습니다.

### 예시: Bridge 네트워크와 컨테이너 취약점 악용

#### Bridge 네트워크

Bridge 네트워크는 Docker의 기본 가상 네트워크입니다. 각 컨테이너는 Bridge 네트워크 안에서 고유의 가상 사설 IP 주소를 할당 받으며, 통신 역시 Bridge 네트워크를 통해 이루어집니다. 

따라서 외부 네트워크에서는 NAT(Netword Address Translation)을 통해 내부 IP 주소를 외부 IP 주소로 변환하는 과정이 이루어집니다. 이후 Docker 엔진은 컨테이너의 요청을 호스트 시스템의 IP를 사용해 인터넷에 전달합니다. 응답이 온다면, Docker 엔진은 다시 컨테이너의 사설 IP 주소로 변환해 전달합니다.

만약 Bridge 네트워크에서 컨테이너로 올라간 이미지에 취약점이 있다면 어떤 일이 발생할 수 있을까요?

#### 취약점 악용 예시 (CVE-2022-0543)

Bridge 네트워크 내에서 Spring Boot 컨테이너, MySQL 컨테이너, Redis 컨테이너가 이용된다고 가정합시다. Spring Boot는 MySQL을 DB로 이용하고, Redis를 캐시 스토리지로 이용합니다.

공격자는 네트워크 스캔을 통해서 Redis의 기본 포트인 6379가 열려있음을 발견했습니다. 그런데 Redis 이미지가 [CVE-2022-0543](https://nvd.nist.gov/vuln/detail/CVE-2022-0543) 취약점이 오래된 버전이었기에, 공격자는 Redis 컨테이너에서 심볼 덮어쓰기를 통해 Lua 스크립트의 샌드박스 제한을 우회하도록 만들었습니다. 따라서 공격자는 시스템 명령어 실행 권한을 얻었습니다.

따라서 공격자는 Redis에서는 데이터를 조작하고, Spring Boot에서는 이용되는 데이터에 악성 데이터를 삽입해 인증을 우회하거나 API 동작을 방해할 수 있게 됩니다.

물론, 포트 노출과 Redis의 보안이 부족한 상황에서 일어나는 예시라는 점은 알아둬야 합니다. 추가로, CVE-2022-0543 취약점은 Debian/Ubuntu 환경에서 배포될 때 Lua 엔진이 특정 라이브러리의 심볼을 덮어쓰는 방식으로 샌드박스 제한을 우회하는 취약점입니다.

#### 네트워크 취약점 개선

포트 노출 차단, Redis 패치 적용 혹은 이미지 버전 변경, 네트워크 격리, 네트워크 ACL 설정, 어플리케이션 차원 보안 강화 등의 방식으로 네트워크 취약점을 개선할 수 있는데요. 특히 포트 노출 차단과  패치 적용이 가장 시급하다고 말할 수 있겠습니다.

그렇다면 컨테이너 이미지의 CVE-2022-0543 취약점을 미리 알 수는 없었던 걸까요? 지금 이용하고 있는 이미지에 잠재해 있는 리스크를 어떻게 파악할 수 있을까요? 이번 글에서는 취약점을 탐지하기 위해 이용하는 대표적인 도구 중 하나인 Snyk을 이용해 실제 취약점을 발견해보겠습니다.

## Snyk

### Snyk이란?

Snyk은 소프트웨어 개발 과정에서 보안 취약점을 파악하고 대처하도록 돕는 보안 플랫폼이자 툴입니다. 애플리케이션 코드, 오픈 소스 종속성, 컨테이너 이미지, 클라우드 인프라 등 다양한 분야에서 취약점을 탐지하고, 효율적으로 해결할 수 있도록 돕습니다. 

Snyk에서 제시하는 취약성 심각도는 Low, Medium, High, Critical로 나누어지는데요. 이 기준은 [Oracle Linux Vulenerability Impact](https://linux.oracle.com/security/classification/index.html)를 따릅니다.

- Low: 애플리케이션이 공격하기위해 다음 취약점과 함께 사용할 수 있는 취약점 매핑을 허용하는 일부 데이터를 노출할 수 있습니다.
- Medium: 일부 조건에서 공격자가 애플리케이션의 민감한 데이터에 액세스하도록 허용할 수 있습니다. (리스크 프로파일 기반 정기 업데이트 권고)
- High: 공격자가 애플리케이션의 민감한 데이터에 액세스하도록 허용할 수 있습니다. (최대한 빠르게 조치 권고)
- Critical: 공격자가 민감한 데이터에 액세스하고 애플리케이션에서 코드를 실행할 수 있습니다. (즉각 조치 권고)

추가적으로, `docker scout` 이전에 이용되었던 `docker scan`에서는 내부적으로 Snyk을 이용해 취약점 검사를 진행했다는 점 역시 알아두어도 좋겠습니다.

### Snyk으로 취약점 탐지하기

#### Snyk 사전 환경

Windows 환경에서는 Docker 이미지 취약점 스캔을 위해 Snyk CLI를 이용할 수도 있고, docker 내의 extension에서 추가해 이용할 수도 있습니다. 둘 모두 인증을 위해 Snyk 회원가입이 필요합니다. Snyk CLI 설치는 [Snyk 공식 문서](https://docs.snyk.io/snyk-cli/install-or-update-the-snyk-cli)를 참고해주세요.

#### 이미지 취약점 스캔

Docker 컨테이너로 올려진 `redis:latest` 이미지에 취약점이 없는지 살펴보겠습니다. `snyk container test redis:latest` 명령어를 git bash에 작성해봅시다. 

```
Testing redis:latest...

(심각도 낮은 취약성 제외)

✗ High severity vulnerability found in pam/libpam0g
  Description: Improper Authentication
  Info: https://security.snyk.io/vuln/SNYK-DEBIAN12-PAM-8352887
  Introduced through: pam/libpam0g@1.5.2-6+deb12u1, shadow/login@1:4.13+dfsg1-1+b1, util-linux@2.38.1-5+deb12u1, adduser@3.134, pam/libpam-modules-bin@1.5.2-6+deb12u1, pam/libpam-modules@1.5.2-6+deb12u1, pam/libpam-runtime@1.5.2-6+deb12u1
  From: pam/libpam0g@1.5.2-6+deb12u1
  From: shadow/login@1:4.13+dfsg1-1+b1 > pam/libpam0g@1.5.2-6+deb12u1
  From: util-linux@2.38.1-5+deb12u1 > pam/libpam0g@1.5.2-6+deb12u1
  and 11 more...

✗ Critical severity vulnerability found in zlib/zlib1g
  Description: Integer Overflow or Wraparound
  Info: https://security.snyk.io/vuln/SNYK-DEBIAN12-ZLIB-6008963
  Introduced through: zlib/zlib1g@1:1.2.13.dfsg-1, util-linux@2.38.1-5+deb12u1, apt@2.6.1, dash@0.5.12-2
  From: zlib/zlib1g@1:1.2.13.dfsg-1
  From: util-linux@2.38.1-5+deb12u1 > zlib/zlib1g@1:1.2.13.dfsg-1
  From: apt@2.6.1 > apt/libapt-pkg6.0@2.6.1 > zlib/zlib1g@1:1.2.13.dfsg-1
  and 2 more...
```

위 결과는 Redis 이미지를 이루는 구성요소 중 하나인 `docker-image|redis`의 취약점 분석 결과입니다. (다른 `gomodules` 패키지 매니저는 제외했습니다.) 실제로 Critical한 취약점이 발견되었습니다. Info 필드에서 볼 수 있듯, Snyk에서는 [SNYK-DEBIAN12-ZLIB-6008963](https://security.snyk.io/vuln/SNYK-DEBIAN12-ZLIB-6008963) 으로 분류했습니다.

```
Organization:      glenn-syj
Package manager:   deb
Project name:      docker-image|redis
Docker image:      redis:latest
Platform:          linux/amd64
Licenses:          enabled

Tested 89 dependencies for known issues, found 40 issues.
```

위는 제가 진행한 테스트의 결과입니다. 

- Organization: Snyk 계정의 조직 이름
- Package manager: Redis 이미지에서 사용된 패키지 관리자
- Project name: Snyk이 스캔한 대상의 프로젝트 이름
- Docker image: 스캔 대상이 되는 Docker 이미지의 태그
- Platform: 빌드된 운영체제 및 이용 대상
- Licenses: 스캔 과정에서의 라이선스 정보 확인 기능 활성화 여부


```
Snyk wasn’t able to auto detect the base image, use `--file` option to get base image remediation advice.
Example: $ snyk container test redis:latest --file=path/to/Dockerfile
```

이러한 문구는 `redis:latest` 이미지를 스캔하며 베이스 이미지를 파악하지 못했음을 의미합니다. `--file` 옵션을 통해서 redis의 도커파일을 넣어주게 되면 base image에서의 취약점까지 검사합니다. 추가로, `--json` 옵션을 통해서 스캔 결과를 json 형태로 상세히 받을 수 있습니다.

물론 위 결과에서는 `redis-latest`는 이미지 내부의 레이어를 따라가며 패키지 매니저인 `debian:bookworm-slim`를 검사하므로, 베이스 이미지를 찾아내지 못해도 `debian:bookworm-slim`의 취약점은 검사됩니다.

```
Base Image          Vulnerabilities  Severity
redis:6.0.0-buster  160              15 critical, 35 high, 30 medium, 80 low

Recommendations for base image upgrade:

Alternative image types
Base Image               Vulnerabilities  Severity
redis:6.2.16-alpine3.20  0                0 critical, 0 high, 0 medium, 0 low

```

만약 `redis:latest` 대신 `redis:6.0.0`을 검사한다면 위와 같이 베이스 이미지의 교체를 권고하는 결과가 나타나게 됩니다.

## Snyk이 Docker Image를 분석하는 방식

### Dockerfile과 이미지 레이어

Dockerfile은 Docker 이미지를 생성하기 위한 스크립트 파일으로, `FROM`, `RUN`,`COPY`, `ADD` 등 각 명령어가 실행되면서 파일 시스템이 변경됩니다. 이러한 변경사항이 레이어로 추가된다고 볼 수 있습니다.

```dockerfile
# Base Image
FROM debian:bookworm-slim

# 레이어: 패키지 업데이트
RUN apt-get update

# 레이어: curl 설치
RUN apt-get install -y curl

# 레이어: 애플리케이션 복사
COPY app /usr/src/app

# 명령어: 실행 명령 설정
CMD ["curl", "--help"]
```

위와 같은 예시로서의 더미 dockerfile이 있다고 칩시다. 먼저, `FROM`을 통해 이미 레이어 단위로 빌드된 상태로 이미지를 가져옵니다. 이후에 `RUN`, `COPY` 명령어 등을 통해 새로운 레이어가 추가됩니다. `CMD`는 메타데이터에 영향을 미치므로, 파일 시스템에는 변화를 주지 않기에 레이어를 생성하지 않습니다.

### Docker Image 내부 레이어 분석 과정

#### 1. Docker 이미지 가져오기

Snyk은 Docker 데몬의 API를 통해 로컬 혹은 원격 Docker Registry에 있는 이미지를 가져옵니다. 만약 이미지 이름과 일치하는 이미지가 로컬에 캐싱되어 있다면 해당 이미지를 쓰고, 아니라면 Docker 레지스트리에서 가져옵니다. 앞서 살펴본 이미지 레이어는 변경 사항을 캡처하는 스냅샷 단위라고 볼 수 있습니다. 

#### 2. 레이어 분해 (Layer Decomposition)

```
 docker image inspect --format "{{json .RootFS.Layers}}" acme/my-final-image:1.0
[
  "sha256:72e830a4dff5f0d5225cdc0a320e85ab1ce06ea5673acfe8d83a7645cbd0e9cf",
  "sha256:07b4a9068b6af337e8b8f1f1dae3dd14185b2c0003a9a1f0a6fd2587495b204a",
  "sha256:cc644054967e516db4689b5282ee98e4bc4b11ea2255c9630309f559ab96562e",
  "sha256:e84fb818852626e89a09f5143dbc31fe7f0e0a6a24cd8d2eb68062b904337af4"
]
```

Docker Image를 구성하는 레이어를 해체하고, 시스템의 변경사항을 분석합니다. 레이어는 `sha256` 해시로 식별됩니다. 이후 각 레이어를 분석하여 시스템의 변경사항을 파악합니다.

Docker는 스토리지 드라이버를 통해 (`overlay2` 드라이버로 가정하겠습니다!), 읽기 전용인 Lowerdir과 읽기-쓰기가 가능한 Upperdir로 나누어 활용합니다. 전자는 변경된 파일을 포함한 스냅샷이고, 후자는 컨테이너 실행 시 변경된 파일 시스템을 기록하는 레이어이기도 합니다. 

이 두 레이어는 merged 디렉토리에서 병합되어 컨테이너 실행 시 파일 시스템의 액세스 지점이 됩니다. 따라서 파일 시스템 변경 사항은 Lowerdir과 Upperdir의 병합 과정에서 차이점이 추적되어 레이어 변경사항이 기록되는데요. Lowerdir에서 변경되지 않은 파일은 유지되고, Upperdir에서 추가되거나 수정된 파일은 새로운 레이어로 기록됩니다.

Snyk 역시 이러한 레이어 구조를 이용하여, 변경사항을 추적하고 취약점을 찾아낼 것으로 추측할 수 있겠습니다.

#### 3. 패키지 정보 추출

Snyk은 각 레이어에서 운영 체제와 패키지 매니저 정보를 검색합니다. 패키지 매니저만이 아니라, `apt-get install`과 같은 명령어 실행 결과와 설치된 소프트웨어 목록 역시 분석합니다.

#### 4. 파일 시스템 검사 및 취약점 데이터베이스와 매칭

이제 각 레이어에서 파일 시스템의 모든 경로를 탐색해 알려진 바이너리와 라이브러리를 식별합니다. SHA-256 해시 계산 후, Snyk의 취약점 데이터베이스와 대조하게 되는데요. 메타데이터와 위험도 높은 설정에 대해 정적 분석도 이루어집니다. 이러한 방식에 따라 패키지 이름과 버전에 따라 취약점을 매칭합니다. 여기에서 심각도 및 (가능하다면) 수정 가능한 버전 정보를 포함해 취약점 검사 결과로 돌려주게 됩니다.

#### 5. 베이스이미지 분석

Dockerfile의 `FROM` 명령어로 지정된 베이스이미지가 포함된 레이어를 개별적으로 스캔하여 취약점을 탐지합니다. 만약 베이스 이미지가 탐지되지 않았다면, 베이스 이미지 교체 권고는 나오지 않습니다. 

<br/>

## 나가며

이번 글에서는 클라우드 환경 보안이 중요해지는 상황에서, Snyk을 이용해 Docker 이미지에서의 취약점 탐색 및 검사의 바탕이 되는 도커 이미지 레이어에 대해 살펴보았습니다. Docker 이미지가 구성되는 방식을 이해하고, Snyk 역시 이러한 구조를 이용해 취약점을 검사함을 알 수 있었습니다.

Snyk은 Docker 이미지 내의 패키지 매니저, 운영 체제, 라이브러리 등 다양한 요소들을 분석하여 취약점을 검출하는데요. 이를 통해 취약점을 감지하고, 필요한 보안 패치를 권고합니다.

컨테이너 보안에서는 단순히 이미지를 스캔하는 것에서 그치지 않고, Snyk 등의 툴을 이용하여 보안 취약점에 빠르게 대응할 수 있도록 하는 환경의 구축이 필요하다는 생각이 듭니다.

## 참고자료

[https://docs.docker.com/engine/storage/drivers/](https://docs.docker.com/engine/storage/drivers/)

[https://security.snyk.io/vuln/SNYK-DEBIAN12-ZLIB-6008963](https://security.snyk.io/vuln/SNYK-DEBIAN12-ZLIB-6008963)

[https://snyk.io/learn/container-security/container-scanning/](https://snyk.io/learn/container-security/container-scanning/)

[https://snyk.io/learn/docker-security-scanning/](https://snyk.io/learn/docker-security-scanning/)

[https://docs.snyk.io/scan-with-snyk/snyk-container/use-snyk-container/detect-the-container-base-image](https://docs.snyk.io/scan-with-snyk/snyk-container/use-snyk-container/detect-the-container-base-image)

[https://nvd.nist.gov/vuln/detail/CVE-2022-0543](https://nvd.nist.gov/vuln/detail/CVE-2022-0543)

[https://docs.docker.com/engine/storage/drivers/overlayfs-driver/](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/)