# 트러블 슈팅: WebSocket 방 퇴장 처리 구조 문제 (20251017)

## 📌 문제 상황

### 초기 구현의 문제점

사용자가 퇴장 버튼을 누르면 다음과 같은 중복 처리가 발생했습니다:
```
1. 퇴장 버튼 클릭
   → STOMP 메시지 전송: /app/room/{id}/leave
   → GameRoomLeaveController에서 퇴장 처리 (DB에서 참가자 삭제)

2. WebSocket 연결 종료
   → SessionDisconnectEvent 발생
   → WebSocketEventsListener에서 다시 퇴장 처리 시도
   → BusinessException: PARTICIPANT_NOT_IN_ROOM 발생 (이미 삭제됨)
```

**결과:**
- 불필요한 예외 발생 (100%)
- 코드 중복 및 복잡도 증가
- DB 쿼리 낭비

---

## 🔍 원인 분석

### 잘못된 이해
> "퇴장 메시지를 보낸 후 연결을 끊어야 한다"

### 올바른 이해
> "WebSocket 연결 종료 자체가 퇴장이다"

**WebSocket의 특성:**
- HTTP와 달리 Stateful한 지속 연결
- 연결 종료 = 세션 종료
- `SessionDisconnectEvent`가 자동으로 발생

**문제의 핵심:**
- `GameRoomLeaveController`와 `WebSocketEventsListener`가 동일한 역할 수행
- 두 컴포넌트 모두 퇴장 처리 로직 실행
- 역할 중복으로 인한 중복 처리

---

## ✅ 해결 방법

### GameRoomLeaveController 제거

**핵심 개선:**
> WebSocket 연결 종료만으로 퇴장 처리 완료

#### 1) 프론트엔드 단순화
```javascript
// ❌ 변경 전: 메시지 전송 + 연결 종료
function handleLeaveRoom(roomId) {
  // 퇴장 메시지 전송
  stompClient.publish({
    destination: `/app/room/${roomId}/leave`
  });
  
  // 연결 종료
  setTimeout(() => {
    stompClient.deactivate();
  }, 100);
}

// ✅ 변경 후: 연결만 종료
function handleLeaveRoom() {
  stompClient.deactivate();
  // WebSocketEventsListener가 자동으로 모든 처리 수행
}
```

#### 2) 백엔드 구조

**변경 전:**
```
GameRoomLeaveController (삭제)
    ↓
WebSocketEventsListener
    ↓
WebSocketSessionService
    ↓
GameRoomLeaveService
```

**변경 후:**
```
WebSocketEventsListener (진입점)
    ↓
WebSocketSessionService (세션 관리)
    ↓
GameRoomLeaveService (비즈니스 로직)
```

#### 3) 핵심 코드

**WebSocketEventsListener.java**
```java
@EventListener
public void onDisconnect(SessionDisconnectEvent event) {
    // 사용자 정보 추출
    DisconnectInfo info = extractDisconnectInfo(event);
    if (info == null) return;
    
    // 세션 서비스에 처리 위임
    sessionManager.handleSessionDisconnect(info.userId(), info.sessionId());
}
```

**WebSocketSessionService.java**
```java
public boolean handleSessionDisconnect(Long userId, String sessionId) {
    try {
        // 활성 세션 검증
        if (!userSessionRegistry.isActive(userId, sessionId)) {
            log.info("비활성 세션 무시");
            return false;
        }
        
        // 방 퇴장 처리
        handleRoomLeave(userId);
        return true;
        
    } finally {
        // 세션 정리 (항상 실행)
        userSessionRegistry.unbind(userId, sessionId);
    }
}
```

**GameRoomLeaveService.java**
```java
@Transactional
public ParticipantEventResponse leaveCurrentRoomByUser(Long userId) {
    // 참가자 조회 (단일 조회)
    GameParticipant participant = participantRepository
        .findByParticipant_Id(userId)
        .orElseThrow(() -> new BusinessException(ErrorCode.PARTICIPANT_NOT_IN_ROOM));
    
    // 방장 여부에 따라 처리 분기
    return leaveRoom(participant);
}

private ParticipantEventResponse leaveRoom(GameParticipant participant) {
    boolean isHost = participant.isHost();
    
    if (isHost) {
        return handleHostLeave(room, participantResponse);
    } else {
        return handleParticipantLeave(participant, room, participantResponse);
    }
}
```

---

## 📊 개선 결과

| 항목 | 변경 전 | 변경 후 | 개선율 |
|------|---------|---------|--------|
| **코드 라인** | 150줄 | 100줄 | 33% ↓ |
| **DB 쿼리** | 5회 | 3회 | 40% ↓ |
| **예외 발생** | 100% | 0% | 100% ↓ |

---

## 💡 핵심 교훈

### 1. WebSocket 생명주기 이해
- WebSocket 연결 종료 = 세션 종료
- `SessionDisconnectEvent` 자동 발생
- 별도의 퇴장 메시지 불필요

### 2. 단일 책임 원칙 (SRP)
- **Listener**: 이벤트 감지 및 추출
- **SessionService**: 세션 생명주기 관리
- **LeaveService**: 비즈니스 로직 처리

### 3. 단순함이 최고
```
복잡한 설계: Controller + Listener (중복 처리)
단순한 설계: Listener만으로 충분
```

---

## 📝 결론

**문제:**
- Controller와 Listener의 역할 중복
- 중복 퇴장 처리로 인한 예외 발생
- 불필요한 코드 복잡도

**해결:**
- Controller 제거로 구조 단순화
- WebSocket 연결 종료 = 방 퇴장
- 단일 진입점으로 명확한 흐름

**결과:**
- 코드 33% 감소
- 쿼리 40% 감소
- 예외 0%로 개선

**핵심:**
> **"WebSocket 연결 종료 자체가 퇴장이다"**
>
> 가장 자연스럽고 단순한 설계가 최선이다.

