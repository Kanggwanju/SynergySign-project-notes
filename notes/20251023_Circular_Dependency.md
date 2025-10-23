# 순환 참조 오류 해결책

## 🔍 순환 참조 발생 이유

```
WebSocketSessionService (세션 관리)
    ↓ 의존
GameRoomLeaveService (퇴장 처리)
    ↓ 의존  
WebSocketSessionService (세션 정리)
    ← 순환!
```

### 구체적인 호출 흐름
1. **WebSocketSessionService** → `handleRoomLeave()` → `leaveService.leaveCurrentRoomByUser()`
2. **GameRoomLeaveService** → `handleHostLeave()` → `sessionService.cleanupMultipleSessions()`

---

## ✅ 추천 해결 방법: **Spring Events 사용**

이 방법이 가장 적합한 이유:
- 두 서비스의 책임이 명확히 분리됨
- 비동기 처리 가능 (선택적)
- 확장성 좋음 (다른 리스너 추가 가능)

### 1️⃣ 이벤트 클래스 생성

```java
package app.signbell.backend.event;

import lombok.Getter;
import lombok.RequiredArgsConstructor;

import java.util.List;

/**
 * 방장 퇴장으로 방이 종료될 때 발행되는 이벤트
 */
@Getter
@RequiredArgsConstructor
public class RoomClosedEvent {
    private final Long roomId;
    private final List<Long> remainingUserIds;
}
```

### 2️⃣ GameRoomLeaveService 수정

```java
@Service
@Slf4j
public class GameRoomLeaveService {

    private final GameParticipantRepository participantRepository;
    private final GameRoomRepository gameRoomRepository;
    private final QuizStateCache quizStateCache;
    private final ApplicationEventPublisher eventPublisher;  // ← 이벤트 발행자 추가

    public GameRoomLeaveService(
            GameParticipantRepository participantRepository,
            GameRoomRepository gameRoomRepository,
            QuizStateCache quizStateCache,
            ApplicationEventPublisher eventPublisher  // ← @Lazy 제거!
    ) {
        this.participantRepository = participantRepository;
        this.gameRoomRepository = gameRoomRepository;
        this.quizStateCache = quizStateCache;
        this.eventPublisher = eventPublisher;
    }

    private ParticipantEventResponse handleHostLeave(GameRoom room, ParticipantResponse hostResponse) {
        Long roomId = room.getId();
        log.info("방장 퇴장 감지 - 방 종료 처리 시작. roomId: {}", roomId);

        // 1. 방장 제외한 다른 참가자들의 userId 목록 조회
        List<Long> otherParticipantUserIds = participantRepository
                .findByGameRoom_Id(roomId)
                .stream()
                .filter(p -> !p.isHost())
                .map(p -> p.getParticipant().getId())
                .toList();

        log.info("방 종료 대상 참가자 수: {}", otherParticipantUserIds.size());

        // 2. 남은 모든 참가자를 한 번의 쿼리로 삭제
        int deletedCount = participantRepository.deleteAllByGameRoom(room);
        log.info("방 종료 시 제거된 참가자 수: {}", deletedCount);

        // 3. 방 종료 처리
        room.closeRoom();

        // 4. 업데이트된 방 정보 저장
        gameRoomRepository.save(room);

        // 5. 방 종료 이벤트 발행 (세션 정리는 이벤트 리스너가 처리)
        if (!otherParticipantUserIds.isEmpty()) {
            eventPublisher.publishEvent(new RoomClosedEvent(roomId, otherParticipantUserIds));
            log.info("방 종료 이벤트 발행 - roomId: {}, 대상 참가자: {}", roomId, otherParticipantUserIds.size());
        }

        log.info("방장 퇴장으로 방 종료 완료 - roomId: {}, 제거된 참가자 수: {}",
                roomId, deletedCount);

        // 6. 방 종료 이벤트 응답 객체 생성 및 반환
        return ParticipantEventResponse.builder()
                .eventType("ROOM_CLOSED")
                .participant(hostResponse)
                .currentParticipants(0)
                .gameRoomId(roomId)
                .roomClosed(true)
                .build();
    }
}
```

### 3️⃣ WebSocketSessionService에 이벤트 리스너 추가

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class WebSocketSessionService {

    private final GameRoomLeaveService leaveService;
    private final SimpMessagingTemplate messagingTemplate;
    private final UserSessionRegistry userSessionRegistry;

    // 기존 메서드들...

    /**
     * 방 종료 이벤트 리스너
     * 
     * GameRoomLeaveService에서 방장이 퇴장해 방이 종료될 때,
     * 남은 참가자들의 세션을 정리합니다.
     * 
     * @param event 방 종료 이벤트
     */
    @EventListener
    public void handleRoomClosedEvent(RoomClosedEvent event) {
        log.info("방 종료 이벤트 수신 - roomId: {}, 대상 참가자 수: {}", 
                event.getRoomId(), event.getRemainingUserIds().size());
        
        cleanupMultipleSessions(event.getRemainingUserIds(), event.getRoomId());
    }

    // cleanupMultipleSessions 메서드는 그대로 유지
}
```

---

## 📊 변경 사항 요약

| 항목 | 기존 (`@Lazy`) | 개선 (Event) |
|------|---------------|-------------|
| **순환 참조** | 여전히 존재 (지연 초기화로 회피) | 완전히 제거 |
| **의존성 방향** | 양방향 | 단방향 (이벤트 기반) |
| **결합도** | 높음 (직접 호출) | 낮음 (이벤트 통신) |
| **테스트 용이성** | 어려움 | 쉬움 (독립적 테스트) |
| **확장성** | 제한적 | 높음 (리스너 추가 가능) |
| **트랜잭션** | 동일 트랜잭션 | 선택 가능 (비동기도 가능) |

---

## 🎯 왜 이 방법이 더 나은가?

1. **책임 분리**: `GameRoomLeaveService`는 "퇴장 처리"만, `WebSocketSessionService`는 "세션 관리"만
2. **의존성 단방향화**: 이벤트 발행자는 리스너를 몰라도 됨
3. **테스트 용이**: 각 서비스를 독립적으로 테스트 가능
4. **확장 가능**: 나중에 다른 처리(로깅, 알림 등)도 리스너로 쉽게 추가

---

## 💡 추가 개선 포인트

코드를 보면서 발견한 다른 개선 포인트:

### 1. 비동기 처리 고려
```java
@Async
@EventListener
public void handleRoomClosedEvent(RoomClosedEvent event) {
    // 세션 정리가 무거운 작업이면 비동기로
}
```

### 2. TransactionalEventListener 사용
```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleRoomClosedEvent(RoomClosedEvent event) {
    // 트랜잭션 커밋 후에만 실행되도록 보장
}
```

**결론: `@Lazy`는 임시방편이었고, Spring Events가 정석 해결책입니다!** 🎉