---
title: 비즈니스 로직에서 테스트 코드까지의 트랜잭션 전파 방식 비교하기
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

우선, 기본값으로 지정된 `@Transactional(propagation=Propagation.REQUIRED)`에 대해 세 가지 경우를 살펴본 다음 `@Transactional` 어노테이션에 대해서 깊게 들어가보겠습니다. 그 전에, 먼저 `@Transactional` 어노테이션의 기본 동작을 살펴보겠습니다.

## @Transactional 동작 이해하기

저는 구체적인 동작을 살펴보기 전에, 한 가지 잊어서는 안되는 사실을 말하고 싶습니다. 바로 어노테이션은 마법처럼 동작을 진행해주는 주문이 아니라, 구현되어 있는 코드의 집합으로 동작한다는 사실입니다.

### @Transactional 어노테이션 코드

예를 들어, `@Transactional` 어노테이션은 다음과 같은 코드로 구현되어 있습니다.

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@Reflective
public @interface Transactional {

    // 트랜잭션 관리자 지정
	@AliasFor("transactionManager")
	String value() default "";

	@AliasFor("value")
	String transactionManager() default "";

    // 트랜잭션에 모니터링이나 디버깅을 위한 레이블 지정
	String[] label() default {};

    // 트랜잭션 전파 방식 지정
	Propagation propagation() default Propagation.REQUIRED;

    // 트랜잭션 격리 수준 설정
	Isolation isolation() default Isolation.DEFAULT;

    // 트랜잭션 타임아웃 설정
	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;

    // 트랜잭션 타임아웃 문자열 설정
	String timeoutString() default "";

    // 트랜잭션 읽기 전용 여부 설정
	boolean readOnly() default false;

    // 롤백 대상 클래스 지정 (클래스)
	Class<? extends Throwable>[] rollbackFor() default {};

    // 롤백 대상 클래스 이름 지정 (문자열)
	String[] rollbackForClassName() default {};

    // 롤백 대상 예외 클래스 이름 지정 (클래스)
	Class<? extends Throwable>[] noRollbackFor() default {};

    // 롤백 대상 예외 클래스 이름 지정 (문자열)
	String[] noRollbackForClassName() default {};

}
```

메타 어노테이션에 대한 깊은 설명은 `@Transactional`에 관한 주제에서 벗어나니, 간단하게만 살펴보겠습니다.

- 클래스와 메서드에 사용 가능함 (`@Target({ElementType.TYPE, ElementType.METHOD})`)
- 런타임 시점에 동작함 (`@Retention(RetentionPolicy.RUNTIME)`)
- 상속된 클래스에서도 어노테이션을 유지함 (`@Inherited`)
- JavaDoc에 문서화가 되어있음 (`@Documented`)
- 리플렉션을 요구함 (`@Reflective`)

메서드에 대한 설명은 각 메서드 위에 원본 영문 주석을 지우고 요약본으로 적어두었습니다. 메서드 중에서 오늘 질문의 대상은 바로 `propagation` 옵션입니다. (`isolation`은 이번 글에서 다루지 않겠습니다.)

### Propagation 옵션

Spring 프레임워크에서 `propagation` 옵션은 트랜잭션 전파 방식을 지정합니다. 트랜잭션 전파라고 함은 트랜잭션이 이미 진행되고 있는 경우 어떻게 트랜잭션을 처리할지를 결정하는 것을 의미합니다. 다음과 같은 옵션이 있습니다. 기본값은 `Propagation.REQUIRED`입니다.

- `Propagation.REQUIRED`

  기본 전파 속성으로, 현재 트랜잭션이 존재하면 그 트랜잭션에 참여하고 없으면 새로운 트랜잭션을 시작하는 옵션입니다. REQUIRED라는 말에서 알 수 있듯, 없으면 만들고 있다면 조건이 충족되어 넘어간다고 생각할 수 있습니다.

- `Propagation.REQUIRES_NEW`

  항상 새로운 트랜잭션을 시작하는 옵션입니다. 현재 트랜잭션이 존재하면 그 트랜잭션을 잠시 보류하고 새롭게 독립적인 트랜잭션을 시작합니다.

- `Propagation.NESTED`

  현재 트랜잭션이 존재하면 중첩된 트랜잭션을 시작하고, 없으면 새로운 트랜잭션을 시작합니다. 중첩된 트랜잭션은 부모 트랜잭션의 커밋/롤백에 영향을 받습니다. 세이브포인트를 이용해 부분 롤백이 필요한 경우에 이용됩니다. 따라서 REQUIRES_NEW와 달리 독립적이지 않습니다.

- `Propagation.MANDATORY`

  현재 트랜잭션이 존재하지 않으면 예외를 발생시키는 옵션입니다. 트랜잭션이 반드시 필요한 비즈니스 로직에서 이용됩니다.

- `Propagation.SUPPORTS`

  현재 트랜잭션이 존재하면 참여하고 없으면 트랜잭션 없이 진행합니다.

- `Propagation.NOT_SUPPORTED`

  현재 트랜잭션이 존재하면 잠시 보류하고 트랜잭션 없이 진행합니다. 즉, 트랜잭션이 잠시 중단되었다 해당 옵션을 가진 어노테이션의 클래스 혹은 메서드가 끝나면 다시 트랜잭션이 이어지게 됩니다.

- `Propagation.NEVER`

  현재 트랜잭션이 존재하면 예외를 발생시키는 옵션입니다. 트랜잭션이 없어야 할 때 이용됩니다.
