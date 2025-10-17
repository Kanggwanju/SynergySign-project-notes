# 🎯 **SignBell 실시간 퀴즈 WebSocket API 명세서 (간소화 버전)**

---

## 1️⃣ 개요 (Overview)

SignBell의 실시간 퀴즈 기능은 **STOMP 프로토콜 기반 WebSocket 통신**을 사용합니다.
프론트엔드(React)는 `/app` 경로를 통해 서버에 요청을 보내고,
서버는 `/topic` 또는 `/user/queue` 구독 채널을 통해 결과를 브로드캐스트 또는 개인 응답 형태로 전송합니다.

---

## 2️⃣ 연결 정보 (Connection Info)

| 구분                     | 경로                   | 설명                 |
| ---------------------- | -------------------- | ------------------ |
| **Application Prefix** | `/app`               | Client → Server 전송 |
| **Broadcast Prefix**   | `/topic`             | Server → 모든 참가자    |
| **User Queue Prefix**  | `/user/queue`        | Server → 특정 사용자    |
| **Error Channel**      | `/user/queue/errors` | 에러 메시지 전용          |

---

## 3️⃣ 공통 데이터 구조 (Common Data Format)

| 필드               | 타입        | 설명                                              |
| ---------------- | --------- | ----------------------------------------------- |
| `success`        | `Boolean` | 요청 처리 성공 여부                                     |
| `message`        | `String`  | 처리 결과 메시지                                       |
| `timestamp`      | `String`  | 서버 전송 시각                                        |
| `data`           | `Object`  | 이벤트별 세부 데이터                                     |
| `data.eventType` | `String`  | 이벤트 구분 (`QUIZ_STARTED`, `CHALLENGE_ACQUIRED` 등) |

---

## 4️⃣ 구독 및 이벤트 흐름 요약 (Subscriptions Summary)

| 구독 경로                              | 대상   | 설명                        |
| ---------------------------------- | ---- | ------------------------- |
| `/user/queue/room`                 | 개인   | 방 입장 결과                   |
| `/topic/room/{gameRoomId}/participant` | 방 전체 | 참가자 입퇴장 및 준비 상태 변경 알림     |
| `/user/queue/room-closed`          | 개인   | 방장 퇴장 시 강제 종료             |
| `/user/queue/errors`               | 개인   | 에러 알림                     |
| `/topic/room/{gameRoomId}/quiz`    | 방 전체 | 퀴즈 시작, 도전, 결과 등 퀴즈 관련 이벤트 |

---

## 5️⃣ WebSocket 이벤트 명세 (Event Specification)

---

### 🟩 5.1 방 입장 (Join Room)

**Client → Server**

```
SEND /app/room/{roomId}/join
{
  "roomId": 101,
  "nickname": "판주"
}
```

**Server → User**

```
SUBSCRIBE /user/queue/room
{
  "success": true,
  "message": "입장 성공",
  "data": {
    "eventType": "ROOM_JOINED",
    "roomId": 101,
    "participant": { "id": "user123", "nickname": "판주" }
  }
}
```

**Error Cases**

* `404 NOT_FOUND_ROOM`
* `400 ROOM_ALREADY_FULL`

---

### 🟨 5.2 참가자 준비 상태 변경 (Update Ready Status)

**Client → Server**

```
SEND /app/room/{roomId}/participant/ready
{
  "roomId": 101,
  "isReady": true
}
```

**Server → Topic**

```
SUBSCRIBE /topic/room/{roomId}/participant
{
  "data": {
    "eventType": "PARTICIPANT_READY_UPDATED",
    "userId": "user123",
    "nickname": "판주",
    "isReady": true,
    "allReady": false
  }
}
```

**설명:**

* 참가자가 “준비하기 / 준비 해제” 버튼을 클릭하면 서버는 해당 사용자의 `isReady` 상태를 업데이트하고, 방의 모든 클라이언트에 이를 브로드캐스트합니다.
* `allReady`가 `true`이면 모든 참가자가 준비 완료 상태이며, **방장은 이때 퀴즈 시작 버튼을 활성화**할 수 있습니다.

**Error Cases**

* `400 INVALID_ROOM_STATE`: 게임이 이미 시작된 경우
* `404 NOT_FOUND_PARTICIPANT`: 참가자 정보가 존재하지 않음

---

### 🟧 5.3 방 퇴장 (Leave Room)

**Client → Server**

```
SEND /app/room/{roomId}/leave
```

**Server → Topic**

```
SUBSCRIBE /topic/room/{roomId}/participant
{
  "data": {
    "eventType": "PARTICIPANT_LEFT",
    "userId": "user123",
    "nickname": "판주"
  }
}
```

**Server → User (본인)**

```
SUBSCRIBE /user/queue/room
{
  "success": true,
  "message": "방 퇴장 성공",
  "data": {
    "eventType": "ROOM_LEFT",
    "roomId": 101
  }
}
```

---

### 🟦 5.4 퀴즈 시작 (Start Quiz) — 방장만

**Client → Server**

```
SEND /app/room/{roomId}/quiz/start
```

**Server → Topic**

```
SUBSCRIBE /topic/room/{roomId}/quiz
{
  "data": {
    "eventType": "QUIZ_STARTED",
    "round": 1,
    "totalQuestions": 8,
    "questions": [
      { "questionNumber": 1, "quizWordId": 101, "title": "안녕하세요" },
      ...
    ],
    "answerTimeLimit": 8000
  }
}
```

**Error Cases**

* `403 NOT_ROOM_HOST`
* `400 ROOM_ALREADY_STARTED`
* `400 ROOM_MIN_PARTICIPANTS_NOT_MET`

---

### 🟥 5.5 정답 도전하기 (Challenge)

**Client → Server**

```
SEND /app/room/{roomId}/quiz/challenge
{
  "quizWordId": 101,
  "questionNumber": 1
}
```

**Server → Topic**

```
SUBSCRIBE /topic/room/{roomId}/quiz
{
  "data": {
    "eventType": "CHALLENGE_ACQUIRED",
    "userId": "user456",
    "nickname": "지훈",
    "quizWordId": 101,
    "questionNumber": 1,
    "round": 1,
    "challengeOrder": 1,
    "countdownStart": 3
  }
}
```

**Error Cases**

* `403 CHALLENGE_ALREADY_TAKEN`
* `400 INVALID_QUESTION_NUMBER`

---

### 🟪 5.6 퀴즈 결과 제출 (Result)

**Client → Server**

```
SEND /app/room/{roomId}/quiz/result
{
  "quizWordId": 101,
  "questionNumber": 1,
  "isCorrect": true
}
```

**Server → Topic**

```
SUBSCRIBE /topic/room/{roomId}/quiz
{
  "data": {
    "eventType": "ANSWER_RESULT",
    "userId": "user456",
    "isCorrect": true,
    "score": 50
  }
}
```

---

### 🟫 5.7 퀴즈 종료 (Finish)

**Server → Topic**

```
SUBSCRIBE /topic/room/{roomId}/quiz
{
  "data": {
    "eventType": "QUIZ_FINISHED",
    "results": [
      { "nickname": "판주", "score": 350 },
      { "nickname": "지훈", "score": 300 }
    ]
  }
}
```

---

## 6️⃣ 에러 코드 (Error Codes)

| 코드                                  | 설명               |
| ----------------------------------- | ---------------- |
| `403 NOT_ROOM_HOST`                 | 방장 권한이 없음        |
| `400 ROOM_ALREADY_STARTED`          | 이미 시작된 방         |
| `400 ROOM_MIN_PARTICIPANTS_NOT_MET` | 최소 인원 부족         |
| `403 CHALLENGE_ALREADY_TAKEN`       | 도전 권한 이미 선점됨     |
| `400 INVALID_QUESTION_NUMBER`       | 유효하지 않은 문제 번호    |
| `404 NOT_FOUND_ROOM`                | 존재하지 않는 방        |
| `400 INVALID_ROOM_STATE`            | 잘못된 방 상태 (준비 불가) |
| `404 NOT_FOUND_PARTICIPANT`         | 참가자 정보가 존재하지 않음  |

---

## 7️⃣ 변경 이력 (Changelog)

| 버전     | 날짜         | 변경 내용                           |
| ------ | ---------- | ------------------------------- |
| v1.0.0 | 2025-10-15 | 실시간 퀴즈 WebSocket API 초안 작성      |
| v1.1.0 | 2025-10-17 | 참가자 isReady 상태 변경 기능 및 퇴장 기능 추가 |
