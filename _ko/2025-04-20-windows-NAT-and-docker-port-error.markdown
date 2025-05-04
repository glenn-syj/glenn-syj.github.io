---
title: Docker 포트 에러로 알아보는 Windows NAT
lang: ko
layout: post
---

## 들어가며

- Windows 환경에서 WSL을 이용해 Docker가 구동될 때, 해당 포트를 이용할 수 없다는 에러를 해결하는 과정과 그 뒤에 존재하는 NAT 개념을 살펴보는 글입니다.
- 최종 업데이트: 25/05/03

## HTTP 500 에러와 포트 바인딩에서 시작하기

```
(HTTP code 500) server error - Ports are not available: exposing port TCP 0.0.0.0:8065 -> 0.0.0.0:0: listen tcp 0.0.0.0:8065: bind: An attempt was made to access a socket in a way forbidden by its access permissions.
```

윈도우즈 환경에서 도커를 이용하며 분명히 지난 실행에서는 잘 되었는데, 갑작스럽게 포트를 이용할 수 없다는 에러가 발생하고는 합니다. 심지어는 `netstat` 명령어로 확인해보아도 해당 포트를 이용하는 프로세스를 찾을 수 없었습니다.

검색을 통해서 찾아낸 해결책은 간단했습니다. `Windows NAT`가 바로 문제였습니다. 해결 자체는 쉬웠습니다.

(1) `net stop winnat`
(2) 컨테이너 시작
(3) `net start winnat`

그런데, Windows NAT란 도대체 뭐길래 포트 바인딩에서 문제를 일으킨 걸까요? 이런 문제가 발생할 때마다 winnat을 끄고, 시작하고, 켜는 과정을 반복해야할까요?

비록 클라우드 환경에서는 대부분의 운영체제가 리눅스지만, 저 역시 노트북에서 윈도우즈를 이용하고 있으므로 살펴보고자 합니다.
