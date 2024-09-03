---
title: Hibernate와 함께하는 JPA 첫걸음 (2) 엔티티 매니저
lang: ko
layout: post
---

## 들어가며

### Disclaimer

- Last Update: 2024/09/03

- Java Persistence API와 hibernate를 이용하는 과정에서, 둘에 대해 스스로 가지고 있던 오해를 풀고 더욱 잘 이해하려고 노력하는 글입니다.
- 관련된 학습 코드는 [github repo](https://github.com/glenn-syj/jpa-playground.git)에서 확인할 수 있습니다.
- 지난 글에서 이어지는 글입니다.

<br/>

## JPA의 핵심: EntityManager와 영속성 컨텍스트

### EntityManager

표준 명세로서의 JPA, JPA 구현체로서 Hibernate, 추상화된 모듈로서 Spring Data JPA는 모두 EntityManager 인터페이스를 핵심으로 삼고 있습니다. EntityManager라는 말은 당연하게도 Entity를 관리하고 다룬다고 예상할 수 있는데요. 여기에서 Entity는 RDBMS에서 테이블에 대응되어 매핑되는 자바 객체를 의미합니다.

EntityManager 인터페이스는 엔티티의 생명주기나 인스턴스 자체를 관리합니다. 또한, `PersistenceUnit`에서 정의된 데이터베이스와 연결되어 연산을 수행하고, 트랜잭션 역시 관리합니다. 이러한 기능은 모두 영속성 컨텍스트(Persistence Context)와 연관되어 있습니다.

특히, [EntityManager Java doc](https://docs.oracle.com/javaee%2F7%2Fapi%2F%2F/javax/persistence/EntityManager.html)에서는 EntityManager에 대해 아래와 같이 서술합니다.

> Interface used to interact with the persistence context.
>
> An EntityManager instance is associated with a persistence context. A persistence context is a set of entity instances in which for any persistent entity identity there is a unique entity instance. Within the persistence context, the entity instances and their lifecycle are managed. The EntityManager API is used to create and remove persistent entity instances, to find entities by their primary key, and to query over entities.
> 
> The set of entities that can be managed by a given EntityManager instance is defined by a persistence unit. A persistence unit defines the set of all classes that are related or grouped by the application, and which must be colocated in their mapping to a single database.

### Persistence Context

영속성 컨텍스트는 각 엔티티 인스턴스가 중복되지 않은 상태로 유지되는 엔티티 인스턴스의 집합이라고 볼 수 있습니다. EntityManager 역시 영속성 컨텍스트를 통해 엔티티의 생명주기를 관리하는 셈입니다. 이러한 영속성 컨텍스트는 곧 1차 캐시로 불리기도 합니다.

#### 영속성 컨텍스트와 스코프

일반적으로 영속성 컨텍스트의 스코프는 트랜잭션 단위지만, 더욱 확장해 이용하는 방법도 있습니다. 트랜잭션 수준 스코프에서는 트랜잭션이 끝나면, 영속성 컨텍스트에 영속된 엔티티가 영속 저장소(persistent storage)에 flush됩니다. 

트랜잭션 내에서 연산이 수행될 때, `EntityManager`는 영속성 컨텍스트가 존재하는지 확인하는 과정도 있습니다. 만약 존재하지 않는다면, 영속성 컨텍스트를 생성합니다. Spring에서는 `@PersistenceContext` 어노테이션이 `PersistenceContextType.TRANSACTION`을 기본값으로 이용함으로써 트랜잭션 수준 스코프를 지원합니다.

반면 확장된 스코프에서는 영속성 컨텍스트가 여러 트랜잭션에 걸쳐 뻗어 나갑니다. 확장된 스코프에서는 트랜잭션 없이도 엔티티를 영속시킬 수는 있지만, 트랜잭션 없이는 flush할 수 없습니다. Spring에서 확장된 스코프를 설정하려면 `@PersistenceContext(type = PersistenceContextType.EXTENDED)`를 이용할 수 있습니다.

하지만 확장된 스코프에서 영속성 컨텍스트를 이용할 때는 주의를 더욱 기울여야 합니다. 특히 무상태 세션 빈(Stateless Session Bean)에서는 확장된 스코프라도 영속성 컨텍스트가  동작할 수 있습니다. (설명하자면 글의 범위를 넘어서므로, [Baeldung](https://www.baeldung.com/jpa-hibernate-persistence-context)의 글을 참고해주세요!) 

## Hibernate에서의 핵심 개념 구현

### Hibernate EntityManager 살펴보기

```java
// hibernate
public class EntityManagerImpl extends AbstractEntityManagerImpl {
	private static final Logger log = LoggerFactory.getLogger( EntityManagerImpl.class );

	protected Session session;
	protected boolean open;
	protected boolean discardOnClose;
	private Class sessionInterceptorClass;

    // constructor, methods, ...
}
```

Hibernate에서는 `EnityManagerImpl` 클래스에서는 최종적으로 JPA의 `EntityManager` 인터페이스가 구현됩니다. 이는 추상 클래스 `AbstractEntityManagerImpl`을 상속하고 있는데요. JPA 인터페이스 `EntityManager`는 어떤 과정을 거쳐 `EntityManagerImpl`에 구현되는 걸까요? 이를 위해 역으로 올라가 파악한 상속 혹은 구현 구조를 `EntityManager`를 중심으로 설명해보겠습니다.

#### HibernateEntityManager와 HibernateEntityManagerFactory

```java
public interface HibernateEntityManager extends EntityManager {
	/**
	 * Retrieve a reference to the Hibernate {@link Session} used by this {@link EntityManager}.
	 * @return
	 */
	public Session getSession();
}
```

```java
public interface HibernateEntityManagerFactory extends EntityManagerFactory, Serializable {
	public SessionFactory getSessionFactory();
}
```

JPA의 `EntityManager` 인터페이스는 `HibernateEntityManager` 인터페이스에서 상속되며 처음 나타납니다. `HibernateEntityManager` 인터페이스에서는 `EntityManager`의 인터페이스에 정의된 메서드와 함께 `Session` 클래스를 이용할 수 있도록 합니다.

`Session` 클래스는 Hibernate의 핵심 개념 중 하나로, JPA의 `EntityManager`와 유사한 기능을 합니다. Hibernate는 `Session`을 통해 JPA에서는 명시되어 있지 않은 2차 캐시, 배치 처리, 플러시 모드 설정, 쿼리 캐시, 커스텀 인터셉터, 동적 프록시 등의 다양한 기능을 지원합니다.

기존 JPA에서의 `EntityManager`-`EntityManagerFactory`의 구조와 같이, `HibernateEntityManager` 구현체는 `EntityManager`를 상속받은 `HibernateEntityManagerFactory`를 통해 생성됩니다.  

#### HibernateEntityManagerImplementor

```java
public interface HibernateEntityManagerImplementor extends HibernateEntityManager {

	public HibernateEntityManagerFactory getFactory();

	boolean isTransactionInProgress();

	public void handlePersistenceException(PersistenceException e);

	public void throwPersistenceException(PersistenceException e);

	public RuntimeException convert(HibernateException e, LockOptions lockOptions);

	public RuntimeException convert(HibernateException e);

	public void throwPersistenceException(HibernateException e);

	public PersistenceException wrapStaleStateException(StaleStateException e);

	public LockOptions getLockRequest(LockModeType lockModeType, Map<String, Object> properties);

    // Enum options ...

	public <T> TypedQuery<T> createQuery(String jpaqlString, Class<T> resultClass, Selection selection, Options options);
}
```

`HibernateEntityManagerImplementor` 인터페이스는 `HibernateEntityManager` 인터페이스를 상속함으로써 Hibernate의 기능을 JPA 표준 명세와 통합합니다. 

특히 `convert(HibernateException e)` 메서드는 Hibernate 명세의 예외를 JPA 명세의 예외로 변환해 처리하도록 합니다. 

또한, `getLockRequest(LockModeType lockModeType, Map<String, Object> properties)`는 JPA의 잠금 모드를 Hibernate의 `LockOptions`로 변환해 이용합니다. 

뿐만 아니라, `isTransactionInProgress()`와 같이 트랜잭션 상태를 확인하는 추가 기능도 제공합니다.

#### AbstractEntityManagerImpl

```java
public abstract class AbstractEntityManagerImpl implements HibernateEntityManagerImplementor, Serializable {
	private static final Logger log = LoggerFactory.getLogger( AbstractEntityManagerImpl.class );

	private static final List<String> entityManagerSpecificProperties = new ArrayList<String>();

	static {
		entityManagerSpecificProperties.add( AvailableSettings.LOCK_SCOPE );
		entityManagerSpecificProperties.add( AvailableSettings.LOCK_TIMEOUT );
		entityManagerSpecificProperties.add( AvailableSettings.FLUSH_MODE );
		entityManagerSpecificProperties.add( AvailableSettings.SHARED_CACHE_RETRIEVE_MODE );
		entityManagerSpecificProperties.add( AvailableSettings.SHARED_CACHE_STORE_MODE );
		entityManagerSpecificProperties.add( QueryHints.SPEC_HINT_TIMEOUT );
	}

	private EntityManagerFactoryImpl entityManagerFactory;
	protected transient TransactionImpl tx = new TransactionImpl( this );
	protected PersistenceContextType persistenceContextType;
	private PersistenceUnitTransactionType transactionType;
	private Map<String, Object> properties;
	private LockOptions lockOptions;

    // constructor, methods, ...
}
```

`AbstractEntityManagerImpl`은 Hibernate에서의 `EntityManagerImpl`의 기본 뼈대를 제공하면서 JPA 표준 기능을 구현합니다. 여기에는 트랜잭션 관리, 락 옵션 설정 등의 JPA 사양 및 Hibernate 기능이 구체화되어있습니다.

```java
public void persist(Object entity) {
    checkTransactionNeeded();
    try {
        getSession().persist( entity );
    }
    catch ( MappingException e ) {
        throw new IllegalArgumentException( e.getMessage() );
    }
    catch ( RuntimeException e ) {
        throw convert( e );
    }
}
```

당연하게도 JPA 명세의 `persist()`, `merge()`, `remove()` 등의 메소드가 구현되어 있는데요. `Session`이 Hibernate의 핵심개념인 만큼, JPA 표준 기능 역시 `Session`을 통해 구현되고 있다는 점이 주목할만 합니다. 앞서 `HibernateEntityManagerImplemtor`에서 살펴보았던 `convert()` 메서드도 예외 처리에서 이용되고 있습니다.

## References

https://docs.oracle.com/javaee%2F7%2Fapi%2F%2F/javax/persistence/EntityManager.html

https://www.baeldung.com/jpa-hibernate-persistence-context