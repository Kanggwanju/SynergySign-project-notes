# Design Document

## Overview

이 문서는 퀴즈 게임 진행 중 현재 도전자가 준비 시간(PREPARE) 또는 수어 동작 시간(SIGNING) 중에 퇴장했을 때, 다음 도전자가 타이머를 초기화된 상태로 시작할 수 있도록 하는 기능의 설계를 다룹니다.

현재 시스템은 `CompletableFuture.delayedExecutor`를 사용하여 타이머를 스케줄링하고 있으며, 도전자가 퇴장해도 이미 스케줄링된 타이머가 계속 실행되어 다음 도전자가 남은 시간을 이어받는 문제가 있습니다.

### 핵심 문제

1. **타이머 스케줄링 방식**: 현재 `startPrepareTimer`와 `startSigningTimer` 메서드는 0~5초까지의 모든 타이머 이벤트를 미리 스케줄링합니다
2. **취소 불가능**: `CompletableFuture.delayedExecutor`로 스케줄링된 작업은 취소할 수 없습니다
3. **상태 불일치**: 도전자가 퇴장해도 타이머는 계속 실행되어 다음 도전자에게 잘못된 시간이 전달됩니다

### 해결 방안

타이머를 취소 가능한 `ScheduledFuture`로 관리하고, 도전자 퇴장 시 진행 중인 타이머를 중단한 후 다음 도전자에게 초기화된 타이머를 제공합니다.

## Architecture

### 시스템 구성 요소

```
┌─────────────────────────────────────────────────────────────┐
│                     Frontend (React)                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  QuizGamePage.jsx                                     │   │
│  │  - handleParticipantLeft (도전자 퇴장 감지)           │   │
│  │  - handleTimerUpdate (타이머 UI 업데이트)             │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ WebSocket
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  Backend (Spring Boot)                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  QuizService                                          │   │
│  │  - handleParticipantLeft (퇴장 처리)                  │   │
│  │  - cancelTimer (타이머 취소)                          │   │
│  │  - restartTimer (타이머 재시작)                       │   │
│  │  - startPrepareTimer (준비 타이머)                    │   │
│  │  - startSigningTimer (수어 타이머)                    │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  QuizTimerManager (새로 추가)                         │   │
│  │  - Map<String, ScheduledFuture<?>> activeTimers       │   │
│  │  - scheduleTimer (타이머 스케줄링)                    │   │
│  │  - cancelTimer (타이머 취소)                          │   │
│  │  - cancelAllTimersForQuestion (문제별 타이머 취소)    │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  QuizStateCache                                       │   │
│  │  - getCurrentChallenger (현재 도전자 조회)            │   │
│  │  - setCurrentChallenger (현재 도전자 설정)            │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 데이터 흐름

#### 정상 흐름 (도전자 퇴장 없음)

```
1. 도전자 A 차례 시작
   → startPrepareTimer(roomId, questionNumber, userA)
   → 5초 카운트다운 (5, 4, 3, 2, 1, 0)
   
2. 준비 시간 종료
   → startSigningTimer(roomId, questionNumber, userA)
   → 5초 카운트다운 (5, 4, 3, 2, 1, 0)
   
3. 수어 시간 종료
   → AI 검증 → 정답/오답 처리
```

#### 도전자 퇴장 시 흐름

```
1. 도전자 A 차례 시작
   → startPrepareTimer(roomId, questionNumber, userA)
   → 타이머 진행 중 (5, 4, 3...)
   
2. 도전자 A 퇴장 (3초 남음)
   → handleParticipantLeft(roomId, userA)
   → cancelTimer(roomId, questionNumber, "PREPARE")
   → 다음 도전자 B 확인
   
3. 도전자 B에게 차례 넘김
   → startPrepareTimer(roomId, questionNumber, userB)
   → 새로운 5초 카운트다운 시작 (5, 4, 3, 2, 1, 0)
```

## Components and Interfaces

### 1. QuizTimerManager (새로 추가)

타이머 생명주기를 관리하는 새로운 컴포넌트입니다.

```java
@Component
@Slf4j
public class QuizTimerManager {
    
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(
            100,  // 최대 50개 방 × 2개 타이머 = 100
            new ThreadFactoryBuilder()
                .setNameFormat("quiz-timer-%d")
                .setDaemon(true)
                .build()
        );
    
    // Key: "roomId:questionNumber:timerType" (예: "123:1:PREPARE")
    private final Map<String, List<ScheduledFuture<?>>> activeTimers = 
        new ConcurrentHashMap<>();
    
    /**
     * 타이머 스케줄링
     * 
     * @param roomId 방 ID
     * @param questionNumber 문제 번호
     * @param timerType 타이머 타입 (PREPARE, SIGNING)
     * @param tasks 실행할 작업 리스트 (초별)
     */
    public void scheduleTimer(
        Long roomId, 
        Integer questionNumber, 
        String timerType,
        List<Runnable> tasks
    ) {
        String key = buildKey(roomId, questionNumber, timerType);
        
        // 기존 타이머가 있으면 취소
        cancelTimer(roomId, questionNumber, timerType);
        
        List<ScheduledFuture<?>> futures = new ArrayList<>();
        
        for (int i = 0; i < tasks.size(); i++) {
            ScheduledFuture<?> future = scheduler.schedule(
                tasks.get(i),
                i,
                TimeUnit.SECONDS
            );
            futures.add(future);
        }
        
        activeTimers.put(key, futures);
        log.info("타이머 스케줄링 완료 - key: {}, tasks: {}", key, tasks.size());
    }
    
    /**
     * 특정 타이머 취소
     */
    public void cancelTimer(
        Long roomId, 
        Integer questionNumber, 
        String timerType
    ) {
        String key = buildKey(roomId, questionNumber, timerType);
        List<ScheduledFuture<?>> futures = activeTimers.remove(key);
        
        if (futures != null) {
            int cancelledCount = 0;
            for (ScheduledFuture<?> future : futures) {
                if (!future.isDone() && !future.isCancelled()) {
                    future.cancel(false);
                    cancelledCount++;
                }
            }
            log.info("타이머 취소 완료 - key: {}, cancelled: {}/{}", 
                key, cancelledCount, futures.size());
        }
    }
    
    /**
     * 특정 문제의 모든 타이머 취소
     */
    public void cancelAllTimersForQuestion(Long roomId, Integer questionNumber) {
        cancelTimer(roomId, questionNumber, "PREPARE");
        cancelTimer(roomId, questionNumber, "SIGNING");
    }
    
    private String buildKey(Long roomId, Integer questionNumber, String timerType) {
        return String.format("%d:%d:%s", roomId, questionNumber, timerType);
    }
}
```

### 2. QuizService 수정

기존 타이머 메서드를 `QuizTimerManager`를 사용하도록 수정합니다.

```java
@Service
@RequiredArgsConstructor
public class QuizService {
    
    private final QuizTimerManager timerManager;
    // ... 기존 필드들
    
    /**
     * 수어 준비 타이머 시작 (5초) - 수정됨
     */
    private void startPrepareTimer(
        Long roomId, 
        Integer questionNumber, 
        Long challengerUserId
    ) {
        List<Runnable> tasks = new ArrayList<>();
        
        for (int i = 5; i >= 0; i--) {
            final int remainingSeconds = i;
            tasks.add(() -> {
                try {
                    messagingTemplate.convertAndSend(
                        "/topic/room/" + roomId + "/quiz/timer",
                        ApiResponse.success("타이머 업데이트",
                            TimerUpdateResponse.builder()
                                .timerType("PREPARE")
                                .remainingSeconds(remainingSeconds)
                                .questionNumber(questionNumber)
                                .challengerUserId(challengerUserId)
                                .build()));
                } catch (Exception e) {
                    log.error("타이머 전송 중 오류", e);
                }
            });
        }
        
        timerManager.scheduleTimer(roomId, questionNumber, "PREPARE", tasks);
        
        // 5초 후 수어 표현 타이머 시작
        scheduler.schedule(
            () -> startSigningTimer(roomId, questionNumber, challengerUserId),
            5,
            TimeUnit.SECONDS
        );
    }
    
    /**
     * 수어 표현 타이머 시작 (5초) - 수정됨
     */
    private void startSigningTimer(
        Long roomId, 
        Integer questionNumber, 
        Long challengerUserId
    ) {
        List<Runnable> tasks = new ArrayList<>();
        
        for (int i = 5; i >= 0; i--) {
            final int remainingSeconds = i;
            tasks.add(() -> {
                try {
                    messagingTemplate.convertAndSend(
                        "/topic/room/" + roomId + "/quiz/timer",
                        ApiResponse.success("타이머 업데이트",
                            TimerUpdateResponse.builder()
                                .timerType("SIGNING")
                                .remainingSeconds(remainingSeconds)
                                .questionNumber(questionNumber)
                                .challengerUserId(challengerUserId)
                                .build()));
                } catch (Exception e) {
                    log.error("타이머 전송 중 오류", e);
                }
            });
        }
        
        timerManager.scheduleTimer(roomId, questionNumber, "SIGNING", tasks);
    }
    
    /**
     * 참가자 퇴장 처리 - 수정됨
     */
    @Transactional
    public void handleParticipantLeft(Long roomId, Long userId) {
        QuizStateCache.GameRoomState roomState = 
            quizStateCache.getOrCreateRoomState(roomId);
        
        // 현재 진행 중인 문제 번호 확인
        Integer currentQuestion = roomState.getCurrentQuestionNumber();
        
        if (currentQuestion == null) {
            // 게임이 시작되지 않았거나 종료됨
            return;
        }
        
        // 현재 도전자인지 확인
        Long currentChallenger = roomState.getCurrentChallenger(currentQuestion);
        
        if (currentChallenger != null && currentChallenger.equals(userId)) {
            log.info("현재 도전자 퇴장 - userId: {}, question: {}", 
                userId, currentQuestion);
            
            // 진행 중인 타이머 취소
            timerManager.cancelAllTimersForQuestion(roomId, currentQuestion);
            
            // 다음 도전자 확인
            Long nextChallenger = roomState.getNextChallenger(currentQuestion);
            
            if (nextChallenger != null) {
                // 다음 도전자에게 차례 넘김 (타이머 초기화)
                roomState.setCurrentChallenger(currentQuestion, nextChallenger);
                
                User challenger = userRepository.findById(nextChallenger)
                    .orElseThrow(() -> new BusinessException(
                        ErrorCode.USER_NOT_FOUND));
                
                // 다음 도전자 알림
                messagingTemplate.convertAndSend(
                    "/topic/room/" + roomId + "/quiz",
                    ApiResponse.success("다음 도전자 차례",
                        NextChallengerResponse.builder()
                            .userId(nextChallenger)
                            .nickname(challenger.getNickname())
                            .profileImage(challenger.getProfileImageUrl())
                            .questionNumber(currentQuestion)
                            .build()));
                
                // 새로운 준비 타이머 시작 (5초)
                startPrepareTimer(roomId, currentQuestion, nextChallenger);
                
                // 5초 후 수어 타이머 시작
                CompletableFuture.delayedExecutor(5, TimeUnit.SECONDS)
                    .execute(() -> startSigningTimer(
                        roomId, currentQuestion, nextChallenger));
            } else {
                // 다음 도전자 없음 - 다음 문제로 이동
                transactionService.moveToNextQuestion(roomId, currentQuestion);
            }
        }
        
        // 참가자 목록에서 제거 등 기존 로직 계속...
    }
}
```

### 3. QuizStateCache 수정

현재 문제 번호를 추적하기 위한 메서드 추가가 필요합니다.

```java
public class QuizStateCache {
    
    public static class GameRoomState {
        // 기존 필드들...
        
        private Integer currentQuestionNumber; // 🆕 추가
        
        public Integer getCurrentQuestionNumber() {
            return currentQuestionNumber;
        }
        
        public void setCurrentQuestionNumber(Integer questionNumber) {
            this.currentQuestionNumber = questionNumber;
        }
    }
}
```

### 4. Frontend 수정

프론트엔드는 최소한의 수정만 필요합니다. 백엔드에서 올바른 타이머 값을 전송하므로 기존 `handleTimerUpdate` 로직을 그대로 사용할 수 있습니다.

```javascript
// QuizGamePage.jsx - 수정 불필요
const handleTimerUpdate = useCallback((data) => {
  if (data.success && data.data) {
    const { timerType, remainingSeconds } = data.data;

    if (timerType === 'CHALLENGE') {
      gameState.setTimer(remainingSeconds);
    } else if (timerType === 'PREPARE') {
      gameState.setSolvingTimer(remainingSeconds);
    } else if (timerType === 'SIGNING') {
      gameState.setSigningTimer(remainingSeconds);
    }
  }
}, [gameState]);

// handleParticipantLeft도 기존 로직 유지
// 백엔드에서 타이머 재시작을 처리하므로 프론트엔드는 
// NEXT_CHALLENGER 이벤트와 TIMER_UPDATE 이벤트만 수신하면 됨
```

## Data Models

### TimerKey

타이머를 식별하기 위한 키 구조:

```
Format: "{roomId}:{questionNumber}:{timerType}"
Example: "123:1:PREPARE"
         "123:1:SIGNING"
         "123:2:PREPARE"
```

### TimerUpdateResponse (기존)

```java
@Builder
public class TimerUpdateResponse {
    private String timerType;        // "CHALLENGE", "PREPARE", "SIGNING"
    private Integer remainingSeconds; // 남은 시간 (초)
    private Integer questionNumber;   // 문제 번호
    private Long challengerUserId;    // 도전자 ID (PREPARE, SIGNING만)
}
```

### NextChallengerResponse (기존)

```java
@Builder
public class NextChallengerResponse {
    private Long userId;
    private String nickname;
    private String profileImage;
    private Integer questionNumber;
}
```

## Error Handling

### 1. 타이머 취소 실패

타이머 취소 실패는 **치명적이지 않은 오류**로 처리합니다. 이유는 다음과 같습니다:

1. **이미 완료된 타이머**: 타이머가 이미 실행 완료되었다면 취소할 필요가 없습니다
2. **새 타이머 시작**: 어차피 새로운 타이머가 시작되므로, 이전 타이머의 메시지는 클라이언트에서 무시됩니다
3. **상태 불일치 방지**: 타이머 취소 실패로 게임을 중단하면 더 큰 문제가 발생합니다

**안전장치**:
- 타이머 메시지 전송 시 현재 도전자 ID를 포함하여, 클라이언트가 잘못된 타이머 메시지를 필터링할 수 있도록 합니다
- 백엔드에서 타이머 메시지 전송 전에 현재 도전자가 여전히 유효한지 확인합니다

```java
public void cancelTimer(Long roomId, Integer questionNumber, String timerType) {
    try {
        String key = buildKey(roomId, questionNumber, timerType);
        List<ScheduledFuture<?>> futures = activeTimers.remove(key);
        
        if (futures != null) {
            int cancelledCount = 0;
            int failedCount = 0;
            
            for (ScheduledFuture<?> future : futures) {
                try {
                    if (!future.isDone() && !future.isCancelled()) {
                        boolean cancelled = future.cancel(false);
                        if (cancelled) {
                            cancelledCount++;
                        } else {
                            failedCount++;
                        }
                    }
                } catch (Exception e) {
                    failedCount++;
                    log.warn("개별 타이머 취소 실패 - key: {}", key, e);
                }
            }
            
            log.info("타이머 취소 완료 - key: {}, 성공: {}, 실패: {}, 전체: {}", 
                key, cancelledCount, failedCount, futures.size());
        }
    } catch (Exception e) {
        log.error("타이머 취소 중 예외 발생 - roomId: {}, question: {}, type: {}", 
            roomId, questionNumber, timerType, e);
        // 게임은 계속 진행 (새 타이머가 시작되므로)
    }
}

// 타이머 메시지 전송 시 검증 추가
private void sendTimerUpdate(
    Long roomId, 
    Integer questionNumber, 
    String timerType,
    Integer remainingSeconds,
    Long challengerUserId
) {
    // 현재 도전자가 여전히 유효한지 확인
    QuizStateCache.GameRoomState roomState = 
        quizStateCache.getOrCreateRoomState(roomId);
    Long currentChallenger = roomState.getCurrentChallenger(questionNumber);
    
    // 도전자가 변경되었으면 메시지 전송 안 함
    if (currentChallenger == null || !currentChallenger.equals(challengerUserId)) {
        log.debug("타이머 메시지 전송 스킵 - 도전자 변경됨 - roomId: {}, question: {}", 
            roomId, questionNumber);
        return;
    }
    
    try {
        messagingTemplate.convertAndSend(
            "/topic/room/" + roomId + "/quiz/timer",
            ApiResponse.success("타이머 업데이트",
                TimerUpdateResponse.builder()
                    .timerType(timerType)
                    .remainingSeconds(remainingSeconds)
                    .questionNumber(questionNumber)
                    .challengerUserId(challengerUserId)
                    .build()));
    } catch (Exception e) {
        log.error("타이머 메시지 전송 실패 - roomId: {}, remaining: {}", 
            roomId, remainingSeconds, e);
    }
}
```

### 2. 다음 도전자 없음

```java
if (nextChallenger == null) {
    log.info("다음 도전자 없음 - 다음 문제로 이동 - roomId: {}, question: {}", 
        roomId, currentQuestion);
    
    // 타이머 정리
    timerManager.cancelAllTimersForQuestion(roomId, currentQuestion);
    
    // 다음 문제로 이동
    transactionService.moveToNextQuestion(roomId, currentQuestion);
}
```

### 3. 동시 퇴장 처리

여러 참가자가 동시에 퇴장하는 경우, `@Transactional`과 `synchronized` 블록을 사용하여 순차 처리합니다.

```java
private final Object participantLeftLock = new Object();

@Transactional
public void handleParticipantLeft(Long roomId, Long userId) {
    synchronized (participantLeftLock) {
        // 퇴장 처리 로직...
    }
}
```

### 4. 타이머 전송 실패

```java
tasks.add(() -> {
    try {
        messagingTemplate.convertAndSend(...);
    } catch (Exception e) {
        log.error("타이머 전송 실패 - roomId: {}, remaining: {}", 
            roomId, remainingSeconds, e);
        // WebSocket 전송 실패는 무시 (클라이언트 재연결 시 복구)
    }
});
```

## Testing Strategy

### 1. Unit Tests

#### QuizTimerManager 테스트

```java
@Test
void testScheduleTimer() {
    // Given
    Long roomId = 1L;
    Integer questionNumber = 1;
    String timerType = "PREPARE";
    List<Runnable> tasks = Arrays.asList(
        () -> System.out.println("5"),
        () -> System.out.println("4"),
        () -> System.out.println("3")
    );
    
    // When
    timerManager.scheduleTimer(roomId, questionNumber, timerType, tasks);
    
    // Then
    // 타이머가 스케줄링되었는지 확인
    assertNotNull(timerManager.getActiveTimer(roomId, questionNumber, timerType));
}

@Test
void testCancelTimer() {
    // Given
    timerManager.scheduleTimer(1L, 1, "PREPARE", tasks);
    
    // When
    timerManager.cancelTimer(1L, 1, "PREPARE");
    
    // Then
    assertNull(timerManager.getActiveTimer(1L, 1, "PREPARE"));
}
```

#### QuizService 테스트

```java
@Test
void testHandleParticipantLeft_CurrentChallenger() {
    // Given
    Long roomId = 1L;
    Long userId = 100L;
    Long nextUserId = 200L;
    Integer questionNumber = 1;
    
    // 현재 도전자 설정
    roomState.setCurrentChallenger(questionNumber, userId);
    roomState.addChallenger(questionNumber, nextUserId);
    
    // When
    quizService.handleParticipantLeft(roomId, userId);
    
    // Then
    // 타이머가 취소되었는지 확인
    verify(timerManager).cancelAllTimersForQuestion(roomId, questionNumber);
    
    // 다음 도전자가 설정되었는지 확인
    assertEquals(nextUserId, roomState.getCurrentChallenger(questionNumber));
    
    // 새 타이머가 시작되었는지 확인
    verify(timerManager).scheduleTimer(eq(roomId), eq(questionNumber), 
        eq("PREPARE"), any());
}
```

### 2. Integration Tests

```java
@SpringBootTest
@AutoConfigureMockMvc
class QuizTimerIntegrationTest {
    
    @Test
    void testTimerResetOnDisconnect() throws Exception {
        // 1. 게임 시작
        // 2. 도전자 A 차례 시작
        // 3. 준비 타이머 3초 대기
        // 4. 도전자 A 퇴장
        // 5. 다음 도전자 B에게 5초 타이머가 시작되는지 확인
    }
}
```

### 3. Manual Tests

1. **정상 시나리오**: 도전자가 퇴장하지 않고 정상적으로 게임 진행
2. **준비 시간 중 퇴장**: 도전자가 PREPARE 타이머 중간에 퇴장
3. **수어 시간 중 퇴장**: 도전자가 SIGNING 타이머 중간에 퇴장
4. **마지막 도전자 퇴장**: 다음 도전자가 없을 때 퇴장
5. **동시 퇴장**: 여러 참가자가 동시에 퇴장

## Implementation Notes

### 1. 타이머 정리 시점

- 도전자 퇴장 시
- 정답 제출 후 다음 문제로 이동 시
- 게임 종료 시
- 방 삭제 시

### 2. 메모리 관리

`QuizTimerManager`의 `activeTimers` Map은 게임 종료 시 정리되어야 합니다.

```java
public void cleanupRoom(Long roomId) {
    activeTimers.entrySet().removeIf(entry -> 
        entry.getKey().startsWith(roomId + ":"));
}
```

### 3. 로깅 전략

- 타이머 시작: INFO 레벨
- 타이머 취소: INFO 레벨
- 타이머 전송 실패: ERROR 레벨
- 도전자 퇴장: INFO 레벨

### 4. 성능 고려사항

**스레드 풀 크기 계산**:
- 각 방당 동시 실행 타이머: 최대 2개 (PREPARE + SIGNING)
- 각 타이머당 스케줄링된 작업: 6개 (5, 4, 3, 2, 1, 0초)
- 예상 동시 진행 방: 최대 50개
- 필요한 스레드: 50방 × 2타이머 = 100개 (여유 있게 설정)

**실제 설정**:
```java
private final ScheduledExecutorService scheduler = 
    Executors.newScheduledThreadPool(
        100,  // 코어 스레드 수
        new ThreadFactoryBuilder()
            .setNameFormat("quiz-timer-%d")
            .setDaemon(true)
            .build()
    );
```

**참고**: `ScheduledExecutorService`는 스케줄링된 작업을 큐에 저장하고, 실행 시점에만 스레드를 사용합니다. 따라서 6개의 작업이 스케줄링되어도 동시에 실행되는 것은 1개뿐입니다 (1초 간격으로 순차 실행).

**기타 최적화**:
- `ConcurrentHashMap` 사용으로 동시성 보장
- 타이머 키 문자열 생성 최소화 (캐싱 고려)
- 완료된 `ScheduledFuture` 자동 정리

## Deployment Considerations

### 1. 배포 전 체크리스트

- [ ] QuizTimerManager 빈 등록 확인
- [ ] 기존 타이머 로직 제거 확인
- [ ] 로그 레벨 설정 확인
- [ ] 스레드 풀 크기 설정 확인

### 2. 롤백 계획

문제 발생 시 기존 `CompletableFuture.delayedExecutor` 방식으로 롤백 가능하도록 코드 백업

### 3. 모니터링

- 활성 타이머 수 모니터링
- 타이머 취소 실패 횟수 모니터링
- 평균 타이머 실행 시간 모니터링

## Future Enhancements

1. **타이머 일시정지 기능**: 네트워크 불안정 시 타이머 일시정지
2. **타이머 보정**: 클라이언트-서버 시간 차이 보정
3. **타이머 히스토리**: 타이머 이벤트 히스토리 저장 및 분석
4. **동적 타이머 시간**: 난이도에 따라 타이머 시간 조정
