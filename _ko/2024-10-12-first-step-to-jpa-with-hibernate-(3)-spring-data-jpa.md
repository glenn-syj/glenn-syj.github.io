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

Spring Data JPA가 추상화를 진행하는 핵심적인 인터페이스는 바로 `Repository`입니다. 흔히들 이용하는 고수준 인터페이스인 `JpaRepository<T, ID>`의 구조를 우선 살펴볼까요?

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

위는 `JpaRepository` 인터페이스의 첫
