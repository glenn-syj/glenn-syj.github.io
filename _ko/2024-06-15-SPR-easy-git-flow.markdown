---
title: 프로젝트 스프린트 2주차 (1) - git flow 쉽게 이용하기
lang: ko
layout: post
---

## 들어가며
### Disclaimer
- Last Update: 2024/06/30

- 현재 진행하는 쇼핑몰 제작 프로젝트에서 팀장과 백엔드를 맡고 있습니다. 
- 이번 글은 협업 과정에서의 비효율을 줄이기 위해, git flow 브랜치 생성과 커밋 메시지 관련 자동화를 진행한 경험을 다룹니다.

<br/>
### 배경
아무리 git에서 git flow 명령어를 지원한다고 하더라도, 브랜치를 생성할 때 발생하는 휴먼 에러는 막기 쉽지 않습니다. 이번 프로젝트의 경우, JIRA를 업무 관리 협업 툴로 이용하고 있는데요. 업무 프로세스는 다음과 같은 과정을 따릅니다.

1. JIRA 티켓 생성
2. git 브랜치 생성
3. commit 후 PR 생성 
4. 코드 리뷰 후 approve
5. merge 후 브랜치 삭제

여기에서, 자동화로 비효율을 빠르게 줄일 수 있는 부분은 **2. git branch 생성**과 **3. commit 후 PR 생성** 에 있다고 보았습니다. **1. JIRA 티켓 생성**이나 **4. 코드 리뷰 후 approve**, **5. merge 후 브랜치 삭제**는 수동 확인이 필수불가결하다는 점 또는 지금 수준에서는 충분히 효율적이라고 판단하였습니다.

**2. git branch 생성**은 [지난 git flow 설명 글](https://glenn-syj.github.io/ko/posts/2024/06/09/git-flow-seminar/)에서의 브랜치 네이밍 컨벤션이 JIRA 이슈를 포함시키는 방향으로 수정되었다는 점이 중요했습니다. `feature/{team}/{jira-prefix}-{issue-number}-{feat-name}`와 같은 방식으로 이용하려다보니, 이름이 길어진 브랜치를 생성하면 휴먼 에러가 발생하기 쉬웠습니다. 또한, 지난 경험에서 판단해볼 때 아무리 git flow 명령어를 이용하더라도 수기로 작성하면 귀찮다는 점은 매한가지였습니다.

**3. commit 후 PR 생성**은 JIRA에서 제공하는 업무 상태 자동화와 연관됩니다. JIRA에서는 커밋 메시지에 이슈 번호가 포함되면 진행 상태를 In Progress로, PR 후 merge가 되면 상태를 Done으로 바꾸는 기능을 제공합니다. 그러나 커밋 메시지에 일일이 JIRA 이슈 번호를 넣는 것은 역시 에러가 발생하기 쉬우면서도 귀찮은 일입니다.

따라서 이번에는 (1) git flow 브랜치 컨벤션에 들어가는 노력 감소와 (2) JIRA-Git(hub) 연동 환경에서의 자동화를 목표로 비효율을 개선하고자 했습니다. 중심 기능에서는 chatGPT의 도움을 받아 만든 **bash shell script**와 **git hooks**, **JIRA automation**, **python**이 큰 역할을 했습니다.

이번 글에서는 특히 **2. git branch 생성**과 관련하여 git flow 브랜치 생성을 자동화한 경험만을 다룹니다. JIRA automation과 git hooks를 이용하는 글은 다음에 다루겠습니다.

<br/>
## git flow 브랜치 생성 자동화
### 기존 방식의 문제점

**누락과 오타는 수기 입력에서 비롯한다**

`git flow init` 명령어 이후에, `git flow feature start ...`과 같은 명령어를 이용하는 것은 그리 어렵지 않습니다. 그러나 작성 이후에 `/` 슬래쉬나 `-` 하이픈의 누락, 브랜치 명에서의 오타 등을 발견한다면 다시 삭제하고 만드는 과정은 확실히 귀찮습니다. 사실 `git flow feature start ...`는 사실상 브랜치 명을 일일이 쓰는 것과 동일하다는 점도 그렇습니다.

즉, 브랜치 생성 시 입력 과정에서의 가장 큰 문제는 수기 입력이었습니다. 수기 입력은 타이핑이 시작되는 순간부터,  오타나 누락의 유무에 정신을 쏟게 만듭니다. 잘못 생성된 브랜치는 삭제도 해야하므로, 사실 생성은 그저 생성에서 멈춘다고 할 수 없습니다. 그래서 (1) 최대한 특수문자를 입력하지 않으면서 (2) 브랜치 컨벤션에 맞출 수 있는 방식으로 자동화가 진행되어야 했습니다.

**develop 브랜치 최신화**

다음 버전의 개발이 진행되는 develop가 최신화되지 않는 상황에서의 불안함도 대부분 공유하고 있었습니다. 당연히 해당 브랜치에서 develop를 pull해오면 되긴 하지만, 저는 심리적 안정성도 비효율을 줄이는 큰 요인이라고 생각합니다. 소규모 프로젝트에서 충돌이 발생한다면, 초기에 확인할 수록 좋다고 판단했습니다.

<br/>
### easy-git-flow 도입

**easy-git-flow**

[easy-git-flow](https://github.com/glenn-syj/easy-git-flow)는 제가 프로젝트에서 이용하기 위해 만든 bash 파일을 확장해 오픈소스 프로젝트로 진행해보려는 시도입니다. (따라서 bash 파일 생성 이후 사후적으로 만들어졌습니다.) 그러나 현재 (2024.06.30) 가지는 기능과 결은 같으며, 한글 주석은 제거되었고 영문 주석은 지속적으로 다듬어야하는 상태입니다. 

참고: 이번에는 JIRA를 이용하는 소규모 프로젝트이기에, git flow에서 제공하는 심화 기능을 이용하지 않아도 괜찮다고 판단하였습니다.

<br/>
- 수기 입력 대신 option 선택

```bash

# branch_types: an array where the branch types are stored
branch_types=( "feature" "bugfix" "release" "hotfix" )

# team_types: an array where the team types are stored
team_types=( "" "back" "front" )

...

echo -e "${GREEN}[1] feature [2] bugfix [3] release [4] hotfix ${RESET}"
echo -e "${YELLOW}enter your branch type${RESET} [default = 1]: \c"
read branch_idx

...

echo -e "${GREEN}[1] none [2] back [3] front${RESET}"
echo -e "${YELLOW}enter your team${RESET} [default = 1]: \c" 
read team_idx

```

가장 중요한 특징 중 하나로 위의 브랜치 타입만이 아니라, 팀 타입까지 옵션처럼 이용할 수 있게 하였습니다. 팀 타입에서 none을 선택하면 팀 타입이 브랜치 명에 포함되지 않습니다. 이는 front와 back를 굳이 구분하지 않는 컨벤션에서 이용하기 위함입니다. 해당 번호를 입력받으면, 이에 걸맞는 브랜치 타입이나 팀 타입이 브랜치명 결과에 추가됩니다.

<br/>
- No hyphen, No slash, and No upper case

```bash
# step done: successfully get the input for a jira issue number variable
result="$result/$jira_prefix-$issue_num"

...

# join each words with replacing blanks as a single '-' and turn them into lower-cases
brief_desc=$(echo "$brief_desc" | tr -s ' ' '-')
brief_desc=$(echo "$brief_desc" | tr "[A-Z]" "[a-z]")
```

하이픈이나 슬래쉬를 입력하는 일은 여간 귀찮지 않습니다. 또는 브랜치 이름에 의도적이지 않지만 Caps Lock이나 쉬프트 탓에 대문자가 들어간다면 삭제하고 싶기 마련입니다. easy-git-flow의 flow.sh에서는 자동으로 하이픈을 작성하거나, 공백을 하이픈으로 대체합니다. 또한, 브랜치 이름에 포함되는 요약적 설명에서 대문자를 소문자로 변환합니다.

<br/>
- 간단한 검증 로직과 확인 절차

```bash 
 # target_version should be a string like 'x.x.x' where x is a number
if [[ "$target_version" =~ ^[0-9]+.[0-9]+.[0-9]+$ ]]; then    
				result="$result/$target_version"
				break
```

처음부터 잘못된 값이 입력되는 경우도 있습니다. 게다가 불특정 다수에게서 아직 많은 피드백을 듣지 못했으므로, 이용자에게 얼마나 친화적인지 파악하기 힘들었습니다. 그래서 일단 최대한 간단하게 검증 로직을 추가했습니다.

위 코드는 `hotfix` 브랜치와 `release` 브랜치에서 이름에 버전을 추가할 때 쓰이는 검증 로직입니다. 정규식 `^[0-9]+.[0-9]+.[0-9]+$`을 이용해 입력받은 변수 `target_version`에 대해, `한 자리 수. 여러 자리 수. 여러 자리 수`와 같이 버전을 입력하도록 만들었습니다. `.`은 다른 브랜치에 비해 `hotfix`와 `release` 브랜치는 생성 빈도가 낮으며 시급함은 높다고 판단해 입력하도록 두었습니다.

```bash
# branch name checking manually 
echo -e "\nbranch name: ${GREEN}$result${RESET}" 
echo -e "${YELLOW}press enter to continue.${RESET}"
read consent;

if [[ $consent == "" ]]; then
    git checkout -b "$result"
else 
    echo -e "${RED}The process is canceled.${RESET}"
fi
```

그리고 마지막으로, 바로 브랜치를 생성하는 대신 생성될 브랜치 명을 확인하고 해당 브랜치로 이동하도록 했습니다. 특히 브랜치를 잘못 만드는 경우, 삭제할 필요가 없어서 제게도 굉장히 유용한 기능이었습니다.

<br/>
- develop 브랜치 최신화

```bash
# switch to develop and pull origin develop
git checkout develop
current_branch=$(git rev-parse --abbrev-ref HEAD)

if [[ $current_branch != "develop" ]]; then
    echo -e "${RED}No develop branch found.${RESET}"
    exit;
fi

echo "Pulling origin develop ..."
git pull origin develop
echo -e "\n"
```

브랜치 이름에 필요한 요소들을 입력받기 전에 우선적으로 `develop` 브랜치가 최신화되어있는지 확인하고 진행합니다. 먼저 `develop` 브랜치로  이동했을 때, `$(git rev-parse --abbrev-ref HEAD)` 명령어를 통해 가져온 현재 브랜치이름이 develop가 아니라면 shell script가 종료됩니다.  이는 develop 브랜치가 존재하지 않거나, git과 관련되어 현재 브랜치를 떠날 수 없는 경우를 고려했습니다.

이후에는 원격 저장소 내 `develop` 브랜치에서 로컬 저장소 내 `develop` 브랜치로 pull함으로써 최신화를 진행합니다. 하지만 현재 버전에서는 원격 저장소에 `develop`가 존재하지 않는 경우는 고려하지 못했습니다. 대신, `git flow init` 명령어를 통해 초기에 git flow를 잘 초기화했다면 문제가 없을 것으로 판단했습니다.

<br/>
### 느낀점

**문서화의 중요성**

지난 번 git flow 세미나를 진행한 뒤에, 자동화와 관련된 세미나를 진행했습니다. 요약하자면 간단하게 코드를 설명하고, 이용하는 방법을 알려주는 시간이었습니다. 하지만 해당 세미나 이후, 문서화가 빠르게 진행되지 않아 팀원들이 사용에 추가적인 어려움을 겪었습니다. 비록 다른 저장소를 팠지만, 문서화가 간단한 quick start라도 우선적으로 작성될 필요가 있습니다.

**자동화의 효율과 그 가능성**

사실 모든 업무를 자동화하는 것은, 아무것도 자동화하는 것보다도 위험하다고 생각합니다. 이는 자동화가 단순히 프로세스에 들이는 노력을 감소시키는 일이 아닌 까닭입니다. 자동화 과정에 들어가는 비용, 자동화에서 비롯되는 오류나 검증 부족으로 인한 리스크, 그리고 추가적인 학습 시간을 고려해야합니다.

이번 프로젝트에서는 코드 퀄리티와 별개로 자동화를 혼자서 진행했음에도, 모두가 유용하게 이용했기에 충분히 효율적이라고 판단했습니다. 특히 모두가 편하다고 이야기하며 파일을 이용할 때, 팀의 효율을 위해 기여하는 경험이 기억에 남습니다.

**shell script를 두려워하지 말자 with AI**

linux 명령어 몇몇은 알고 있지만, shell script 자체를 작성하는 경험은 없었습니다. 이번에는 이를 두려워하기보다, chatGPT의 도움을 받아 shell script 문법을 학습하며 원하는 결과물을 내어보고 싶었습니다. 특히 프로젝트를 위해서라면, shell script를 처음부터 배우는 것보다 필요한 내용을 위주로 코드에서 학습을 시작하는 편이 낫다고 판단했습니다.

결과적으로 bash 파일을 작성하며 bash 파일에서의 문법이 얼마나 강경한(strict) 지를 깨달을 수 있었습니다. 조건문, 비교, 변수 이용, 배열 생성에 관한 문법만이 아니라 띄어쓰기 하나에도 민감하게 작성해야한다는 점을 배웠습니다. 이러한 배움은 chatGPT가 없었다면 해내긴 했어도 더욱 많은 심리적 투자가 필요했을 것입니다. 

결론적으로 chatGPT를 통해서 shell script에 대한 학습적 허들을 낮출 수 있었습니다. 생성형 AI를 이용해, 저는 다른 분야에서도 학습 허들을 낮출 수 있다고 생각합니다. 물론, 적절한 자료를 통해 정보를 확인하고 새로운 관점을 찾는 일이 필수적입니다.
## References

https://stackoverflow.com/questions/2111042/how-to-get-the-name-of-the-current-git-branch-into-a-variable-in-a-shell-script

https://kodekloud.com/blog/regex-shell-script/
