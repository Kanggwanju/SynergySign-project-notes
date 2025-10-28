# [Feat]: 대기방 참가자 점수 DB 동기화 및 표시

## 제안 배경 (Background)

현재 퀴즈 대기방에서 모든 참가자의 점수가 하드코딩된 `0`으로 표시되어 실제 DB의 점수와 동기화되지 않는 문제가 있습니다. 

사용자가 게임을 완료하고 점수를 획득한 후 다시 대기방에 입장했을 때, 자신과 다른 참가자들의 실제 점수를 확인할 수 없어 게임 성과를 비교하거나 경쟁 의식을 느끼기 어렵습니다. 이는 사용자 경험과 데이터 정합성 측면에서 개선이 필요한 부분입니다.

---

## 제안할 기능 (Proposed Feature)

대기방에 입장한 모든 참가자의 실제 누적 점수(totalScore)를 DB에서 조회하여 화면에 표시합니다.

**사용자 경험 변화:**
- 대기방 입장 시 자신의 최신 점수를 즉시 확인 가능
- 다른 참가자들의 점수를 확인하여 실력 비교 가능
- 게임 완료 후 대기방 복귀 시 업데이트된 점수가 자동으로 반영됨
- 새로운 참가자가 입장할 때도 해당 참가자의 점수가 실시간으로 표시됨

---

## 주요 작업 목록 (To-Do List)

### 백엔드
- [ ] 작업 1: ParticipantResponse DTO에 totalScore 필드 추가
- [ ] 작업 2: from() 메서드에서 User.totalScore 매핑 및 null 체크

### 프론트엔드
- [ ] 작업 3: useWaitingRoom 훅의 setInitialParticipants 함수 수정
- [ ] 작업 4: useWaitingRoom 훅의 addParticipant 함수 수정
- [ ] 작업 5: QuizWaitingRoom 컴포넌트의 handleRoomJoin 함수 수정
- [ ] 작업 6: QuizWaitingRoom 컴포넌트의 handleParticipantEvent 함수 수정

### 테스트 (선택적)
- [ ] 작업 7: 백엔드 단위 테스트 작성
- [ ] 작업 8: 프론트엔드 단위 테스트 작성
- [ ] 작업 9: 엔드투엔드 테스트 (신규 사용자, 게임 완료 후 복귀, 여러 참가자)

---

## 기술적인 구현 방안 (Implementation Details)

### 아키텍처 개요
백엔드에서 참가자 정보를 전달할 때 각 참가자의 totalScore를 포함하고, 프론트엔드에서 이를 받아 UI에 표시하는 방식입니다.

### 백엔드 구현
**파일:** `backend/src/main/java/app/signbell/backend/dto/response/ParticipantResponse.java`

```java
@Getter
@Builder
public class ParticipantResponse {
    private Long userId;
    private String nickname;
    private String profileImageUrl;
    private boolean isHost;
    private boolean isReady;
    private Long totalScore; // 🔑 추가

    public static ParticipantResponse from(GameParticipant participant) {
        return ParticipantResponse.builder()
                .userId(participant.getParticipant().getId())
                .nickname(participant.getParticipant().getNickname())
                .profileImageUrl(participant.getParticipant().getProfileImageUrl())
                .isHost(participant.isHost())
                .isReady(participant.isReady())
                .totalScore(participant.getParticipant().getTotalScore() != null 
                    ? participant.getParticipant().getTotalScore() 
                    : 0L) // 🔑 추가 (null 체크)
                .build();
    }
}
```

### 프론트엔드 구현
**파일:** `frontend/src/hooks/useWaitingRoom.js`

```javascript
const setInitialParticipants = useCallback((participantsList, myUserId, isWebcamOn) => {
  const formattedParticipants = participantsList.map(p => ({
    id: p.userId,
    userId: p.userId,
    nickname: p.nickname,
    profileImageUrl: p.profileImageUrl,
    score: p.totalScore ?? 0, // 🔑 백엔드에서 받은 totalScore 사용
    isMe: p.userId === myUserId,
    isHost: p.host,
    isReady: p.ready,
    webcamStatus: p.userId === myUserId ? (isWebcamOn ? 'on' : 'off') : 'off'
  }));
  setParticipants(formattedParticipants);
}, []);
```

### 데이터 플로우
1. 사용자가 대기방 입장 요청
2. 백엔드에서 GameParticipant 목록 조회 (User.totalScore 포함)
3. ParticipantResponse.from()에서 totalScore 매핑
4. WebSocket으로 프론트엔드에 전달
5. useWaitingRoom 훅에서 totalScore → score 매핑
6. UI에 점수 표시

### 게임 완료 후 복귀 시나리오
- 게임 종료 시 `QuizService.endGame()`에서 `user.updateTotalScore()` 호출
- User 엔티티의 totalScore가 DB에 업데이트됨
- 대기방 재입장 시 자동으로 최신 점수 조회 및 표시

---

## 스크린샷 또는 와이어프레임 (Screenshots or Wireframes)

### 현재 상태
- 모든 참가자의 점수가 "0점"으로 하드코딩되어 표시됨

### 개선 후
- 각 참가자의 실제 누적 점수가 표시됨
- 예: "사용자1: 1500점", "사용자2: 2300점", "사용자3: 800점"

---

## 참고 자료 (References)

### 관련 파일
- **스펙 문서:** `.kiro/specs/waiting-room-score-sync/`
  - `requirements.md` - 요구사항 정의
  - `design.md` - 설계 문서
  - `tasks.md` - 구현 계획

### 관련 엔티티 및 DTO
- `backend/src/main/java/app/signbell/backend/entity/User.java` - totalScore 필드
- `backend/src/main/java/app/signbell/backend/entity/GameParticipant.java` - User 참조
- `backend/src/main/java/app/signbell/backend/dto/response/ParticipantResponse.java` - 수정 대상

### 관련 서비스
- `backend/src/main/java/app/signbell/backend/service/GameRoomJoinService.java` - 대기방 입장 로직
- `backend/src/main/java/app/signbell/backend/service/QuizService.java` - 게임 종료 및 점수 업데이트

### 관련 프론트엔드 파일
- `frontend/src/hooks/useWaitingRoom.js` - 대기방 상태 관리
- `frontend/src/pages/quiz/QuizWaitingRoom.jsx` - 대기방 페이지
- `frontend/src/components/quiz/WaitingRoomParticipantCard.jsx` - 점수 표시 UI
