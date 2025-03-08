---
title: 전략 패턴으로 금액 정산 비즈니스 로직 성능 비교하기
lang: ko
layout: post
---

## 들어가며

- 전략 패턴을 활용하여 간단한 엔티티 조회 및 생성 과정에서의 로직 성능을 비교해봅니다.

- 티끌(Tikkle)은 '시간'이라는 내부 재화를 통해 단체 내 구성원 간 서로 도움을 쉽게 주고받기 위해 만든 플랫폼 웹사이트입니다. 구성원은 도움을 주고 받으며 '시간'을 소비하고 획득하며, 나아가 환전 기능을 통해 '시간'을 랭킹 포인트로 바꿀 수 있습니다. 정산 기능은 이러한 활동의 무결성을 검증하고, 이상 거래를 감지하는 밑바탕이 되는 중요한 기능입니다.

- 계좌 정산 작업의 밑바탕이 되는, Account 엔티티의 마지막 상태를 저장하는 BalanceSnapshot 엔티티 생성 방식 비교에 따른 성능 차이를 비교합니다.

- 자세한 내용은 아래 링크에서 확인하실 수 있습니다.

  - [https://github.com/tikkle-a501/tikkle/issues/20](https://github.com/tikkle-a501/tikkle/issues/20)
  - [https://github.com/tikkle-a501/tikkle/pull/21](https://github.com/tikkle-a501/tikkle/pull/21)
  - [https://github.com/tikkle-a501/tikkle/pull/22](https://github.com/tikkle-a501/tikkle/pull/22)

- 최종 수정일: 2025-03-08

## 전략 패턴 선정

### 서비스 고려사항

저희 프로젝트 서비스 티끌(Tikkle)에서는 매일 00:00에 다음날의 계좌 정산 작업을 위한 스냅샷을 생성하는 작업이 있습니다. 정산 관련 모든 프로세스는 23:30 ~ 익일 00:30, 서비스 중단 시간에 이루어지기로 기획했습니다.

그러므로 정산 프로세스의 시간을 줄인다면 서버 다운타임을 최대한 줄일 수 있을 것이라 생각합니다. 나아가 플랫폼에서 활용하는 재화와 관련된 거래 활동을 제외하면 추후 서버를 내리지 않는 방식을 채택할 수도 있겠습니다. 따라서 존재하는 모든 계좌에 대한 정산 작업 및 그 사전 작업에 대해서 최대한 비용을 줄이는 선택지를 찾고자 합니다.

### 기술적 고려사항

부족하게나마 테스트 코드를 짜면서, 여러 메서드의 성능을 비교하려 서비스 컴포넌트에서 메서드를 여러 개 작성했던 경험이 있는데요. 제가 기존에 느꼈던 문제점은 다음과 같습니다.

- 개방 폐쇄 원칙 위반

  개방 폐쇄 원칙(Open Closed Principle)은 클래스에 대해서 확장에는 열려있어야 하고, 수정에는 닫혀있어야 한다는 원칙입니다. 즉, 기존 코드를 변경하지 않고 기능을 추가할 수 있어야 하는데요. 하지만 서비스 클래스에 여러 메서드를 작성하게 된다면 기존 코드를 변경하는 것이 불가피해집니다. 새로운 방식을 이용하는 코드를 추가할 때마다 서비스 클래스를 직접 수정해야 하니까요.

- 단일 책임 원칙 위반

  Spring에서 서비스 클래스는 비즈니스 로직을 담당해야 하는데, 성능 테스트를 위해서 서비스에 여러 메서드를 작성하는 것이 객체 지향에서의 SRP(Single Responsibility Principle) 원칙을 위배하는 게 아닐까, 하는 의문이 들었습니다. 특히, 비교를 위해 여러 메서드에서 사용되는 모든 Repository에 의존하며 결합도를 높인다는 점도 문제라고 생각했습니다.

그렇다면, SRP를 지키면서 서비스 클래스에 application 프로필 설정 혹은 `@Qualifier` 어노테이션을 이용해 의존성을 주입함으로써 테스트를 진행하면 되지 않을까요?

- 애플리케이션 재시작 오버헤드

  그러나 오직 하나의 빈 주입을 이용한다면, 컴파일 시 정적으로 결정된 단일 전략만 사용이 가능합니다. 따라서 런타임 시 전략을 자유롭게 적용하기 위해서는 서로 다른 빈을 주입하기 위해, 테스트 코드 시 애플리케이션을 재시작하므로 불편을 낳습니다. 테스트 데이터를 준비하는 과정에서도 마찬가지입니다.

따라서 단일 책임 원칙을 준수함은 물론, 런타임에서 전략을 자유롭게 적용할 수 있는 방법이 필요했습니다. 바로 전략 패턴을 도입하는 일이었습니다.

### 전략 패턴에 대해서

#### 전략 패턴의 정의

전략 패턴은 런타임에 유연하게 알고리즘을 택할 수 있도록, 알고리즘의 집합을 정의하고 서로 교체할 수 있도록 하는 디자인 패턴인데요.

전략 패턴은 세 가지 요소로 이루어집니다. 전략 인터페이스(Strategy Interface), 구체적인 전략(Concrete Strategies), 컨텍스트(Context)입니다.

1. 전략 인터페이스

전략 인터페이스는 구체적인 전략이 따라야 할 행동 계약을 정해놓은 공통 인터페이스입니다.

2. 구체적인 전략

구체적인 전략은 인터페이스의 계약을 준수하는 실제 알고리즘으로, 전략 인터페이스를 구현하는 클래스입니다. 각 전략은 특정 알고리즘에 대한 책임만 가집니다.

3. 컨텍스트

컨텍스트는 전략 객체를 사용하는 역할을 하는 클래스입니다. 컨텍스트는 전략 인터페이스를 사용하여 알고리즘을 선택하고 실행합니다. 컨텍스트는 알고리즘과 관련된 전략을 이용하는 책임만 가집니다.

#### 전략 패턴을 적용하기 전에

1. 알고리즘 캡슐화

전략 패턴은 각 알고리즘을 구체적인 전략 클래스로 캡슐화합니다. 컨텍스트는 전략 객체를 이용하지만, 알고리즘의 세부 구현에 대해서 알지 못합니다. 따라서 알고리즘의 내부 로직이 변경되어도 컨텍스트의 코드는 변경되지 않습니다.

2. 인터페이스 기반 의존성 역전

모든 전략 클래스은 공통 인터페이스를 구현하고, 컨텍스트는 이 인터페이스를 사용하여 전략을 전달받습니다. 따라서 컨텍스트는 전략 클래스의 구체적인 구현에 의존하지 않습니다.

3. 상속보다는 구성

전략 패턴은 상속보다는 컴포지션을 이용해 객체의 행위를 변경합니다. 컴포지션은 다른 클래스의 인스턴스를 포함하고(HAS-A), 인스턴스는 실행 중에 교체될 수 있으며, 인터페이스를 통해 느슨하게 결합되어 있습니다. 반면, 상속은 IS-A 관계를 따르고, 그 관계는 컴파일 타임에 결정되며, 하위 클래스가 상위 클래스에 강하게 의존합니다.

## 계좌 스냅샷 생성 전략 패턴

객체 중심적인 개발을 시도하며 사용했던 JPA는 객체 지향적으로 데이터를 다룰 수 있지만, 정산 로직을 작성하면서는 과연 최선의 선택이 맞는지 의문이 들었습니다. 특히, 정산 로직과 계좌 스냅샷은 엔티티 조회 및 생성이라는 흐름이 비슷하므로, 계좌 스냅샷에서 더 적절한 대안은 정산 로직 자체에서도 적용할 수 있다고도 가정했습니다.

### 현재의 정산 로직

시간과 랭킹 포인트를 소지하고 있는 Account 엔티티는 Member와 일 대 일 관계로 매핑되어 있습니다. 저희의 목표는 매일 존재하는 모든 계좌의 시간 및 랭킹 포인트 재화 상태가 재화에 대한 거래 내역과 환전 내역에 일치하는 상태인지 확인하는 것이었는데요. 정산 로직은 아래와 같습니다.

1. 도움을 주고 받으며 Account에 직접 시간 재화량에 대한 변경, 환전을 통해 랭킹 포인트 재화량에 대한 변경이 발생

2. 해당 변경 발생 시 로그 테이블에 거래 내역과 환전 내역을 남김

3. 이전일 정산 작업 이후 생성된 BalanceSnapshot 엔티티에 기반한 검증 작업

   - 당일 가입한 Member의 Account는 가입 시 초기 설정값에 대한 스냅샷
   - 비교: 전일 BalanceSnapshot 엔티티 기록 + 당일 거래 및 환전 로그 기록값 == 당일 Account 재화량

4. 이후 이상 거래 혹은 이상 계좌 감지

그러나 서비스 고려사항과 기술적 고려사항을 떠올리며, 과연 JPA를 사용하는 게 맞는지 확신이 들지 않았습니다. 오히려 자료를 조사하면서, JDBC를 직접 이용하는 편이 훨씬 나을 것 같았습니다.

JPA가 비효율적이라면 왜 비효율적인지, JDBC가 효율적이라면 얼마나 효율적인지 명확한 근거가 필요하다고 생각했습니다. 따라서 테스트에서 런타임에 유연하게 전략을 변경해 어플리케이션 재시작을 줄이고 알고리즘 작성 시의 유연성도 높이는 전략 패턴을 적용해보고자 했습니다.

### 전략 패턴 도식

![Tikkle_StartegyPatternUML](https://private-user-images.githubusercontent.com/65771798/420578167-132cecb7-4678-4f3f-bb69-e74c1758a701.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDE0MjQ1MDQsIm5iZiI6MTc0MTQyNDIwNCwicGF0aCI6Ii82NTc3MTc5OC80MjA1NzgxNjctMTMyY2VjYjctNDY3OC00ZjNmLWJiNjktZTc0YzE3NThhNzAxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNTAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjUwMzA4VDA4NTY0NFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTAxMDU1OTFiZjRjMGYyYTAzMDM0MzNjNDgzOTk0MWE3YjQ0Nzc4NmZlOTBkZDBkNTViMWE3ZjFlMGU4MDQ3ZGQmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.iNg8U9zfcLJ7kGn4PrtYuqdP5KayRqDgoCGYhFK8E2A)

앞서 설명한 정의에서와 같이 전략 패턴의 요소는 다음과 같이 작성했습니다.

- 컨텍스트: `SettlementService`
- 전략 인터페이스: `SnapshotStrategy`
- 구체적인 전략: `JpaSnapshotStrategy`,
  `StreamSnapshotStrategy`,
  `PagingSnapshotStrategy`,
  `JdbcSnapshotStrategy`

`SettlementService`는 `SnapshotStrategy` 타입의 필드 인스턴스를 가지는데요. 위 인터페이스를 구현한 구체적인 전략의 `createSnapshot()` 메서드 내용은 알지 못하도록 캡슐화 되어 있습니다.

그런데 분명히, 앞서 상속보다는 컴포지션을 이용한다는 설명과 달리 UML에서는 어그리게이션으로 표현했습니다.

이는 전략 패턴이 일반적으로 컴포지션을 사용하지만 Spring 환경에서는 집합 관계에 가깝다는 점을 표현하고 싶었습니다.

Spring 환경 상 구체적인 전략이 Spring 컨테이너에 등록되어 관리되기에, `SettlementService`와 `SnapshotStrategy` 인터페이스 구현체 Bean은 서로 독립적입니다.

Spring 컨테이너가 각 Bean의 생성과 소멸을 독립적으로 관리하므로, 컨텍스트가 소멸해도 전략 객체는 계속 존재할 수 있습니다.

즉, 컴포지션이 강한 HAS-A 관계로 생명 주기를 공유하는 반면, Spring 환경 상 이번에 구현된 전략 패턴은 생명 주기가 독립적이므로 어그리게이션으로 표현했습니다.

### 전략 패턴 적용

### 엔티티

#### 컨텍스트

`SettlementService.java`

```java
@Service
@RequiredArgsConstructor
public class SettlementService {

  // ... 관련된 코드만 표시
  private SnapshotStrategyType snapshotStrategyType;
  private SnapshotStrategy snapshotStrategy;

  @Transactional
  @Scheduled(cron = "0 0 0 * * *")
  public void createSnapshots() {
      logger.info("Creating snapshots with strategy: {} ({})",
              snapshotStrategyType.name(), snapshotStrategyType.getDescription());

      long startTime = System.nanoTime();
      snapshotStrategy.createSnapShots();
      long endTime = System.nanoTime();

      logger.info("Completed snapshot creation in {} ms", TimeUnit.NANOSECONDS.toMillis(endTime - startTime));
  }
}
```

컨텍스트인 `SettlementService`는 전략 인터페이스인 `SnapshotStrategy`를 필드로 가지고 있습니다. `SnapshotStrategyType`은 기록을 측정하고 관리하기 위한 Enum 클래스입니다.

#### 전략 인터페이스

`SnapshotStrategy.java`

```java
public interface SnapshotStrategy {
    void createSnapShots();
    String getStrategyName();
}
```

전략 인터페이스인 `SnapshotStrategy`는 전략 패턴의 핵심 요소입니다. 이 인터페이스는 모든 구체적인 전략이 구현해야 하는 메서드를 정의합니다.

#### 구체적인 전략

스냅샷 생성 전략을 고민한 과정에서의 간단한 정리는 [이 링크](https://github.com/tikkle-a501/tikkle/issues/20)를 참고해주세요.

`JpaSnapshotStrategy.java`

```java
@Component("jpaSnapshotStrategy")
@RequiredArgsConstructor
public class JpaSnapshotStrategy implements SnapshotStrategy {

    private final AccountRepository accountRepository;
    private final BalanceSnapshotRepository balanceSnapshotRepository;
    private static final Logger logger = LoggerFactory.getLogger(JpaSnapshotStrategy.class);

    @Override
    // 작성: 기본적인 findAll(), saveAll() 메서드를 이용한 전략
    public void createSnapShots() {
        List<Account> accounts = accountRepository.findAll();
        List<BalanceSnapshot> snapshots = accounts.stream()
                .map(account -> BalanceSnapshot.builder()
                        .accountId(account.getId())
                        .timeQnt(account.getTimeQnt())
                        .rankingPoint(account.getRankingPoint())
                        .build())
                .toList();

        balanceSnapshotRepository.saveAll(snapshots);
    }

    @Override
    public String getStrategyName() {
        return SnapshotStrategyType.JPA.getDescription();
    }
}
```

가장 처음 기능을 작성하며, 로직의 흐름을 나타내기 위해 빠르게 작성한 코드입니다. 기본적으로 BalanceSnapshot 엔티티, Account 엔티티와 각각 매핑된 JpaRepository의 `findAll()`과 `saveAll()` 메서드를 이용합니다.

당연하게도, 이 코드는 문제 의식의 출발점이 된 코드입니다. 메모리 부족 등의 이유로 데이터를 모두 로드하는 것이 불가능하거나 비용적으로 비효율적임을 걱정했습니다.

`StreamSnapshotStrategy.java`

```java
@Component("streamSnapshotStrategy")
@RequiredArgsConstructor
public class StreamSnapshotStrategy implements SnapshotStrategy {

    private final AccountCustomRepository accountCustomRepository;
    private final BalanceSnapshotRepository balanceSnapshotRepository;

    @PersistenceContext
    private EntityManager entityManager;

    private static final Logger logger = LoggerFactory.getLogger(StreamSnapshotStrategy.class);

    @Override
    public void createSnapShots() {
        logger.info("Creating snapshots using Stream API strategy");

        int fetchSize = 1000;
        int processedCount = 0;

        try (Stream<Account> accountStream = accountCustomRepository.findAllWithStream(fetchSize)) {
            List<BalanceSnapshot> batchSnapshots = new ArrayList<>(fetchSize);
            Iterator<Account> iterator = accountStream.iterator();

            while (iterator.hasNext()) {
                Account account = iterator.next();
                BalanceSnapshot snapshot = convertToSnapshot(account);
                batchSnapshots.add(snapshot);

                if (batchSnapshots.size() >= fetchSize) {
                    processedCount += batchSnapshots.size();
                    balanceSnapshotRepository.saveAll(batchSnapshots);
                    entityManager.flush();
                    entityManager.clear();
                    batchSnapshots.clear();
                }
            }

            if (!batchSnapshots.isEmpty()) {
                processedCount += batchSnapshots.size();
                balanceSnapshotRepository.saveAll(batchSnapshots);
                entityManager.flush();
                entityManager.clear();
            }
        }

        logger.info("Created {} snapshots using Stream API strategy", processedCount);
    }

    @Override
    public String getStrategyName() {
        return SnapshotStrategyType.STREAM.getDescription();
    }

    // ...
}
```

`StreamSnapshotStrategy`는 JPA Stream API는 설정된 fetch size에 따라서 일정량의 데이터를 메모리에 로드하고, 처리 후 다음 청크를 가져오는 방식입니다. saveAll() 메서드 역시 청크 내 데이터의 개수만큼만 반복적으로 처리합니다. 그러므로 이 전략에서는 Stream이 유지되는 동안 DB와의 연결이 유지되어야 합니다.

`entityManager`를 이용해 트랜잭션 내에서 영속성 컨텍스트에 데이터가 남아있어 메모리가 낭비되지 않도록 해야합니다. 특히 `Iterator`를 이용하여 배치 단위로 계좌 엔티티를 불러오고 스냅샷을 저장합니다. 따라서 트랜잭션 내에서 `flush()`를 통해 지속적으로 변경사항을 커밋하고, `clear()`를 통해 영속성 컨텍스트를 초기화하는 작업이 필요합니다.

`PagingSnapshotStrategy.java`

```java
@Component("pagingSnapshotStrategy")
@RequiredArgsConstructor
public class PagingSnapshotStrategy implements SnapshotStrategy {

    private final AccountRepository accountRepository;
    private final BalanceSnapshotRepository balanceSnapshotRepository;

    @PersistenceContext
    private EntityManager entityManager;

    private static final Logger logger = LoggerFactory.getLogger(PagingSnapshotStrategy.class);

    @Override
    @Transactional
    public void createSnapShots() {

        logger.info("Creating snapshots using Paging strategy");

        int pageSize = 1000;
        int pageNumber = 0;
        int totalProcessed = 0;
        boolean hasMoreData = true;

        while (hasMoreData) {

            // 1. COUNT 쿼리 없는 Slice로 account 가져오기
            Pageable pageable = PageRequest.of(pageNumber, pageSize);
            Slice<Account> accountSlice = accountRepository.findAllAsSlice(pageable);

            // 2. 해당 Slice 내에서 BalanceSnapshot으로 변경
            List<BalanceSnapshot> batchSnapshots = accountSlice.getContent().stream()
                    .map(this::convertToSnapshot)
                    .collect(Collectors.toList());

            // 3. 영속화 이후 영속성 컨텍스트 초기화
            balanceSnapshotRepository.saveAll(batchSnapshots);
            entityManager.flush();
            entityManager.clear();

            totalProcessed += batchSnapshots.size();
            logger.debug("Processed page {} with {} accounts", pageNumber, batchSnapshots.size());

            // 4. 다음 Slice 준비
            pageNumber++;
            hasMoreData = accountSlice.hasNext();
        }

        logger.info("Created {} snapshots using Paging strategy", totalProcessed);
    }

    @Override
    public String getStrategyName() {
        return SnapshotStrategyType.PAGING.getDescription();
    }

    // convertToSnapshot() 메서드는 생략
}
```

`PagingSnapshotStrategy`는 페이징 처리를 이용합니다. 스냅샷 생성 로직에서는 따로 전체 페이지가 필요한 것이 아니므로, `Page` 대신 `Slice`를 이용해 COUNT 쿼리에 따르는 오버헤드를 줄였습니다.

```java
@Query("SELECT a FROM Account a")
Slice<Account> findAllAsSlice(Pageable pageable);
```

JPA Stream과 달리, 페이징 처리를 이용하면 각 페이지는 독립적인 쿼리로 처리됩니다. 그러나 `createSnapshot()` 자체가 `@Transactional`을 통해서 하나의 트랜잭션으로 처리되므로, `StreamSnapshotStrategy`와 비슷하게 영속성 컨텍스트를 관리했습니다.

`JdbcSnapshotStrategy.java`

```java
@Component("jdbcSnapshotStrategy")
@RequiredArgsConstructor
public class JdbcSnapshotStrategy implements SnapshotStrategy {

    private final BalanceSnapshotCustomRepository balanceSnapshotCustomRepository;
    private static final Logger logger = LoggerFactory.getLogger(JdbcSnapshotStrategy.class);

    @Override
    @Transactional
    public void createSnapShots() {
        logger.info("Creating snapshots using JDBC");
        int insertedCount = balanceSnapshotCustomRepository.insertDirectlyFromAccounts();
        logger.info("Created {} snapshots using JDBC directly SELECT INTO", insertedCount);
    }

    @Override
    public String getStrategyName() {
        return SnapshotStrategyType.JDBC.getDescription();
    }

    // ...
}
```

`JdbcSnapshotStrategy.java`는 Spring 6.1부터 이용 가능한 JDBC Client를 이용하여 직접 데이터에 접근하는 방식입니다. 위에서 이용하는 `balanceSnapshotCustomRepository.insertDirectlyFromAccounts()` 메서드는 아래와 같습니다.

```java
@Override
public int insertDirectlyFromAccounts() {
    logger.info("Starting INSERT INTO SELECT of balance snapshots directly from account table");

    // INSERT INTO SELECT 문으로 애플리케이션 메모리 사용 없이 DB 내부에서 직접 데이터 변환 및 삽입
    int rowsAffected = jdbcClient.sql(
                    "INSERT INTO balance_snapshot (id, account_id, time_qnt, ranking_point, created_at) " +
                            "SELECT " +
                            "  CONCAT(UNHEX(LPAD(HEX(ROUND(UNIX_TIMESTAMP(NOW(3)) * 1000)), 12, '0')), " +
                            "  UNHEX(LEFT(MD5(CONCAT(UUID(), RAND())), 20))), " + // 랜덤 부분 (80비트 = 10바이트)
                            "  id, time_qnt, ranking_point, NOW() " +
                            "FROM accounts")
            .update();

    logger.info("INSERT INTO SELECT completed, inserted {} records", rowsAffected);

    return rowsAffected;
}
```

가장 중요한 점은 BalanceSnapshot 엔티티가 필요로 하는 필드는 Account 엔티티에 모두 존재하므로, 위와 같이 직접 데이터를 변환해 삽입할 수 있다는 점입니다. 따라서 처음에는 JdbcTemplate에서 지원하는 Bulk Insert를 고려해보았으나, 데이터에 대한 추가적인 변환이나 작업이 필요없다는 점에 착안해 직접 INSERT INTO SELECT 문을 이용했습니다.

ID로 이용하고 있는 ULID를 기존 어플리케이션 단에서 생성한 것과 달리, 위는 MariaDB 내부에서 직접 생성해야하므로 id에 대해 ULID를 BINARY(16)으로 저장하도록 했습니다.

이 방법에서는데이터가 애플리케이션 메모리로 로드되는 대신 DB 엔진 내에서 작업이 수행되며, 트랜잭션 오버헤드도 최소화 될 것으로 기대했습니다.

## References

- [https://en.wikipedia.org/wiki/Strategy_pattern](https://en.wikipedia.org/wiki/Strategy_pattern)
