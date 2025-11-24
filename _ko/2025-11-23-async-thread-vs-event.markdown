---
title: 비동기를 처리하는 두 가지 방식 - 스레드와 이벤트
lang: ko
layout: post
---

## 들어가며

- 이 글은 비동기 처리를 다루는 두 방식인 스레드 기반 방식과 이벤트 루프 기반 방식이 가지는 차이를 다룬다.
- 최종 수정일: 25/11/24

업무에서 FastAPI를 사용하게 되면서 Python의 async/await 기반 이벤트 루프 모델을 경험하게 되었고, 이전에 경험한 Spring MVC 기반의 Thread Pool + CompletableFuture 기반 비동기 처리와 비교할 기회가 생겼다.

Spring MVC는 Thread-Per-Request + Blocking I/O 구조를 사용하기 때문에, 비동기 처리도 본질적으로 스레드 기반 병렬성을 벗어나지 못한다. 요청은 워커 스레드 1개에서 동기적으로 실행되고, JDBC 역시 기본적으로 Blocking I/O 방식이다. 따라서 컨테이너를 Tomcat 대신 Netty로 교체하거나, Spring WebFlux + R2DBC 같은 완전한 non-blocking 스택을 사용하지 않는 다면 비동기의 효과는 결국 스레드를 다른 풀로 분산시키는 수준이라고 이해하고 있다.

반면, FastAPI에서는 async/await 기반 이벤트 루프 비동기 모델을 표면적으로는 쉽게 사용할 수 있지만, 아직 내부 동작 방식은 완전히 이해하지 못했다. 처음에는 psycopg2 같은 blocking DB 드라이버로 구현했으나, 이후 asyncpg로 바꾸어 non-blocking I/O 환경도 직접 경험해보았다.

이 글에서는 FastAPI가 이용하는 이벤트 루프 기반 비동기가 Spring MVC가 이용하는 스레드 기반 비동기와 실제로 어떤 차이가 있는지, 특히 I/O 중심 작업에서의 차이를 더 깊이 살펴보고 비교하고자 한다. 이 과정에서 기본적인 비동기 모델의 차이와 함께, 프레임워크가 가지는 차이도 부수적으로 살펴보고자 한다.

## 비동기 이해하기

### 개념

비동기 프로그래밍은 하나의 작업이 완료되기를 기다리지 않고, 다음 작업을 이어서 실행할 수 있도록 하는 프로그래밍 모델이다. 비동기 프로그래밍에서는 제어 흐름이 비선형적이고, 특정 작업의 완료를 기다리는 동안 스레드가 블로킹되지 않는다. 비동기 모델이 동기 모델과 어떤 차이가 있는지 살펴보면 더욱 이해가 쉽다.

- 제어 흐름의 분리

비동기 모델에서는 작업의 시작과 결과의 처리가 분리되는 것이 특징이다. 동기 모델에서 함수의 호출과 반환이 스택에 쌓인다면, 비동기 모델에서는 작업 지시 후 제어권이 호출부에 반환된다. 이러한 결과는 완료 시점에 비동기 처리 메커니즘을 통해 다루어진다.

- 논블로킹 I/O

동기 모델에서의 I/O 작업이 이루어지면 호출부의 스레드가 대기 상태에 들어간다. 이를 블로킹 I/O라고 부를 수 있다. 반면, 비동기 모델에서는 I/O 작업을 위임하고 바로 제어권을 반환 받는다. 이를 논블로킹 I/O라고 할 수 있다.

비동기 모델이 논블로킹 I/O를 특징으로 가지지만, 기술적으로 논블로킹 I/O 자체가 비동기는 아님에 유의해야한다. 오히려 논블로킹 I/O가 제어 흐름의 분리를 위해서 채택되었다고 보는 것이 더 명확하다. 비동기 처리는 완료 시점에서의 처리 메커니즘까지 포함되는 개념이라고 보면 쉽다.

### 필요성

비동기 모델의 필요성을 떠올리기 위해서 비동기 모델이 존재하지 않을 때를 상상해보면 좋을 것 같다. 모든 요청이 동기적으로 처리된다면 어떨까? 특히 값비싼 I/O 요청이라면? 놀랍게도 세상에는 이미 C10K 문제라는 것이 존재했다.

- C10K 문제

C10K 문제는 1999년 댄 케겔(Dan Kegel)이 제시한 문제로, 다음과 같은 뜻을 가진다. 1만(10k)개의 동시(Concurrent) 클라이언트(Client)를 어떻게 효율적으로 처리할 것인가? 당시에 우세했던 Thread-Per-Request 방식으로는 이러한 문제를 풀기가 어려웠다. 어떤 문제가 생길까?

친절하게도 ![올리브영 테크블로그 - 고전 돌아보기, C10K 문제](https://oliveyoung.tech/2023-10-02/c10-problem/)에서 "1만 명이 접속하는 채팅 서버"라는 예시로 c10k 문제를 다루어주었으니, 간략하게만 설명하고 넘어가고자 한다.

> Thread가 1만개가 실행되면 어떤 문제가 있을까요?

> 먼저, 메모리가 문제가 됩니다. thread가 하나 생성되면, 각 thread는 별도의 메모리 스택을 가지게 됩니다.
> 물론 process보다는 적지만, Java 64bit VM의 경우 1개의 thread는 1024kbyte(1MB)를 사용한다고 합니다.
> 1만명이면 10Gbyte 정도가 기본으로 필요하겠네요? 괜찮습니다. 이 정도는 버틸만 합니다. 우리 서버는 64Gbyte 메모리를 가지고 있거든요.

첫째, 각 스레드 당 스택 사용량으로 인한 메모리 오버헤드가 발생한다.

> (아직도 뒷골이 서늘합니다.) 잘 알고 있다시피, thread를 너무 많이 만들면 context switching 과정에서 경합(racing condition)이 발생합니다.
> 각 thread가 자신의 execution time을 할당 받기 위해서 서로 경쟁을 하지요.
> 소켓에 데이터가 들어왔는지 확인하고 읽어들이기 위해서 1만개의 thread는 모두 polling 경쟁을 벌이게 됩니다.
> 아무도 채팅창에서 키보드를 치고 있지 않지만, 제가 만든 서버의 thread는 서로 CPU 자원을 차지하기 위해서 아우성을 지릅니다.

둘째, 스레드 간의 CPU 자원 경쟁과 컨텍스트 스위칭 과정에서의 경쟁이 발생한다.

이제 다시 이번 글의 '개념' 파트를 돌아보자. 비동기 모델에서는 작업의 시작과 결과의 분리가 이루어진다. '논블로킹 I/O'로 1만 개의 대기 스레드를 없애 물리적 자원 문제(스레드 폭발)를 해결하고, '제어 흐름의 분리'가 단일 스레드로 1만 개의 독립적인 작업 흐름을 관리해 논리적 동시성 문제(요청 구분)를 해결하면 어떨까?

이것이 바로 대기가 긴 I/O 바운드 작업에서 비동기가 필요한 이유다. 하지만 반대로 말하자면, CPU 바운드 작업에서는 비동기가 필요하지 않다는 뜻이기도 하다. CPU 바운드 작업에서는 대기보다는 연산이 우위를 점하기 때문이다.

### 모델 비교: 스레드 vs. 이벤트 루프

#### 이벤트 루프 모델

앞서 말했듯, 비동기 처리를 통해 단일 스레드로도 C10K 문제에 대응할 수 있다고 치자. 그렇다면 어떻게 스레드를 1만 개나 만들지 않고도 1만 개의 요청을 구분하면서 작업 흐름을 관리할 것인가?

이벤트 루프(Event Loop)라는 대표적인 비동기 모델이 있다. 이름에서 잘 드러나듯이, 이벤트 루프는 종료되기 전까지 계속 "반복"해 "이벤트" 메시지를 수신하고 처리한다. 여기서 이벤트 메시지는 C10K 문제에서의 각 요청의 완료 신호로 대응할 수 있다. 의사 코드는 다음과 같다.

```
loop
    message := get_next_message()
    process_message(message)
while message != quit
```

적은 수의 스레드가 위와 같이 이벤트 루프를 통해 요청을 관리한다. 하지만 이벤트 루프가 CPU-heavy한 연산으로 블로킹되면 전체 요청 처리 속도가 급격히 떨어진다. 따라서 이벤트 루프 기반 시스템에서는 I/O 작업은 커널 레벨의 논블로킹 이벤트 큐를 통해 수행하고, CPU-heavy 작업은 별도 워커 스레드 풀에서 처리하는 경우가 많다.

#### 스레드 기반 모델

스레드 기반 모델(Thread-Per-Request 방식)이라고 해서 비동기 처리를 전혀 사용할 수 없는 것은 아니다. 스레드 기반 모델에서도 나중에 결과를 받을 수 있는 객체를 이용하면 비동기적으로 작업 결과를 처리할 수 있기 때문이다. 예를 들어, Java의 Future, Rust의 Task, JavaScript의 Promise 등이 있다.

즉, 작업 완료를 기다리는 동안 호출 측 코드가 블로킹되지 않아 다른 작업을 진행할 수 있다. 그러나 물리적 스레드 자체는 요청 수만큼 존재하기 때문에, I/O가 많은 환경에서는 스레드 폭발 문제가 여전히 발생할 수 있다.

다만 매번 스레드를 새로 생성하는 것은 아니며, 스레드 풀을 사용해 이미 생성된 유휴 스레드를 재사용하는 경우가 많다. 예를 들어, 스레드 풀 기반의 비동기 실행을 지원하여 스레드 관리 비용을 줄일 수 있다.

#### 하이브리드 모델

이벤트 루프 모델이 I/O-bound 작업에서 효율적인 것처럼, 스레드 기반 모델은 CPU-heavy 작업을 병렬로 처리할 수 있다는 장점이 있다. 이는 곧 서로의 단점을 상쇄함을 의미한다.

따라서 현대 서버는 두 모델을 결합하는 하이브리드 구조를 많이 사용한다. I/O 처리는 이벤트 루프에서 논블로킹 I/O로 처리해 수만 개 요청을 동시에 처리하고,
CPU-heavy 연산은 별도 워커 스레드 풀에서 병렬 처리해 이벤트 루프의 블로킹을 방지하고 연산 성능을 높인다.

## 사례로 보는 비동기 처리

### Spring MVC: 스레드 풀 초과

#### 가정

1. 전자상거래 플랫폼에서 하루 동안 블랙프라이데이 이벤트가 진행되고 있다.
2. 동시에 수만 명의 사용자가 상품 검색, 장바구니 조회, 결제 요청을 서버에 보낼 것이다.
3. 서버는 Spring MVC 기반으로, Thread-per-request 모델과 스레드 풀(크기 200)로 동작하고 있다.
4. 각 요청 처리 중 일부는 외부 결제 API 호출이나 상품 재고 DB 조회처럼 I/O 지연이 긴 작업을 포함한다.
5. 서버 인스턴스의 수평적 확장은 고려하지 않는다.

#### 문제 상황

1. 200개의 요청은 스레드 풀에서 즉시 처리할 수 있다.
2. 나머지 요청은 스레드 풀 대기 큐에 쌓이며, I/O 작업이 길어지면 스레드가 블로킹 상태로 유지된다.
3. 동시에 새로운 요청이 계속 들어오면서 대기 큐가 포화되고 응답 지연과 타임아웃이 발생한다.
4. 서버 측 로그에는 스레드 고갈과 타임아웃 에러가 찍혀있다.

#### 어떻게 해결할까?

- 스레드 풀 조정

```yaml
server:
  tomcat:
    max-threads: 500 # 또는 그 이상
    min-spare-threads: 50
```

단기적으로 가장 빠르게 이를 해결할 수 있는 것은 스레드 풀의 크기를 키우는 것이다. 블로킹된 스레드 수만큼 여유 스레드를 확보하면 대기 큐에 쌓이는 요청을 줄일 수 있다.

하지만 앞서 살펴보았듯, 스레드 수가 늘어나면 컨텍스트 스위칭 비용과 함께 메모리 측면에서의 오버헤드가 간과할 수 없을 정도로 커질 것이다.

- I/O 작업 비동기 처리

```java
// PaymentService 내부
public CompletableFuture<String> callExternalPaymentAPI() {
    return webClient.get()
        .uri("/payment")
        .retrieve()
        .bodyToMono(String.class) // WebClient의 논블로킹 Mono 반환
        // Mono를 CompletableFuture로 변환
        .toFuture();
}

// Controller
@GetMapping("/order")
public CompletableFuture<String> processOrder() {
    // Non-blocking I/O 작업
    return paymentService.callExternalPaymentAPI();
}
```

Spring MVC가 기본적으로 Thread-Per-Request라는 점을 다시 떠올려보자. 그렇다면 가장 먼저 클라이언트의 HTTP 요청을 실제로 처리하는 워커 스레드, 서블릿 스레드가 존재할 것이다.

서블릿 컨테이너는 내부의 스레드 풀에서 사용 가능한 서블릿 스레드 하나를 꺼내어 클라이언트의 요청에 할당한다. 동기 모델에서는 할당된 서블릿 스레가 길고 긴 외부 I/O 요청이 끝날 때까지 블로킹되어 어떠한 작업도 처리하지 못하는 대기 상태일 것이다.

하지만 위와 같이 비동기 모델을 이용한다면 블로킹되지 않기에 비록 I/O 요청이 끝날 때 응답이 전해지더라도, 그 사이에 다른 작업을 처리할 수 있게 된다.

### FastAPI: 이벤트 루프 블로킹

#### 가정

1. 사용자가 PDF 또는 문서를 업로드하면, 서버가 이를 분석하고 검색 가능한 형태로 제공하는 AI 문서 분석/검색 플랫폼이다.
2. 단일 FastAPI 이벤트 루프(asyncio 기반)에서 요청을 처리한다.
3. 동시에 수천~수만 명의 사용자가 문서 업로드, 질의, 검색 요청을 보낸다.
4. 각 요청 처리에는 다음과 같은 작업이 포함된다.
   - CPU-heavy 연산: PDF/문서에서 텍스트 추출, 전처리, 벡터화
   - DB 조회: 실수로 동기 psycopg2 사용
   - 외부 LLM API 호출: 질문-답변 생성 (invoke 동기 호출)

#### 문제 상황

1. 이벤트 루프는 단일 스레드로 동작하므로, CPU-heavy 작업이 발생하면 다른 요청 처리 지연
2. 동기 DB 호출(psycopg2)과 동기 LLM API 호출(invoke)으로 인해 이벤트 루프 블로킹
3. 블로킹 상태에서 다른 요청이 들어오면 응답 지연 및 타임아웃 발생
4. 서버 로그에는 이벤트 루프 블로킹 경고 및 응답 지연 기록

#### 어떻게 해결할까?

- I/O 요청을 논블로킹으로 전환

```python
import httpx
from fastapi import FastAPI

app = FastAPI()

@app.get("/qa")
async def query_llm(prompt: str):
    async with httpx.AsyncClient() as client:
        resp = await client.post("https://llm.api/generate", json={"prompt": prompt})
        return resp.json()
```

httpx는 requests 라이브러리의 비동기 버전으로, HTTP 요청을 보낼 때 스레드를 블로킹하지 않고 논블로킹 I/O를 이용한다. 또한, await 키워드는 LLM API 응답을 기다리는 동안 이벤트 루프에 제어권을 돌려준다.

따라서 이벤트 루프는 대기 시간 동안 다른 수많은 요청을 받거나 처리하는 작업을 계속 수행할 수 있으므로, 서버의 동시 처리 능력이 향상된다.

- CPU-heavy 작업을 별도 스레드/프로세스 풀에서 실행

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=10)

def process_pdf(file_path: str):
    # PDF 텍스트 추출, 전처리, 벡터화 등 CPU-heavy 작업
    return processed_data

@app.post("/upload")
async def upload_pdf(file_path: str):
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(executor, process_pdf, file_path)
    return {"processed": result}
```

ThreadPoolExecutor는 CPU 바운드 작업을 실행하기 위한 별도의 워커 스레드 풀을 생성한다. `loop.run_in_executor()`는 CPU 집약적인 `process_pdf()` 함수를 워커 스레드 풀로 위임한다. 마찬가지로 `await` 키워드를 통해서 작업이 완료되기를 기다린다.

여기에서는 CPU 바운드 작업인 PDF 처리가 백그라운드에서 병렬로 실행되므로 서버의 반응성과 동시 처리 능력이 유지된다.

- DB 조회 비동기화

```python
import asyncpg

async def fetch_document(doc_id: int):
    conn = await asyncpg.connect(...)
    result = await conn.fetchrow("SELECT * FROM documents WHERE id=$1", doc_id)
    await conn.close()
    return result
```

전통적인 DB 커넥터와 달리, 비동기 PostgreSQL 드라이버인 `asyncpg`는 `asyncio`를 지원해 DB 연결과 쿼리 실행 모두 논블로킹 방식으로 진행된다. 따라서 DB와 관련된 I/O 작업이 블로킹되지 않는다.

대신, DB 커넥션 풀을 개발자가 관리해야 한다는 점에 유의해야 한다. 풀 생성 및 해제 로직을 적절히 구현하지 않으면 DB 서버에 과부하가 걸리거나 성능 문제가 발생할 수 있다.

## 더 나아가기 - 코루틴과 가상스레드

최근 JDK 21부터 Java에서도 코루틴과 유사한 가상 스레드가 정식으로 릴리즈에 포함된 것으로 알고 있다. 그래서 비동기 처리를 살펴보는 김에 간단하게 내용을 짚고 넘어가도 좋을듯해 더 나아가기에 추가한다.

### 코루틴

#### 코루틴이란?

코루틴은 함수 수준에서 실행을 일시 중단(suspend)하고 나중에 다시 재개(resume)할 수 있는 단위를 의미한다. 스레드라기 보다는, 스레드 내에서 컨텍스트 스위칭 없이 동작하는 하나의 논블로킹 작업 단위라고 보면 된다.

Python 예시에서 이용하던 async/await에서 await로 일시중단하고, async로 이벤트 발생 시 재개하는 것을 떠올려보면 쉽다. 그런데 어떻게 중단과 재개가 반복될 수 있는 걸까?

#### 코루틴의 원리

코루틴에서 중단과 재개가 가능한 이유는 스레드 안에서 독립적으로 코루틴을 관리하는 컨텍스트가 존재하기 때문이다. 스레드는 생성 시에 OS 레벨에서 독립된 스택과 레지스터를 가지게 되지만, 코루틴은 스레드의 스택이나 레지스터에 의존하지 않는다.

대신 각 코루틴이 자체 컨텍스트 객체를 가지고 있다. 이 객체 안에 지역 변수나 실행 위치, 대기 중인 비동기 작업이 저장되어 있다고 보면 된다. 따라서 OS와 링크되는 스레드보다 훨씬 가볍다.

Python asyncio에서는 코루틴 객체가 힙에 올라가고, Kotlin coroutine에서도 `Continuation` 객체로 힙에 올라간다. python `await` 이용 시 중단 지점에서 지역 변수와 실행 위치가 코루틴 객체에 저장되고, kotlin `Continuation`에서도 `suspend` 호출 시마다 상태 머신을 생성한다.

여기에서 CPS(Continuation Passing Style)이라는 방식이 쓰인다. CPS는 함수가 값을 반환하는 대신, 다음에 무엇을 할 지 나타내는 함수를 인자로 받아 실행하는 방식이다. 이렇게 된다면 제어 흐름이 함수로 넘어가는 것이 아니라 여전히 코루틴 객체가 가지고 있게 된다.

```python
def add_cps(x, y, cont):
    cont(x + y)  # 결과를 continuation(콜백)에 전달

# continuation 정의
def print_result(result):
    print(result)

add_cps(1, 2, print_result)
```

따라서 코루틴에서는 다음과 같은 방식이 흔히 쓰인다.

1. 현재 실행 상태 + 다음 작업을 Continuation 객체에 저장한다.
2. I/O가 완료되면, 이벤트 루프가 Continuation을 호출하여 코루틴을 이어서 실행한다.

### 가상 스레드

#### 가상 스레드란?

Java 21부터 도입된 가상 스레드(Virtual Thread)는 경량 스레드라고 생각하면 쉽다. 기존 OS 스레드와 달리, 대량의 스레드를 동시에 생성할 수 있도록 설계되었다. 여기에서는 동기 코드를 그대로 사용할 수 있다.

기존 스레드는 OS에서 스택과 레지스터를 할당받는다. 따라서 수천 개 이상 생성하면 메모리와 컨텍스트 스위칭 비용이 커진다. 가상 스레드는 JVM이 스레드 스택을 관리하며, 필요할 때만 OS 스레드에 매핑된다. 블로킹 I/O가 발생하더라도 JVM 런타임이 다른 가상 스레드를 실행시키므로, 단일 스레드처럼 많은 동시 작업을 효율적으로 처리할 수 있다.

#### 가상스레드의 원리

가상 스레드는 코루틴처럼 명시적인 키워드 없이도 I/O 대기 시간 동안 스레드 자원을 낭비하지 않도록 설계되었다. 이 설계는 런타임에서 OS 스레드에의 마운트, 언마운트, 리마운트 메커니즘을 통해 동작한다. 가상 스레드와 관계를 맺는 OS 스레드를 플랫폼 스레드(Platform Thread)라고 부를 수 있다.

1. 마운트(Mount): 가상 스레드는 OS 스레드인 플랫폼 스레드(Platform Thread)에 올라타서 실행을 시작한다.
2. I/O 블로킹 감지 및 언마운트 (Unmount): 가상 스레드가 Blocking I/O 작업을 수행할 때, JVM은 이 호출을 가로채 가상 스레드의 상태 (스택, 지역 변수 등)를 힙(Heap)에 저장하고 플랫폼 스레드에서 언마운트시킨다.
3. 플랫폼 스레드 재사용: 언마운트된 플랫폼 스레드는 즉시 JVM의 다른 가상 스레드를 마운트하여 실행한다. (즉, I/O 대기 시간 동안 OS Thread가 낭비되지 않음)
4. I/O 완료 및 리마운트 (Remount): I/O 작업이 완료되면, JVM은 저장된 상태를 복원하고 가상 스레드를 다시 플랫폼 스레드에 마운트하여 실행을 재개한다.

```java
Thread vThread = Thread.ofVirtual().start(() -> {
    // 동기 코드 그대로 작성
    blockingIO();
});
```

> Virtual threads are not faster threads; they do not run code any faster than platform threads.
> They exist to provide scale (higher throughput), not speed (lower latency).

쉽게 말하자면, OS 스레드 1개를 여러 가상 스레드가 번갈아가며 빌려 쓰는 구조이다. 가상 스레드의 목적은 확장성을 통해서 동시 처리량을 높이고 I/O 작업에서 낭비되는 스레드를 줄이는 것에 목적이 있음을 알 수 있다.

가상 스레드는 대부분의 블로킹 I/O에서 자유롭게 언마운트/리마운트되지만, Pinning 상태에서는 언마운트가 제한됨에 유의해야한다. 언마운트가 제한되면 OS 스레드 재사용효과가 떨어지기 때문이다. Pinning 상태가 되는 경우는 다음과 같다.

- synchronized 블록 안에서 가상 스레드가 실행될 때: JVM이 해당 블록의 락 보호를 위해 가상 스레드를 핀 상태로 유지
- 네이티브 메서드나 외부 함수 호출: JVM은 안전을 위해 가상 스레드를 핀 처리

### 코루틴과 가상스레드 비교 표

| 항목      | 코루틴                         | 가상 스레드                              |
| --------- | ------------------------------ | ---------------------------------------- |
| 구현 단위 | 함수/작업 수준                 | 스레드 수준                              |
| 스택      | 공유/힙에 저장                 | JVM이 관리하는 가벼운 스택               |
| 키워드    | 명시적 (suspend/await 필요)    | 암묵적 (기존 코드 그대로)                |
| I/O 처리  | 이벤트 루프 기반 논블로킹      | JVM이 블로킹 감지, 다른 가상 스레드 실행 |
| 장점      | 경량, 제어 유연                | 동기 코드 그대로, 수만 개 스레드 가능    |
| 단점      | 명시적 suspend 필요, 언어 종속 | OS 스레드 매핑 시 Pinning 주의           |

## 나가며

## References

https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/
https://github.com/python/cpython/blob/e457d60daafe66534283e0f79c81517634408e57/InternalDocs/asyncio.md#L4

https://oliveyoung.tech/2023-10-02/c10-problem/

https://en.wikipedia.org/wiki/Event_loop
https://en.wikipedia.org/wiki/Futures_and_promises

https://techblog.woowahan.com/15398/
https://dev.gmarket.com/82

https://en.wikipedia.org/wiki/Coroutine

https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html
