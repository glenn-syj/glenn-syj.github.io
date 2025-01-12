---
title: Elasticsearch 코드로 알아보는 생명주기 상태 관리
lang: ko
layout: post
---

## 들어가며

- Last Update: 2025/01/12

- Elasticsearch Server에서 핵심이 되는 클래스인 `Node.java`를 살펴보며, 생명주기 상태 관리를 알아봅니다.
- Lucene 검색엔진을 이용하지 않고, Elasticsearch의 경량화된 버전을 직접 구현해보는 프로젝트의 사전 작업에 가까운 글입니다.

## 생명주기 상태 관리

Elasticsearch보다 경량화된 문서 기반 NoSQL 데이터베이스를 직접 설계하며 구현하는 프로젝트를 진행중인데요. 이 과정에서 Elasticsearch의 소스코드를 살펴보게 되는 것은 설계 시 어려움을 극복하기 위해서도 필수적인 작업입니다.

특히, 처음 설계를 시작하며 서버 자체 혹은 인스턴스의 생명주기 상태 관리를 어떻게 구현할지 고민하게 되었습니다. 따라서 Elasticsearch의 소스코드를 살펴보며 생명주기 상태 관리를 어떻게 구현하고 있는지 알아보고, 이를 참고하여 프로젝트를 진행해보고자 합니다.

### 생명주기의 중요성

데이터베이스 시스템에서 생명주기 상태 관리는 시스템의 안정성과 신뢰성을 보장하는 핵심 요소인데요. 특히, 분산 시스템과 고가용성보다는 크게 상태 기반 제어와 리소스 관리에 중점을 두어 생명주기 상태 관리의 중요성을 파악했습니다.

1. 상태 기반 로직 제어

데이터베이스 서버의 동작이 상태 기반 로직에 따라 결정되도록 선택했습니다. 예를 들어, 데이터베이스 서버는 상태를 확인하고, 상태에 따라 로직을 제어하고 예외를 던집니다. `STARTED` 상태에서만 쿼리를 받아들이고, `STOPPED` 상태에서는 쿼리를 거부하듯이 말입니다.

2. 리소스 최적화

시스템에서는 할당과 해제를 거치면서 리소스를 최적화하는 것이 중요한데요. 이는 메모리와 디스크 공간을 효율적으로 사용하기 위함입니다. 시스템이 초기화되었을 때에는 최소한의 리소스만이 투입되어야 하고, 시스템이 활성화되어 작업을 수행할 때는 적절한 최대한의 리소스를 할당해야 합니다. 마찬가지로, 시스템이 중지되었을 때와 완전히 종료되었을 때에도 차등적인 리소스 해제가 필수적임을 알 수 있겠습니다.

## Elasticsearch 노드의 생명주기

Elasticsearch에서 Node는 클러스터를 구성하는 단위이자 데이터를 저장하고, 검색 및 집계 요청을 처리하는 역할을 가지고 있습니다. 따라서 이를 가장 핵심적인 단위로 볼 수 있는데요. 이번 단락에서는 Node의 생명주기를 다뤄보고자 합니다.

### Node와 Lifecycle 개요

```java
// server/src/main/java/org/elasticsearch/node/Node.java
public class Node implements Closeable {
    ...
    private final Lifecycle lifecycle = new Lifecycle();
    ...
}
```

Node는 내부적으로 Lifecycle 객체를 이용해 생명주기의 상태를 관리합니다. 이러한 Lifecycle 클래스는 상태 전환 가능 여부 확인 및 전환 메서드를 제공하고, 불가 혹은 실패 시 예외를 던지는 역할을 합니다. 아래는 Lifecycle 클래스의 코드입니다.

```java
// server/src/main/java/org/elasticsearch/common/component/Lifecycle.java
public final class Lifecycle {

    public enum State {
        INITIALIZED,
        STOPPED,
        STARTED,
        CLOSED
    }

    private volatile State state = State.INITIALIZED;

    public State state() {
        return this.state;
    }

    /**
     * Returns {@code true} if the state is initialized.
     */
    public boolean initialized() {
        return state == State.INITIALIZED;
    }

    ...

    public boolean canMoveToStarted() throws IllegalStateException {
        return switch (state) {
            case INITIALIZED -> true;
            case STARTED -> false;
            case STOPPED -> {
                assert false : "STOPPED -> STARTED";
                throw new IllegalStateException(
                    ElasticsearchProcess.isStopping()
                        ? "Can't start lifecycle object when the Elasticsearch process is shutting down"
                        : "Can't move to started state when stopped"
                );
            }
            case CLOSED -> {
                assert false : "CLOSED -> STARTED";
                throw new IllegalStateException("Can't move to started state when closed");
            }
        };
    }

    public synchronized boolean moveToStarted() throws IllegalStateException {
        if (canMoveToStarted()) {
            state = State.STARTED;
            return true;
        } else {
            return false;
        }
    }

    ...
}
```

`Lifecycle.java`의 주석과 코드에서는 상태 관리에 대한 설명이 구체적으로 작성되어 있습니다. 눈 여겨 볼만한 부분은 다음과 같습니다.

#### Lifecycle 상태 전환 규칙

우선, 코드에서 허용하는 간단한 상태 전환 규칙은 다음과 같습니다. 이러한 규칙은 불리언을 반환하는 `canMoveToStarted()` 메서드 등에서 확인할 수 있습니다.

<li>INITIALIZED -&gt; STARTED, CLOSED</li>
<li>STARTED     -&gt; STOPPED</li>
<li>STOPPED     -&gt; CLOSED</li>
<li>CLOSED      -&gt; </li>
<br/>

한 번 전환된 상태는 다시 되돌릴 수 없습니다. 예를 들어, `STARTED` 상태에서 `STOPPED` 상태로 전환되었다면, `STOPPED` 상태에서 `STARTED` 상태로 전환할 수 없습니다. 또한, 상태 전환의 순서가 강제되어 순차적으로만 진행가능합니다.

```java
synchronized (lifecycle) {
    if (lifecycle.state() == State.STARTED) {
        // 1. STARTED -> STOPPED
        lifecycle.moveToStopped();

        // 2. STOPPED -> CLOSED
        // 다른 스레드가 STOPPED -> STARTED로 변경할 수 없음
        lifecycle.moveToClosed();
    }
}
```

재밌는 점은 `Lifecycle` 객체 자체가 락으로 이용될 수 있다는 점인데요. 위에서 `lifecycle.moveToStopped()` 메서드를 호출하면, `lifecycle` 객체 자체가 락으로 이용되어 다른 스레드가 `STOPPED` 상태로 전환할 수 없게 됩니다.

### Node와 Lifecycle의 상태 관리 패턴

#### final 클래스로서의 Lifecycle

Lifecycle 클래스는 Node 클래스와 달리 불변 클래스로 선언되어 있는데요. 이는 다음과 같은 이점을 가진다고 볼 수 있습니다.

1. 상태 전환 규칙의 안정성 보장

   `final` 키워드는 상속을 통한 오버라이딩을 막아 엄격한 상태 전환 규칙이 훼손되는 것을 방지합니다. 또한, 하위 클래스에서 `canMoveToStarted()`, `moveToStarted()` 등의 상태 전환 관련 메서드를 재정의하여 의도하지 않은 변경에 따른 부작용을 방지할 수 있습니다. 또한, `synchronized` 키워드로 선언된 메서드들의 동작 역시 하위 클래스에서 변경할 수 없게 함으로써 스레드 안전성이 깨지는 것도 방지합니다.

2. 설계 의도의 명확한 전달

   위의 안정성 보장에서도 드러나듯이, Lifecycle 클래스가 확장이 아니라 독립적인 상태 관리를 목적으로 하는 컴포넌트임을 명시하는 셈입니다.

`final` 키워드는 특히 Elasticsearch와 같은 분산 시스템에서 컴포넌트의 생명주기를 안전하고 예측 가능하게 관리하는 데 중요한 역할을 하는 셈입니다.

#### state 선언 시의 volatile 키워드

`State` 타입의 상태 변수는 `volatile` 키워드로 선언되어 있는데요. 이는 멀티 스레드 환경에서 변수의 가시성을 보장하는 역할을 합니다. 여기에서 가시성이라 함은, `state` 변수가 변경될 때 다른 스레드에서도 이 변화를 인식함을 의미합니다. `volatile` 키워드는 모든 스레드가 공유하는 중앙 저장소인 메인 메모리에 변수를 저장하고, 각 스레드는 이 메모리에서 변수를 읽어오도록 합니다.

코드를 통한 예시를 살펴보겠습니다. 먼저 상태 전환에서 `volatile` 키워드를 이용하지 않은 클래스입니다. 아래 코드를 실행하면 어떤 결과가 출력될까요?

```java
public class NonVolatileNode {
    private boolean runningState = true;

    public void start() {
        new Thread(() -> {
            System.out.println("Thread started.");
            while (runningState) {
                // 작업 수행
            }
            System.out.println("Thread stopped.");
        }).start();
    }

    public void stop() {
        System.out.println("Stopping thread...");
        runningState = false;
    }

    public static void main(String[] args) throws InterruptedException {
        NonVolatileNode node = new NonVolatileNode();
        node.start();

        Thread.sleep(1000); // 1초 후에 스레드를 멈춤
        node.stop();

        Thread.sleep(1000); // 스레드가 멈출 시간을 줌
        System.out.println("Main thread finished.");
    }
}
```

```
Thread started.
Stopping thread...
Main thread finished.
```

여기에서 중요한 점은 결과에서 `start()` 메서드 내의 `while (runningState)` 루프가 종료되지 않는다는 점입니다. 이는 `runningState`의 상태 변경이 다른 스레드에 즉시 반영되지 않는 까닭입니다.

즉, `start()` 메서드에서 새로 생성된 스레드와 `main()` 메서드 내의 스레드는 각각 다른 CPU 코어에서 실행될 수 있기에, 변경된 값을 즉시 감지하지 못할 수 있습니다.

이번에는 `volatile` 키워드를 이용한 코드입니다. 결과에서 어떤 차이가 있을까요?

```java
public class VolatileNode {
    private volatile boolean runningState = true;

    public void start() {
        new Thread(() -> {
            System.out.println("Thread started.");
            while (runningState) {
                // 작업 수행
            }
            System.out.println("Thread stopped.");
        }).start();
    }

    public void stop() {
        System.out.println("Stopping thread...");
        runningState = false;
    }

    public static void main(String[] args) throws InterruptedException {
        VolatileNode node = new VolatileNode();
        node.start();

        Thread.sleep(1000); // 1초 후에 스레드를 멈춤
        node.stop();

        Thread.sleep(1000); // 스레드가 멈출 시간을 줌
        System.out.println("Main thread finished.");
    }
}
```

```
Thread started.
Stopping thread...
Thread stopped.
Main thread finished.
```

이번에는 `Thread stopped.`이 출력되었습니다. 이는 `runningState` 변수의 변경이 메인 메모리에 기록되고, 다른 스레드가 메인 메모리에서 값을 직접 읽어오므로 가시성이 보장되는 까닭입니다. 따라서 `volatile` 키워드는 스레드 간의 상태 동기화를 보장합니다.

#### 상태 전환 메서드에서의 synchronized 키워드

이번에는 `moveToStarted()` 등 상태 전환 메서드에서 사용된 `synchronized` 키워드의 존재 이유를 살펴보고자 합니다.

앞서 `volatile` 키워드를 살펴보며, 메인 메모리를 이용함으로써 가시성을 보장함을 살펴본 바 있습니다. 그러나 `volatile` 키워드는 읽기와 쓰기 작업을 원자적으로 수행하지는 못합니다.

여기에서 원자적이라 함은, 하나의 스레드가 변수를 읽거나 쓰는 동안 다른 스레드가 동일한 변수를 읽거나 쓰지 못하도록 보장하는 것을 의미합니다. 익숙한 락(lock)이 원자적 연산을 보장하는 대표적인 기법 중 하나입니다.

예를 들어, 다음 코드는 `count` 값을 증가시키는 작업을 수행합니다. 이 때, `count` 값은 `volatile` 키워드로 선언되어 있기에 모든 스레드가 메인 메모리의 값을 변경하고 읽어옵니다.

```java
public class VolatileCounter {
    private volatile int count = 0;

    public void increment() {
        count++; // 비원자적 연산
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args) throws InterruptedException {
        VolatileCounter counter = new VolatileCounter();
        int numberOfThreads = 1000;
        Thread[] threads = new Thread[numberOfThreads];

        // 각 스레드가 1000번씩 increment() 호출
        for (int i = 0; i < numberOfThreads; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    counter.increment();
                }
            });
            threads[i].start();
        }

        // 모든 스레드가 종료될 때까지 대기
        for (int i = 0; i < numberOfThreads; i++) {
            threads[i].join();
        }

        // 기대하는 결과는 1000 * 1000 = 1,000,000
        System.out.println("Final count: " + counter.getCount());
    }
}
```

```
Final count: 996238
```

기대하는 결과가 1_000_000임과 달리, 실제 결과는 1_000_000 보다 작은 값을 출력합니다. 이는 `count++` 연산이 원자적으로 수행되지 않기 때문입니다.

즉, `volatile` 키워드를 이용하더라도 값에 대해 읽기와 쓰기가 잦다면, 여러 스레드에서 동시에 다른 값을 읽거나 쓰는 경우가 발생하게 됩니다.

이러한 문제는 `synchronized` 키워드를 이용함으로써 해결할 수 있습니다. 아래 코드는 `synchronized` 키워드를 이용해 `count` 값을 증가시키는 작업을 수행합니다.

```java
public class SynchronizedCounter {
    private int count = 0;

    public synchronized void increment() {
        count++; // 원자적 연산
    }

    public synchronized int getCount() {
        return count;
    }

    public static void main(String[] args) throws InterruptedException {
        SynchronizedCounter counter = new SynchronizedCounter();
        int numberOfThreads = 1000;
        Thread[] threads = new Thread[numberOfThreads];

        // 각 스레드가 1000번씩 increment() 호출
        for (int i = 0; i < numberOfThreads; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    counter.increment();
                }
            });
            threads[i].start();
        }

        // 모든 스레드가 종료될 때까지 대기
        for (int i = 0; i < numberOfThreads; i++) {
            threads[i].join();
        }

        // 기대하는 결과는 1000 * 1000 = 1,000,000
        System.out.println("Final count: " + counter.getCount());
    }
}
```

```
Final count: 1000000
```

위 코드는 `synchronized` 키워드를 이용함으로써 `count` 값을 증가시키는 작업을 원자적으로 수행합니다. 이는 모니터 락 개념을 이용해 원자성을 보장합니다.

사실 위 코드에서 `volatile` 키워드 역시 빠져 있는데, 이는 메모리 배리어와 happens-before 관계를 통해 메모리 가시성이 보장되는 까닭입니다.

`synchronized` 키워드의 메커니즘에 대한 자세한 내용은 다른 글에서 다루고, 이번에는 위와 같은 `synchronized`의 역할만 살펴보고 넘어가겠습니다.

#### volatile과 synchronized 동시 이용

그렇다면 왜 elasticsearch에서는 `volatile`과 `synchronized`를 동시에 이용하는 것일까요? 이는 메모리 가시성과 원자성을 적절한 성능을 확보하며 보장하기 위함이라고 생각합니다.

즉, `volatile`은 읽기에 더욱 초점을 맞추어 메모리 가시성을 보장하고, `synchronized`는 쓰기에 더욱 초점을 맞추어 원자성을 보장하려는 목적으로 보입니다.

이는 상태 전환 여부를 확인하는 `canMoveToStarted()`과 같은 메서드에서는 굳이 `synchronized` 키워드가 붙어있지 않음에서 확인할 수 있습니다.

반면, 변수 값을 바꿈으로써 상태 전환을 수행하는 `moveToStarted()` 메서드에서는 `synchronized` 키워드가 붙어있음 역시 확인할 수 있습니다.

특히, `synchronized` 키워드를 단순 읽기 작업에만 이용된다면 락 획득과 해제에 따른 오버헤드 및 스레드 경합 시의 대기로 인한 성능 저하가 우려되기 때문이라고 생각합니다.

이렇듯 `volatile`과 `synchronized` 키워드를 동시에 이용함으로써, `Lifecycle` 클래스는 `Node` 클래스의 상태를 효율적으로 확인하고 안전하게 전환하는 과정의 핵심이 됩니다.

## 나가며

이번 글에서는 Elasticsearch의 `Node` 클래스와 `Lifecycle` 클래스를 살펴보며, 생명주기 상태 관리 방식을 알아보았습니다. 이를 통해 경량화된 문서 기반 NoSQL 데이터베이스 프로젝트에서 생명주기 상태 관리를 구현할 때 참고할 수 있는 패턴을 알 수 있었습니다.

특히, Java에서 지원하는 `final`, `volatile`, `synchronized` 키워드를 살펴봄으로써 상태 관리 패턴을 이해하는 데 도움이 되었습니다.
