---
title: 재미있는 JPA - 영속성 레이어 with Redis
lang: ko
layout: post
---

## 들어가며

### Disclaimer

- Last Update: 2024/08/06

- 현재 진행중인 SSAFY 내 프로젝트에서, 게시글 조회수와 좋아요 수 변경 관련 기존 코드는 RDBMS에 바로 쿼리를 보내는 식으로 작성되었는데요. 인-메모리 데이터 저장소 Redis를 이용하면서도 JPA의 영속성 컨텍스트를 활용해 더 효율적인 방식을 고민하는 과정을 담은 글입니다.

<br/>

## 재미있는 JPA

### 문제 상황

매 조회마다 RDBMS에 (이번 프로젝트에서는 MySQL에) UPDATE 쿼리를 날려 DB에 담긴 게시글의 조회수를 올린다면, 어떤 문제가 있을까요? 특히 모놀리식한 환경에서는 갑자기 트래픽이 올라가 데이터베이스 부하가 늘거나, 잠금 경합(Lock Contention)이 발생하고, 트랜잭션 충돌이 일어날 지도 모릅니다.

기존 코드는 RDBMS에서 바로 UPDATE 문을 날리는 식으로 구성되어 있었는데요. 정확히는 `@Transactional` 어노테이션을 활용하고 있었습니다. 이는 곧 트랜잭션 내에서 예외가 발생하지 않는다면 엔티티의 변경사항이 감지되고, 트랜잭션이 종료되면서 커밋되어 데이터베이스에 반영됨을 의미합니다.

제 목표는 RDBMS로 향하는 불필요한 쿼리 호출을 줄이면서, 좋아요 수와 조회수라는 정보의 특성을 살려서 전달하는 것이었는데요. 특히 좋아요 수와 조회수는 사용자에게 도움이 되지만, 다른 정보보다는 엄격한 데이터 정합성을 비교적 요구하지 않는다고 판단했습니다. 나아가 멀티스레딩 환경에서도 문제가 없도록 코드를 작성해보려고 했습니다.

### JPA 영속성 컨텍스트와 @Transactional

먼저, JPA에는 영속성 컨텍스트(Persitence Context)라는 재미있는 개념이 존재합니다. 사실 영속성 컨텍스트는 JPA 등장 이전, Java 7에서도 존재하는 개념입니다. 실제로 `EntityManager` 클래스 Java Doc에는 영속성 컨텍스트를 아래와 같이 이야기합니다.

> A persistence context is a set of entity instances in which for any persistent entity identity there is a unique entity instance. Within the persistence context, the entity instances and their lifecycle are managed.

Spring Data JPA에서 흔히 이용되는 `repository.find...()` 문을 생각해봅시다.

### References

https://www.baeldung.com/jpa-hibernate-persistence-context

https://docs.oracle.com/javaee/7/api/javax/persistence/EntityManager.html

<br/>
