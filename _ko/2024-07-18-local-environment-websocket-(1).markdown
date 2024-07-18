---
title: 로컬 환경에서의 웹소켓 (1) HTTP
lang: ko
layout: post
---

## 들어가며
### Disclaimer
- Last Update: 2024/07/18

- 실습을 통한 WebSocket 통신 구성과 메시지 전송 로직 이해를 다루기 위한 프로젝트에서 비롯된 글입니다.
- [https://github.com/glenn-syj/my-own-websocket](https://github.com/glenn-syj/my-own-websocket)에서 최신 레포지토리를 확인할 수 있습니다.
- 비록 HTTPS 기반 코드이지만, 해당 레포지토리의 `f12de17` 커밋에 대해 cherry-pick 할 시 동일한 UI를 이용할 수 있습니다.

<br/>

## HTTP 환경에서의 웹소켓

### HTTP 환경

localhost로 이용되는 로컬 환경에서는 HTTP 환경을 따로 설정할 필요가 없습니다. 클라이언트와 서버 모두 `http://` 로 시작하는 주소를 이용해서 통신을 진행합니다. 간단하게 Spring Framework에서 제공하는 WebSocket 및 STOMP 가이드를 따라, 웹소켓 통신을 진행해보겠습니다. 클라이언트로는 Vue 프레임워크를, 서버로는 Spring Boot를 이용하고 있습니다. 코드와 함께, WebSocket과 STOMP를 이용한 통신 코드 중 일부를 살펴보겠습니다.

### 핵심 코드

**HelloMessage.java**

```java
public class HelloMessage {

  private String name;

  public HelloMessage() {
  }

  public HelloMessage(String name) {
    this.name = name;
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }
}
```

HelloMessage는 클라이언트에서 서버로 보내는 메시지의 내용을 나타내는 POJO 클래스입니다.

**Greeting.java**

```java
public class Greeting {

  private String content;

  public Greeting() {
  }

  public Greeting(String content) {
    this.content = content;
  }

  public String getContent() {
    return content;
  }

}
```

Greeting은 서버에서 클라이언트로 보내는 메시지의 내용을 나타내는 POJO 클래스입니다.

**WebSocketConfig.java**

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/gs-guide-websocket")
                .setAllowedOrigins("http://localhost:5173")
                .withSockJS();
    }

}
```

WebSocketConfig은 WebSocket 통신 환경을 설정하는 파일입니다. `configureMessageBroker` 메서드는 클라이언트가 `/app` 접두사를 이용한 경로로 메시지를 서보로 전송하도록 합니다. 또한, 웹소켓 메시지에 대해 `/topic` 접두사를 이용해 적절한 목적지로 보내도록 설정합니다. `registerStompEndpoints` 메서드는 클라이언트 측에서 핸드쉐이크를 처리하는 URL 엔드포인트를 지정합니다.

**WebConfig.java**

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("https://localhost:5173") // 허용할 출처
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true);
    }
}
```

WebConfig에서는 `addCorsMappings` 메서드를 통해 CORS(교차 출처 리소스 공유, Cross Origin Resource Sharing) 정책에서 허용할 출처를 설정합니다. 현재 프로젝트는 웹소켓에 목적이 있으므로, 위 코드 정도로 작성해줍니다.

**GreetingController.java**

```java
@Controller
public class GreetingController {

    @MessageMapping("/hello")
    @SendTo("/topic/greetings")
    public Greeting greeting(HelloMessage message) throws Exception {
        Thread.sleep(1000); // simulated delay
        return new Greeting("Hello, " + HtmlUtils.htmlEscape(message.getName()) + "!");
    }

}
```

GreetingController는 `@MessageMapping("/hello")` 어노테이션을 통해 `/app/hello` 경로에 대해서 메시지를 매핑합니다. 여기에서 `/app` 은 이전에 살펴보았던 WebSocketConfig 파일에서 설정했으므로 자동으로 추가됩니다. 

`@SendTo("/topic/greetings")` 어노테이션은 해당 경로의 주제를 구독하고 있는 클라이언트에게 전송 메서드가 반환하는 Greeting 메시지를 전송합니다. STOMP와 pub/sub 모델에 대해서는 다른 글에서 설명하겠습니다.

**HomeView.vue**

```javascript
import { ref } from 'vue';
import SockJS from 'sockjs-client/dist/sockjs.min.js'
import { Client as StompJsClient } from '@stomp/stompjs';

const name = ref('');
const greetings = ref([]);
const connected = ref(false);

let stompClient;
let socket;

const connect = () => {
  if (!socket) {
    socket = new SockJS('https://localhost:443/gs-guide-websocket'); // SockJS 객체를 한 번만 생성
  }
  stompClient = new StompJsClient({
    webSocketFactory: () => socket,
    connectHeaders: {},
    debug: function (str) {
      console.log(str);
    },
    reconnectDelay: 5000,
    heartbeatIncoming: 4000,
    heartbeatOutgoing: 4000,
  });
  stompClient.onConnect = (frame) => {
    setConnected(true);
    console.log('Connected: ' + frame);
    stompClient.subscribe('/topic/greetings', (greeting) => {
      showGreeting(JSON.parse(greeting.body).content);
    });
  };
  stompClient.onWebSocketError = (error) => {
    console.error('Error with websocket', error);
  };
  stompClient.onStompError = (frame) => {
    console.error('Broker reported error: ' + frame.headers['message']);
    console.error('Additional details: ' + frame.body);
  };
  stompClient.activate();
};
const disconnect = () => {
  if (stompClient) {
    stompClient.deactivate();
  }
  setConnected(false);
  console.log('Disconnected');
};
const sendName = () => {
  if (stompClient && stompClient.connected) {
    stompClient.publish({
      destination: '/app/hello',
      body: JSON.stringify({ name: name.value }),
    });
  }
};
const setConnected = (connect) => {
  connected.value = connect;
};
const showGreeting = (message) => {
  greetings.value.push(message);
};
```

위 코드는 HomeView의 `<script setup>` 코드입니다. HomeView 라우터를 통해 `/` 경로로 바로 매핑되어 보여지는 Vue 파일인데요. greetings를 통해서 서버에서 전달된 메시지를 받아 저장해 보여줍니다.

`connect` 와 `disconnect` 는 WebSocket 연결 설정을 관리합니다. `connect` 에서는 SockJS 인스턴스를 생성하고, STOMP 클라이언트가 생성됩니다. STOMP 클라이언트는 활성화(activate) 또는 비활성화(deactivate)로 연결을 제어할 수 있습니다. `connect` 코드에 관한 자세한 내용은 다른 글에서 다루겠습니다.

`sendName`과 `showGreeting`은 각각 클라이언트에서 서버로 메시지를 전송하는 함수, 서버에서 받은 메시지를 처리하는 함수입니다. `sendName`에서 `stompClient`가 발행하는 메시지의 destination은 앞서 살펴본 `/app/hello`이므로 서버 측에서 수신할 수 있습니다. 이 메시지는 `/topic/greetings/` 토픽에서 발행된 메시지입니다. 

### 전송 준비 완료

이제, 웹소켓 통신을 진행하면 됩니다.

![연결 화면](https://github.com/user-attachments/assets/9844dc66-a0b6-4276-bb3a-be85a769b5fc)

connect 버튼을 누르면 웹소켓 통신이 연결됩니다.

![연결 및 구독](https://github.com/user-attachments/assets/18e5d04f-cff7-40a5-936b-37d465d970d8)

위 Vue 코드에서 stomClient의 생성과 동시에 토픽을 구독하므로 이와 관련된 메시지도 전송 받습니다. 

![이름 전송 화면](https://github.com/user-attachments/assets/423e2e87-ffe2-4561-8b8b-298f8e5b163d)

입력폼에 name을 입력하고 전송하면, 해당 메시지가 서버 측으로 전송됩니다.

![이름 전송 메시지 ](https://github.com/user-attachments/assets/aa6707c7-8f10-4663-8204-b36c8fd8a5bf)

이름 전송에 대한 메시지입니다. STOMP 프로토콜에서 SEND와 MESSAGE 커맨드를 이용하는 것을 알 수 있습니다.

![연결 해제 메시지](https://github.com/user-attachments/assets/dbc6bb0d-adbc-4a10-a365-f4acfbb55195)

연결이 해제될 때에도 RECEIPT 커맨드를 이용해 시그널을 주고 받음을 알 수 있습니다.

### Reference

https://spring.io/guides/gs/messaging-stomp-websocket

학습을 위한 코드 작성에서 chatGPT를 활용했습니다.

<br/>
