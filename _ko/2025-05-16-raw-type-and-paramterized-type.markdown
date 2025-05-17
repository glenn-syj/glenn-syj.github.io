---
title: WebClient 오류에서 시작하는 Raw Type과 ParameterizedTypeReference
lang: ko
layout: post
---

## 들어가며

- 타입 안전성을 보장하지 않는 Raw Type과 이를 해결하기 위한 ParameterizedTypeReference에 대해 알아보는 글입니다.
- 특히, Spring Boot에서 `WebClient`를 이용해 배열 형태로 response body를 받는 경우에 대하여 살펴보고, 이에서 단순히 `List.class`를 이용했을 때의 문제 상황에서 시작합니다.
- 나아가 `WebClient` 오류에 기반해 역직렬화 과정에서 제네릭과 타입 소거가 미치는 영향을 살펴봅니다.
- 최종 수정: 25/05/18

## Raw Type 이해하기

Raw Type은 제네릭이 등장하기 전인 JDK 5.0 이전 버전과의 호환성을 위해 도입되었는데요. Raw Type은 Java에서 제네릭 클래스나 인터페이스에서 타입 아규먼트가 지정되어있지 않은 경우를 의미합니다.

아래 예시를 보시면 이해가 쉬울 것입니다.

```java
// actual type argument passed
List<String> list = new ArrayList<>();
// raw type of List<T>
List rawList = new ArrayList();
```

위에서 `List<String>`은 타입 아규먼트가 `String` 으로 지정된 제네릭 클래스이고, `List`는 타입 아규먼트가 지정되어있지 않은 Raw Type입니다. 코드를 통해서 두 `List` 타입의 차이를 확인해보겠습니다.

### 코드 예시

```java
import java.io.*;
import java.util.*;

public class Main {

    static class Box {
        public int x = 10;
        public int y = 20;

        Box() {
        }
    }

    public static void main(String[] args) {

        // Parameterized Type (타입 안전)
        List<String> parameterizedList = new ArrayList<>();
        parameterizedList.add("Hello");
        // parameterizedList.add(123); // 컴파일 오류! String만 허용

        String str = parameterizedList.get(0);
        System.out.println(str);

        // Raw Type (타입 안전성 저하)
        List rawList = new ArrayList(); // 컴파일러 경고 발생 가능성
        rawList.add("World");
        rawList.add(456); // 다른 타입의 객체도 추가 가능
        rawList.add(new Box());

        Object obj1 = rawList.get(0);
        if (obj1 instanceof String) {
            String item1Casted = (String) obj1;
            System.out.println(item1Casted);
        }

        // Object obj2 = rawList.get(1);
        // String item2Casted = (String) obj2; // 여기서 ClassCastException 발생!
        // System.out.println(item2Casted);

        Box box = (Box) rawList.get(2);
        System.out.println(box.x);
    }
}
```

```
출력 결과
---

Hello
World
10

```

`List<String>`은 `String` 타입이 아닌 요소가 들어가면 컴파일 오류가 발생하지만, `List`는 타입 아규먼트가 지정되어있지 않은 Raw Type이기 때문에 컴파일 오류가 발생하지 않습니다.

또한, `List`는 타입 아규먼트가 지정되어있지 않기 때문에 다양한 타입의 객체를 저장할 수 있습니다. 위 예시에서도 `String`, `Integer`, `Box` 타입의 객체를 저장하고 있습니다.

하지만 주석 처리 해둔 코드처럼 타입 아규먼트가 지정되어있지 않은 경우에는 기본적으로 `Object`로 처리하기 때문에, 형변환을 해줘야 합니다. 이렇게 형변환을 해주지 않으면 런타임 오류가 발생할 수 있습니다.

즉, Raw Type은 타입 안전성을 보장하지 않습니다. 컴파일러 역시 Raw Type을 이용하면 unchecked 경고를 일으키기도 합니다.

## WebClient와 함께 살펴보는 Raw Type

```java
// Riot API Key는 riotKorWebClient의 defaultHeader()에 설정
public List<TftLeagueEntryResponse> getLeagueEntries(String puuid) {
    return handleApiCall(
            riotKorWebClient.get()
                    .uri("/tft/league/v1/by-puuid/{puuid}", puuid)
                    .retrieve()
                    .accept(MediaType.APPLICATION_JSON)
                    .bodyToMono(List.class),
            "소환사의 TFT 리그 정보를 찾을 수 없습니다: " + puuid
    );
}
```

위 코드는 제가 직접 문제를 겪었던, Riot 계정의 TFT 리그 엔트리 정보를 찾는 메서드입니다. 이 메서드는 `WebClient`를 이용하여 웹 서비스에 요청을 보낸 뒤 응답을 `List<TftLeagueEntryResponse>` 타입으로 받습니다.

Response Body는 아래와 같이 배열로 둘러 싸인 JSON으로 반환됩니다. 이 때, 배열 안에 들어가는 요소는 모두 `TftLeagueEntryResponse` 타입으로 변환되어야 합니다.

```json
[
  {
    "puuid": "puuid",
    "leagueId": "leagueId",
    "queueType": "RANKED_TFT",
    "tier": "PLATINUM",
    "rank": "II",
    "summonerId": "summonerId",
    "leaguePoints": 45,
    "wins": 84,
    "losses": 95,
    "veteran": false,
    "inactive": false,
    "freshBlood": false,
    "hotStreak": false
  }
]
```

위 테스트 코드에서 중요한 것은 바로 `bodyToMono(List.class)` 부분입니다. 이 부분은 제네릭 타입을 이용하여 타입 안전성을 보장하는 것이 아니라, Raw Type을 이용하기에 타입 안전성이 보장되지 않음을 예측할 수 있습니다. 응답 역시도 `List` 타입으로 받아오게 되듯이 말입니다.

그렇다면 이제 테스트 코드를 통해 이를 확인해보겠습니다.

### 테스트 코드를 통한 확인

```java
@Test
void TFT_랭크_엔트리_실제API호출_테스트() throws InterruptedException {
    // given
    String gameName = "승상싱";
    String tagLine = "KR1";

    String puuid = riotAccountClient.getAccountInfo(gameName, tagLine).puuid();

    // when
    List<TftLeagueEntryResponse> response = tftApiClient.getLeagueEntries(puuid);

    log.debug("puuid returned: {}", puuid);
    log.debug(response.toString());

    /*
        에러 발생!
    */
    TftLeagueEntryResponse entry0 = (TftLeagueEntryResponse) response.get(0);

    // then
    assertNotNull(response);
    assertThat(Objects.equals(entry0.puuid(), puuid));
}
```

가장 처음 살펴보았던, `List rawList`와 같은 방식이라면 분명히 `entry0`은 `TftLeagueEntryResponse` 타입으로 변환되어야 합니다. 하지만 위 테스트 코드를 실행하면 `에러 발생!` 부분에서 `ClassCastException`이 발생하는 것을 확인할 수 있습니다.

```
java.lang.ClassCastException: class java.util.LinkedHashMap cannot be cast to class com.glennsyj.rivals.api.tft.model.TftLeagueEntryResponse (이하 생략)
```

이러한 에러는 왜 발생할까요? 해결 방안을 살펴보기 전에, 우선 `WebClient`가 어떻게 위 Riot API 경로의 응답을 받아들이고 `ClassCastException`을 띄우는지 그 과정을 살펴보겠습니다.

### WebClient 오류 과정 살펴보기

`WebClient`는 Spring WebFlux에서 제공하는 논블로킹, 리액티브 HTTP 클라이언트입니다. 동기 및 블로킹 방식의 RestTemplate의 대체로 사용가능하며, 추후 `RestClient`가 등장하기는 했지만, 추후 서비스가 복잡해질 것을 고려하여 `WebClient`를 택했습니다.

여기에서 `bodyToMono()` 부분은 전체 응답은 필요 없이 응답 본문만을 가져오기 위해 사용됩니다. 이때, HTTP 요청이 성공적으로 수행되고 응답이 도착하면 `WebClient`는 응답 본문을 가져오고, Java 객체로 역직렬화하려고 시도합니다. 이후 변환된 Java 객체를 `Mono` 타입으로 감싸서 반환합니다. (직관적으로 `bodyToMono()`라는 메서드 명을 떠올리면 쉽습니다.)

이 역직렬화 과정에서 문제가 발생합니다. 이전 코드에서 `bodyToMono(List.class)`는 `WebClient`에게 응답 본문을 Raw Type인 `List` 타입으로 역직렬화하도록 지시하는데요. 여기에서 기본 구현체인 `DefaultWebClient`는 내부적으로 Jackson과 같은 JSON 라이브러리를 사용해 JSON 응답을 Java 객체로 변환하고자 합니다.

```json
[
  {
    "puuid": "puuid",
    "leagueId": "leagueId",
    "queueType": "RANKED_TFT",
    "tier": "PLATINUM",
    "rank": "II",
    "summonerId": "summonerId",
    "leaguePoints": 45,
    ...
  }
]
```

하지만 Jackson은 이 응답 본문에 대해서 `List.class`라는 정보만 가지고 있기 때문에, JSON 배열 내의 객체를 특정 구현체로 반환할 수 없습니다. 그래서 Jackson은 POJO 대신 각 JSON 객체를 키-값 쌍을 가지는 `LinkedHashMap`으로 변환을 시도합니다.

```java
/*
    에러 발생!
    LinkedHashMap은 TftLeagueEntryResponse로 형 변환 불가능
*/
TftLeagueEntryResponse entry0 = (TftLeagueEntryResponse) response.get(0);
```

즉, 기존의 의도와 다르게 `bodyToMono(List.class)`는 실제로는 `Mono<List<LinkedHashMap>>`의 형식으로 반환을 시도하게 됩니다. 이러한 이유로 테스트 코드에서는 `ClassCastException`이 발생하게 됩니다.

특히, Java에서 제네릭은 컴파일 시점에만 타입 정보를 강제하고 런타임에는 타입 정보가 소거되므로 아무리 `List<TftLeagueEntryResponse>`라는 제네릭 클래스를 지정할 수 없습니다. 곧, `bodyToMono(class<T> clazz)` 시그니처에서 `List<TftLeagueEntryResponse>`라는 완전한 타입 정보를 가진 객체를 전달할 수 없다는 뜻입니다.

그렇다면 이대로 포기해야 할까요? 아닙니다. 이러한 문제를 해결하기 위해서는 바로 `ParameterizedTypeReference`를 이용하면 됩니다.

## ParameterizedTypeReference를 이용한 문제 해결

Spring Framework에서는 런타임에서도 제네릭 타입 정보를 보존하고 조회할 수 있도록 `ParameterizedTypeReference`를 제공합니다. `ParameterizedTypeReference`는 익명 내부 클래스를 활용해서 제네릭 타입 정보를 보존하고 이용할 수 있도록 합니다.

```java
public abstract class ParameterizedTypeReference<T> {

	private final Type type;

	protected ParameterizedTypeReference() {

        // 익명 서브 클래스의 제네릭 슈퍼클래스 타입을 가져옴
		Class<?> parameterizedTypeReferenceSubclass = findParameterizedTypeReferenceSubclass(getClass());
		Type type = parameterizedTypeReferenceSubclass.getGenericSuperclass();

        // 파라미터화된 타입인지 확인
		Assert.isInstanceOf(ParameterizedType.class, type, "Type must be a parameterized type");

        // 실제 타입 인자를 가져옴
		ParameterizedType parameterizedType = (ParameterizedType) type;
		Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
		Assert.isTrue(actualTypeArguments.length == 1, "Number of type arguments must be 1");
		this.type = actualTypeArguments[0];
	}

	private ParameterizedTypeReference(Type type) {
		this.type = type;
	}

	// getter, equals, hashCode, toString 생략

	// forType, findParameterizedTypeReferenceSubclass 메서드 생략

}
```

익명 내부 클래스를 생성할 때, 컴파일러는 제네릭 타입 정보를 클래스 파일에 저장합니다. `findParameterizedTypeReferenceSubclass()` 메서드는 상속 계층 구조를 따라 `ParameterizedTypeReference` 클래스의 서브 클래스를 찾는 재귀적인 메서드입니다.

```java
private static Class<?> findParameterizedTypeReferenceSubclass(Class<?> child) {

    // 현재 클래스의 부모 클래스 가져오기
    Class<?> parent = child.getSuperclass();

    // Object 클래스까지 도달했다면 `ParameterizedTypeReference` 클래스가 아닌 것이므로 예외 발생
    if (Object.class == parent) {
        throw new IllegalStateException("Expected ParameterizedTypeReference superclass");
    }
    // 직접적인 부모가 `ParameterizedTypeReference` 클래스라면 현재 클래스 반환
    else if (ParameterizedTypeReference.class == parent) {
        return child;
    }
    // 그 외의 경우 부모 클래스에 대해 재귀적으로 검사
    else {
        return findParameterizedTypeReferenceSubclass(parent);
    }
}
```

### 중첩 구조 타입 처리

이러한 구조는 `ParameterizedTypeReference<Map<String, List<String>>>`과 같이 복잡하게 중첩된 제네릭 타입의 경우도 Refelction API를 이용해 전체 타입 정보를 보존하도록 하는데요.

```java
// 익명 내부 클래스로 ParameterizedTypeReference 생성
ParameterizedTypeReference<Map<String, List<String>>> typeRef = new ParameterizedTypeReference<Map<String, List<String>>>() {};
```

위와 같이 `Map<String, List<String>>` 타입에 대한 `ParameterizedTypeReference` 객체인 `typeRef`를 생성했습니다. 이 `typeRef` 객체는 내부적으로 `Map<String, List<String>>` 전체에 대한 `Type` 정보를 `type` 필드에 저장하고 있습니다.

이 저장된 타입 정보는 `typeRef.getType()` 메서드를 통해 가져올 수 있습니다. 이렇게 가져온 `Type` 객체(여기서는 `actualTypeFromRef`라고 명명하겠습니다)를 사용하여 중첩된 제네릭 구조를 분석할 수 있습니다. 아래 코드는 그 내부 구조를 분석하는 과정입니다.

```java
Type actualTypeFromRef = typeRef.getType();

// 최상위 레벨: type이 Map<String, List<String>>
ParameterizedType mapType = (ParameterizedType) actualTypeFromRef;
/*
    Map의 타입 파라미터를 배열로 가져옴
    mapTypeArgs[0] = String.class (키 타입)
    mapTypeArgs[1] = ParameterizedType(List<String>) (값 타입)
*/
Type[] mapTypeArgs = mapType.getActualTypeArguments();

// 중첩 레벨: type이 List<String>
if (mapTypeArgs[1] instanceof ParameterizedType) {
    ParameterizedType listType = (ParameterizedType) mapTypeArgs[1];

    /*
        List의 타입 파라미터를 배열로 가져옴
        listTypeArgs[0] = String.class
    */
    Type[] listTypeArgs = listType.getActualTypeArguments();
}
```

이처럼 `ParameterizedType` 인터페이스의 `getActualTypeArguments()` 메서드를 재귀적으로 활용하면, 아무리 복잡하게 중첩된 제네릭 타입이라도 그 계층 구조를 따라 각 레벨의 실제 타입 인자들을 정확히 파악할 수 있습니다. `ParameterizedTypeReference`는 이러한 방식으로 중첩된 상태에서도 전체 제네릭 타입 정보를 보존합니다.

### 문제 해결

이제 `ParameterizedTypeReference`를 이용해 문제를 해결할 수 있습니다.

```java
public List<TftLeagueEntryResponse> getLeagueEntries(String puuid) {
    return handleApiCall(
            riotKorWebClient.get()
                    .uri("/tft/league/v1/by-puuid/{puuid}", puuid)
                    .accept(MediaType.APPLICATION_JSON)
                    .retrieve()
                    .bodyToMono(new ParameterizedTypeReference<List<TftLeagueEntryResponse>>() {
                    }),
            "소환사의 TFT 리그 정보를 찾을 수 없습니다: " + puuid
    );
}
```

## 나가며

지금까지 Java의 Raw Type이 타입 안전성을 어떻게 저해할 수 있는지, 특히 `WebClient`와 같은 HTTP 클라이언트로 제네릭 컬렉션 타입의 응답을 처리할 때 `List.class`와 같이 Raw Type을 사용하면 어떤 문제가 발생하는지 살펴보았습니다.

문제를 해결하기 위해 Spring Framework가 제공하는 `ParameterizedTypeReference`는 익명 내부 클래스의 특성을 활용하여 런타임에도 완전한 제네릭 타입 정보를 보존할 수 있음 역시도 알게 되었습니다.

특히, `ParameterizedTypeReference`를 사용함으로써 `WebClient`는 내부적으로 Jackson과 같은 라이브러리를 통해 `List<TftLeagueEntryResponse>`와 같이 중첩된 제네릭 타입까지도 정확하게 역직렬화할 수 있게 됩니다.

Raw Type 때문에 JSON 객체가 의도한 DTO가 아닌 `LinkedHashMap`으로 변환되어 `ClassCastException`이 발생했다는 점도 인상 깊었습니다. 이는 역직렬화 과정에서도 제네릭과 타입 소거가 미치는 영향이 크다는 것을 알게되는 계기가 되었습니다.

결국, Raw Type의 사용은 가급적 지양하고, 제네릭 타입을 다룰 때는 `ParameterizedTypeReference`와 같이 타입 정보를 명확히 전달할 수 있는 방법을 이용할 필요가 있겠습니다.

비록 `Mono`란 무엇인가에 대한 설명, 왜 `ParameterizedTypeReference`를 익명 클래스로 생성하는 가에 대한 바이트코드 분석은 다루지 못했지만, 이번 글은 여기에서 마치도록 하겠습니다.

## References

[https://docs.oracle.com/javase/tutorial/java/generics/rawTypes.html](https://docs.oracle.com/javase/tutorial/java/generics/rawTypes.html)

[https://github.com/spring-projects/spring-framework/blob/main/spring-core/src/main/java/org/springframework/core/ParameterizedTypeReference.java](https://github.com/spring-projects/spring-framework/blob/main/spring-core/src/main/java/org/springframework/core/ParameterizedTypeReference.java)

[https://developer.riotgames.com/apis](https://developer.riotgames.com/apis)

[https://github.com/glenn-syj/rivals](https://github.com/glenn-syj/rivals)
