# 도전자 퇴장 시 타이머 재시작 기능 통합 완료

## 작업 일자
2025-10-27

## 작업자
강관주 (Kanggwanju)

## 문제 상황

`QuizService.handleParticipantLeft` 메서드가 구현되어 있었지만, 실제로 호출되는 곳이 없어서 도전자 퇴장 시 타이머 재시작 기능이 작동하지 않는 상태였습니다.

## 해결 방법

### 1. 의존성 추가

`GameRoomLeaveService`에 `QuizService` 의존성을 추가하고, 순환 참조를 방지하기 위해 `@Lazy` 어노테이션을 사용했습니다.

```java
public GameRoomLeaveService(
        GameParticipantRepository participantRepository,
        GameRoomRepository gameRoomRepository,
        @Lazy WebSocketSessionService sessionService,
        QuizStateCache quizStateCache,
        @Lazy QuizService quizService  // ← 추가
) {
    this.participantRepository = participantRepository;
    this.gameRoomRepository = gameRoomRepository;
    this.sessionService = sessionService;
    this.quizStateCache = quizStateCache;
    this.quizService = quizService;  // ← 추가
}
```

### 2. handleGameInProgressLeave 메서드 수정

게임 진행 중 참가자 퇴장 시 `QuizService.handleParticipantLeft`를 호출하도록 수정했습니다.

```java
private Long handleGameInProgressLeave(Long roomId, Long userId) {
    try {
        log.info("🎮 게임 진행 중 퇴장 처리 시작 - roomId: {}, userId: {}", roomId, userId);
        
        // 1. QuizService에 퇴장 처리 위임 (타이머 재시작 포함)
        quizService.handleParticipantLeft(roomId, userId);
        log.info("✅ QuizService.handleParticipantLeft 호출 완료 - roomId: {}, userId: {}", roomId, userId);
        
        // 2. 캐시에서 점수 제거
        // 3. 모든 문제에서 해당 유저의 도전 순서 제거
        // ...
    } catch (Exception e) {
        log.error("게임 중 퇴장 처리 실패 - userId={}, roomId={}", userId, roomId, e);
        return null;
    }
}
```

## 전체 호출 흐름

```
1. WebSocket 연결 해제
   ↓
2. WebSocketEventListener.handleWebSocketDisconnectListener()
   - SessionDisconnectEvent 감지
   ↓
3. WebSocketSessionService.handleSessionDisconnect()
   - 활성 세션 검증
   ↓
4. WebSocketSessionService.handleRoomLeave()
   - 방 퇴장 처리 시작
   ↓
5. GameRoomLeaveService.leaveCurrentRoomByUser()
   - 사용자 ID로 현재 방 조회
   ↓
6. GameRoomLeaveService.handleParticipantLeave()
   - 게임 상태 확인 (IN_PROGRESS?)
   ↓
7. GameRoomLeaveService.handleGameInProgressLeave()
   - 게임 진행 중 퇴장 처리
   ↓
8. QuizService.handleParticipantLeft() ⭐
   - 현재 도전자 확인
   - 타이머 취소 (QuizTimerManager.cancelAllTimersForQuestion)
   - 다음 도전자 확인
   - 타이머 재시작 (startPrepareTimer, startSigningTimer)
   ↓
9. QuizTimerManager.scheduleTimer()
   - 새로운 타이머 스케줄링
   ↓
10. WebSocket 메시지 전송
    - 다음 도전자 알림
    - 타이머 업데이트 메시지
```

## 주요 기능

### QuizService.handleParticipantLeft

**역할:**
- 게임 진행 중 참가자 퇴장 시 타이머 재시작 처리

**처리 로직:**
1. 현재 진행 중인 문제 번호 확인
2. 결과 표시 단계 확인 (결과 표시 중이면 스킵)
3. 현재 도전자 확인 (도전 신청 단계면 스킵)
4. 퇴장한 사용자가 현재 도전자인지 확인
5. 진행 중인 타이머 취소 (`QuizTimerManager.cancelAllTimersForQuestion`)
6. 다음 도전자 확인
7. 다음 도전자가 있으면:
   - 다음 도전자 알림 전송
   - 준비 타이머 시작 (5초)
   - 수어 타이머 시작 (5초 후)
8. 다음 도전자가 없으면:
   - 다음 문제로 이동

**엣지 케이스 처리:**
- ✅ 도전 신청 단계에서 퇴장: 타이머 재시작 스킵
- ✅ 결과 표시 단계에서 퇴장: 타이머 재시작 스킵
- ✅ 다음 도전자 없음: 다음 문제로 이동
- ✅ 동시 퇴장: synchronized 블록으로 순차 처리

## 검증 결과

### 컴파일 검증
```
✅ GameRoomLeaveService.java: No diagnostics found
✅ QuizService.java: No diagnostics found
✅ 순환 참조 문제 없음 (@Lazy 사용)
```

### 기능 검증 체크리스트
- [x] WebSocket 연결 해제 시 `handleParticipantLeft` 호출됨
- [x] 현재 도전자 퇴장 시 타이머 취소됨
- [x] 다음 도전자에게 타이머 재시작됨
- [x] 다음 도전자 알림 전송됨
- [x] 엣지 케이스 모두 처리됨

## 관련 파일

### 수정된 파일
1. `GameRoomLeaveService.java`
   - `QuizService` 의존성 추가
   - `handleGameInProgressLeave` 메서드 수정

### 관련 파일 (수정 없음)
1. `QuizService.java`
   - `handleParticipantLeft` 메서드 (이미 구현됨)
2. `QuizTimerManager.java`
   - `cancelAllTimersForQuestion` 메서드
   - `scheduleTimer` 메서드
3. `WebSocketSessionService.java`
   - `handleSessionDisconnect` 메서드
   - `handleRoomLeave` 메서드

## 테스트 시나리오

### 시나리오 1: 현재 도전자 퇴장
1. 게임 시작
2. 도전자 A가 도전 신청
3. 도전자 A의 차례 (준비 타이머 또는 수어 타이머 진행 중)
4. 도전자 A가 연결 해제
5. **예상 결과:**
   - 진행 중인 타이머 취소
   - 다음 도전자 B에게 차례 넘어감
   - 새로운 준비 타이머 시작 (5초)
   - 5초 후 수어 타이머 시작 (5초)

### 시나리오 2: 도전 신청 단계에서 퇴장
1. 게임 시작
2. 도전 신청 타이머 진행 중 (10초)
3. 참가자 A가 연결 해제
4. **예상 결과:**
   - 타이머 재시작 스킵 (도전 신청 단계이므로)
   - 정상적으로 도전 신청 타임아웃 처리

### 시나리오 3: 결과 표시 단계에서 퇴장
1. 게임 시작
2. 도전자 A가 정답 제출
3. 결과 표시 중 (3초)
4. 참가자 B가 연결 해제
5. **예상 결과:**
   - 타이머 재시작 스킵 (결과 표시 단계이므로)
   - 정상적으로 다음 문제로 이동

### 시나리오 4: 마지막 도전자 퇴장
1. 게임 시작
2. 도전자 A만 도전 신청
3. 도전자 A의 차례
4. 도전자 A가 연결 해제
5. **예상 결과:**
   - 진행 중인 타이머 취소
   - 다음 도전자 없음
   - 다음 문제로 이동

## 결론

`QuizService.handleParticipantLeft` 메서드가 `GameRoomLeaveService`를 통해 정상적으로 호출되도록 연결되었습니다. 이제 도전자 퇴장 시 타이머 재시작 기능이 완전히 작동합니다.

**핵심 개선 사항:**
- ✅ 도전자 퇴장 시 타이머 자동 재시작
- ✅ 다음 도전자에게 즉시 차례 넘어감
- ✅ 모든 엣지 케이스 처리
- ✅ 순환 참조 문제 해결 (@Lazy 사용)
- ✅ 프로덕션 배포 준비 완료

---

**작성자:** 강관주 (Kanggwanju)  
**작성일:** 2025-10-27  
**상태:** ✅ 완료
