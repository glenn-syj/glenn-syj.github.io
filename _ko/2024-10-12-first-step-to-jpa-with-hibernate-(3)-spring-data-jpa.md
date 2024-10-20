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

Spring Data JPA가 추상화를 진행하는 핵심적인 인터페이스는 바로 `Repository`입니다. Spring Data JPA 내에서 이용되는 `...Repository` 인터페이스의 최상위 부모에는 `Repository<T, ID>` 인터페이스가 있습니다. Spring Data JPA는 `Repository`를 상속하는 레포지토리만 스캔합니다.

`Repository` 인터페이스는 도메인 엔티티와 데이터베이스 간 데이터 액세스 계층 정의에 이용되어, 엔티티 클래스와 DB 테이블의 매핑에 기반해 데이터베이스와 상호작용 할 수 있습니다.

우선 흔히 이용되는 고수준 인터페이스인 `JpaRepository<T, ID>`의 구조를 살펴볼까요? 대표적인 구현체로는 `SimpleJpaRepository`가 있지만, 이번에는 기능에 중심해 살펴볼 예정이므로 인터페이스를 다뤄보겠습니다.

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

대신, Exception을 반환한다는 점은 물론 트랜잭션 내에서만 안전하게 사용할 수 있음에 주의해야 합니다. 게다가 프록시 객체이므로, `equals()` 및 `hashCode()`를 이용하는 비교에서도 원하지 않는 결과를 낼 수 있습니다. 이는 JPA가 엔티티의 동일성을 확인할 때, `equals()`를 이용한다는 점에서 문제적입니다.

추가적인 차이점과 동작 방식은 다른 글에서 보충해 보겠습니다.

### ListCrudRepository와 CrudRepository

```java
// ListCrudRepository.java

@NoRepositoryBean
public interface ListCrudRepository<T, ID> extends CrudRepository<T, ID> {

	<S extends T> List<S> saveAll(Iterable<S> entities);

	List<T> findAll();

	List<T> findAllById(Iterable<ID> ids);

}
```

```java
// CrudRepository.java

@NoRepositoryBean
public interface CrudRepository<T, ID> extends Repository<T, ID> {

	<S extends T> S save(S entity);

	<S extends T> Iterable<S> saveAll(Iterable<S> entities);

	Optional<T> findById(ID id);

	boolean existsById(ID id);

	Iterable<T> findAll();

	Iterable<T> findAllById(Iterable<ID> ids);

	long count();

	void deleteById(ID id);

	void delete(T entity);

	void deleteAllById(Iterable<? extends ID> ids);

	void deleteAll(Iterable<? extends T> entities);

	void deleteAll();
}
```

`JpaRepository` 인터페이스가 상속하는 `ListCrudRepository` 인터페이스는 사실 `CrudRepository`를 `List`라는 콜렉션으로 반환값을 이용할 수 있게 만든 인터페이스입니다. 주로 다수의 엔티티로 `List`로 처리하는 경우가 많으므로, 명시적인 형변환 과정을 줄이고도 사용의 편의성을 챙겼습니다. 또한, 이 인터페이스들 사이에서도 `@NoRepositoryBean`의 존재를 다시 확인해볼 수 있습니다.

### ListPagingAndSortingRepository와 PagingAndSortingRepository

```java
@NoRepositoryBean
public interface ListPagingAndSortingRepository<T, ID> extends PagingAndSortingRepository<T, ID> {

	List<T> findAll(Sort sort);

}
```

```java
@NoRepositoryBean
public interface PagingAndSortingRepository<T, ID> extends Repository<T, ID> {

	Iterable<T> findAll(Sort sort);

	Page<T> findAll(Pageable pageable);
}
```

`ListPagingAndSortingRepository`도 마찬가지로 `PagingAndSortingRepository`에서의 `Iterable`을 `List`로 이용할 수 있도록 하는 레포지토리입니다. `PagingAndSortingRepository`는 페이징과 정렬을 추상화해둔 레포지토리인데요. `CrudRepository`와 함께 이용되어 추가적인 기능을 제공하는 레포지토리라고 이해할 수 있습니다.

흥미로운 점은 `Sort`, `Pageable`, `Page<T>` 등의 Spring Data 프레임워크 내에서의 자료구조입니다. 이러한 자료구조는 정렬과 페이징에서 어떻게 편의성을 제공하고 있을까요?

#### (1) Sort 클래스

```java
// Sort.java

public class Sort implements Streamable<org.springframework.data.domain.Sort.Order>, Serializable {

	private static final @Serial long serialVersionUID = 5737186511678863905L;

	private static final Sort UNSORTED = Sort.by(new Order[0]);

	public static final Direction DEFAULT_DIRECTION = Direction.ASC;

	private final List<Order> orders;

	protected Sort(List<Order> orders) {
		this.orders = orders;
	}

	private Sort(Direction direction, @Nullable List<String> properties) {

		if (properties == null || properties.isEmpty()) {
			throw new IllegalArgumentException("You have to provide at least one property to sort by");
		}

		this.orders = properties.stream() //
				.map(it -> new Order(direction, it)) //
				.collect(Collectors.toList());
	}

	public static Sort by(String... properties) {

		Assert.notNull(properties, "Properties must not be null");

		return properties.length == 0 //
				? Sort.unsorted() //
				: new Sort(DEFAULT_DIRECTION, Arrays.asList(properties));
	}

	public static Sort by(List<Order> orders) {

		Assert.notNull(orders, "Orders must not be null");

		return orders.isEmpty() ? Sort.unsorted() : new Sort(orders);
	}

	// ... 추가적인 메소드 생략


}
```

Sort 클래스는 정렬 기준을 지정하기 위해 이용되는 클래스인데요. 특정 필드에 대해 오름차순 이나 내림차순으로 정렬함은 물론, 비교 순서를 명확히할 수 있습니다. 기본적으로는 오름차순을 이용하고, `by()` 메서드를 통해서 특정한 정렬 기준에 맞는 `Sort` 클래스를 반환할 수 있습니다.

내부적으로 정적 클래스인 `Order`에는 direction, property, ignoreCase, nullHandling 등의 필드가 서술되어 있습니다. direction을 통해서는 정렬 방향(`ASC`, `DESC`)를 설정할 수 있으며, ignoreCase를 통해서는 대소문자 구분 유무를, nullHandling에서는 null 값을 정렬 시 처리하는 방식(`NULLS_FIRST`, `NULLS_NATIVE`)을 선택할 수 있습니다.

#### (2) Pageable과 Page<T>

```java

public interface Pageable {

	static Pageable unpaged() {
		return unpaged(Sort.unsorted());
	}

	static Pageable unpaged(Sort sort) {
		return Unpaged.sorted(sort);
	}

	static Pageable ofSize(int pageSize) {
		return PageRequest.of(0, pageSize);
	}

	default boolean isPaged() {
		return true;
	}

	default boolean isUnpaged() {
		return !isPaged();
	}

	int getPageNumber();

	int getPageSize();

	long getOffset();

	Sort getSort();

	// ...

	Pageable next();

	Pageable previousOrFirst();

	Pageable first();

	Pageable withPage(int pageNumber);

	boolean hasPrevious();

	// ...
}

```

`Pageable` 인터페이스는 페이징과 정렬 정보를 전달하기 위해 사용되는 인터페이스입니다. 페이지 번호, 페이지 크기, 정렬 기준 등을 설정함으로써 응답 시에 페이징된 결과를 `Page<T>` 인터페이스 구현체에 담게 됩니다.

`Pageable` 인터페이스는 주로 `PageRequest` 클래스로 구현됩니다. 특히, `next()` 메서드와 `previous()` 메서드는 설정을 그대로 가져가면서 다음 혹은 이전 페이지에 해당하는 요청을 생성합니다.

```java
public interface Page<T> extends Slice<T> {

	static <T> Page<T> empty() {
		return empty(Pageable.unpaged());
	}

	static <T> Page<T> empty(Pageable pageable) {
		return new PageImpl<>(Collections.emptyList(), pageable, 0);
	}

	int getTotalPages();

	long getTotalElements();

	<U> Page<U> map(Function<? super T, ? extends U> converter);
}
```

`Page` 인터페이스는 결과 데이터는 물론, 전체 데이터 개수, 전체 페이지 수, 현재 페이지 번호 등의 메타 정보를 포함하도록 합니다. 여기에서 흥미로운 점은 `Slice` 인터페이스를 상속하고 있다는 점인데요. `Slice` 인터페이스는 다음 페이지 여부와, 현재 페이지 데이터를 반환할 뿐이므로 더 간편한 반면, `Page` 인터페이스는 `Slice` 인터페이스의 기능과 더불어 페이지 계산을 용이하게 합니다. `Page`의 대표적인 구현체로 `PageImpl`이 이용됩니다.

하지만 `Page<T>`는 성능 상 오버헤드가 있는데요. 총 데이터 개수와 총 페이지 수를 계산하기 위해 추가적인 COUNT 쿼리를 작성하는 까닭입니다. 즉, 데이터 페이징 쿼리와 함께 COUNT 쿼리가 실행되어, 두 번의 쿼리가 진행되는 셈입니다. COUNT 쿼리 자체가 전체 데이터를 스캔한다면, 테이블 규모가 커질 수록 더욱 비효율적임에도 유의해야 합니다.

### QueryByExampleExecutor<T>

```java

public interface QueryByExampleExecutor<T> {

	<S extends T> Optional<S> findOne(Example<S> example);

	<S extends T> Iterable<S> findAll(Example<S> example);

	<S extends T> Iterable<S> findAll(Example<S> example, Sort sort);

	<S extends T> Page<S> findAll(Example<S> example, Pageable pageable);

	<S extends T> long count(Example<S> example);

	<S extends T> boolean exists(Example<S> example);

	<S extends T, R> R findBy(Example<S> example, Function<FluentQuery.FetchableFluentQuery<S>, R> queryFunction);
}
```

`QueryByExampleExecutor` 는 `Example` 인스턴스에 기반해 쿼리를 실행하도록 하는 인터페이스입니다. 여기서 `Example`이란, 예시적인 엔티티를 이용해 동적 쿼리를 간편하게 생성하고 실행하도록 돕는 인터페이스입니다.

```java
public interface Example<T> {

	static <T> Example<T> of(T probe) {
		return new TypedExample<>(probe, ExampleMatcher.matching());
	}

	static <T> Example<T> of(T probe, ExampleMatcher matcher) {
		return new TypedExample<>(probe, matcher);
	}

	T getProbe();

	ExampleMatcher getMatcher();

	@SuppressWarnings("unchecked")
	default Class<T> getProbeType() {
		return (Class<T>) ProxyUtils.getUserClass(getProbe().getClass());
	}
}
```

정적 메소드인 `Example.of()`는 `Example` 인터페이스의 구현체인 `TypedExample`을 생성합니다. 엔티티 객체와 필드 값을 설정한 뒤 파라미터로 넘겨주는 방식으로 이용할 수 있습니다.

Spring Data JPA 공식문서에서는 다음과 같은 경우에 QBE(Query By Example)의 사용을 추천합니다.

- 정적 또는 동적 제약 조건으로 데이터를 검색할 때
- 도메인 객체에 잦은 리팩토링이 발생할 때 (쿼리 수정의 필요성 사라짐)
- 데이터베이스 API와 독립적으로 동작할 때 (데이터베이스 종속성 감소)

그러나 편리한 인터페이스와 반대로, 제한 사항도 강함에 유의해야 합니다.

- 복잡한 논리 조건 처리 불가 (OR, 중첩, 그룹화)
- 문자열 매칭은 데이터베이스마다 다를 수 있음 (종속성 고려 필요)
- 정확한 값 매칭 지원 (범위 검색은 지원하지 않음)
