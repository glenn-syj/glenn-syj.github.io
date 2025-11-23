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

### 모델 비교: 스레드 vs. 이벤트 루프

## 사례로 보는 비동기 처리

### Spring MVC

### FastAPI

## 사고 실험: 스레드 vs. 이벤트 루프

## 더 나아가기

## References
