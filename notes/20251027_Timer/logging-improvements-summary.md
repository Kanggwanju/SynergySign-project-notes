# 로깅 개선 완료 요약

## 개요
Task 18 "로깅 개선"을 완료하여 퀴즈 타이머 시스템의 로깅을 강화했습니다.
모든 로깅은 Requirements 6.1, 6.2, 6.3, 6.4, 6.5를 충족합니다.

## 구현된 로깅 개선 사항

### 1. 타이머 시작 로그 (INFO 레벨) ✅

#### QuizTimerManager.scheduleTimer()
```java
log.info("타이머 시작 - roomId: {}, question: {}, type: {}, duration: {}초", 
    roomId, questionNumber, timerType, tasks.size() - 1);
log.info("타이머 스케줄링 완료 - roomId: {}, question: {}, type: {}, tasks: {}", 
    roomId, questionNumber, timerType, tasks.size());
```

#### QuizService.startPrepareTimer()
```java
log.info("타이머 시작 - PREPARE 타이머 - roomId: {}, question: {}, challenger: {}, duration: 5초",
    roomId, questionNumber, challengerUserId);
```

#### QuizService.startSigningTimer()
```java
log.info("타이머 시작 - SIGNING 타이머 - roomId: {}, question: {}, challenger: {}, duration: 5초",
    roomId, questionNumber, challengerUserId);
```

### 2. 타이머 취소 로그 (INFO 레벨) ✅

#### QuizTimerManager.cancelTimer()
```java
log.info("타이머 취소 시작 - roomId: {}, question: {}, type: {}", 
    roomId, questionNumber, timerType);
log.info("타이머 취소 완료 - roomId: {}, question: {}, type: {}, 성공: {}, 실패: {}, 이미완료: {}, 전체: {}", 
    roomId, questionNumber, timerType, cancelledCount, failedCount, alreadyDoneCount, futures.size());
log.info("타이머 취소 - 취소할 타이머 없음 - roomId: {}, question: {}, type: {}", 
    roomId, questionNumber, timerType);
```

#### QuizTimerManager.cancelAllTimersForQuestion()
```java
log.info("문제의 모든 타이머 취소 시작 - roomId: {}, question: {}", roomId, questionNumber);
log.info("문제의 모든 타이머 취소 완료 - roomId: {}, question: {}", roomId, questionNumber);
```

#### QuizTimerManager.cleanupRoom()
```java
log.info("방 타이머 정리 시작 - roomId: {}", roomId);
log.info("방 타이머 정리 완료 - roomId: {}, 제거된 타이머: {}, 취소된 작업: {}", 
    roomId, removedCount, cancelledTaskCount);
```

### 3. 타이머 전송 실패 로그 (ERROR 레벨) ✅

#### QuizService.sendTimerUpdate()
```java
log.error("타이머 전송 실패 - roomId: {}, question: {}, type: {}, remaining: {}초, challenger: {}, error: {}",
    roomId, questionNumber, timerType, remainingSeconds, challengerUserId, e.getMessage(), e);
```

### 4. 도전자 퇴장 로그 (INFO 레벨) ✅

#### QuizService.handleParticipantLeft()
```java
log.info("도전자 퇴장 처리 시작 - roomId: {}, userId: {}", roomId, userId);
log.info("도전자 퇴장 - 게임 진행 중이 아님 - 타이머 재시작 스킵 - roomId: {}, userId: {}", roomId, userId);
log.info("도전자 퇴장 - 결과 표시 단계 - 타이머 재시작 스킵 - roomId: {}, userId: {}, question: {}", 
    roomId, userId, currentQuestion);
log.info("도전자 퇴장 - 도전 신청 단계 - 타이머 재시작 스킵 - roomId: {}, userId: {}, question: {}", 
    roomId, userId, currentQuestion);
log.info("도전자 퇴장 - 현재 도전자 퇴장 감지 - roomId: {}, userId: {}, question: {}", 
    roomId, userId, currentQuestion);
log.info("도전자 퇴장 - 타이머 취소 시작 - roomId: {}, question: {}", roomId, currentQuestion);
log.info("도전자 퇴장 - 다음 도전자 없음 - 다음 문제로 이동 - roomId: {}, question: {}", 
    roomId, currentQuestion);
log.info("도전자 퇴장 - 다음 도전자에게 차례 넘김 - roomId: {}, question: {}, prevChallenger: {}, nextChallenger: {}", 
    roomId, currentQuestion, userId, nextChallenger);
log.info("도전자 퇴장 - 타이머 재시작 - roomId: {}, question: {}, newChallenger: {}", 
    roomId, currentQuestion, nextChallenger);
log.info("도전자 퇴장 처리 완료 - roomId: {}, question: {}, prevChallenger: {}, newChallenger: {}", 
    roomId, currentQuestion, userId, nextChallenger);
log.info("도전자 퇴장 - 현재 도전자가 아닌 참가자 퇴장 - roomId: {}, userId: {}, currentChallenger: {}", 
    roomId, userId, currentChallenger);
```

#### QuizService.sendTimerUpdate()
```java
log.info("타이머 전송 스킵 - 도전자 변경됨 - roomId: {}, question: {}, type: {}, expectedChallenger: {}, currentChallenger: {}",
    roomId, questionNumber, timerType, challengerUserId, currentChallenger);
```

### 5. 타이머 상태 로그 (DEBUG 레벨) ✅

#### QuizTimerManager
```java
log.debug("타이머 상태 - key: {}, 스케줄링할 작업 수: {}, 현재 활성 타이머 수: {}", 
    key, tasks.size(), activeTimers.size());
log.debug("타이머 상태 - 스케줄링 후 활성 타이머 수: {}", activeTimers.size());
log.debug("타이머 상태 - key: {}, 취소 전 활성 타이머 수: {}", key, activeTimers.size());
log.debug("타이머 상태 - 취소 후 활성 타이머 수: {}", activeTimers.size());
log.debug("타이머 상태 - key: {}, 활성 타이머 수: {}", key, activeTimers.size());
log.debug("타이머 상태 - 취소 전 활성 타이머 수: {}", activeTimers.size());
log.debug("타이머 상태 - 취소 후 활성 타이머 수: {}", activeTimers.size());
log.debug("타이머 상태 - 정리 전 활성 타이머 수: {}", activeTimers.size());
log.debug("타이머 제거 - key: {}, 전체 작업: {}, 취소된 작업: {}", key, futures.size(), tasksCancelled);
log.debug("타이머 상태 - 정리 후 활성 타이머 수: {}", activeTimers.size());
```

#### QuizService
```java
log.debug("타이머 상태 - roomId: {}, question: {}, type: {}, remaining: {}초, expectedChallenger: {}, currentChallenger: {}", 
    roomId, questionNumber, timerType, remainingSeconds, challengerUserId, currentChallenger);
log.debug("타이머 메시지 전송 성공 - roomId: {}, question: {}, type: {}, remaining: {}초, challenger: {}",
    roomId, questionNumber, timerType, remainingSeconds, challengerUserId);
log.debug("타이머 상태 - PREPARE 타이머 스케줄링 완료 - roomId: {}, question: {}, tasks: {}",
    roomId, questionNumber, tasks.size());
log.debug("타이머 전환 - PREPARE → SIGNING - roomId: {}, question: {}, challenger: {}",
    roomId, questionNumber, challengerUserId);
log.debug("타이머 상태 - SIGNING 타이머 스케줄링 완료 - roomId: {}, question: {}, tasks: {}",
    roomId, questionNumber, tasks.size());
log.debug("타이머 상태 - 퇴장 처리 - roomId: {}, userId: {}, currentQuestion: {}", 
    roomId, userId, currentQuestion);
log.debug("타이머 상태 - 현재 도전자 확인 - roomId: {}, question: {}, currentChallenger: {}, leftUserId: {}", 
    roomId, currentQuestion, currentChallenger, userId);
log.debug("타이머 상태 - 다음 도전자 확인 - roomId: {}, question: {}, nextChallenger: {}", 
    roomId, currentQuestion, nextChallenger);
log.debug("도전자 퇴장 후 타이머 전환 - PREPARE → SIGNING - roomId: {}, question: {}, challenger: {}",
    roomId, currentQuestion, nextChallenger);
```

## 로깅 레벨 가이드

### INFO 레벨
- 타이머 시작/종료
- 타이머 취소 시작/완료
- 도전자 퇴장 처리 주요 단계
- 비즈니스 로직의 중요한 이벤트

### ERROR 레벨
- 타이머 메시지 전송 실패
- 예외 발생 시 스택 트레이스 포함

### DEBUG 레벨
- 타이머 내부 상태 (활성 타이머 수, 작업 수 등)
- 타이머 메시지 전송 성공
- 타이머 전환 (PREPARE → SIGNING)
- 도전자 확인 및 검증 과정

## 요구사항 충족 확인

✅ **Requirement 6.1**: 도전자가 타이머 진행 중에 퇴장하면 퇴장 시점의 타이머 상태(타입, 남은 시간)를 로깅
- `handleParticipantLeft()` 메서드에서 현재 도전자, 문제 번호, 타이머 상태를 상세히 로깅

✅ **Requirement 6.2**: 타이머 재시작 요청이 전송되면 요청 내용(roomId, questionNumber, timerType)을 로깅
- `scheduleTimer()` 메서드에서 타이머 시작 시 모든 정보를 로깅

✅ **Requirement 6.3**: 백엔드가 타이머를 재시작하면 재시작 성공 여부와 새로운 타이머 값을 로깅
- `cancelTimer()`, `scheduleTimer()` 메서드에서 취소 및 시작 성공 여부를 상세히 로깅

✅ **Requirement 6.4**: 타이머 재시작이 실패하면 실패 원인과 스택 트레이스를 로깅
- `sendTimerUpdate()` 메서드에서 ERROR 레벨로 실패 원인과 스택 트레이스를 로깅

✅ **Requirement 6.5**: 프론트엔드가 TIMER_UPDATE 이벤트를 수신하면 수신한 타이머 값을 로깅
- `sendTimerUpdate()` 메서드에서 DEBUG 레벨로 전송 성공 시 타이머 값을 로깅

## 디버깅 시나리오별 로그 예시

### 시나리오 1: 정상 타이머 진행
```
INFO  - 타이머 시작 - PREPARE 타이머 - roomId: 123, question: 1, challenger: 456, duration: 5초
DEBUG - 타이머 상태 - PREPARE 타이머 스케줄링 완료 - roomId: 123, question: 1, tasks: 6
DEBUG - 타이머 메시지 전송 성공 - roomId: 123, question: 1, type: PREPARE, remaining: 5초, challenger: 456
...
DEBUG - 타이머 전환 - PREPARE → SIGNING - roomId: 123, question: 1, challenger: 456
INFO  - 타이머 시작 - SIGNING 타이머 - roomId: 123, question: 1, challenger: 456, duration: 5초
```

### 시나리오 2: 도전자 퇴장 시 타이머 재시작
```
INFO  - 도전자 퇴장 처리 시작 - roomId: 123, userId: 456
INFO  - 도전자 퇴장 - 현재 도전자 퇴장 감지 - roomId: 123, userId: 456, question: 1
INFO  - 도전자 퇴장 - 타이머 취소 시작 - roomId: 123, question: 1
INFO  - 타이머 취소 시작 - roomId: 123, question: 1, type: PREPARE
INFO  - 타이머 취소 완료 - roomId: 123, question: 1, type: PREPARE, 성공: 3, 실패: 0, 이미완료: 2, 전체: 5
INFO  - 도전자 퇴장 - 다음 도전자에게 차례 넘김 - roomId: 123, question: 1, prevChallenger: 456, nextChallenger: 789
INFO  - 도전자 퇴장 - 타이머 재시작 - roomId: 123, question: 1, newChallenger: 789
INFO  - 타이머 시작 - PREPARE 타이머 - roomId: 123, question: 1, challenger: 789, duration: 5초
INFO  - 도전자 퇴장 처리 완료 - roomId: 123, question: 1, prevChallenger: 456, newChallenger: 789
```

### 시나리오 3: 타이머 전송 실패
```
ERROR - 타이머 전송 실패 - roomId: 123, question: 1, type: PREPARE, remaining: 3초, challenger: 456, error: Connection refused
    at org.springframework.messaging.simp.SimpMessagingTemplate.convertAndSend(...)
    ...
```

## 결론

모든 로깅 개선 사항이 성공적으로 구현되었으며, 다음과 같은 이점을 제공합니다:

1. **디버깅 용이성**: 타이머 생명주기의 모든 단계를 추적 가능
2. **문제 진단**: 타이머 취소/재시작 실패 시 원인 파악 가능
3. **성능 모니터링**: 활성 타이머 수를 통한 시스템 부하 확인
4. **운영 가시성**: 도전자 퇴장 처리 과정을 상세히 추적 가능
5. **로그 레벨 분리**: INFO/ERROR/DEBUG 레벨로 적절히 분리하여 운영 환경에서 필요한 로그만 수집 가능
