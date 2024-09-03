---
title: Hibernate와 함께하는 JPA 첫걸음 (1) 개념과 흐름
lang: ko
layout: post
---

## 들어가며

### Disclaimer

- Last Update: 2024/09/03

- Java Persistence API와 hibernate를 이용하는 과정에서, 둘에 대해 스스로 가지고 있던 오해를 풀고 더욱 잘 이해하려고 노력하는 글입니다.
- 관련된 학습 코드는 [github repo](https://github.com/glenn-syj/jpa-playground.git)에서 확인할 수 있습니다.

<br/>

## Java Persistence API와 Hibernate

### 흔한 오해

흔히 JPA로 줄여서 부르는 Java Persistence API(Jakarta Persistence API)를 처음 마주하게 되면, JPA가 곧 Hibernate고, Hibernate가 곧 JPA라고 생각하기 쉽습니다. 게다가 Spring Data JPA 모듈이 곧 JPA라고 여겨지기도 합니다.

저 역시 프로젝트에서 JPA를 이용하면서도 깊게는 이해하지 못하고 있었는데요. 이러한 오해는 JPA와 Hibernate가 등장한 배경을 이해하기 어렵게 만들 뿐만 아니라, 코드를 깊게 이해하지 못하고 사용만 하는 결과를 낳을 위험이 있습니다.

그렇다면, JPA란 무엇이고 Hibernate란 무엇일까요? 왜 JPA를 설명하는 수많은 글에서 Hibernate를 함께 이야기할까요? 이번 글에서는 JPA와 Hibernate를 전반적으로 살펴보려고 합니다.

### Java Persistence API

#### 개념

JPA는 자바에서 객체와 관계형 데이터베이스 간의 매핑, ORM(Object-Relational Mapping)을 관리하기 위한 표준 명세입니다. 여기에서 표준 명세라 함은, ORM 인터페이스나 규칙은 정의하지만 구현체를 제공하지 않는다는 뜻으로 볼 수 있습니다.

JPA 명세에서는 Entity, EntityManager, PersistentContext 등의 핵심 인터페이스를 이용해 주로 영속성 컨텍스트 관리, 변경 사항 추적(Dirty Checking), 엔티티 생명주기 관리 등을 다룹니다.

#### 배경

JPA는 엔터프라이즈급 어플리케이션 개발에서 OOP(Object-Oriented Programming)과 RDBMS(Relational Database Management System) 사이에서 생기는 패러다임 불일치를 해소하고자 등장했습니다. 정확히는 EJB 2.x에서도 CMP(Container-Managed Persistence)를 데이터베이스 접근 방식으로 제공했지만, 비효율적인 탓도 컸습니다.

```java
public class User {
    private Long id;
    private String username;
    private String email;

    // Constructors, getters, and setters
    // ...
}
```

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(255),
    email VARCHAR(255)
);
```

익숙한 POJO 클래스 User에 대해서 아래와 같은 데이터베이스 테이블이 정의될 수 있는데요. MyBatis나 JdbcTemplate과 같은 SQL Mapper에서는 직접 SQL 쿼리를 작성하고 수동으로 데이터에 매핑하게 됩니다. 언뜻 보자면 어떤 문제가 발생하는지 알기 어렵습니다.

하지만 객체를 DB 테이블을 중심으로 만들며 SQL Mapper를 이용하게 된다면 OOP와 RDBMS 사이에는 불일치가 발생합니다. 특히 세분성, 상속, 데이터 탐색, 동일성 비교, 연관성이 대표적입니다.

RDBMS에는 Java가 제공하는 상속 기능이 없으며, Java에는 RDBMS처럼 외래키를 통한 자유로운 참조가 불가능하며, RDBMS 레코드가 매핑된 객체는 추가적인 처리가 없다면 같은 레코드라도 다른 객체로 인식되기 쉽습니다.

JPA는 이러한 문제를 해결하기 위해 등장했다고 볼 수 있습니다. 그러나 ORM 프레임워크가 JPA보다 먼저 등장했으며, JPA가 사실 ORM 프레임워크(특히 Hibernate)에 기반해 작성되었다는 점도 알아두면 좋겠습니다.

### Hibernate

#### 개념

[Hibernate](https://hibernate.org/orm/)는 JPA 구현체이자, Java 진영의 대표적인 ORM 프레임워크입니다. Hibernate는 Java 객체를 DB 테이블에 자동으로 매핑하고, 시스템 초기화 시점에 SQL을 대부분 생성합니다. Hibernate는 캐싱, 세션 관리, 확장성 등에서도 장점을 가집니다.

Hibernate가 JPA를 따르는 구현체(Provider)라는 것은, Spring에서 JPA 구현체가 필요할 때 Hibernate를 이용할 수 있다는 뜻입니다. JPA 기반 repository를 쉽게 이용하도록 돕는 Spring Data JPA는 Hibernate를 기본 구현체로 설정합니다.

#### 특징

또한, Hibernate는 공식 문서에서 Hibernate API의 특징을 다음과 같이 기술합니다.

> an implementation of the JPA-defined APIs, most importantly, of the interfaces EntityManagerFactory and EntityManager, and of the JPA-defined O/R mapping annotations,
>
> a native API exposing the full set of available functionality, centered around the interfaces SessionFactory, which extends EntityManagerFactory, and Session, which extends EntityManager, and
>
> a set of mapping annotations which augment the O/R mapping annotations defined by JPA, and which may be used with the JPA-defined interfaces, or with the native API.

첫째, JPA 명세를 따르는 API 구현체입니다. 여기에는 `EntityManagerFactory`와 `EntityManager`, 그리고 Object Relational 매핑 어노테이션이 포함됩니다.

둘째, JPA의 `EntityManagerFactory`를 상속받은 `SessionFactory` 인터페이스와 `EntityManager` 인터페이스를 상속받은 `Session`을 중심으로 Hibernate가 지원하는 native API를 통해 추가적인 기능을 이용할 수 있습니다.

셋째, JPA에 명시된 Object Relational 매핑 어노테이션과 함께 추가적인 어노테이션을 제공합니다. 이 어노테이션들은 Hibernate의 JPA에서 정의된 인터페이스나 Hibernate native API와 함께 사용할 수 있습니다.

#### 배경

Hibernate 역시 OOP와 RDBMS 사이의 불일치를 해결하려는 노력으로 등장했습니다. 2001년, Gavin King이 EJB 2.x 스타일의 엔티티 빈을 이용하는 방식의 대안으로 Hibernate의 시작을 알렸습니다. 2003년에는 Hibernate2가 나왔고, 2005년에는 Hibernate 3.0이 나왔습니다.

Hibernate는 JPA가 도입된 이후, 3.2버전부터 JPA를 완전히 지원하면서 JPA 표준을 따르는 ORM 프레임워크가 되었습니다. 2010년 JPA 2.0부터는 Hibernate 3.5 이상 버전이 JPA의 인증된 구현체(a certified implementation)이 되었습니다.

또한 앞서 설명했듯이 JPA 역시 Hibernate를 참고한 까닭에, JPA 표준과 Hibernate native API 사이에 [명명이 겹치는 경우](https://docs.jboss.org/hibernate/orm/6.6/introduction/html_single/Hibernate_Introduction.html#hibernate-and-jpa)도 있습니다.

| Hibernate                               | JPA                               |
| --------------------------------------- | --------------------------------- |
| `org.hibernate.annotations.CascadeType` | `javax.persistence.CascadeType`   |
| `org.hibernate.FlushMode`               | `javax.persistence.FlushModeType` |
| `org.hibernate.annotations.FetchMode`   | `javax.persistence.FetchType`     |
| `org.hibernate.query.Query`             | `javax.persistence.Query`         |
| `org.hibernate.Cache`                   | `javax.persistence.Cache`         |
| `@org.hibernate.annotations.NamedQuery` | `@javax.persistence.NamedQuery`   |
| `@org.hibernate.annotations.Cache`      | `@javax.persistence.Cacheable`    |

## Spring Framework에서의 흐름

### 핵심 흐름

Spring Framework 내에서 JPA, Hibernate, Spring Data JPA는 계층적으로 연결되어 있습니다. 앞서 정리했듯이, JPA는 표준 명세, Hibernate는 JPA 구현체, Spring Data JPA는 더욱 추상화된 모듈입니다. 이번 단락에서는 특히 JPA에서 핵심적인 개념들이 어떻게 Bean으로서 올라가고 동작하는지를 개괄하겠습니다.

#### Spring Framework에서 EntityManagerFactory까지

`EntityManagerFactory`는 JPA에서 데이터베이스와 상호작용하는 기본 인터페이스인 `EntityManager`를 생성합니다. 일반적으로 `EntityManagerFactory`는 싱글톤으로 이용되며, DB 연결 설정 및 ORM 매핑 정보를 관리합니다.

[Spring](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/jpa/LocalContainerEntityManagerFactoryBean.html)에서는 `LocalContainerEntityManagerFactoryBean`을 통하거나 또는 `LocalEntityManagerFactoryBean`으로 `EntityManagerFactory`를 빈으로 등록합니다. 전자가 후자보다 더 유연하고, 기본적으로 Spring에서는 전자를 이용해 JPA 표준과 Spring의 유연성을 모두 활용할 수 있습니다.

```java
@Bean
public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
    LocalContainerEntityManagerFactoryBean factoryBean = new LocalContainerEntityManagerFactoryBean();

    // 주입된 DataSource 설정
    factoryBean.setDataSource(dataSource);

    // JPA Vendor Adapter 설정
    HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
    vendorAdapter.setShowSql(true);  // SQL을 로그에 출력
    vendorAdapter.setGenerateDdl(true);  // Hibernate가 DDL을 생성하도록 설정
    factoryBean.setJpaVendorAdapter(vendorAdapter);

    // 엔티티가 있는 패키지 경로 설정
    factoryBean.setPackagesToScan("com.example.model");

    return factoryBean;
}
```

`LocalContainerEntityManagerFactoryBean`는 `persistence.xml`를 읽어들인 정보에 바탕해 JPA 설정 정보를 담고 있는 `PersistenceUnitInfo` 객체를 만들어 냅니다. 데이터베이스 연결을 위한 `JDBC DataSource`은 물론, 지연 로딩 등을 지원하기 위해 바이트 코드를 조작하는 로드 타임 위빙 `LoadTimeWeaver` 설정도 함께 담을 수 있습니다. `JpaVendorAdapter`는 JPA 구현체를 지정합니다.

```java
@Bean
public DataSource dataSource() {
    HikariDataSource dataSource = new HikariDataSource();
    dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
    dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
    dataSource.setUsername("root");
    dataSource.setPassword("password");
    dataSource.setMaximumPoolSize(10); // 최대 풀 크기 설정
    return dataSource;
}
```

여기에서 데이터 베이스 연결 풀, 커넥션 풀(Connection Pool)도 생성됩니다. 이는 DB와의 연결을 재사용하기 위해 미리 여러 연결을 생성해두고, 필요할 때 연결을 할당하는 방식으로 동작합니다. `HikariDataSource()`를 통해 이용되는 HikariCP가 대표적입니다.

```java
public class HibernatePersistenceProvider implements PersistenceProvider {
	private static final EntityManagerMessageLogger log = messageLogger( HibernatePersistenceProvider.class );

	private final PersistenceUtilHelper.MetadataCache cache = new PersistenceUtilHelper.MetadataCache();

    // ... implSpec
	@Override
	public EntityManagerFactory createEntityManagerFactory(String persistenceUnitName, Map properties) {
		log.tracef( "Starting createEntityManagerFactory for persistenceUnitName %s", persistenceUnitName );
		final EntityManagerFactoryBuilder builder = getEntityManagerFactoryBuilderOrNull( persistenceUnitName, properties );
		if ( builder == null ) {
			log.trace( "Could not obtain matching EntityManagerFactoryBuilder, returning null" );
			return null;
		}
		return builder.build();
	}

    // ... builder(), wrap(), ...
}
```

`EntityManagerFactory`는 `PersistenceUnitInfo`에 기반해 생성됩니다. `PersistenceUnitInfo` 객체와 추가 설정은 `PersistenceProvider` 구현체를 통해 `EntityManagerFactory`를 생성해 빈으로 등록합니다. 이 과정은 이용하는 JPA 구현체에 의존하는데요. Spring Boot는 `HibernateJpaAutoConfiguration`을 통해 Hibernate를 JPA 구현체로 사용하도록 설정한 뒤 주입함도 알아두면 좋겠습니다.

#### EntityManager

이후 `EntityManagerFactory`는 영속성 컨텍스트(Persistence Context)를 관리하는 `EntityManager` 인스턴스를 생성합니다. 영속성 컨텍스트란 메모리 내에서 엔티티 객체를 관리하는 저장소라고 생각해도 좋습니다. 여기에서 `EntityManager`는 스레드 안전성이 보장되지 않으므로, 각 스레드를 공유해서는 안됩니다.

`EntityManager`는 데이터 액세스 계층에서 `EntityManager`는 `persist()`, `merge()`, `remove()`, `find()` 등으로 엔티티의 영속상태를 관리합니다. 여기에서 엔티티는 비영속(transient), 영속(persistent), 준영속(detached), 삭제(removed)라는 영속 상태를 가집니다.

`EntityManager`는 1차 캐시로 동작하면서, 동일한 트랜잭션 내에서는 동일한 엔티티에 대해 DB에 반복적으로 접근하지 않고 영속성 컨텍스트에 영속된 엔티티를 반환합니다.

또한, `flush()` 메서드를 호출하거나 트랜잭션이 커밋될 때, 컨텍스트 내에 올라와 있는 엔티티들의 변경사항이 감지되어 반영됩니다. 이를 더티 체킹(dirty checking)이라고 부릅니다.

여기에이때, `EntityManagerFactory`를 기반으로 트랜잭션을 관리하는 데 이용되는 `JpaTransactionManager`는 트랜잭션의 시작과 종료를 제어합니다. `JpaTransactionManager`는 `EntityManager`를 스레드에 바인딩해, 스레드마다 각각의 `EntityManager`를 이용하게 함으로써 스레드 안전성을 보합니다. 만약 여러 데이터베이스를 이용한다면, JTA(Java Transaction API)를 이용하는 `JtaTransactinoalManager`를 사용할 수 있습니다.

트랜잭션이 종료되면 `EntityManager`도 종료되고, 그에 따라 관련된 영속성 컨텍스트도 함께 종료됩니다. 혹은 Hibernate에서 2차 캐시를 설정해 여러 트랜잭션 간 엔티티 상태를 공유할 수도 있습니다.

만약 애플리케이션이 종료된다면, Spring 컨테이너는 `EntityManagerFactory`를 포함해 모든 빈을 종료하고 정리합니다. `DataSource`와 관련된 모든 리소스도 정리되어 데이터베이스 연결이 해제되고, 데이터베이스 연결 풀도 닫힙니다.

## 나가며

이번 글에서는 JPA와 Hibernate의 기본 개념과 Spring Framework 내에서의 생성 및 동작 흐름을 다루었습니다. 특히 JPA와 Hibernate가 맺는 관계는 물론, Spring Framework에서 어떻게 JPA 명세의 핵심 개념들이 Hibernate에 기반해 빈으로 컨테이너에 올라가는지 살펴보았는데요. JPA 핵심 개념 간의 관계를 살펴볼 수 있는 좋은 기회였습니다.

다음 글에서는 Spring Data JPA도 함께 다루어, `EntityManager`가 작동하는 방식과 함께 JPA를 이용하는 여러 방법을 살펴보겠습니다. 아래는 다음 시간에 살펴볼 테스트 코드니, 시간이 되시는 분들은 먼저 비교해보셔도 좋겠습니다.

```java
@SpringBootTest
@Transactional
public class MemberEntityManagerTest {

    @PersistenceContext
    private EntityManager entityManager;

    @Test
    public void testCreateAndFindMember() {
        // Create a Member
        Member member = Member.builder()
                .username("test")
                .email("test@example.com")
                .build();

        // Persist the entity
        entityManager.persist(member);
        entityManager.flush(); // Force the changes to be applied to the database

        // Find the Member
        Member foundMember = entityManager.find(Member.class, member.getId());
        assertThat(foundMember).isNotNull();
        assertThat(foundMember.getUsername()).isEqualTo("test");

        // Retrieve all Members using JPQL
        List<Member> members = entityManager.createQuery("SELECT m FROM Member m", Member.class)
                .getResultList();
        assertThat(members).isNotEmpty();

        // Remove the entity
        entityManager.remove(foundMember);
        entityManager.flush();

        // Verify that the Member has been deleted
        Member deletedMember = entityManager.find(Member.class, member.getId());
        assertThat(deletedMember).isNull();
    }

}
```

```java
@SpringBootTest
@Transactional
public class MemberRepositoryTest {

    @Autowired
    private MemberRepository memberRepository;

    @Test
    public void testCreateAndFindMember() {

        // Create a new Member instance using the builder pattern
        Member member = Member.builder()
                .username("test")
                .email("test@test.com")
                .build();

        // Save the Member entity to the database using the repository
        memberRepository.save(member);

        // Retrieve all Members from the database
        List<Member> members = memberRepository.findAll();

        // Verify that the members list is not empty and the username of the first member is "test"
        assertThat(members).isNotEmpty();
        assertThat(members.get(0).getUsername()).isEqualTo("test");

        // Delete the Member entity from the database using the repository
        memberRepository.delete(member);

        // Verify that the Member has been successfully deleted by checking that the ID is no longer present
        assertThat(memberRepository.findById(member.getId())).isEmpty();
    }

}
```

## References

https://en.wikipedia.org/wiki/Jakarta_Persistence

https://spring.io/projects/spring-data-jpa

https://en.wikipedia.org/wiki/Object%E2%80%93relational_impedance_mismatch

https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/instrument/classloading/LoadTimeWeaver.html
