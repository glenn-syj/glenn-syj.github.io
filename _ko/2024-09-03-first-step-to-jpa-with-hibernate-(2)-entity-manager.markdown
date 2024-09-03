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

일반적으로 영속성 컨텍스트의 스코프는 트랜잭션 단위지만, 더욱 확장해 이용하는 방법도 있습니다. 트랜잭션 수준 스코프에서는 트랜잭션이 끝나면, 영속성 컨텍스트에 영속된 엔티티가 영속 저장소(persistent storage)에 flush됩니다. 트랜잭션 내에서 연산이 수행될 때, `EntityManager`는 영속성 컨텍스트가 존재하는지 확인합니다. 만약 존재하지 않는다면, 영속성 컨텍스트를 생성합니다. Spring에서는 `@PersistenceContext` 어노테이션이 `PersistenceContextType.TRANSACTION`을 기본값으로 이용함으로써 트랜잭션 수준 스코프를 지원합니다.

반면 확장된 스코프에서는 영속성 컨텍스트가 여러 트랜잭션에 걸쳐 뻗어 나갑니다. 확장된 스코프에서는 트랜잭션 없이도 엔티티를 영속시킬 수는 있지만, 트랜잭션 없이는 flush할 수 없습니다. Spring에서 확장된 스코프를 설정하려면 `@PersistenceContext(type = PersistenceContextType.EXTENDED)`를 이용할 수 있습니다.

하지만 확장된 스코프에서 영속성 컨텍스트를 이용할 때는 주의를 더욱 기울여야 합니다. 특히 무상태 세션 빈(Stateless Session Bean)에서는 확장된 스코프라도 영속성 컨텍스트가  동작할 수 있습니다. (설명하자면 글의 범위를 넘어서므로, [Baeldung](https://www.baeldung.com/jpa-hibernate-persistence-context)의 글을 참고해주세요!) 




### 



## References

https://docs.oracle.com/javaee%2F7%2Fapi%2F%2F/javax/persistence/EntityManager.html

https://www.baeldung.com/jpa-hibernate-persistence-context