---
title: PNPM과 Next.js standalone 모드 (1) - PNPM 이해하기
lang: ko
layout: post
---

## 들어가며

- TFT 기반 라이벌리 서비스 Rivals를 개발 및 배포하는 과정에서 PNPM을 이용하며 겪었던 문제와 원인을 파악하는 시리즈입니다.
- 이번 글은 패키지 매니저와 pnpm의 구조에 대해 알아가는 목적으로 작성되었습니다.
- 최종 수정: 25/06/19

## 패키지 매니저와 PNPM

저는 최근 Rivals라는 Riot TFT(전략적 팀 전투) 기반 라이벌리 제공 서비스를 개발하고 있습니다. 이 프로젝트에서는 npm이나 yarn보다 빌드 속도 상 우위에 있다는 이야기를 많이 들어 이번에는 pnpm을 패키지 매니저로 선정했습니다.

저는 pnpm 패키지 매니저 기반 next.js에서의 standalone 방식을 이용할 때, `server.js` 실행 시 모듈 임포트와 관련된 에러를 마주했었는데요. 배포에 우선 순위를 두어 일단 `.npmrc` 파일에 `node-linker=hoisted` 옵션을 추가하는 방식으로 해결했었습니다.

그러나 이러한 해결책이 과연 진정한 해결책일까요? 위 옵션을 이용한다면 pnpm만이 가지는 장점이 사라지는 게 아닐까요? 이번 글은 이러한 의심에서 시작해봅니다. 가장 먼저, 패키지 매니저에서부터 살펴보겠습니다.

### 패키지 매니저의 개념과 기능

패키지 매니저는 개발에 필요한 라이브러리나 프레임워크의 의존성을 관리하는 도구입니다. 특히, 명시된 의존성 정보를 바탕으로 패키지를 설치하고, 소스코드에서는 명시된 특정 버전의 라이브러리를 이용할 수 있도록 합니다.

대표적으로 npm, yarn, pnpm이 패키지 매니저로 활약하고 있습니다. Java에서의 gradle도 패키지 매니징과 함께 빌드 매니징을 지원하고 있구요. Hombrew나 apt 역시 패키지 매니저라 볼 수 있습니다. 이번 글에서는 Node.js 생태계 내에서의 패키지 매니저를 다룹니다.

패키지 매니저의 기본적인 기능은 무엇일까요? 이는 "만약 패키지 매니저가 없었다면 어떻게 의존성 관리를 진행해야할까?"라는 질문에서 시작하면 좋습니다. MDN Web Docs에서는 패키지 매니저를 쓰지 않는 경우에 관해 이야기합니다.

```
1. 정확한 패키지의 자바스크립트 파일을 모두 찾는다.
2. 해당 패키지에 알려진 취약점이 없는지 살펴본다.
3. 패키지를 다운로드하고 프로젝트 내의 정확한 경로에 넣는다.
4. 어플리케이션에서 패키지를 코드에 넣는다.
5. 1-4를 패키지의 모든 하위 의존성에 대해 반복한다.
6. 패키지를 삭제하고 싶다면 다시 모든 파일을 삭제한다.
```

즉, 패키지 매니저는 패키지 설치부터 삭제 등의 관리, 취약점 확인 등의 보안까지 전반적으로 담당한다고 말할 수 있겠습니다.

### 패키지 매니저의 동작 방식

패키지 매니저는 크게 세 단계를 거쳐 의존성을 관리합니다. 바로 Resolution(의존성 해석), Fetching(다운로드), Linking(연결)입니다. 각 단계별로 살펴보겠습니다.

#### 1. Resolution (의존성 해석)

패키지 매니저는 먼저 `package.json`에 명시된 의존성을 분석하고 해석합니다.

- 직접 의존성과 하위 의존성을 모두 파악
- 버전 범위를 확인하고 실제 설치할 버전을 결정
- 의존성 충돌이 있는 경우 해결 방법 결정
- 결과물로 lock 파일 생성

#### 2. Fetching (다운로드)

Resolution 단계에서 결정된 패키지들을 다운로드하는 단계입니다.

- 패키지 레지스트리(보통 npm)에서 타볼(tarball) 형태로 다운로드
- 무결성 검사 및 캐시 처리

#### 3. Linking (연결)

다운로드된 패키지들을 프로젝트에서 사용할 수 있도록 연결하는 단계입니다.

- 실제 파일에 대한 접근 경로 생성
- 운영체제와 파일 시스템에 따라 적절한 링크 방식 선택

이러한 세 단계는 패키지 매니저마다 다른 방식으로 구현됩니다.

### PNPM의 주특징

지난 단락에서 `node-linker=hoisted` 옵션에 대해서 언급했습니다. 이 옵션을 본다면 먼저 의문이 들기 마련입니다. 듣자하니 앞서 설명한 **링크(linking)**와 관련된 옵션 같습니다. 그런데 `hoisted`라는 것은 어떤 설정값일까요? 이 옵션을 이해하기 위해서는 먼저 PNPM 패키지 매니저의 동작 방식을 알아야 합니다.

PNPM은 Perforamant NPM의 약어입니다. PNPM이 다른 패키지 매니저와 가장 대비되는 특성은 콘텐츠 주소 지정 가능 저장소(Content-Addressable Store)와 링크 방식입니다.

#### 수평적 효율성 - 콘텐츠 주소 지정 가능 저장소

PNPM으로 설치되는 모든 패키지 파일은 시스템 내의 전역 단일 저장소에 저장되는데요. 이는 콘텐츠 주소 지정 가능한(Content-Addressable) 방식으로 관리됩니다. 파일의 이름이나 위치가 아닌 파일 내용의 해시값을 기반으로 파일이 저장되고 식별된다는 뜻입니다. 아래는 [pnpm.io](pnpm.io)에서 가져온 이미지입니다.

![Content-addressable store](/assets/images/250605+pnpm-content-addressable-store.svg)

npm에서는 의존성 패키지 A를 이용하는 100개의 프로젝트를 위해서 100번 복사되어 이용됩니다. 이는 곧 패키지 A에 변화가 생길 때 npm은 100개 전체에 대해서 클론을 진행해야함을 의미합니다. 하지만 pnpm은 변경된 파일 하나에 대해서만 전역적으로 저장하면 끝입니다.

저는 이를 두고 '수평적 효율성'이라고 말하고 싶은데요. 시스템 내에서 여러 다른 프로젝트가 동일한 패키지를 필요로 할 때, 해당 파일 콘텐츠를 전역 저장소에서 단 한 번만 저장한 것으로 이용하기 때문입니다. 또한, 의존성 간에 이용하는 파일이 동일한 경우도 저장소에 한 번만 저장되기에 중복을 제거할 수 있습니다. 이러한 방식으로 프로젝트 수나 의존성 수가 증가하는 '수평적인 확장'에서 pnpm은 디스크 공간을 절약합니다.

#### 수직적 효율성 - 하드 링크와 심볼릭 링크

pnpm은 `node_modules` 디렉토리를 이용하는 방식에서도 근본적인 차이가 있습니다.. `pnpm install` 명령이 실행되면 의존성 관리는 다음과 같이 이루어집니다.

pnpm은 먼저 시스템 내 전역 저장소에 있는 패키지 파일들에 대해 해당 프로젝트의 `node_modules/.pnpm` 디렉토리 내에 하드 링크를 생성합니다. 아래 코드는 실제로 pnpm에서 하드 링크를 생성하는 코드입니다. 자세한 내용은 [pnpm repository의 /fs/hard-link-dir](https://github.com/pnpm/pnpm/blob/main/fs/hard-link-dir/src/index.ts)링크를 참고해주세요.

```typescript
/* pnpm/pnpm repostiory: /fs/hard-link-dir/src/index.ts
    구조 파악을 위해 일부 생략한 뒤에 주석을 추가하였습니다.
*/

// 외부에서의 진입점이 되는 함수로
export function hardLinkDir(src: string, destDirs: string[]): void {
  if (destDirs.length === 0) return;
  // Don't try to hard link the source directory to itself
  destDirs = destDirs.filter((destDir) => path.relative(destDir, src) !== "");
  _hardLinkDir(src, destDirs, true);
}

function _hardLinkDir(src: string, destDirs: string[], isRoot?: boolean) {
  let files: string[] = [];

  // 1. 원본 디렉토리 처리 (src 디렉토리 파일 목록 읽어오기)
  try {
    files = fs.readdirSync(src);
  } catch (err: unknown) {
    // ... error handling for ENOENT
    return;
  }

  // 2. src 내부 파일 및 디렉토리 순회
  for (const file of files) {
    // node_modules는 pnpm이 별도로 처리하므로 넘어감
    if (file === "node_modules") continue;
    const srcFile = path.join(src, file);

    // 2-1. 현재 항목이 디렉토리인 경우
    if (fs.lstatSync(srcFile).isDirectory()) {
      const destSubdirs = destDirs.map((destDir) => {
        const destSubdir = path.join(destDir, file);
        // ... create directory
        return destSubdir;
      });

      // 함수 재귀적 실행
      _hardLinkDir(srcFile, destSubdirs);
      continue;
    }

    // 2-2. 현재 항목이 디렉토리가 아니라 파일일 때
    for (const destDir of destDirs) {
      const destFile = path.join(destDir, file);
      try {
        // linkOrCopyFile 호출
        linkOrCopyFile(srcFile, destFile);
      } catch (err: unknown) {
        // ... error handling for ENOENT
      }
    }
  }
}

function linkOrCopyFile(srcFile: string, destFile: string): void {
  try {
    //
    linkOrCopy(srcFile, destFile);
  } catch (err: unknown) {
    // ... ENOENT 에러 핸들링: 디렉토리 생성 및 재시도
  }
}

// 3. 하드 링크 생성 핵심 로직
function linkOrCopy(srcFile: string, destFile: string): void {
  try {
    // 3-1. srcFile을 destFile로 하드링크 시도
    fs.linkSync(srcFile, destFile);
  } catch (err: unknown) {
    // ... 3-2. EXDEV 에러 (Cross-Device Link) 발생 시 다른 파일 시스템이므로 생성 불가
    // 하드 링크 대신 파일 복사 이용
    fs.copyFileSync(srcFile, destFile);
  }
}
```

이 과정에서 전역 콘텐츠 주소 지정 저장소가 이용됩니다. 아래 도식에서 `source`는 전역 콘텐츠 주소 지정 저장소라고 볼 수 있습니다.

![Hard Link Symlink](/assets/images/250605+pnpm+hardlink+symlink.webp)

하드 링크는 원본 파일과 동일한 inode를 가리키므로, 실제 디스크 공간을 추가로 차지하지 않으면서 파일에 직접 접근하는 것과 같은 효과를 냅니다. 이 `.pnpm` 디렉토리 안에는 프로젝트가 필요로 하는 모든 의존성 패키지(직접 의존성 및 하위 의존성)가 각각의 폴더 구조로 정리된 채로 존재합니다.

다음으로 pnpm은 `.pnpm` 디렉토리의 하드 링크된 패키지들을 기반으로 심볼릭 링크를 생성합니다. 이때 가장 중요한 특징은 루트 `node_modules` 디렉토리에는 오직 `package.json`에 명시된 직접 의존성만이 심볼릭 링크로 존재한다는 점입니다.

이제 예시를 간단한 예시를 통해서 살펴보겠습니다. 아래와 같은 `package.json`이 있다고 가정해보겠습니다.

```json
{
  "dependencies": {
    "foo": "1.0.0" // 루트 package.json에서는 foo만 명시
  }
}
```

`pnpm install`을 진행하면 `node_modules` 구조는 아래와 같이 구성됩니다. `bar`와 `baz`가 `foo`의 의존성이라고도 가정하겠습니다.

```
node_modules/
├── .pnpm/ # pnpm의 저장소
│ ├── foo@1.0.0/ # foo 패키지 디렉토리
│ │ └── node_modules/ # foo와 foo의 의존성들의 심볼릭 링크가 모인 곳
│ │   ├── foo/ # -> ../../foo@1.0.0 (심볼릭 링크)
│ │   ├── bar/ # -> ../../bar@1.0.0 (심볼릭 링크)
│ │   └── baz/ # -> ../../baz@1.0.0 (심볼릭 링크)
│ │
│ ├── bar@1.0.0/ # bar 패키지의 실제 파일들 (전역 저장소에서 하드 링크됨)
│ └── baz@1.0.0/ # baz 패키지의 실제 파일들 (전역 저장소에서 하드 링크됨)
│
├── foo/ # -> .pnpm/foo@1.0.0/node_modules/foo (심볼릭 링크)
├── bar/ # -> .pnpm/foo@1.0.0/node_modules/bar (심볼릭 링크)
└── baz/ # -> .pnpm/foo@1.0.0/node_modules/baz (심볼릭 링크)
```

앞서 살펴본 하드 링크와 심볼릭 링크에 기반한 구조를 살펴보면 다음과 같은 점을 알 수 있는데요.

- 실제 패키지 파일들은 `.pnpm` 디렉토리 안에 하드 링크로 존재합니다. (전역 저장소와 동일한 파일을 가리킴)
- foo가 사용하는 의존성(bar, baz)들은 `.pnpm/foo@1.0.0/node_modules/` 안에서만 접근 가능합니다
- 프로젝트 코드에서는 오직 루트의 `foo`만 접근 가능하므로 유령 의존성이 방지됩니다. (유령 의존성은 다음 장에서 살펴보겠습니다.)

pnpm은 패키지를 위와 같이 관리하기에 실제 파일은 하드 링크로 공유하면서도 각 패키지는 자신의 의존성을 독립적으로 가질 수 있습니다.

## node-linker=hoisted 옵션

`node-linker=hoisted` 옵션을 이해하기 전에 pnpm을 간단하게 살펴보았습니다. 이 옵션을 활성화하면, pnpm의 엄격한 심볼릭 링크 구조를 포기하고 npm이나 yarn 스타일의 호이스팅을 사용하게 됩니다. (기본값은 `isolated`입니다.) 호이스팅을 적용한 `node_modules` 구조는 다음과 같습니다.

```
node_modules/
├── foo/                  # 직접 의존성 (package.json에 명시됨)
│                         # -> .pnpm/foo@1.0.0/ 심볼릭 링크
│
├── bar/                  # 호이스팅된 하위 의존성 (foo가 의존)
│                         # -> .pnpm/bar@1.0.0/ 심볼릭 링크
│
├── baz/                  # 호이스팅된 하위 의존성 (bar가 의존)
│                         # -> .pnpm/baz@1.0.0/ 심볼릭 링크
│
└── .pnpm/               # 실제 패키지 파일들이 하드 링크로 저장된 공간
    ├── foo@1.0.0/       # foo 패키지의 실제 파일들 (전역 저장소와 하드 링크 공유)
    │                    # package.json에는 bar를 의존성으로 명시
    │
    ├── bar@1.0.0/       # bar 패키지의 실제 파일들 (전역 저장소와 하드 링크 공유)
    │                    # package.json에는 baz를 의존성으로 명시
    │
    └── baz@1.0.0/       # baz 패키지의 실제 파일들 (전역 저장소와 하드 링크 공유)
```

`package.json`에서는 분명히 `foo`만 직접 의존성으로 명시하고 있지만, 호이스팅으로 인해 모든 패키지가 루트 `node_modules`에서 접근 가능해지는 결과를 낳게 됩니다.

### 유령 의존성

위 hoisted된 구조에서 `require('bar')`나 `require('baz')`는 분명히 프로젝트 패키지에 명시되어 있지 않지만 이용할 수 있습니다. 이를 두고 우리는 유령 의존성(phantom dependency)라고 부릅니다. npm이나 yarn classic에서는 유령 의존성 문제가 발생했습니다.

추가로, 현재 yarn berry에서는 `.pnp.cjs`를 통해 의존성 목록을 기술하는 PnP 방식을 통해 유령 의존성 문제가 해결된 상태입니다.

### hoisted의 trade-off

#### 얻는 것

pnpm에서 node-linker=hoisted 옵션을 이용한다면 동일한 hoisted 구조를 이용함으로써 기존 도구나 라이브러리와의 충돌이 감소합니다. 또한, Next.js의 standalone 모드와 같은 특수 빌드 환경에서 잘 동작합니다.

#### 잃는 것

그러나 pnpm의 엄격한 링크 관리가 가지는 장점이 사라지면서 유령 의존성 문제가 다시 발생할 위험이 생깁니다. 또한, 잠재적인 버전 충돌 위험 등의 영향이 발생하며 유지보수성이 저하될 수 있습니다.

## 1편 부록: Windows 환경과 Linux BTRFS 환경에서의 링크

### Windows 환경에서의 Junction

```typescript
/* 
    https://github.com/pnpm/symlink-dir/blob/main/src/index.ts#L7-L11
*/

const IS_WINDOWS =
  process.platform === "win32" ||
  /^(msys|cygwin)$/.test(<string>process.env.OSTYPE);

// Always use "junctions" on Windows. Even though support for "symbolic links" was added in Vista+, users by default
// lack permission to create them
const symlinkType = IS_WINDOWS ? "junction" : "dir";
```

위 코드는 [pnpm/symlink-dir] (https://github.com/pnpm/symlink-dir/blob/main/src/index.ts) 레포지토리에서 가져온 심볼릭 링크 생성에 쓰이는 코드입니다. 재밌는 점은 Windows 환경에서는 심볼릭 링크가 아니라 Junction이 이용된다는 것입니다.

Junction은 로컬 디렉토리를 가리키는 NTFS 링크로, 심볼릭 링크와 달리 관리자 권한이 필요하지 않습니다. Windows 같은 경우는 일반 유저가 심볼릭 링크를 생성할 수 없다는 점이 중요한 듯 보입니다. 또한, Junction에서 파일을 링크할 수 없다는 단점은 로컬 디렉토리만을 이용하는 pnpm에서는 문제가 되지 않습니다.

### Linux BTRFS 환경에서의 링크

BTRFS(B-tree File System)은 B-tree 자료구조 기반의 Linux의 Copy-on-Write 방식을 기본으로 이용하는 파일 시스템입니다. BTRFS 파티션에서는 내부적으로 독립적인 inode 번호 공간을 가지는 서브 볼륨을 만들 수 있는데요. 문제는 pnpm이 다른 서브 볼륨간에는 하드 링크를 생성할 수 없다는 데서 발생합니다.

이를 해결하기 위해서 pnpm은 reflink를 이용합니다. 여기에서 reflink는 실제 데이터를 복사하지 않고 동일한 inode 블록을 가리키는 새로운 참조만 생성합니다. 여기에서 Copy-On-Write 메커니즘을 이용해 수정할 때만 실제 데이터 복사가 일어나고, 나머지 수정되지 않은 부분은 여전히 원본 파일의 데이터 블록을 공유합니다.

pnpm은 이를 활용해 BTRFS의 서브 볼륨 간 패키지 공유를 진행한다고 합니다.

## 나가며

이번 글은 문제를 제시하기 전, 패키지 매니저 pnpm을 먼저 이해하는 취지에서 작성되었습니다. 핵심 특징인 콘텐츠 주소 지정 가능 저장소와 링크 방식에 대해서도 살펴보았는데요. 특히 하드 링크와 심볼릭 링크를 통한 효율적인 패키지 관리 방식을 이해할 수 있었습니다.

pnpm은 전역 저장소와 프로젝트 간의 하드 링크를 통해 디스크 공간을 절약하고, 심볼릭 링크를 통해 의존성을 엄격하게 관리합니다. 이러한 구조는 유령 의존성 문제를 해결하면서도 저장 공간을 효율적으로 사용할 수 있게 해줍니다.

또한 플랫폼별 특성도 고려되어 있음을 확인했습니다. Windows에서는 권한 문제를 피하기 위해 Junction을 사용하고, Linux의 BTRFS 환경에서는 서브볼륨 간의 제약을 reflink로 해결하는 등 각 환경에 최적화된 전략을 채택하고 있습니다.

다만 이러한 pnpm의 장점들은 `node-linker=hoisted` 옵션을 사용하면 일부 포기해야 할 수 있습니다.

다음 글에서는 pnpm을 Next.js의 standalone 모드와 함께 사용할 때 발생하는 문제와 손쉬운 해결책으로서의 `node-linker=hoisted`가 중심이 되겠습니다. 나아가, 과연 `node-linker=hoisted`가 최선일지도 고민해보겠습니다.

## Refereneces

https://pnpm.io/

https://toss.tech/article/lightning-talks-package-manager

https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Client-side_tools/Package_management

https://dev.to/chlorine/how-does-pnpm-work-5mh

https://toss.tech/article/node-modules-and-yarn-berry

https://learn.microsoft.com/en-us/windows/win32/fileio/hard-links-and-junctions
