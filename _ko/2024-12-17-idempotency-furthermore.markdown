---
title: 멱등성과 함께하는 HTTP 메서드 설계
lang: ko
layout: post
---

## 들어가며

- 멱등성을 중심으로 HTTP 메서드와 비롯한 안정성, 캐시 가능성 등의 성격을 살펴봅니다. 이후 멱등성을 고려한 HTTP 메서드 설계 방법을 살펴보겠습니다.
- nullipotent는 의미를 살리기 위해 무영향성으로 번역했습니다.

## 멱등성과 무영향성

### 기본 정의

멱등성(Idempotency)이란 어떤 연산을 여러 번 수행해도 결과가 변하지 않는 성질을 말합니다. `f(x) = x * 0`라는 함수가 있을 때, `f(x)`를 여러 번 수행해도 결과는 항상 0이라는 점이 멱등성을 만족한다고 할 수 있습니다.

반면, 무영향성(Nullipotency)이란 어떤 연산을 수행해도 결과가 변하지 않는 성질을 말합니다. `f(x) = x + 0`이라는 함수가 있을 때, `f(x)`를 수행해도 결과가 변하지 않는 것이 무영향성을 만족한다고 할 수 있습니다.

즉, 멱등성은 연산을 여러 번 수행해도 결과가 동일한 것이고, 무영향성은 연산을 수행해도 결과에 영향을 미치지 않는 성질을 의미한다고 볼 수 있겠습니다. 당연하게도, 이 둘은 가장 첫 연산에서 차이를 드러내는데요. 멱등성은 첫 연산에서는 결과의 변화를 일으킬 수 있지만, 무영향성은 첫 연산에서는 결과의 변화를 일으키지 않습니다.

### 프로그래밍 패러다임 속에서

#### 선언형 프로그래밍

선언형 프로그래밍에서 멱등성은 결과의 일관성을 보장한다는 의의를 가집니다. 코드의 안정성과 예측 가능성을 높이는 것이 중요한 까닭입니다. 대표적인 선언형 언어인 SQL의 경우, 멱등성을 만족하는 UPDATE 문은 여러 번 수행해도 결과가 동일해야 합니다. 여러 번 요청이 들어와도 데이터베이스의 상태가 변하지 않아야 하니까요.

무영향성은 마찬가지로 연산이 시스템의 상태를 전혀 변경하지 않기에, 안전하게 데이터를 조회하고 시스템의 신뢰성을 높입니다.

#### 함수형 프로그래밍

함수형 프로그래밍에서 멱등성은 함수의 결과가 항상 동일하다는 것을 의미하는데요. 함수의 입력이 동일하면 함수의 결과도 동일해야 합니다. 이는 함수의 순수성을 보장하고, 예측 가능한 결과를 얻을 수 있게 합니다. 함수의 순수성이라고 함은, 함수가 외부 상태에 의존하지 않는다는 것을 의미합니다. 즉, 함수의 결과는 함수의 입력에만 의존하며, 외부 상태에 의존하지 않는다는 것입니다. 이는 함수의 결과가 항상 동일하게 나오는 것을 보장합니다.

무영향성은 함수가 외부 상태에 의존하지 않는다는 것을 의미합니다. 즉, 함수의 결과는 함수의 입력에만 의존하며, 외부 상태에 의존하지 않는다는 것입니다. 이는 함수의 결과가 항상 동일하게 나오는 것을 보장합니다.

## HTTP 내 멱등성과 안정성, 그리고 캐시 가능성 이해

| HTTP Method | Idempotency | Safety | Cacheability         |
| ----------- | ----------- | ------ | -------------------- |
| GET         | Yes         | Yes    | Yes                  |
| DELETE      | Yes         | No     | No                   |
| POST        | No          | No     | Yes (명시적 설계 시) |
| PUT         | Yes         | No     | No                   |
| PATCH       | No          | No     | No                   |

### 멱등성

멱등성은 동일한 HTTP 요청을 여러 번 수행해도 서버의 상태가 변하지 않는 것을 의미합니다. 예를 들어, GET 요청은 동일한 요청을 여러 번 수행해도 결과가 동일합니다. GET, PUT, DELETE 메서드는 멱등성을 보장한다고 말할 수 있습니다.

### 안정성

안정성은 HTTP 요청이 서버의 상태를 변화시키지 않음을 의미합니다. 앞서 살펴본 무영향성(nullipotent)과 유사합니다. 예를 들어, GET 요청은 동일한 요청을 여러 번 수행해도 결과가 동일합니다. GET 메서드는 안정성을 보장한다고 말할 수 있습니다.

반면, PUT과 DELETE 메서드는 안정성을 보장하지 않습니다. PUT 메서드는 서버의 상태를 변화시키며, DELETE 메서드는 서버의 상태를 제거합니다. 따라서 안정성을 보장하지 않습니다.

### 캐시 가능성

캐시 가능성은 각 메서드가 반환하는 응답의 캐시 가능 여부를 나타냅니다. 예를 들어, GET 요청은 기본적으로 캐시 가능성이 있습니다. 여기에서 POST 메서드는 전통적으로 캐싱되지 않았으나, HTTP/1.1 사양을 정의하는 RFC 7231부터 POST 요청의 응답이 캐시될 수 있는 가능성을 열어뒀음에 주의해야 합니다. POST 요청은 기본적으로 캐시되어 있지 않도록 설계되어 있습니다. 비멱등성, 데이터 안전성 등의 이유로 위험이 있는 까닭인데요. POST 캐싱을 위해서는 명시적인 설계가 필요합니다.

```
Responses to POST requests are only cacheable when they include
explicit freshness information (see Section 4.2.1 of [RFC7234]).
However, POST caching is not widely implemented. For cases where an
origin server wishes the client to be able to cache the result of a
POST in a way that can be reused by a later GET, the origin server
MAY send a 200 (OK) response containing the result and a
Content-Location header field that has the same value as the POST's
effective request URI (Section 3.1.4.2).
```

POST 요청은 명시적인 신선도(freshness) 정보를 헤더에 포함하는 경우에 캐시될 수 있지만, 널리 구현되어 있지는 않습니다. 서버가 POST 응답을 후속 GET 요청에서 재사용하고자 할 때는, 서버는 200 OK 응답을 반환하고 헤더에 POST 요청에서의 유효한 URI를 Content-Location 헤더에 포함해야 합니다.

## 멱등성을 고려한 HTTP 메서드 설계

앞서 POST나 PATCH 메서드는 멱등성을 보장하지 않음을 살펴본 바 있습니다. 멱등성을 보장하지 않는 메서드가 어떤 문제를 발생시킬 수 있을까요? 여기에서 멱등성이 fault-tolerent한 API 설계의 중심이 된다는 점을 살펴볼 필요가 있습니다.

### 비멱등적 문제점

제품 결제를 위해 POST 요청이 들어오는 상황을 가정해보겠습니다. POST 요청은 멱등적이지 않으므로, 동일한 요청이 여러 번 들어오면 여러 번 결제가 이루어질 수 있습니다.

전송된 POST 요청이 타임 아웃 된다면요? POST 요청이 리소스를 생성했는지, 요청이 서버에 도착했는지, 타임아웃이 요청 전송에서 발생했는지, 타임아웃이 응답 전송에서 발생했는지 알기 어렵습니다. 게다가 다시 요청을 전송해도 되는 걸까요?

### 멱등키를 통한 멱등성 보장

위와 같은 문제 상황에서, 멱등성을 보장하기 위한 대표적인 방법으로 멱등키(Idempotency Key)를 이용할 수 있습니다. 대표적으로 헤더를 통한 UUID 기반의 멱등키가 있습니다.

#### 1. 클라이언트 측 요청

```http
POST /api/v1/payment HTTP/1.1
Content-Type: application/json
Idempotency-Key: 123e4567-e89b-12d3-a456-426614174000

{
  "amount": 1000,
  "currency": "USD"
}
```

위와 같은 요청에 대해서, 서버는 이제 멱등키를 통해 동일한 요청인지 검증할 수 있습니다. 어떻게 검증할 수 있을까요?

#### 2. 서버 측 응답

Spring 서버는 먼저 요청 헤더에서 멱등성 키를 확인합니다. 컨트롤러에서는 `@RequestHeader("Idempotency-Key")` 어노테이션을 통해 멱등성 키를 받아, 이 키가 이미 처리된 요청인지 RDBMS 혹은 Redis와 같은 캐시 저장소를 통해 검증합니다.

만약 이미 처리된 요청이라면, 저장된 이전 응답을 그대로 반환합니다. 새로운 요청인 경우에는 실제 비즈니스 로직을 처리하고 그 결과를 저장한 후 응답합니다. 이러한 방식으로 클라이언트는 네트워크 오류 등으로 인한 재시도에도 동일한 결과를 안전하게 받을 수 있습니다.

여기에서도 예외 처리와 키 관리는 중요한데요. 예외 처리는 키 누락, 키 형식 오류, 키 만료 등 예외 상황에 대한 처리를 분리하고 보장함으로써 클라이언트에 적절한 대응을 지시합니다. 키 관리는 키 저장소를 구현하고, 사용된 키를 정리하고, 추가적으로는 모니터링과 알림으로 관리하며 키의 유효성을 안정적으로 보장합니다.

## References

- [Wikipedia - Idempotence](https://en.wikipedia.org/wiki/Idempotence)
- [RFC 7231](https://www.rfc-editor.org/rfc/rfc7231.html)
- [MDN web - POST](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/POST)
- [Spring - RequestHeader](https://docs.spring.io/spring-framework/reference/web/webflux/controller/ann-methods/requestheader.html)
