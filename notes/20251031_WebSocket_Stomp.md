# STOMP.js 선택 이유 - Q&A 참고 자료

## 핵심 답변 (30초 버전)
"STOMP.js를 선택한 이유는 Spring Boot와의 통합이 쉽고, 
pub/sub 패턴으로 메시지 라우팅이 명확하기 때문입니다.
단순 WebSocket보다 구조화된 메시징 프로토콜이 필요했고, 
최신 브라우저 환경에서는 네이티브 WebSocket만으로도 충분히 안정적이라고 판단했습니다."

---

## 상세 답변

### 1. 왜 순수 WebSocket이 아닌 STOMP를 사용했나요?

**순수 WebSocket의 한계:**
- 단순히 양방향 통신만 제공
- 메시지 형식, 라우팅 규칙이 없음
- 직접 프로토콜을 설계해야 함

**STOMP의 장점:**
- 표준화된 메시징 프로토콜
- 목적지 기반 라우팅 (`/topic`, `/queue`)
- 헤더, 바디 구조화
- Spring Boot의 `@MessageMapping`과 자연스럽게 연동

```javascript
// 순수 WebSocket - 모든 것을 직접 구현해야 함
socket.send(JSON.stringify({type: 'chat', room: 'A', message: 'hello'}));

// STOMP - 명확한 구조
stompClient.send('/app/chat/room/A', {}, JSON.stringify({message: 'hello'}));
```

---

### 2. SockJS 없이 네이티브 WebSocket만 사용한 이유는?

**핵심 판단:**
- 프로젝트 타겟 사용자가 최신 브라우저 사용 (Chrome, Edge, Safari, Firefox 최신 버전)
- 모든 주요 브라우저가 WebSocket을 지원 (IE 11도 지원)
- 불필요한 라이브러리 추가를 피하고 싶었음

**최신 브라우저 WebSocket 지원 현황:**
- Chrome: 2011년부터 지원 (버전 16+)
- Firefox: 2012년부터 지원 (버전 11+)
- Safari: 2013년부터 지원 (버전 7+)
- Edge: 모든 버전 지원

**SockJS를 쓰지 않은 이유:**

1. **타겟 환경이 명확함**
    - 사내 시스템 / 대학 프로젝트 / 특정 고객층
    - 최신 브라우저 사용 보장

2. **불필요한 복잡도 제거**
   ```javascript
   // SockJS 사용 시
   const socket = new SockJS('/ws');  // 추가 라이브러리
   const stompClient = Stomp.over(socket);
   
   // 우리 방식 (네이티브 WebSocket)
   const socket = new WebSocket('ws://localhost:8080/ws');
   const stompClient = Stomp.over(socket);  // 간단하고 직관적
   ```

3. **번들 크기 최적화**
    - SockJS 라이브러리: ~40KB (gzipped)
    - 네이티브 WebSocket: 0KB (브라우저 내장)

4. **성능 우선**
    - 폴백 메커니즘 불필요 = 오버헤드 제거
    - WebSocket 직접 사용이 가장 빠름

**만약 WebSocket 연결이 실패한다면?**
- 에러 핸들링으로 사용자에게 안내
- 브라우저 업데이트 권장
- 실제로 이런 경우는 거의 없음 (현대 브라우저는 모두 지원)

```javascript
const socket = new WebSocket('ws://localhost:8080/ws');

socket.onerror = (error) => {
    console.error('WebSocket 연결 실패:', error);
    alert('실시간 기능을 사용하려면 브라우저를 업데이트해주세요.');
};
```

---

### 3. STOMP.js만 사용한 것이 위험하지 않나요?

**전혀 위험하지 않습니다. 오히려 장점이 많습니다:**

**1. 단순성 (Simplicity)**
```javascript
// 우리 방식 - 명확하고 직관적
const socket = new WebSocket('ws://localhost:8080/ws');
const stompClient = Stomp.over(socket);

// vs SockJS 추가 시 - 한 단계 더 복잡
const socket = new SockJS('/ws');
const stompClient = Stomp.over(socket);
```

**2. 번들 크기**
- STOMP.js만: ~15KB
- STOMP.js + SockJS: ~55KB
- **차이: 40KB 절약 (약 73% 감소)**

**3. 성능**
- 네이티브 WebSocket은 브라우저에 최적화되어 있음
- 폴백 메커니즘 체크 과정이 없어서 연결 속도가 빠름
- 메모리 사용량도 적음

**4. 디버깅**
- 브라우저 DevTools에서 WebSocket 탭으로 바로 확인 가능
- 추가 레이어가 없어서 문제 추적이 쉬움

**5. 실제 통계**
- [Can I Use](https://caniuse.com/websockets) 기준 WebSocket 지원률: **98.67%**
- 지원하지 않는 브라우저: IE 9 이하, Opera Mini
- 우리 프로젝트 타겟 사용자는 이런 브라우저를 사용하지 않음

**언제 SockJS가 필요한가?**
- 매우 오래된 브라우저 지원이 필수인 경우 (IE 9 이하)
- 특수한 네트워크 환경 (WebSocket을 차단하는 프록시)
- 글로벌 서비스로 다양한 환경 대응 필요

**우리는 해당사항 없음 → SockJS 불필요**

---

### 4. 다른 대안은 없었나요?

| 기술 | 장점 | 단점 | 선택하지 않은 이유 |
|------|------|------|-------------------|
| **Socket.IO** | 자동 재연결, 방 관리 | 무겁고, Spring과 통합 복잡 | 백엔드가 Spring Boot |
| **순수 WebSocket** | 가볍고 빠름 | 프로토콜 직접 구현 필요 | 개발 시간 부족 |
| **Server-Sent Events** | 단순함 | 단방향 통신만 가능 | 양방향 채팅 필요 |
| **GraphQL Subscriptions** | 현대적 | 러닝 커브, 오버스펙 | 프로젝트 범위 초과 |

---

---

### 5. Spring Boot와의 통합 예시

**백엔드 (Spring Boot):**
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // SockJS 없이 순수 WebSocket 엔드포인트만 등록
        registry.addEndpoint("/ws")
                .setAllowedOrigins("*");
    }
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
    }
}

@Controller
public class ChatController {
    
    @MessageMapping("/chat/{roomId}")
    @SendTo("/topic/chat/{roomId}")
    public ChatMessage sendMessage(@Payload ChatMessage message) {
        return message;
    }
}
```

**프론트엔드 (STOMP.js + 네이티브 WebSocket):**
```javascript
// 네이티브 WebSocket 사용
const socket = new WebSocket('ws://localhost:8080/ws');
const stompClient = Stomp.over(socket);

stompClient.connect({}, function() {
    // 구독
    stompClient.subscribe('/topic/chat/room1', function(message) {
        console.log(JSON.parse(message.body));
    });
    
    // 메시지 전송
    stompClient.send('/app/chat/room1', {}, 
        JSON.stringify({user: 'John', text: 'Hello'})
    );
});

// 에러 핸들링
socket.onerror = (error) => {
    console.error('WebSocket 에러:', error);
};

socket.onclose = () => {
    console.log('WebSocket 연결 종료');
};
```

**핵심:**
- SockJS 없이도 STOMP 프로토콜은 정상 작동
- 백엔드의 `@MessageMapping`과 프론트의 `send()` 경로가 1:1 매칭
- 더 가볍고 직관적인 코드

---

### 6. 실제 프로젝트에서의 이점

**우리 프로젝트에 적용한 부분:**
- 실시간 채팅방 구독/해제
- 사용자별 개인 알림 (`/queue/user/{userId}`)
- 브로드캐스트 메시지 (`/topic/announcements`)

**코드 가독성:**
```javascript
// 명확한 의도 표현
stompClient.subscribe('/topic/chat/room123', handleMessage);
stompClient.subscribe('/queue/notifications', handleNotification);
stompClient.send('/app/chat/send', {}, messageData);
```

---

### 7. 성능은 괜찮나요?

**벤치마크 기준:**
- 동시 연결: 10,000+ 가능
- 메시지 레이턴시: <100ms (로컬 네트워크)
- 오버헤드: STOMP 헤더로 인한 약간의 추가 (무시할 수준)

**우리 프로젝트 규모:**
- 예상 동시 접속자: ~500명
- 메시지 빈도: 초당 ~100개
- **결론:** 충분히 적합

---

### 8. 단점이나 한계는?

**솔직하게 말하면:**

1. **학습 곡선**
    - STOMP 프로토콜 이해 필요
    - 하지만 문서가 잘 되어 있음

2. **오버헤드**
    - 순수 WebSocket보다 약간 무거움
    - 실시간 게임 같은 초저지연이 필요한 경우는 부적합

3. **브라우저 호환성 리스크 (이론적)**
    - 매우 오래된 환경에서는 작동하지 않을 수 있음
    - 하지만 현실적으로 거의 발생하지 않음 (98.67% 지원률)

4. **네트워크 문제 대응**
    - WebSocket이 차단되는 특수한 환경에서는 대안이 없음
    - 하지만 일반적인 인터넷 환경에서는 문제없음

**우리 프로젝트에는 이러한 단점이 큰 문제가 되지 않음**
- 타겟 사용자의 브라우저 환경 파악됨
- 특수한 네트워크 환경 아님
- 채팅 서비스이므로 초저지연 불필요

---

## 예상 추가 질문

### Q: "SockJS 없이 STOMP만 써도 문제없나요?"
A: "네, 전혀 문제없습니다. 최신 브라우저는 모두 WebSocket을 네이티브로 지원하기 때문에 SockJS의 폴백 메커니즘이 필요하지 않습니다. 오히려 불필요한 라이브러리를 제거해서 번들 크기를 줄이고 성능을 개선할 수 있었습니다."

### Q: "만약 WebSocket 연결이 실패하면 어떻게 하나요?"
A: "현대 브라우저에서는 거의 발생하지 않지만, 연결 실패 시 에러 핸들링으로 사용자에게 알림을 보내도록 구현했습니다. 우리 프로젝트의 타겟 사용자는 최신 브라우저를 사용하기 때문에 이런 경우는 극히 드뭅니다."

### Q: "Socket.IO는 왜 안 썼나요?"
A: "Socket.IO는 기능이 풍부하지만, 우리 백엔드가 Spring Boot라서 통합이 복잡합니다. STOMP는 Spring의 네이티브 지원이 있어서 더 자연스럽게 연동됩니다. 또한 Socket.IO는 자체 프로토콜을 사용하지만, STOMP는 표준 프로토콜이라 유지보수가 더 쉽습니다."

### Q: "WebSocket만 쓰면 더 가볍지 않나요?"
A: "맞습니다. 하지만 메시지 라우팅, 구독 관리를 직접 구현해야 해서 개발 시간이 더 걸립니다. STOMP를 사용하면 이런 기능들이 이미 구현되어 있어서 개발 생산성이 훨씬 높습니다."

### Q: "IE 11 같은 오래된 브라우저는 지원 안 하나요?"
A: "IE 11도 WebSocket을 지원합니다(IE 10+). 만약 정말 오래된 브라우저(IE 9 이하)를 지원해야 한다면 SockJS를 추가하는 것도 고려할 수 있지만, 우리 프로젝트에서는 그럴 필요가 없었습니다."

### Q: "프로덕션 환경에서 안정성은 괜찮나요?"
A: "네, WebSocket은 이미 성숙한 기술입니다. 2011년부터 지원되어 왔고, 현재는 모든 주요 브라우저와 서버에서 안정적으로 동작합니다. 실제로 많은 대규모 서비스들도 SockJS 없이 네이티브 WebSocket을 사용합니다."

### Q: "나중에 SockJS를 추가하기 어렵지 않나요?"
A: "전혀 어렵지 않습니다. 프론트엔드에서 `new WebSocket()` 부분만 `new SockJS()`로 바꾸고, 백엔드에서 `.withSockJS()`만 추가하면 됩니다. 필요하다면 언제든 추가할 수 있습니다."

---

## 마무리 멘트

"STOMP.js는 실시간 메시징에 특화된 프로토콜로, Spring Boot와의 호환성이 뛰어나고 구조화된 메시지 처리가 가능합니다. SockJS 없이 네이티브 WebSocket을 사용한 것은 최신 브라우저 환경에서 불필요한 복잡도를 제거하고 성능을 최적화하기 위한 선택이었습니다. 우리 프로젝트의 요구사항과 타겟 환경을 고려했을 때 가장 적합한 구조입니다."

---

## 참고 자료

- STOMP 공식 문서: https://stomp.github.io/
- Spring WebSocket 문서: https://docs.spring.io/spring-framework/reference/web/websocket.html
- SockJS GitHub: https://github.com/sockjs/sockjs-client

---

**작성일:** 2025-10-31  
**용도:** 발표 Q&A 대비용 참고 자료