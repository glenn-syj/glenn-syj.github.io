---
title: 포트 에러로 시작하는 Docker Desktop 네트워크 뜯어보기
lang: ko
layout: post
---

## 들어가며

- Windows 환경에서 WSL을 이용해 Docker가 구동될 때, 해당 포트를 이용할 수 없다는 에러를 해결하는 과정과 그 뒤에 존재하는 네트워크 개념과 도커 데스크탑의 구조를 살펴보는 글입니다.
- 최종 업데이트: 25/05/09

## 네트워크: HTTP 500 에러와 포트 바인딩에서 시작하기

```
(HTTP code 500) server error - Ports are not available: exposing port TCP 0.0.0.0:8065 -> 0.0.0.0:0: listen tcp 0.0.0.0:8065: bind: An attempt was made to access a socket in a way forbidden by its access permissions.
```

윈도우즈 환경에서 도커를 이용하며 분명히 지난 실행에서는 잘 되었는데, 갑작스럽게 포트를 이용할 수 없다는 에러가 발생하고는 합니다. 심지어는 `netstat` 명령어로 확인해보아도 해당 포트를 이용하는 프로세스를 찾을 수 없었습니다.

검색을 통해서 찾아낸 해결책은 간단했습니다. 대부분의 해결책에서는 `Windows NAT`가 바로 열쇠였습니다. 해결 자체는 쉬웠습니다. (8065 포트는 시스템 예약 포트였습니다.)

```
(1) `net stop winnat`
(2) 컨테이너 재시작
(3) `net start winnat`
```

그런데, Windows NAT란 도대체 뭐길래 포트 바인딩에서 문제를 일으킨 걸까요? 이런 문제가 발생할 때마다 winnat을 끄고, 시작하고, 켜는 과정을 반복해야할까요? 정말로 Windows NAT가 문제인 걸까요?

비록 클라우드 환경에서는 인스턴스에서 이용하는 대부분의 운영체제가 리눅스지만, 저 역시 노트북에서 윈도우즈를 이용하고 있으므로 Windows NAT와 도커 데스크탑에 대해서도 살펴보고자 합니다.

정말로 Docker Desktop 이용 과정에서의 Windows NAT가 문제인지 살펴보기 전에, 먼저 Windows NAT와 그를 둘러싼 환경에 대해서 살펴보고자 합니다.

### 1. Windows NAT

Windows NAT에서 NAT는 Network Address Translation의 약어입니다. Windows NAT는 사설 네트워크의 여러 장치가 하나의 공인 IP를 공유하여 외부 네트워크와 통신할 수 있게 해주는데요. 이러한 통신을 위해서는 사설 네트워크와 공인 네트워크 사이의 매핑이 필요합니다.

Windows NAT는 Windows Host와 WSL 사이의 네트워크 통신을 관리하고 중개하는데요. 여기에서 외부에서의 호스트 IP와 내부 네트워크 IP 간에서 주소 변환이 이루어집니다. 이와 함께 해당 호스트 IP와 네트워크 주소 IP에서의 포트 매핑도 이루어지구요.

일반적인 WSL2 네트워킹에서는 Windows NAT가 이용됩니다. (추후 설명하겠지만, 도커 데스크탑에서 WSL2 백엔드를 이용할 때는 다르게 동작합니다.)

### 2. WSL2

> "WSL 2는 Linux 배포를 설치할 때 기본 배포판 유형입니다. WSL 2는 가상화 기술을 사용하여 경량 유틸리티 VM(가상 머신) 내에서 Linux 커널을 실행합니다. Linux 배포판은 WSL 2 관리형 VM 내에서 격리된 컨테이너로 실행됩니다."
>
> [https://learn.microsoft.com/ko-kr/windows/wsl/about#what-is-wsl-2](https://learn.microsoft.com/ko-kr/windows/wsl/about#what-is-wsl-2)

도커 데스크탑은 Microsoft에서 제공하는 WSL2에 기반해 동적 메모리 할당이나 I/O 성능 향상 등 윈도우 환경에서의 성능을 향상하고 리소스를 관리합니다. 여기에서 Microsoft가 밝히듯이, WSL2는 NAT 기반 아키텍처를 이용합니다. (아래가 도커 데스크탑의 네트워크 구조가 아님에는 유의해야 합니다.)

```
[외부 네트워크]
      |
[Windows Host]
      |
[Hyper-V 가상 스위치]
      |
[vEthernet (WSL)]
      |
[WSL2 VM]
      |
[eth0 (리눅스 내부)]
```

WSL2는 경량 가상머신(VM)으로 동작하는데, 이는 Windows 호스트와 통신하기 위해 자체적인 가상 네트워크 어댑터(vEthernet)를 생성해 이용합니다. 이 가상 어댑터를 통해서 WSL2 VM은 독립된 IP 주소를 할당받고, Windows Host와는 기본적으로 NAT 기반 아키텍처로 통신합니다.

Docker Desktop의 컨테이너는 WSL2에 기반하기 때문에 실제로 WSL2 VM 내부에서 실행됩니다. 따라서 도커 네트워크는 VM 내부에 존재하는 것이고, 컨테이너도 네트워크 안에서 IP를 할당받아 통신합니다.

기본적으로 Windows NAT에서 이용하는 포트 포워딩은 `localhost`를 통해서 접근하도록 매핑됩니다. WSL2의 포트 포워딩 기능을 통해 Windows 호스트의 localhost에서 WSL2 VM 내부의 서비스에 접근할 수 있습니다.

### 3. vEthernet 어댑터와 eth0

WSL2가 실행되면 Windows는 Hyper-V에 기반한 "vEthernet Adapter (WSL)"이라는 이름의 가상 네트워크 어댑터를 자동으로 생성합니다.

![vEthernet Adapter (WSL)](/assets/images/250420+vEthernet+adapter.png)

이 어댑터는 Windows 호스트와 WSL2 VM(가상머신) 사이의 네트워크 브리지 역할을 하며, NAT를 통해 서로 통신할 수 있도록 하는데요.

WSL2 내부(리눅스 환경)에서는 `eth0`이라는 네트워크 인터페이스도 자동으로 생성됩니다. 이 `eth0` 네트워크는 Hyper-V가 제공하는 가상 네트워크 어댑터에 연결되어 있으며, WSL2 VM이 부팅될 때 자동으로 생성됩니다.

`eth0` 인터페이스에는 WSL2 VM만의 고유한 내부 IP가 할당되며, 이 IP를 통해 리눅스 환경 내에서 네트워크 통신이 이루어집니다.

그러면 Hyper-V 가상 스위치는 어떻게 이 두 네트워크를 브리징하는 것일까요?

### 4. Hyper-V 가상 스위치

Hyper-V 가상 스위치는 실제 네트워크 스위치처럼 동작하지만, 소프트웨어적으로 구현되어 Hyper-V 환경에서 통신을 중계하는 역할을 하는 가상 스위치입니다.

Hyper-V 가상 스위치는 연결 수준에 따라 세 가지 유형으로 분류됩니다.

- 외부 스위치: 가상 머신과 물리적 네트워크(외부 네트워크) 간의 통신을 중계하는 스위치
- 내부 스위치: 가상 머신과 호스트 간, 그리고 가상 머신끼리의 통신을 중계하는 스위치
- 프라이빗 스위치: 가상 머신끼리만 통신할 수 있도록 중계하는 스위치

WSL2와 관련된 Microsoft 공식 문서에서는 Hyper-V 가상 스위치 유형을 명시하거나 추가적으로 설명하고 있지는 않았습니다.

하지만 다행히도 Hyper-V 공식 문서에서는 NAT가 내부 스위치(internal switch)를 이용해 가상 머신이 컴퓨터 네트워크에 액세스 할 수 있도록 한다고 명시되어 있습니다.

## 네트워크: Docker Desktop 포트 에러 발생 과정 뜯어보기

이해를 돕기 위해, Windows 환경에서 Docker Desktop을 이용할 때 그 내부의 상호작용을 나타내기 위한 Deployment UML 다이어그램을 그려봤습니다.

![Docker Desktop 네트워크 구조 및 에러 발생 흐름](/assets/images/250420+whole+network+deployment+UML.png)

Docker Desktop에서 WSL2 Backend 옵션을 활성화하는 경우에는 Windows NAT가 아니라, 자체적인 네트워크 스택(VPNKit)을 이용한다는 점은 꼭 기억해주시면 좋겠습니다.

이는 `winnat`이 중단된 상태에서도 도커 데스크탑이 정상적으로 동작하고 포트 포워딩이 진행됨을 지지하는 근거이기도 한 까닭입니다.

이어서 제가 겪었던, Mattermost 컨테이너에서 이용하는 8065 포트를 Windows Host의 8065 포트로 포워딩하려고 했을 때 에러가 발생하는 과정도 조망해보겠습니다.

### 에러 발생 과정 상세 설명

**1. 사용자의 Docker 컨테이너 실행 요청**

```bash
docker run -p 8065:8065 mattermost/mattermost-team-edition
```

사용자가 Windows Host에서 Docker CLI를 실행하여 컨테이너를 띄우려 합니다. 위 명령어는 Windows Host의 8065 포트를 컨테이너의 8065 포트에 매핑하도록 요청합니다.

**2. 명령 전달 경로**

```
Docker CLI (Windows Host)
   ↓
Hyper-V Socket (AF_VSOCK)
   ↓
WSL2 VM
   ↓
Docker Engine (dockerd)
```

Docker CLI의 명령은 Hyper-V Socket을 통해 WSL2 VM에 전달됩니다. WSL2 VM 내부에서 Docker Engine이 명령을 수신합니다.

**3. 컨테이너 생성 및 네트워크 설정 준비**

Docker Engine이 먼저 Mattermost 이미지를 확인하고 필요한 경우 pull합니다. 이후 컨테이너를 생성하고 네트워크 네임스페이스를 설정합니다.

생성된 컨테이너는 Docker Bridge Network에 연결되어 172.x.x.x 대역의 내부 IP를 할당받습니다. 컨테이너의 8065 포트는 Docker Bridge Network를 통해 WSL2 VM의 네트워크 인터페이스인 eth0의 8065 포트와 바인딩됩니다.

**4. VPNKit 포트 포워딩 설정 시도**

```
VPNKit
   ↓
Windows Host의 8065 포트 바인딩 시도 (여기서 실패)
   ↓
Hyper-V Socket (AF_VSOCK)
   ↓
tap-vsockd (VM 내부 프로세스)
   ↓
tap device (가상 네트워크 인터페이스)
   ↓
WSL2 VM의 eth0 (이미 컨테이너와 바인딩 된 상태)
```

VPNKit은 WSL2 VM과 Windows Host 간의 네트워크 통신을 담당합니다. 따라서 WSL2 VM의 eth0 인터페이스와 Windows Host의 포트를 연결하려 시도합니다.

**5. 포트 포워딩 실패**

Windows Host의 8065 포트는 문제 상황에서 시스템 예약 포트이므로 이용할 수 없습니다. VPNKit이 Windows Host의 8065 포트를 바인딩하려 할 때 실패합니다.

**6. 에러 전파 및 컨테이너 실행 중단**

```
Windows Host 포트 바인딩 실패
   ↓
VPNKit에서 에러 발생
   ↓
tap-vsockd를 통해 WSL2 VM에 전달
   ↓
Docker Engine에 에러 전달
   ↓
컨테이너 실행 중단 및 롤백
   ↓
Docker CLI로 에러 메시지 반환
```

**7. 최종 에러 메시지 출력**

```
(HTTP code 500) server error - Ports are not available: exposing port TCP 0.0.0.0:8065 -> 0.0.0.0:0: listen tcp 0.0.0.0:8065: bind: An attempt was made to access a socket in a way forbidden by its access permissions.
```

## Windows NAT 재시작이 정말 해결책일까?

`net stop winnat`과 `net start winnat`가 정말 위의 시스템 예약 포트 범위(excluded portrange)로 인한 에러를 해결하기 위한 근본적인 해결책일까요?

저는 그렇지 않다고 답하고 싶습니다. 이는 확률적인 성공에 기대기 때문인데요.

### Windows NAT 재시작이 불충분한 이유

#### 1. 다중적인 서비스의 포트 예약과 이용

Windows NAT만이 아니라, Hyper-V 역시도 포트를 예약할 수 있습니다. 만약 8065 포트가 Windows NAT에서 예약한 포트 범위 중 하나에 속한다면, Windows NAT 서비스를 중단하고 다시 시작하는 것만으로도 문제가 해결될 것입니다.

하지만 만약 Hyper-V가 저희가 이용해야하는 8065 포트를 예약한다면요? 혹은 50000+ 포트 범위를 어플리케이션에서 이용해야하는 상황이라면요? 이러한 질문에는 단순하게 Windows NAT를 재시작하는 것만으로는 해결되지 않습니다.

#### 2. 재부팅 시 무작위적 포트 범위 예약

현재 Windows는 동적 포트를 이용해 재부팅 시에도 예약 포트 범위를 동적으로 할당하는데요. 만약 이용해야 하는 포트가 동적인 포트 범위에 속한다면 언제든 다시 예약될 위험이 있습니다.

즉, 이는 임시방편일 뿐 확실한 해결책이 아니라고 볼 수 있는데요. 다시 말하자면 매번 특정한 해결책을 실행하는 것이 아니라, 포트 범위에 대한 설정 자체를 수정해야한다는 뜻이기도 합니다.

### 동적 포트 범위 재설정하기

#### 1. 포트 범위 확인하기

```
netsh int ipv4 show excludedportrange protocol=tcp
```

![TCP 동적 포트 범위](/assets/images/250420+tcp+port+range.png)

위 명령어의 실행 결과로, 제 Windows 호스트는 1024포트부터 시작해 64511개의 포트를 동적 포트로 할당 가능한 상태로 열어둠을 알게 되었습니다.

즉, 1024-65535 포트 범위가 모두 동적 포트 범위로 할당가능한 포트인 상태입니다.

#### 2. 포트 범위 수정하기

```
netsh int ipv4 set dynamic tcp start=49152 num=16384
netsh int ipv6 set dynamic tcp start=49152 num=16384
```

위 명령어는 동적 포트 범위를 41952-65535로 조정합니다. 근본적인 해결책으로 포트 범위 41952-65535가 선정된 이유는, IANA에서 정한 포트 할당 정책에 기반했기 때문이라고 생각합니다.

Well Known Ports(0-1023)는 IANA에서 직접 할당하고 관리하는 포트로, HTTP(80)이 대표적입니다.

Registsered Ports(1024-49151)는 기업이나 조직이 IANA에 등록하여 사용하거나 일반적으로 사용 가능한 포트입니다.

Dynamic/Private Ports(49152-65535)는 동적 포트 범위로, IANA가 관리하지 않습니다. 특히 [RFC 6335](https://datatracker.ietf.org/doc/html/rfc6335)에서는 다음과 같이 어플리케이션이 해당 포트를 이용하지 말라고(MUST NOT) 강력하게 이야기합니다.

> On the other hand, application software MUST NOT assume that a specific port number in the Dynamic Ports range will always be available for communication at all times, and a port number in that range hence MUST NOT be used as a service identifier.
>
> [https://datatracker.ietf.org/doc/html/rfc6335](https://datatracker.ietf.org/doc/html/rfc6335)

이는 동적 포트 범위를 이용하는 어플리케이션이 해당 포트를 항상 이용할 수 있음을 보장하지 않는다는 것을 의미합니다.

따라서 소프트웨어 어플리케이션이 이를 준수한다면, Windows 호스트의 동적 포트 범위를 Dynamic/Private Ports로 설정할 때 충돌이 발생하지 않는 효과로 이어지겠습니다.

## 부록

Windows 환경에서 시스템 예약 포트 범위를 확인하고, 그 범위에 속하는 포트 포워딩을 Docker Desktop에서 진행하는 코드를 작성해두었습니다. 빠른 PoC를 위해 Cursor를 이용해 작성했으므로, 코드 자체가 완벽하지 않을 수 있음에 유의해주세요.

[https://github.com/glenn-syj/infra-playground/tree/main/docker/port_managing_winnat](https://github.com/glenn-syj/infra-playground/tree/main/docker/port_managing_winnat)

위 repository를 clone하시거나 다운받으신 뒤, `run_test.ps1` 파일을 실행하고 관리자 권한을 허용하면 `test_scenarios.py` 파일의 테스트 시나리오가 진행됩니다.

요구 조건으로는 python 3.10 이상과 pip, docker desktop 설치가 필요합니다.

## 나가며

처음에는 단순한 Docker Desktop의 HTTP 500 에러와 그 해결책을 찾는 것에서 시작했습니다. `net stop winnat` 및 `net start winnat` 명령어로 문제가 해결되는 것을 보고 쉽게 넘어갔지만, 동시에 의문이 들었습니다.

에러 하나에서 시작해 Docker Desktop for Windows가 가지는 구조도 살펴보고, Deployment UML을 통해 통신 과정을 살펴보기도 했습니다.

게다가 처음에는 Docker Desktop이 항상 Windows NAT를 이용하는 줄로만 알았습니다. 그런데 Hyper-V 기반이 아닌 WSL2 백엔드를 이용할 때는 다름을 알게된 후에, UML도 갈아 엎어야 했던 기억이 많이 남습니다.

특히 "왜 Winnat을 종료했는데 도커 데스크탑에서 컨테이너가 잘 매핑되고 실행되지?"라는 의문에서 VPN Kit를 발견했을 때 퍼즐이 풀린 기분이었습니다.

비록 글에서 다루지는 못했지만, WSL1과 WSL2의 차이나 Hyper-V와 하이퍼바이저 타입 등을 학습하는 계기가 되기도 했습니다.

## References

[Docker Desktop WSL 2 backend](https://docs.docker.com/desktop/features/wsl/)

[Configure and troubleshoot the Docker daemon](https://docs.docker.com/engine/daemon/logs/)

[Completely solve the problem of Docker containers not starting or running on Windows 10 due to port](https://medium.com/@sevenall/completely-solve-the-problem-of-docker-containers-not-starting-or-running-on-windows-10-due-to-port-57f16ed6143)

[Docker for Windows Issue #3171 Comment](https://github.com/docker/for-win/issues/3171#issuecomment-554587817)

[VPNKit GitHub Repository](https://github.com/moby/vpnkit)

[VPNKit Ethernet Documentation](https://github.com/moby/vpnkit/blob/master/docs/ethernet.md)

[VPNKit Ports Documentation](https://github.com/moby/vpnkit/blob/master/docs/ports.md)

[Ephemeral Port - Wikipedia](https://en.wikipedia.org/wiki/Ephemeral_port)

[RFC 6335 - Internet Assigned Numbers Authority (IANA) Procedures](https://www.rfc-editor.org/rfc/rfc6335.html)

[Docker, WSL2, and Hyper-V Guests](https://www.elliotsegler.com/docker-wsl2-and-hyper-v-guests.html)
