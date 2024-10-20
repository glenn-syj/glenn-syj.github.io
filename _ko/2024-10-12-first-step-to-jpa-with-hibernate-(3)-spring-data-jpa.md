---
title: Hibernate와 함께하는 JPA 첫걸음 (3) Spring Data JPA
lang: ko
layout: post
---

## 들어가며

### Disclaimer

- Last Update: 2024/09/03

- Java Persistence API와 hibernate를 이용하는 과정에서, 둘에 대해 스스로 가지고 있던 오해를 풀고 더욱 잘 이해하려고 노력하는 글입니다.

<br/>

## Spring Data JPA와 추상화

### Spring Data JPA란?

Spring Data JPA는 추상화를 통해 JPA를 이용한 어플리케이션 개발을 용이하게 하는 프레임워크입니다. 기본적인 CRUD 작업, 쿼리 작성, 트랜잭션 관리를 핵심적으로 추상화해 반복적인 작업을 감소시킵니다. 즉, 복잡한 사항을 감추고 단순한 고수준의 인터페이스를 이용하는 대표적인 추상화가 이루어집니다.

Spring Data JPA가 추상화를 진행하는 핵심적인 인터페이스는 바로 `Repository`입니다. Spring Data JPA는 `Repository`를 상속하는 레포지토리만 스캔한다는 점을 고려하면, 왜 `Repository`가 중요한지 알 수 있습니다. 우선 흔히 이용되는 고수준 인터페이스인 `JpaRepository<T, ID>`의 구조를 살펴볼까요?

### JpaRepsitory에서 시작하기

```java
// 주석은 삭제 했습니다.
// https://github.com/spring-projects/spring-data-jpa/blob/main/spring-data-jpa/src/main/java/org/springframework/data/jpa/repository/JpaRepository.java

@NoRepositoryBean
public interface JpaRepository<T, ID> extends ListCrudRepository<T, ID>, ListPagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {

	void flush();

	<S extends T> S saveAndFlush(S entity);

	<S extends T> List<S> saveAllAndFlush(Iterable<S> entities);

	@Deprecated
	default void deleteInBatch(Iterable<T> entities) {
		deleteAllInBatch(entities);
	}

	void deleteAllInBatch(Iterable<T> entities);

	void deleteAllByIdInBatch(Iterable<ID> ids);

	void deleteAllInBatch();

	@Deprecated
	T getOne(ID id);

	@Deprecated
	T getById(ID id);

	T getReferenceById(ID id);

	@Override
	<S extends T> List<S> findAll(Example<S> example);

	@Override
	<S extends T> List<S> findAll(Example<S> example, Sort sort);
}

```

위는 `JpaRepository` 인터페이스의 소스코드입니다. 여러 흥미로운 특징들을 가지고 있는데요. 상속 구조에서 `ListCrudRepository`와 `ListPagingAndSortRepository`, `QueryByExampleExecutor`로 넘어가기 전 먼저 `JpaRepository`의 특징을 살펴보고 가겠습니다.

#### (1) @NoRepositoryBean

`@NoRepoistoryBean` 어노테이션은 인스턴스가 생성될 때 Repository로서 프록시 객체를 생성하지 않도록 하는 데 쓰입니다. 주로 추가적인 메서드를 중개하는(intermediate) 레포지토리에서 쓰입니다.

만약 이 어노테이션을 적용하지 않는다면, `Repository`를 상속하지만 중개적인 역할을 하는 레포지토리가 일반 레포지토리로 간주되어 Bean으로 인스턴스화 된다고 예상할 수 있습니다. 이러한 경우에는, 구체적인 타입 정보가 없어서 빈 생성 오류가 뜨기 마련입니다.

따라서 `JpaRepository` 상위의 `ListCrudRepository` 등에서도 `@NoRepositoryBean`을 이용합니다. 역으로, `JpaRepository`를 상속해 이를테면 `BaseRepoistory`와 같은 추가적인 기능을 다루는 인터페이스를 만드는 경우에도 이용할 필요가 있습니다.

#### (2) getReferenceById(ID id)

`getReferenceById(ID id)`는 엔티티를 가져온다는 점에서, `findById(ID id)`와 유사합니다. 그러나 `getReferenceById(ID id)`는 지연 로딩(lazy loading)을 통해 프록시 객체를 반환하는 반면, `findById(ID id)`는 엔티티 객체를 반환합니다.

또한, 해당하는 엔티티가 존재하지 않을 경우, `getReferenceById(ID id)`는 Exception을 반환하지만, `findById(ID id)`는 비어있는 `Optional`을 반환합니다.

엔티티의 실제 데이터를 즉시 사용할 필요가 없이 연관 관계만 설정할 경우, 혹은 식별자만 필요한 경우에는 `getReferenceById(ID id)`가 성능 상 이점이 있음을 유추할 수 있겠네요.

대신, Exception을 반환한다는 점은 물론 트랜잭션 내에서만 안전하게 사용할 수 있음에 주의해야 합니다. 게다가 프록시 객체이므로, `equals()` 및 `hashCode()`를 이용하는 비교에서도 원하지 않는 결과를 냄도 마찬가지입니다.

추가적인 차이점과 동작 방식은 다른 글에서 보충해 보겠습니다.

### (3)
