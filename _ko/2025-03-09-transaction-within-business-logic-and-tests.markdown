---
title: 비즈니스 로직에서 테스트까지의 Spring 트랜잭션 (1) - @Transactional
lang: ko
layout: post
---

## 들어가며

- 지난 번의 [전략 패턴으로 금액 정산 비즈니스 로직 성능 비교하기](https://glenn-syj.github.io/ko/posts/2025/03/05/test-performance-with-strategy-pattern/) 포스트의 내용을 같은 팀원에게 공유하고 화상으로 설명을 진행했습니다.

- 이후 Pull Reqeust 내 코드에서 서비스 클래스 코드와 구체적 전략 코드 일부에 `@Transactional` 어노테이션이 붙어있음에 대한 질문이 들어왔습니다. 추가로, `@DataJpaTest`에 대해서도 설명하며 트랜잭션을 함께 살펴보게 되었습니다.

- Spring AOP가 관여하는 부분에 대한 코드는 트랜잭션에 대한 설명을 넘어서는 것 같아, 의도적으로 배제했습니다.

- (1)에서는 주로 `@Transactional` 어노테이션이 동작하는 방식과 그를 구성하는 Spring Transaction에서의 핵심적인 작동을 살펴봅니다.

- 최종 수정일: 25/03/16

## @Transactional에 대한 질문

### 질문 상세

서비스 클래스의 메서드에도 `@Transactional`이 있고, 해당 메서드에서 이용하는 전략 클래스의 메서드에도 `@Transactional`이 왜 있나요? 동작이 다른가요? `@DataJpaTest`가 붙은 테스트 클래스 내의 메서드에서는 왜 `@Transactional`의 옵션이 `NOT_SUPPORTED`인가요?

### 기초에서 시작하기

이는 코드에서 `@Transactional` 어노테이션을 잘못 남겨뒀기 때문에 받은 질문인데요. 특히, 제가 이후 동작에 관해 제대로 설명하지 못했던 기억이 납니다. 그 덕에 팀원과 함께 `@Transactional` 어노테이션과 Spring에서의 트랜잭션을 학습하며 이해하는 계기가 되었습니다.

이번 글에서는 `@Transactional` 어노테이션의 동작에 대해 점층적으로 살펴보겠습니다. 이후 다음 글에서는 실제로 저희 비즈니스 로직과 테스트 코드에서 트랜잭션이 어떻게 관리되었는지 비판적으로 살펴보겠습니다. 이번 글은 이를 위한 기초 작업이라고 생각해주시면 감사하겠습니다.

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

Spring 프레임워크에서 `propagation` 옵션은 트랜잭션 전파 방식을 지정합니다. 트랜잭션 전파라고 함은 트랜잭션이 이미 진행되고 있는 경우 어떻게 트랜잭션을 처리할지를 결정하는 것을 의미합니다. 다음과 같은 옵션이 있습니다. 기본값은 `Propagation.REQUIRED`입니다. 간단하게 어떠한 옵션이 있는지 살펴보겠습니다.

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

### @Transactional 어노테이션을 쓰면 일어나는 일

`@Transactional` 어노테이션을 서비스 클래스의 메서드에 붙이면 어떤 일이 일어날까요?

#### 1. 프록시 객체 생성

- Spring Framework의 AOP(Aspect Oriented Programming)를 통해 해당 클래스의 프록시 객체를 생성합니다.
- `BeanFactoryTransactionAttributeSourceAdvisor` 빈이 `@Transactional` 어노테이션이 붙은 빈을 찾아 프록시로 감쌉니다.
- `AnnotationAwareAspectJAutoProxyCreator` 빈이 프록시 객체를 생성한 뒤 원본 대신 프록시를 컨테이너에 등록합니다.

#### 2. 트랜잭션 인터셉터 동작

- 프록시 객체는 서비스 클래스의 메서드 호출을 가로채서 트랜잭션 처리 로직을 실행합니다.
- `TransactionInterceptor` 클래스가 이를 담당하고, 내부적으로는 `PlatformTransactionManager`를 이용합니다.
- Spring Framework에서는 적절한 `PlatformTransactionManager` 구현체를 찾아 사용합니다.

#### 3. 트랜잭션 시작

- `PlatformTransactionManager` 인터페이스 구현체는 새로운 트랜잭션을 시작하거나 기존 트랜잭션에 참여합니다.
- JDBC 연결을 이용하는 트랜잭션 매니저는 커넥션을 획득한 뒤 auto-commit을 false로 설정합니다.

#### 4. 비즈니스 로직 실행

- 프록시 객체 내부에서 실제로 서비스 클래스의 메서드가 호출이 됩니다.
- 실제 메서드의 비즈니스 로직이 실행되며, 데이터베이스 작업이 수행됩니다.

#### 5. 트랜잭션 완료

- 비즈니스 로직 메서드가 정상적으로 완료되면 트랜잭션을 커밋합니다.
- 예외가 발생하면 롤백을 수행합니다 (rollbackFor 설정에 따라 다릅니다.)
- 사용한 데이터베이스 리소스를 정리합니다.

## @Transactional 깊이 살펴보기

### 트랜잭션의 생명주기

#### 1. TransactionalInterceptor의 호출 가로채기

먼저, `@Transactional` 어노테이션이 붙은 메서드의 프록시 객체는 TransactionInterceptor를 어드바이스로 이용합니다.

```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {

  // ...

  @Override
  public @Nullable Object invoke(MethodInvocation invocation) throws Throwable {

    Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

    return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
  }

}
```

`invoke()`는 Spring AOP에서의 `MethodInterceptor` 인터페이스의 계약인데요. 이 메서드를 통해서 `@Transactional`이 붙은 메서드가 호출될 때 Spring AOP 프록시에 의해 실행되는 메서드입니다.

위에서 나타나는 `invokeWithinTransaction()` 메서드는 `TransactionInterceptor` 클래스가 상속하는 추상 클래스 `TransactionAspectSupport`에서 트랜잭션 공통 처리를 제공하기 위한 메서드입니다.

#### 2. invokeWithinTransaction() - 속성 추출 및 매니저 결정

```java
// 이후 메서드명 생략
protected @Nullable Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
    final InvocationCallback invocation) throws Throwable {

  // If the transaction attribute is null, the method is non-transactional.
  TransactionAttributeSource tas = getTransactionAttributeSource();
  final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
  final TransactionManager tm = determineTransactionManager(txAttr, targetClass);

  // ...
}
```

`invokeWithinTransaction()` 메서드는 리플렉션을 활용해 호출된 메서드의 트랜잭션 속성을 추출합니다.

`TransactionAttributeSource`는 트랜잭션 속성을 추출하는 인터페이스입니다. 이 인터페이스는 메서드 레벨에서 트랜잭션 속성을 정의할 수 있도록 합니다.

그리고 `determineTransactionManager()` 메서드는 트랜잭션 매니저가 어노테이션에서 지정되지 않았다면 적절한 트랜잭션 매니저를 결정합니다. 기본 트랜잭션 매니저로는 `PlatformTransactionManager`의 구현체가 이용됩니다.

이후 `TransactionManager tm`은 트랜잭션 시작, 커밋, 롤백을 관리하도록 역할을 위임받습니다.

#### 3. invokeWithinTransaction() - 트랜잭션 정보 생성 및 트랜잭션 시작

```java
TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);
```

`createTransactionIfNecessary()` 메서드는 트랜잭션 속성과 트랜잭션 매니저를 이용해 트랜잭션 정보를 생성합니다.

만약 트랜잭션 속성 `TransactionAttribute txAttr`이 존재하고 트랜잭션 매니저가 설정된 경우에만 트랜잭션을 시작합니다.

`TransactionStatus` 객체는 `tm.getTransaction()` 메서드를 호출해 생성됩니다. 이 메서드는 내부적으로 구체적인 트랜잭션 매니저의 `doGetTransaction()` 메서드를 이용합니다. 대표적으로 `DataSourceTransactionManager` 클래스 구현체의 코드는 아래와 같습니다.

```java
// DataSourceTransactionManager.java
@Override
protected Object doGetTransaction() {
  DataSourceTransactionObject txObject = new DataSourceTransactionObject();
  txObject.setSavepointAllowed(isNestedTransactionAllowed());
  ConnectionHolder conHolder =
      (ConnectionHolder) TransactionSynchronizationManager.getResource(obtainDataSource());
  txObject.setConnectionHolder(conHolder, false);
  return txObject;
}
```

이후 `tm.getTransaction()` 메서드는 내부에서 반환된 `txObject`를 인자 중 하나로 받는 `startTransaction()` 메서드를 호출합니다. `startTransaction()` 메서드는 내부적으로 구현체에 동작을 위임하는 `doBegin()` 메서드를 이용해 트랜잭션을 생성하고 시작하며, `TransactionStatus` 객체를 생성합니다.

```java
// DataSourceTransactionManager.java
@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {
  DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
  Connection con = null;

  // 생략: connection 객체 생성 및 할당, 예외 처리, timeout 설정, readonly 설정 등
}
```

`startTransaction()`에서는 `doBegin()` 이후 `AbstractPlatformTransactionManager` 클래스의 `prepareSynchronization()` 메서드를 통해 트랜잭션 동기화가 준비됩니다. 여기에서는 `TransactionSynchronizationManager`를 이용해 스레드 내에서 트랜잭션 활성 상태, 격리 수준, 읽기 전용, 동기화 초기화 등을 설정합니다.

`startTransaction()`의 반환값으로 나오는 `TransactionStatus` 객체는 이후 `invokeWithinTransaction()`에서 최종으로 호출되는 return 문에서 내부적으로 `prepareTransactionInfo()` 메서드에서 이용됩니다.

```java
protected TransactionInfo prepareTransactionInfo(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, String joinpointIdentification,
        @Nullable TransactionStatus status) {

    // TransactionInfo 객체는 항상 생성
    TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);

    // 트랜잭션 속성 txAttr이 존재하면
    if (txAttr != null) {
        if (logger.isTraceEnabled()) {
            logger.trace("Getting transaction for [" + txInfo.getJoinpointIdentification() + "]");
        }
        txInfo.newTransactionStatus(status);
    }
    // 트랜잭션 속성 txAttr이 null이면
    else {
        if (logger.isTraceEnabled()) {
            logger.trace("No need to create transaction for [" + joinpointIdentification +
                    "]: This method is not transactional.");
        }
    }

    // 항상 비어있든 아니든 TransactionInfo를 현재 스레드에 바인딩
    txInfo.bindToThread();
    return txInfo;
}
```

트랜잭션이 필요하다면 `TransactionInfo` 객체를 생성하고 트랜잭션 상태를 설정합니다.

트랜잭션이 필요하지 않다면 트랜잭션 정보를 생성하지 않고 비어있는 `TransactionInfo` 객체를 반환합니다.

이어서 `txInfo.newTransactionStatus()` 메서드와 `txInfo.bindToThread()` 메서드를 호출합니다.

### 4. txInfo.newTransactionStatus() - 트랜잭션 상태 설정

`TransactionInfo` 객체는 인자로 받은 `TransactionStatus` 객체를 이용해 트랜잭션 상태를 새로 설정합니다. 대표적인 구현체로 `DefaultTransactionStatus` 클래스를 이용합니다.

```java
public class DefaultTransactionStatus extends AbstractTransactionStatus {

  // 트랜잭션 이름
	private final @Nullable String transactionName;

  // 실제 트랜잭션 객체
	private final @Nullable Object transaction;

  // 트랜잭션 새로 시작 여부 - 메서드 종료시 커밋 / 롤백 여부 결정
	private final boolean newTransaction;

  // 트랜잭션 동기화 설정 여부
	private final boolean newSynchronization;

  // 중첩 트랜잭션 여부
	private final boolean nested;

  // 읽기 전용 여부
	private final boolean readOnly;

  // 디버그 모드 여부
	private final boolean debug;

  // 트랜잭션 시작을 위해 보류된 트랜잭션 리소스 (REQUIRES_NEW 전파 옵션 사용 시)
	private final @Nullable Object suspendedResources;

  // 메서드
```

여기에서 `Object` 타입 필드인 `transaction` 필드는 실제 트랜잭션 객체를 저장합니다. 대표적으로 `TransactionManager` 구현체인 `DataSourceTransactionManager` 클래스의 `doBegin()` 메서드에서 이용됩니다.

앞서 살펴본 `DataSourceTransactionManager` 클래스의 `doGetTransaction()` 메서드에서 생성된 `DataSourceTransactionObject` 객체가 이 필드에 저장된다고 볼 수 있습니다.

#### 5. txInfo.bindToThread() - 트랜잭션 정보 바인딩

이후 `txInfo.bindToThread()` 메서드는 `TransactionInfo` 객체를 현재 스레드에 바인딩합니다.

```java
// TransactionInfo 클래스 내부 메서드

private void bindToThread() {
  // Expose current TransactionStatus, preserving any existing TransactionStatus
  // for restoration after this transaction is complete.
  this.oldTransactionInfo = transactionInfoHolder.get();
  transactionInfoHolder.set(this);
}
```

```java
private static final ThreadLocal<TransactionInfo> transactionInfoHolder =
			new NamedThreadLocal<>("Current aspect-driven transaction");
```

또한, `NamedThreadLocal` 클래스는 `ThreadLocal` 클래스에 toString() 메서드만 오버라이드하여 내부 검사를 위한 이름을 이용합니다. 실제로 Spring에서는 다양한 `NamedThreadLocal` 클래스를 이용해 `ThreadLocal@3a4afd8d`과 같은 이름보다 의미있는 이름으로 디버깅에 도움을 줍니다.

`transactionInfoHolder`는 `TransactionAspectSupport` 클래스의 정적 필드로, 현재 스레드에 바인딩된 `TransactionInfo` 객체를 저장합니다.

`ThreadLocal`은 멀티스레드 환경에서 스레드 간 변수 공유로 인한 동시성 문제를 방지하기 위해 제공하는 클래스인데요. 이는 각 스레드가 다른 스레드와 독립적인 변수 복사본을 가질 수 있게 합니다.

현재 `TransactionStatus`를 드러낸다(expose)고 표현한 것은, 스레드 로컬 변수에 저장해 같은 스레드 내에서 이용할 수 있게 드러낸다는 뜻입니다.

또한, 이전에 스레드에 바인딩된 트랜잭션 정보가 있다면 이를 `oldTransactionInfo`에 저장해 추후 복원에서도 이용할 수 있게 됩니다.

이후 `transactionInfoHolder.set(this)`를 통해 현재 `TransactionInfo` 객체를 `transactionInfoHolder`에 저장해 교체합니다.

`ThreadLocal`을 이용함으로써 Spring Transaction은 트랜잭션 컨텍스트의 흐름을 유연하게 관리하도록 돕는다고 말할 수 있겠습니다. 이는 이후 테스트 메서드를 함께 살펴볼 때 더 자세히 다루겠습니다.

#### 6. 다시 invokeWithinTransaction() - 비즈니스 로직 실행

스레드 로컬에 트랜잭션 컨텍스트가 설정되고 바인딩되면, 다음 인터셉터로 작업은 넘어갑니다.

만약 인터셉터 체인에서 더 이상 실행할 인터셉터가 없다면, 실제 클래스의 타겟 메서드를 호출합니다. 즉, 최종적으로 비즈니스 로직을 실행하고 그 결과가 반환됩니다.

```java
// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
      retVal = invocation.proceedWithInvocation();
```

이 과정에서는 Spring AOP에서 `invocation.proceedWithInvocation()` 메서드를 이용합니다. 이 `invocation`은 처음 람다식으로 넘겨낸 함수형 인터페이스 인자입니다.

#### 7. invokeWithinTransaction() - 트랜잭션 커밋/롤백 (트랜잭션 종료)

```java
// TransactionAspectSupport.java
// 둘러싸는 부분 생략

Object retVal;
try {
  // This is an around advice: Invoke the next interceptor in the chain.
  // This will normally result in a target object being invoked.
  retVal = invocation.proceedWithInvocation();
}
catch (Throwable ex) {
  // target invocation exception
  completeTransactionAfterThrowing(txInfo, ex);
  throw ex;
}
finally {
  cleanupTransactionInfo(txInfo);
}
```

이 코드에서 `retVal`에는 실제 비즈니스 로직을 실행한 값이 최종적으로 저장될 것입니다.

이후 `completeTransactionAfterThrowing()` 메서드는 예외가 발생한 경우 설정에 따라 트랜잭션을 롤백하고, 예외가 발생하지 않은 경우 트랜잭션을 커밋합니다. 자세한 코드는 아래와 같습니다.

```java
// TransactionAspectSupport.java
// 둘러싸는 부분 생략

if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
    try {
      // 이 부분에서 트랜잭션 매니저의 rollback() 메서드를 호출합니다.
      txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
    }
    catch (TransactionSystemException ex2) {
      // ...
    }
    catch (RuntimeException | Error ex2) {
      // ...
    }
  }
else {
  // We don't roll back on this exception.
  // Will still roll back if TransactionStatus.isRollbackOnly() is true.
  try {
    // 이 부분에서 트랜잭션 매니저의 commit() 메서드를 호출합니다.
    txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
  }
  catch (TransactionSystemException ex2) {
    // ...
  }
  catch (RuntimeException | Error ex2) {
    // ...
  }
}
```

여기에서 `rollback()`과 `commit()`은 추상 클래스 `AbstractPlatformTransactionManager`에서 정의된 메서드로, 메서드의 수행을 하위 클래스에 위임합니다.

따라서 `DataSourceTransactionManager` 클래스의 경우에 이용되는 `doRollback()` 메서드와 `doCommit()` 메서드는 다음과 같습니다.

```java
// DataSourceTransactionManager.java
@Override
protected void doRollback(DefaultTransactionStatus status) {
  DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
  Connection con = txObject.getConnectionHolder().getConnection();
  if (status.isDebug()) {
    logger.debug("Rolling back JDBC transaction on Connection [" + con + "]");
  }
  try {
    con.rollback();
  }
  catch (SQLException ex) {
    throw translateException("JDBC rollback", ex);
  }
}
```

```java
// DataSourceTransactionManager.java
@Override
protected void doCommit(DefaultTransactionStatus status) {
  DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
  Connection con = txObject.getConnectionHolder().getConnection();
  if (status.isDebug()) {
    logger.debug("Committing JDBC transaction on Connection [" + con + "]");
  }
  try {
    con.commit();
  }
  catch (SQLException ex) {
    throw translateException("JDBC commit", ex);
  }
}
```

이러한 하위 구현체의 코드를 이용해 `AbstractPlatformTransactionManager` 클래스의 `rollback()`과 `commit()` 메서드는 세이브 포인트 및 롤백 설정, 예외에 따라 트랜잭션을 커밋하거나 롤백합니다.

#### 8. cleanupAfterCompletion() - 트랜잭션 리소스 정리

`AbstractPlatformTransactionManager` 클래스는 `rollback()`과 `commit()` 메서드 종료 후 finally 블록에서 `cleanupAfterCompletion()`을 호출하는데요.

이는 하위 구현체의 `doCleanupAfterCompletion()` 메서드를 호출합니다. 이번에도 `DataSourceTransactionManager` 클래스의 코드를 이용해 살펴보겠습니다.

```java
@Override
protected void doCleanupAfterCompletion(Object transaction) {
  DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;

  // Remove the connection holder from the thread, if exposed.
  if (txObject.isNewConnectionHolder()) {
    TransactionSynchronizationManager.unbindResource(obtainDataSource());
  }

  // Reset connection.
  Connection con = txObject.getConnectionHolder().getConnection();
  try {
    if (txObject.isMustRestoreAutoCommit()) {
      con.setAutoCommit(true);
    }
    DataSourceUtils.resetConnectionAfterTransaction(
        con, txObject.getPreviousIsolationLevel(), txObject.isReadOnly());
  }
  catch (Throwable ex) {
    logger.debug("Could not reset JDBC Connection after transaction", ex);
  }

  if (txObject.isNewConnectionHolder()) {
    if (logger.isDebugEnabled()) {
      logger.debug("Releasing JDBC Connection [" + con + "] after transaction");
    }
    DataSourceUtils.releaseConnection(con, this.dataSource);
  }

  txObject.getConnectionHolder().clear();
}
```

먼저 매개변수로 전달된 `transaction` 객체를 타입 캐스팅합니다.

만약 트랜잭션에서 새로 생성된 커넥션이라면 데이터 소스와 연결 홀더의 바인딩을 해제합니다. 여기에서 이 바인딩이라 함은, `TransactionSynchronizationManager`가 `ThreadLocal<Map<Object, Object>> resources` 형태로 저장하는 매핑을 의미합니다. 이 Map에서 키는 `DataSource` 객체이고 값은 `ConnectionHolder` 객체입니다. 이 바인딩을 해제한다는 것은 물리적인 데이터베이스와의 연결이 아니라, Spring 트랜잭션 내에서의 현재 스레드와 데이터 소스 간 논리적 연결을 해제함을 의미합니다. 이후에는 연결 상태를 초기화하고, autocommit도 다시 true도 되돌립니다.

여기에서 커넥션을 새로 생성했다면 이를 해제하고, 다시 커넥션 풀로 반환해 물리적인 자원을 해제합니다.

- 새로 생성된 커넥션을 사용하는 경우

  - 최초 트랜잭션 시작 (REQUIRED의 첫 호출)
  - REQUIRES_NEW: 항상 새 커넥션을 생성하므로 완전히 정리
  - NOT_SUPPORTED: 트랜잭션 없이 실행되지만, 필요시 새 커넥션 사용 후 정리

- 기존 커넥션을 사용하는 경우

  - REQUIRED (참여하는 경우): 기존 커넥션 유지, 트랜잭션 상태만 정리
  - SUPPORTS, MANDATORY: 기존 커넥션 유지, 트랜잭션 상태만 정리
  - NESTED: 세이브포인트 관련 리소스만 정리, 기존 커넥션 유지

#### 9. cleanUpTransactionInfo() - 트랜잭션 컨텍스트 정리

이제 7. 에서 살펴보았던 finally 블록 안의 코드를 이용해 트랜잭션 정보를 정리합니다.

```java
finally {
  cleanupTransactionInfo(txInfo);
}
```

`cleanupTransactionInfo()` 메서드는 다음과 같이 구성되어 있습니다.

```java
// TransactionAspectSupport.java
protected void cleanupTransactionInfo(@Nullable TransactionInfo txInfo) {
  if (txInfo != null) {
    txInfo.restoreThreadLocalStatus();
  }
}

// TransactionAspectSupport의 static class TransactionInfo
private void restoreThreadLocalStatus() {
  // Use stack to restore old transaction TransactionInfo.
  // Will be null if none was set.
  transactionInfoHolder.set(this.oldTransactionInfo);
}
```

내부적으로 `TransactionInfo` 인스턴스 내 `oldTransactionInfo` 필드에 저장된 `TransactionInfo` 객체를 복원합니다. 즉, 현재 트랜잭션이 시작되기 전의 트랜잭션 정보가 복구되는데요.

이를 통해 중첩 트랜잭션이 종료되면 외부 트랜잭션 정보가 복원되며, 가장 외부 트랜잭션이 종료되면 null이 설정되어서 트랜잭션 컨텍스트가 정리됩니다.

결국 ThreadLocal 내의 `TransactionInfo` 객체가 null로 교체되는 것인데요. Spring Transaction에서는 `TransactionInfo` 내의 `TransactionStatus`를 이용해 컨텍스트 존재 여부를 확인하므로 결국 트랜잭션 컨텍스트가 정리되는 셈입니다.

여기서 트랜잭션 컨텍스트가 정리된다는 것은 Spring의 트랜잭션 관리 시스템에서 현재 스레드에 활성 트랜잭션이 없음을 나타내는 상태가 되며, 이후 새로운 트랜잭션이 시작될 수 있는 상태가 준비되어 있음을 의미합니다.

## 나가며

이번 글에서는 `@Transactional` 어노테이션을 중심으로 어노테이션의 기본적인 동작과 함께, Spring Transaction에서 트랜잭션의 생명주기와 작동 방식을 살펴보았습니다.

다음 글에서는 `@DataJpaTest` 어노테이션이 생성하는 트랜잭션과 함께, 내부적으로 이용되는 `@Transactional` 어노테이션이 어떻게 상호작용하는지 살펴보겠습니다.

게다가 아직 새롭게 던져야 할 질문도 있습니다. 트랜잭션 매니저는 어떻게 선정되는 걸까요? 테스트 코드에서 JPA와 JDBC가 한 번에 이용된다면 어떻게 될까요?

테스트 코드에서의 Spring Transaction이 어떻게 `ThreadLocal`을 이용하는지도 다음 글에서 다뤄보겠습니다.

긴 글 읽어주셔서 감사합니다. 다음 글도 잘 읽어주세요!
