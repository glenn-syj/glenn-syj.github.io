---
title: 비즈니스 로직과 테스트 코드에서의 트랜잭션 처리 방식 비교하기
lang: ko
layout: post
---

## 들어가며

- 지난 번의 [전략 패턴으로 금액 정산 비즈니스 로직 성능 비교하기](https://glenn-syj.github.io/ko/posts/2025/03/05/test-performance-with-strategy-pattern/) 포스트의 내용을 같은 팀원에게 공유하고 화상으로 설명을 진행했습니다.

- 이후 Pull Reqeust 내 코드에서 서비스 클래스 코드와 구체적 전략 코드 일부에 `@Transactional` 어노테이션이 붙어있음에 대한 질문이 들어왔습니다. 추가로, `@DataJpaTest`에 대해서도 설명하며 트랜잭션을 함께 살펴보게 되었습니다.

- 최종 수정일: 25/03/10

## @Transactional에 대한 질문

### 질문 상세

서비스 클래스의 메서드에도 `@Transactional`이 있고, 해당 메서드에서 이용하는 전략 클래스의 메서드에도 `@Transactional`이 왜 있나요? 동작이 다른가요? `@DataJpaTest`가 붙은 테스트 클래스 내의 메서드에서는 왜 `@Transactional`의 옵션이 `NOT_SUPPORTED`인가요?

### 질문에서 시작하기

이는 코드에서 `@Transactional` 어노테이션을 잘못 남겨뒀기 때문에 받은 질문인데요. 그 덕에 팀원과 함께 `@Transactional` 어노테이션과 Spring에서의 트랜잭션을 이해하는 계기가 되었습니다.

### 상황 가정

서비스 S와 레포지토리 R이 있고, S의 메서드 `S.m1()`에서 `R.m2()`를 호출한다고 생각해봅시다.

다음과 같은 세 가지 경우를 떠올릴 수 있는데요.

- `S.m1()`에만 `@Transactional`이 있습니다.
- `R.m2()`에만 `@Transactional`이 있습니다.
- `S.m1()`과 `R.m2()` 모두에 `@Transactional`이 있습니다.

우선, 기본값으로 지정된 `@Transactional(propagation=Propagation.REQUIRED)`에 대해 세 가지 경우를 살펴본 다음 `@Transactional` 어노테이션에 대해서 깊게 살펴보겠습니다.
