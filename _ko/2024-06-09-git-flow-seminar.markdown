---
title: 프로젝트 스프린트 1주차 (1) - git flow 세미나
lang: ko
layout: post
---

## 들어가며
### Disclaimer
- Last Update: 2024/06/09

- 현재 진행하는 쇼핑몰 제작 프로젝트에서 백엔드를 맡고 있습니다. 
- 이번 글은 DB 설계나 심화 알고리즘에서의 고민보다는 프로젝트 팀원을 대상으로 한 git flow 세미나에 기반합니다.

### 배경
SSAFY(삼성 청년 SW 아카데미)에서 같은 반으로 한 학기를 같이 보낸 친구들과, 방학을 맞아 다음 학기를 시작하기 전 프로젝트를 진행하기로 했습니다. 프론트엔드 4명, 백엔드 2명으로 구성되어 있습니다.

프로젝트 협업 환경으로 슬랙, 노션, JIRA, github을 선택했습니다. 특히, 슬랙은 매일 진행되는 데일리 스크럼, 랩업 스크럼에 유용하게 쓰고 있습니다. SSAFY 1학기에서의 프로젝트 경험으로, 모두가 협업 환경과 코드 버전 관리의 중요성을 느끼고 있던 바였습니다.

이번 쇼핑몰 프로젝트에서는 프론트엔드는 인원이 많은 대로, 백엔드는 인원이 적은 대로 다시 한 번 git을 어떻게 잘 이용할 것인지 고민해보았습니다.

결론적으로 이번에는 git flow를 확실히 알고 쓰자는 의미에서, 제가 git flow 세미나를 주최해 보았습니다.

## git flow 세미나
### git flow 선정 이유
#### 1. 친숙성
대부분의 팀원이 git과 github에 익숙했습니다. 특히, 6명 중 대부분이 이전 프로젝트에서 git flow를 도입해보았다는 점도 중요했습니다.

하지만 많은 팀원이 git bash 자체에서 git flow를 이용하지 않고, 직접 브랜치를 생성하는 등의 비효율적인 상황을 겪고 있었습니다.

#### 2. 독립성 유지
Git flow는  프론엔드가 넷, 백엔드는 둘이므로 특히 많은 업무가 병행되어야 한다고 판단했습니다.

특히, 어떤 기능의 일부가 완성되지 않았다고 해서 다른 기능이 개발되지 않는다면 미리 정해둔 개발 일정을 맞출 수 없을 것입니다.

따라서 최대한 독립성을 유지하기에, 각 브랜치에서 문제가 발생해도 큰 영향이 없다는 것 역시 장점이라고 생각했습니다. 

#### 3. 빠르고 쉬운 대처

Git flow가 제공하는 main branch (develop, main)와 support branch (release, feature, hotfix, bugfix, support)는 다양한 시나리오에 쉽게 대처할 수 있도록 돕습니다.

Git flow를 설명하는  자료들에서는 대개 bugfix 브랜치를 굳이 이용하지 않았습니다. 그러나 소규모 단기 프로젝트에서는 기능 개발 내에서 이루어지는 오류 해결과 기능 개발 이후 수정이 필요한 오류 해결이 구분되어야 한다고 보았습니다. 실제로도 지난 프로젝트에서 이를 구분해 bugfix 브랜치를 추가로 이용한 것이 효율적이었다는 회고를 진행했습니다.

#### 4. 기능 별 우선순위 분리

약 1달의 개발 기간 동안, 1주로 나뉘는 네 번의 스프린트가 진행됩니다. 기간이 짧은 만큼, 처음에 계획했던 대로 개발되기는 쉽지 않다고 보았습니다.

따라서 기능을 필수 기능, 핵심 기능, 추가 기능으로 분류하고 우선순위가 높은 기능을 먼저 개발할 필요가 있습니다. 

Git flow는 필수 기능이 모두 구현된 후, 혹은 필수 기능의 개발과 함께 핵심 기능을 개발하는 것에서 이점을 가집니다.

### git flow 설명

#### git flow init
Windows 환경에서도 git version 2.5.3 이후부터는 git flow를 적용하기 위해서 cygwin 등을 이용해 따로 설치를 진행할 필요가 사라졌습니다. 세미나 진행시에는 슬랙의 허들 기능을 통해서, 명령어를 하나씩 쳐보면서 진행했습니다.

![git flow init 명령어](https://github.com/glenn-syj/glenn-syj.github.io/assets/65771798/b847de3d-c83b-40c4-b2ee-7746654da043)

가장 먼저 git bash에서 git flow init 명령어를 입력하면 위와 같은 화면이 나타납니다. 각 파트가 어떤 의미인지 함께 살펴보겠습니다.

```git
Branch name for production releases: [main]
Branch name for "next release" development: [develop]
```

먼저 주요 브랜치입니다. 현재 배포가 되어 진행중인 브랜치는 main이고, 다음 출시 버전을 개발하는 브랜치는 develop입니다. 다음 출시 버전이라고 하면 헷갈릴 수 있겠지만, 가장 처음에는 출시 버전이 없으므로, 마찬가지로  develop 브랜치가 다음 출시 버전을 다룬다고 보면 됩니다.

```git
Feature branches? [feature/]
Bugfix brancehs? [bugfix/]
Release branches? [release/]
Hotfix branches? [hotfix/]
Support branches? [support/]
Version tag prefix? []
```

아래는 supporting branch입니다. 저희 프로젝트에서는 feature, bugfix, release, hotfix 브랜치는 이용하되 support 브랜치는 이용하지 않습니다. 실제로 프로젝트에서 이용중인 컨벤션과 함께 support를 제외한 각 브랜치를 살펴보겠습니다.

#### git flow 브랜치

- 브랜치 역할

```
- main      : 최종 배포가 진행되는 branch
- release   : 현재 출시 버전이 진행되는 branch
- develop   : 다음 출시 버전을 준비하는 branch
- hotfix    : main 브랜치에서 배포 후 긴급 수정이 필요할 때 이용
- bugfix    : develop 브랜치에서 기능 구현 후 버그 수정을 진행할 때
- feature   : develop를 위한 기능 개발 이루어지는 branch (fix가 이루어져도 됨)
```

여기에서 release 브랜치는 각 버전 별 목표 기능이 완수되었을 때, 출시 버전에 대한 품질 체크나 버그를 확인하는 용도로도 쓰인다는 점 역시 추가로 논의해 확립했습니다. 앞서 설명했듯, bugfix 브랜치는 기능 구현 후에 추가적으로 이루어지는 버그 수정에서 개발 중인 feature와 구분됩니다.

- 브랜치 네이밍

```
- main
- release/{version}            : {version}은 1.1.1과 같은 방식으로 표기
- develop
- hotfix/{version}             : {version}은 1.1.1과 같은 방식으로 표기
- bugfix/{team}/{bug-related}  : 
- feature/{team}/{feat-name}   : {feat-name}은 스페이스 대신 하이픈('-')을 이용
```

bugifx 브랜치와 feature 브랜치에서 공통적으로 쓰이는 `{team}`에는 `front` 혹은 `back`이 이용됩니다. 같은 레포지토리에서 작업하기로 논의했으므로, 마주하는 기능이 같더라도 최소한의 구분을 두었습니다.

특히 supporting branch는 `git flow {prefix} start {branch-naming}`과 같은 방식으로 이용할 수 있고, 각 브랜치의 역할에 대한 구분이 팀 구분보다 우선되어야 하기에 `{team}`의 위치를 `{prefix}` 뒤에 두었습니다.

![git flow로 브랜치 생성하기](https://github.com/glenn-syj/glenn-syj.github.io/assets/65771798/d1c72e6c-1949-4d79-afdf-41870ba121a7)

위 캡쳐처럼 브랜치를 생성하고 변경해가며 이용할 수 있습니다. (이 역시 git flow 세미나에서 명령어를 입력해보며 진행했습니다.)

#### git flow를 어떻게 적절히 이용할까?
먼저 JIRA 티켓-커밋-PR을 최대한 적절히 하나로 묶을 수 있는 단위로 잘라서 이용하기로 결정했습니다. 특히 대규모 프로젝트를 경험해 본 적이 없기도 하지만, 소규모 프로젝트의 장점은 빠르게 진행 상황을 확인하고 반영할 수 있다는 점이라고 판단했습니다. 따라서 티켓-커밋-PR을 최대한 일치시켜 확인과 검증에 드는 비용을 줄이고자 했습니다. 물론 집착하지는 말고, 더 늘어나는 경우에는 그대로 올리도록 합의했습니다.

가장 우려스러웠던 부분은 rebase merge의 이용입니다. rebase merge는 히스토리를 깔끔하게 유지하게 만들고, 코드 리뷰를 용이하게 만드는 장점이 있습니다. 하지만 모두가 rebase merge를 적절히 사용해본 경험이 없어, 커밋 해시가 바뀌어 발생하는 오류에 너무 많은 비용이 드는 건 아닐지 고민되었습니다. 결론적으로는 커밋 컨벤션을 잘 작성한다는 가정 하에,  각 변동 사항을 파악하기 쉬운 일반 머지를 이용하기로 했습니다. 

대신, 프로젝트를 진행하며 rebase merge의 필요성을 체감한다면 추가적인 회의를 통해 이를 적용해볼지 새로 논의해볼 수 있겠습니다. 개발 일정에 맞추어 최소한의 기능을 맞추어나가고, feature-develop-release-master로 이어지는 브랜치 전략에서 rebase의 중요성을 느낀다면 먼저 제안해보려고 합니다. 자세한 내용은 프로젝트 진행 후에 말씀 드릴 수 있게 노력해보겠습니다. 

## References

https://techblog.woowahan.com/2553/

https://www.atlassian.com/ko/git/tutorials/merging-vs-rebasing

https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow

https://github.com/glenn-syj/TIL/blob/master/Git/240603_Git_Git-Flow-Explained.md
