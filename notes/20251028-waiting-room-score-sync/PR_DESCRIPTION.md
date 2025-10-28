# Pull Request

> 제목: **[Feat]: 대기방 참가자 점수 DB 동기화 및 표시**

## PR 유형 (Type of PR)

- [x] 기능 (Feature)
- [ ] 버그 수정 (Bugfix)
- [ ] 기능 개선 (Enhancement)
- [ ] 리팩토링 (Refactoring)
- [ ] 문서 (Docs)
- [ ] 기타 (Chore)

---

## PR 요약 (PR Summary)

퀴즈 대기방에서 모든 참가자의 실제 누적 점수(totalScore)를 DB에서 조회하여 화면에 표시하는 기능을 구현했습니다. 기존에는 모든 참가자의 점수가 하드코딩된 `0`으로 표시되었으나, 이제 백엔드에서 각 참가자의 실제 점수를 전달하고 프론트엔드에서 이를 올바르게 표시합니다.

---

## 관련 이슈 (Related Issue)

- Relates to # (이슈 번호를 입력해주세요)

---

## 변경 사항 (Changes)

### 백엔드 변경사항

**1. ParticipantResponse DTO 수정**
- `totalScore` 필드 추가 (Long 타입)
- `from()` 메서드에서 User 엔티티의 totalScore 매핑
- null 체크 후 기본값 0L 설정

```java
// backend/src/main/java/app/signbell/backend/dto/response/ParticipantResponse.java
@Getter
@Builder
public class ParticipantResponse {
    private Long userId;
    private String nickname;
    private String profileImageUrl;
    private boolean isHost;
    private boolean isReady;
    private Long totalScore; // 추가

    public static ParticipantResponse from(GameParticipant participant) {
        return ParticipantResponse.builder()
                // ... 기존 필드들
                .totalScore(participant.getParticipant().getTotalScore() != null 
                    ? participant.getParticipant().getTotalScore() 
                    : 0L) // 추가
                .build();
    }
}
```

### 프론트엔드 변경사항

**2. useWaitingRoom 훅 수정**
- `setInitialParticipants` 함수: 백엔드에서 받은 `totalScore`를 `participant.score`에 매핑
- `addParticipant` 함수: 새 참가자의 `totalScore`를 `participant.score`에 매핑
- 불필요한 `myTotalScore` 파라미터 제거

```javascript
// frontend/src/hooks/useWaitingRoom.js
const setInitialParticipants = useCallback((participantsList, myUserId, isWebcamOn) => {
  const formattedParticipants = participantsList.map(p => ({
    // ... 기존 필드들
    score: p.totalScore ?? 0, // 변경: 백엔드에서 받은 totalScore 사용
  }));
  setParticipants(formattedParticipants);
}, []);

const addParticipant = useCallback((participant, myUserId, isWebcamOn) => {
  const newParticipant = {
    // ... 기존 필드들
    score: participant.totalScore ?? 0, // 변경: 백엔드에서 받은 totalScore 사용
  };
  setParticipants(prev => [...prev, newParticipant]);
}, []);
```

**3. QuizWaitingRoom 컴포넌트 수정**
- `handleRoomJoin` 함수: `setInitialParticipants` 호출 시 `user.totalScore` 파라미터 제거
- `handleParticipantEvent` 함수: `addParticipant` 호출 시 `user.totalScore` 파라미터 제거

```javascript
// frontend/src/pages/quiz/QuizWaitingRoom.jsx
// 변경 전: setInitialParticipants(roomData.participants, myUserId, isWebcamOn, user?.totalScore)
// 변경 후: setInitialParticipants(roomData.participants, myUserId, isWebcamOn)

// 변경 전: addParticipant(newUser, myUserId, isWebcamOn, user?.totalScore)
// 변경 후: addParticipant(newUser, myUserId, isWebcamOn)
```

### 데이터 플로우

1. 사용자가 대기방 입장 요청
2. 백엔드에서 GameParticipant 목록 조회 (User.totalScore 포함)
3. ParticipantResponse.from()에서 totalScore 매핑
4. WebSocket으로 프론트엔드에 전달
5. useWaitingRoom 훅에서 totalScore → score 매핑
6. UI에 점수 표시

---

## 스크린샷 (Screenshots) (Optional)

### 변경 전
- 모든 참가자의 점수가 "0점"으로 하드코딩되어 표시

### 변경 후
- 각 참가자의 실제 누적 점수가 표시됨
- 게임 완료 후 대기방 복귀 시 업데이트된 점수가 자동으로 반영됨

---

## 테스트 방법 (Test Steps)

### 1. 신규 사용자 대기방 입장 테스트
1. 새로운 계정으로 로그인
2. 퀴즈 대기방 생성 또는 입장
3. 자신의 점수가 "0점"으로 표시되는지 확인

### 2. 게임 완료 후 대기방 복귀 테스트
1. 퀴즈 게임을 완료하여 점수 획득 (예: 1500점)
2. 게임 종료 후 대기방으로 복귀
3. 자신의 점수가 업데이트된 점수(1500점)로 표시되는지 확인

### 3. 여러 참가자 점수 표시 테스트
1. 서로 다른 점수를 가진 여러 계정으로 대기방 입장
2. 각 참가자의 점수가 올바르게 표시되는지 확인
3. 새 참가자가 입장할 때 해당 참가자의 점수가 실시간으로 표시되는지 확인

### 4. WebSocket 이벤트 테스트
1. 브라우저 개발자 도구의 Network 탭에서 WebSocket 메시지 확인
2. PARTICIPANT_JOINED 이벤트에 totalScore 필드가 포함되어 있는지 확인
3. 대기방 입장 응답에 모든 참가자의 totalScore가 포함되어 있는지 확인

---

## 셀프 체크리스트 (Self-Checklist)

- [x] 제목 규칙을 지켰습니다.
- [ ] 관련 이슈를 연결했습니다.
- [x] 로컬에서 충분히 테스트했습니다.
- [ ] `dev` 브랜치를 Pull하여 최신 코드를 반영했습니다.
- [ ] CI/CD 파이프라인이 통과했습니다.

---

## 리뷰어에게 (To Reviewers)

### 테스트 완료 사항
- ✅ **프론트엔드 테스트 완료**: 대기방 입장 및 게임 종료 후 대기방 복귀 시나리오에서 점수 UI가 정상 작동하는 것을 확인했습니다.

### 리뷰 포인트
1. **백엔드 DTO 변경**: ParticipantResponse에 totalScore 필드가 추가되었고, null 체크 로직이 올바른지 확인 부탁드립니다.
2. **프론트엔드 데이터 매핑**: useWaitingRoom 훅에서 `totalScore → score` 매핑이 올바르게 이루어지는지 확인 부탁드립니다.
3. **WebSocket 이벤트**: 대기방 입장 및 새 참가자 입장 시 totalScore가 누락되지 않고 전달되는지 확인 부탁드립니다.
4. **기본값 처리**: totalScore가 null이거나 undefined일 때 0으로 fallback 되는 로직이 백엔드와 프론트엔드 모두에 적용되어 있는지 확인 부탁드립니다.

### 참고사항
- 백엔드 변경은 DTO 수정만으로 완료되며, 서비스 로직 변경은 불필요합니다.
- N+1 쿼리 문제는 이미 JOIN FETCH로 해결되어 있습니다.
- 게임 완료 후 점수 업데이트 로직은 이미 QuizService.endGame()에 구현되어 있습니다.
