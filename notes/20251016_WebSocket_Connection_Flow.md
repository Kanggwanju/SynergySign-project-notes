## 📡 WebSocket 연결 및 메시지 처리 흐름

### **1단계: 초기 설정 (WebSocketConfig)**

클라이언트가 WebSocket 연결을 시도하면:

```
클라이언트 → http://localhost:5173 에서 /ws 엔드포인트로 연결 시도
```

- **엔드포인트**: `/ws`로 STOMP 연결 설정
- **메시지 브로커**: `/topic`, `/queue`로 시작하는 구독 경로 활성화
- **애플리케이션 목적지**: `/app`으로 시작하는 메시지 전송 경로
- **핸드셰이크 인터셉터**: `CookieAuthHandshakeInterceptor`가 쿠키 기반 인증 처리

---

### **2단계: CONNECT - 연결 수립**

클라이언트가 STOMP CONNECT 프레임을 보내면:

#### **보안 검증 (WebSocketSecurityConfig)**
- 인증된 사용자인지 확인
- 인증되지 않은 요청은 즉시 거부

#### **단일 세션 정책 적용 (SingleSessionChannelInterceptor)**
```
handleConnect(accessor) 실행:
1. 인증 정보 확인 (accessor.getUser())
2. userId와 sessionId 추출
3. hasOtherActiveSession() 확인
   - 다른 활성 세션이 있으면 → MessagingException 발생 (중복 접속 차단)
   - 없으면 → userSessionRegistry.bind(userId, sessionId) 실행
```

**결과**: 사용자당 하나의 활성 WebSocket 세션만 유지 (단일 탭 강제)

---

### **3단계: SUBSCRIBE - 구독 요청**

클라이언트가 `/topic/room/{roomId}/participant` 같은 경로를 구독하면:

#### **보안 검증**
- `/topic/**`, `/queue/**` 구독은 인증 필요
- 인증되지 않은 구독은 거부

#### **활성 세션 검증 (SingleSessionChannelInterceptor)**
```
handleSubscribe(accessor) 실행:
1. userId와 sessionId 추출
2. isActive(userId, sessionId) 확인
   - 현재 등록된 활성 세션이 아니면 → 차단
   - 활성 세션이면 → 구독 허용
```

**목적**: 오래된 탭이나 비활성 세션에서의 구독 요청을 차단

---

### **4단계: SEND - 메시지 전송**

클라이언트가 `/app/**` 경로로 메시지를 보내면:

#### **보안 검증**
- `/app/**` 전송은 인증 필요

#### **인터셉터 통과**
- `SingleSessionChannelInterceptor`는 SEND 명령에 대해서는 특별한 검증 없이 통과
- 애플리케이션 로직이 메시지 처리

---

### **5단계: DISCONNECT - 연결 종료**

클라이언트 연결이 끊기면:

#### **인터셉터에서 로깅 (SingleSessionChannelInterceptor)**
```
handleDisconnect(accessor):
- 로그만 기록
- 실제 언바인드는 하지 않음 (이벤트 리스너에서 처리)
```

#### **이벤트 리스너 처리 (WebSocketEventsListener)**
```
onDisconnect(event) 실행:
1. userId와 sessionId 추출
2. isActive() 확인
   - 비활성 세션이면 → 무시 (스테일 세션)
   - 활성 세션이면 → 계속 진행
3. leaveCurrentRoomByUser(userId) 호출
   - 참여 중인 게임방에서 자동 퇴장 처리
4. 방의 다른 참여자들에게 알림 전송
5. finally 블록에서 unbind(userId, sessionId)
   - 예외 발생 여부와 관계없이 세션 매핑 정리
```

---

## 🔒 핵심 설계 포인트

### **단일 탭 강제 정책**
```
UserSessionRegistry에서 userId → sessionId 매핑 관리
- 새 탭 열기 → 이전 세션이 있으면 CONNECT 차단
- 브라우저 새로고침 → 새 sessionId로 기존 매핑 덮어쓰기
```

### **스테일 세션 방지**
```
바인딩: 인터셉터 (CONNECT 시점)
검증: 인터셉터 (SUBSCRIBE 시점)
정리: 이벤트 리스너 (finally 블록)
```

이렇게 분리하여 네트워크 지연이나 예외 상황에서도 오래된 매핑이 남지 않도록 보장합니다.

### **메시지 흐름 요약**
```
클라이언트 → /ws (핸드셰이크) 
→ CONNECT (보안 + 단일세션 검증 + 바인딩)
→ SUBSCRIBE (보안 + 활성세션 검증)
→ SEND/RECEIVE (보안 검증)
→ DISCONNECT (이벤트 리스너에서 정리 + 언바인딩)
```

이 구조를 통해 안전하고 일관된 WebSocket 통신 환경을 제공합니다!