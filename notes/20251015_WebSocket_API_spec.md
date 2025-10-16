# SignBell WebSocket API 명세서

본 문서는 SignBell 프로젝트의 실시간 퀴즈 기능을 위한 WebSocket API를 정의합니다.  
STOMP 프로토콜을 사용하며, REST API와의 역할 분리를 명확히 합니다.

* **작성일**: 2025-10-15
* **버전**: v1.8.0
* **프로토콜**: STOMP over WebSocket
* **기반 문서**: 기능 요구사항 명세서 v1.0.0, 사용자 흐름도 명세서 v1.0.1

---

## 목차
1. [아키텍처 개요](#1-아키텍처-개요)
2. [연결 설정](#2-연결-설정)
3. [인증](#3-인증)
4. [메시지 구조](#4-메시지-구조)
5. [REST API 목록](#5-rest-api-목록)
6. [WebSocket 이벤트 목록](#6-websocket-이벤트-목록)
7. [에러 처리](#7-에러-처리)
8. [연결 생명주기](#8-연결-생명주기)
9. [시퀀스 다이어그램](#9-시퀀스-다이어그램)

---

## 1. 아키텍처 개요

### 1.1 REST API vs WebSocket 역할 분리

SignBell은 **하이브리드 접근 방식**을 사용합니다.

| 기능 | 프로토콜 | 이유 |
|------|---------|------|
| 방 생성 | REST API | 단발성 요청-응답, gameRoomId 반환 |
| 방 리스트 조회 | REST API | 페이징 처리 필요, 실시간 업데이트 불필요 |
| 방 입장 | WebSocket | 실시간 양방향 통신 시작 |
| 방 내부 모든 상호작용 | WebSocket | 퀴즈 진행, 참여자 관리, WebRTC 시그널링 |

### 1.3 AI 판정 플로우

```
프론트엔드                FastAPI 서버           백엔드(Spring)
    |                         |                      |
    | MediaPipe로 좌표 추출    |                      |
    |                         |                      |
    |--- HTTP POST ---------->|                      |
    |    (좌표 데이터)         |                      |
    |                         |                      |
    |                         | AI 모델 판정          |
    |                         | (정답/오답)           |
    |                         |                      |
    |<-- HTTP Response -------|                      |
    |    (판정 결과)           |                      |
    |                         |                      |
    |--- WebSocket SEND ----------------------->|
    |    (결과만 전송: isCorrect)                |
    |                         |                      |
    |                         |            점수 계산 (HashMap) |
    |                         |            실시간 랭킹 업데이트 |
    |                         |                      |
    |<-- WebSocket (ANSWER_RESULT) --------------|
    |    (점수, 랭킹)          |                      |
```

**FastAPI 서버 역할:**
- 좌표 데이터 수신
- AI 모델로 수어 동작 판정
- 정답 여부 반환

**백엔드(Spring) 역할:**
- 판정 결과만 수신
- 메모리(HashMap)에서 점수 관리 (1~7번 문제)
- WebSocket으로 실시간 랭킹 브로드캐스트
- 8번 문제 완료 시 DB(game_history)에 최종 점수 저장

```
[사용자 플로우]
1. REST API GET /api/quiz/rooms → 방 리스트 조회
2. REST API POST /api/quiz/rooms → 방 생성 (gameRoomId 획득, currentRound: 1)
3. WebSocket 연결
4. WebSocket SEND /app/room/{gameRoomId}/join → 방 입장
5. 방 내부 모든 상호작용은 WebSocket으로만 처리

[퀴즈 게임 플로우]
6. 방장이 퀴즈 시작 (round: 1)
7. 8문제 진행 (questionNumber: 1~8)
8. 퀴즈 종료 → 대기실 복귀 (currentRound: 2로 증가)
9. 방장이 다시 퀴즈 시작 가능 (round: 2)
10. 반복 가능...
```

---

## 2. 연결 설정

### 2.1 Endpoint
```
ws://localhost:9000/ws        (개발)
wss://api.signbell.com/ws     (프로덕션)
```

### 2.2 프로토콜
- **STOMP over WebSocket**
- SockJS fallback 미지원 (순수 WebSocket만 사용)

### 2.3 메시지 브로커 구조
| 경로 Prefix | 용도 | 설명 |
|------------|------|------|
| `/app` | Client → Server | 클라이언트가 서버로 메시지 전송 시 사용 |
| `/topic` | Server → Clients (Broadcast) | 특정 주제를 구독한 모든 클라이언트에게 전송 |
| `/queue` | Server → Client (Direct) | 특정 큐를 구독한 클라이언트에게 전송 |
| `/user` | Server → User (Personal) | 특정 사용자에게만 전송 |

### 2.4 CORS 설정
```java
// WebSocketConfig.java
.setAllowedOriginPatterns("http://localhost:5173")
```

### 2.5 연결 예시 (JavaScript)
```javascript
import { Client } from '@stomp/stompjs';

const client = new Client({
  brokerURL: 'ws://localhost:9000/ws',

  debug: (str) => {
    console.log('STOMP:', str);
  },

  onConnect: (frame) => {
    console.log('Connected:', frame);

    // 방 입장
    joinRoom(gameRoomId);
  },

  onStompError: (frame) => {
    console.error('STOMP Error:', frame);
  },
});

client.activate();
```

---

## 3. 인증

### 3.1 인증 방식
- **HTTP-Only 쿠키 기반 JWT 인증**
- Access Token을 쿠키에 저장하여 WebSocket 핸드셰이크 시 자동 전송
- 서버는 `CookieAuthHandshakeInterceptor`에서 토큰 검증

### 3.2 인증 흐름
```
1. 사용자 로그인 → Access Token 발급 (HTTP-Only 쿠키)
2. WebSocket 연결 시도
3. 서버가 쿠키에서 Access Token 추출
4. JWT 검증 성공 시 연결 허용
5. WebSocket 세션에 사용자 정보 저장 (Principal)
```

### 3.3 보안 규칙
```java
// WebSocketSecurityConfig.java 기반
- /app/** 전송(SEND): 인증 필요 ✅
 - /topic/**, /queue/** 구독(SUBSCRIBE): 인증 필요 ✅
 - 그 외 메시지: 거부 ❌
```

---

## 4. 메시지 구조

### 4.1 공통 응답 포맷 (ApiResponse)

**성공 응답:**
```json
{
  "success": true,
  "message": "요청이 성공적으로 처리되었습니다.",
  "timestamp": "2025-10-15T14:30:00",
  "data": {
    // 응답 데이터
  }
}
```

**타입 정의 (JavaScript 주석):**
```javascript
/**
 * @typedef {Object} ApiResponse
 * @property {boolean} success
 * @property {string} message
 * @property {string} timestamp - ISO 8601 format
 * @property {*} data
 */
```

### 4.2 에러 응답 포맷 (ErrorResponse)

```json
{
  "timestamp": "2025-10-15T14:30:00",
  "status": 400,
  "error": "BAD_REQUEST",
  "detail": "잘못된 요청입니다.",
  "path": "/app/room/1/join"
}
```

**유효성 검증 에러:**
```json
{
  "timestamp": "2025-10-15T14:30:00",
  "status": 400,
  "error": "VALIDATION_ERROR",
  "detail": "입력값 검증에 실패했습니다.",
  "path": "/app/room/create",
  "validationErrors": [
    {
      "field": "gameTitle",
      "message": "방 제목은 필수입니다.",
      "rejectedValue": null
    }
  ]
}
```

**타입 정의 (JavaScript 주석):**
```javascript
/**
 * @typedef {Object} ErrorResponse
 * @property {string} timestamp
 * @property {number} status
 * @property {string} error
 * @property {string} detail
 * @property {string} path
 * @property {ValidationError[]} [validationErrors]
 */

/**
 * @typedef {Object} ValidationError
 * @property {string} field
 * @property {string} message
 * @property {*} rejectedValue
 */
```

---

## 5. REST API 목록

### 5.1 방 생성

**[POST] /api/quiz/rooms**

**Request:**
```json
{
  "gameTitle": "초보자 환영 방"
}
```

**Request Spec:**
| 필드 | 타입 | 필수 | 검증 규칙 |
|------|------|------|----------|
| `gameTitle` | `String` | ✅ | `@NotBlank`, `@Size(min=1, max=50)` |

**Response:** `ApiResponse<CreateRoomResponse>`
```json
{
  "success": true,
  "message": "퀴즈 방이 생성되었습니다.",
  "timestamp": "2025-10-13T10:30:00",
  "data": {
    "gameRoomId": 123
  }
}
```

**Response Spec (data):**
| 필드 | 타입 | 설명 |
|------|------|------|
| `gameRoomId` | `Long` | 생성된 방 고유 ID |

**Error Cases:**
- `401 Unauthorized`: `UNAUTHORIZED` - 인증되지 않은 사용자
- `401 Unauthorized`: `INVALID_TOKEN` - 유효하지 않은 토큰
- `401 Unauthorized`: `EXPIRED_TOKEN` - 만료된 토큰
- `409 Conflict`: `PARTICIPANT_ALREADY_IN_ROOM` - 이미 다른 방에 참여 중
- `400 Bad Request`: `VALIDATION_ERROR` - 입력값 검증 실패
  - `gameTitle`이 공백: "방 제목은 필수입니다."
  - `gameTitle`이 1~50자 벗어남: "방 제목은 1~50자여야 합니다."

---

### 5.2 방 리스트 조회

**[GET] /api/quiz/rooms?page=0&size=10**

**Query Parameters:**
| 파라미터 | 타입 | 필수 | 기본값 | 검증 |
|---------|------|------|--------|------|
| `page` | `Integer` | ❌ | `0` | 0 이상 |
| `size` | `Integer` | ❌ | `10` | 1~100 |

**Response:** `ApiResponse<RoomListSliceResponse>`
```json
{
  "success": true,
  "message": "퀴즈 방 목록을 조회했습니다.",
  "timestamp": "2025-10-13T10:30:00",
  "data": {
    "roomList": [
      {
        "gameRoomId": 123,
        "gameTitle": "초보자 환영 방",
        "hostNickname": "홍길동",
        "currentParticipants": 2,
        "maxParticipants": 4,
        "currentRound": 1,
        "status": "WAITING"
      },
      {
        "gameRoomId": 124,
        "gameTitle": "고수들의 방",
        "hostNickname": "김철수",
        "currentParticipants": 3,
        "maxParticipants": 4,
        "currentRound": 1,
        "status": "WAITING"
      }
    ],
    "hasNext": false
  }
}
```

**Response Spec (data.roomList[]):**
| 필드 | 타입 | 설명 |
|------|------|------|
| `gameRoomId` | `Long` | 방 고유 ID |
| `gameTitle` | `String` | 방 제목 (1~50자) |
| `hostNickname` | `String` | 방장 닉네임 |
| `currentParticipants` | `Integer` | 현재 인원 (기본값: 1) |
| `maxParticipants` | `Integer` | 최대 인원 (고정값: 4) |
| `currentRound` | `Integer` | 게임 진행 횟수 (기본값: 1) |
| `status` | `String` | 방 상태 (항상 `WAITING`) |

**currentRound 설명:**
- 방 생성 시: 1
- 첫 게임 완료 후 대기실 복귀: 2
- 두 번째 게임 완료 후 대기실 복귀: 3
- 게임이 몇 번 진행되었는지를 나타내는 값

**Response Spec (Slice 정보):**
| 필드 | 타입 | 설명 |
|------|------|------|
| `roomList` | `List<RoomListResponse>` | 방 목록 |
| `hasNext` | `Boolean` | 다음 페이지 존재 여부 |

**특징:**
- `WAITING` 상태의 방만 조회
- 생성일 기준 최신순 정렬 (`createdAt DESC`)
- Slice 방식 페이징 (무한 스크롤용)
- **프론트엔드에서 무한 스크롤로 구현 예정**

**Error Cases:**
- `401 Unauthorized`: `UNAUTHORIZED` - 인증되지 않은 사용자
- `401 Unauthorized`: `INVALID_TOKEN` - 유효하지 않은 토큰
- `401 Unauthorized`: `EXPIRED_TOKEN` - 만료된 토큰
- `400 Bad Request`: `INVALID_INPUT` - 유효하지 않은 파라미터

---

## 6. WebSocket 이벤트 목록

### 6.1 방 입장

**[Client → Server]**
```
SEND /app/room/{gameRoomId}/join
// 본문 없음 (gameRoomId는 경로 변수로 전달)
```

**[Server → Client]** (입장자 개인)
```
SEND /user/queue/room

{
  "success": true,
  "message": "방에 입장했습니다.",
  "timestamp": "2025-10-15T14:30:00",
  "data": {
    "gameRoomId": 123,
    "gameTitle": "초보자 환영 방",
    "hostId": 100,
    "participants": [
      {
        "userId": 100,
        "nickname": "user123",
        "profileImageUrl": "https://...",
        "isHost": true,
        "isReady": true
      },
      {
        "userId": 101,
        "nickname": "user456",
        "profileImageUrl": "https://...",
        "isHost": false,
        "isReady": false
      }
    ],
    "currentParticipants": 2,
    "maxParticipants": 4,
    "currentRound": 1,
    "status": "WAITING"
  }
}
```

**currentRound 의미:**
- 이 방에서 게임이 몇 번 진행되었는지를 나타냄
- 방 생성 시: 1
- 게임 완료 후 대기실 복귀: 2, 3, 4, ...

**[Server → Room]** (방 내 모든 참여자)
```
SUBSCRIBE /topic/room/{gameRoomId}/participant

{
  "success": true,
  "message": "새로운 참가자가 입장했습니다.",
  "timestamp": "2025-10-15T14:30:00",
  "data": {
    "eventType": "PARTICIPANT_JOINED",
    "participant": {
      "userId": 101,
      "nickname": "user456",
      "profileImageUrl": "https://...",
      "isHost": false,
      "isReady": false
    },
    "currentParticipants": 2
  }
}
```

**Error Cases:**
- `404 Not Found`: `ROOM_NOT_FOUND` - 방을 찾을 수 없음
- `400 Bad Request`: `ROOM_FULL` - 방 인원이 가득 참
- `400 Bad Request`: `ROOM_ALREADY_STARTED` - 이미 시작된 방
- `409 Conflict`: `PARTICIPANT_ALREADY_IN_ROOM` - 이미 다른 방에 참여 중
- `403 Forbidden`: `CAMERA_PERMISSION_REQUIRED` - 카메라 권한 필요

---

### 6.2 방 퇴장

#### 6.2.1 퇴장 방법

**클라이언트 동작:**
```javascript
// 퇴장 버튼 클릭 시 WebSocket 연결만 종료
stompClient.deactivate();
navigate('/quiz/rooms');
```

**서버 자동 처리:**
- `SessionDisconnectEvent` 감지
- DB에서 참가자 제거
- 방 인원수 감소
- 세션 정리
- 남은 참가자들에게 알림 전송

---

#### 6.2.2 퇴장 알림

**[Server → Room]**
```
SEND /topic/room/{gameRoomId}/participant

{
  "success": true,
  "message": "사용자가 연결을 끊었습니다.",
  "data": {
    "eventType": "PARTICIPANT_LEFT",
    "participant": {
      "userId": 101,
      "nickname": "user456",
      "profileUrl": "https://example.com/profile.jpg",
      "isHost": false
    },
    "currentParticipants": 2,
    "roomId": 123
  }
}
```

**클라이언트 처리:**
```javascript
if (response.data.eventType === 'PARTICIPANT_LEFT') {
  const { participant, currentParticipants } = response.data;
  
  removeParticipantFromUI(participant.userId);
  updateParticipantCount(currentParticipants);
  toast.info(`${participant.nickname}님이 퇴장했습니다.`);
  
  // 방장이 나간 경우
  if (participant.isHost) {
    toast.warning('방장이 퇴장하여 방이 종료되었습니다.');
    stompClient.deactivate();
    navigate('/quiz/rooms');
  }
}
```

---

#### 6.2.3 방장 퇴장 (방 종료)

**동작:**
- 방장 퇴장 시 **방이 즉시 종료**됨
- 모든 참가자가 자동으로 방 목록으로 이동
- 방장 승계 기능 없음

**[Server → Room]** (방장 퇴장 시)
```
SEND /topic/room/{gameRoomId}/participant

{
  "success": true,
  "message": "방장이 퇴장하여 방이 종료되었습니다.",
  "data": {
    "eventType": "ROOM_CLOSED",
    "participant": {
      "userId": 100,
      "nickname": "host123",
      "isHost": true
    },
    "currentParticipants": 0,
    "roomId": 123,
    "roomClosed": true
  }
}
```

**방장 퇴장 전 확인 권장:**
```javascript
if (confirm('방을 나가시면 방이 종료됩니다. 계속하시겠습니까?')) {
  stompClient.deactivate();
}
```

---

#### 6.2.4 비정상 종료

**상황:**
- 브라우저 닫기
- 네트워크 끊김
- 탭 새로고침

**처리:**
서버가 자동으로 감지하여 6.2.2와 동일하게 처리됨 (추가 작업 불필요)

---

### 6.3 퀴즈 시작 (방장만)

**[Client → Server]**
```
SEND /app/room/{gameRoomId}/quiz/start

{
  "gameRoomId": 123
}
```

**[Server → Room]**
```
SUBSCRIBE /topic/room/{gameRoomId}/quiz

{
  "success": true,
  "message": "퀴즈가 시작되었습니다.",
  "timestamp": "2025-10-15T14:30:00",
  "data": {
    "eventType": "QUIZ_STARTED",
    "round": 1,
    "totalQuestions": 8,
    "questions": [
      {
        "questionNumber": 1,
        "quizWordId": 101,
        "title": "안녕하세요"
      },
      {
        "questionNumber": 2,
        "quizWordId": 205,
        "title": "감사합니다"
      },
      {
        "questionNumber": 3,
        "quizWordId": 312,
        "title": "미안합니다"
      }
      // ... 총 8개 문제
    ],
    "answerTimeLimit": 8000
  }
}
```

**응답 데이터 설명:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `round` | `Integer` | 현재 게임 진행 횟수 (1, 2, 3, ...) |
| `totalQuestions` | `Integer` | 이번 게임의 총 문제 수 (8개 고정) |
| `questions` | `Array` | 8개 문제 리스트 |
| `questions[].questionNumber` | `Integer` | 문제 번호 (1~8) |
| `questions[].quizWordId` | `Long` | 퀴즈 단어 ID (DB의 quiz_word_id) |
| `questions[].title` | `String` | 수어 단어 제목 |
| `answerTimeLimit` | `Integer` | 답변 제한 시간 (밀리초) |

**프론트엔드 처리:**
```javascript
// 1. 퀴즈 시작 이벤트 수신
let questions = [];
let currentQuestionIndex = 0;

const handleQuizEvent = (message) => {
  const response = JSON.parse(message.body);
  
  if (response.data.eventType === 'QUIZ_STARTED') {
    console.log('게임 라운드:', response.data.round);
    
    // 8개 문제 저장
    questions = response.data.questions;
    currentQuestionIndex = 0;
    
    // 첫 번째 문제 출제
    setTimeout(() => {
      issueQuestion(questions[0]);
    }, 3000); // 3초 대기
  }
};

// 2. 문제 출제 (클라이언트 주도)
const issueQuestion = (question) => {
  console.log(`${question.questionNumber}번 문제: ${question.title}`);
  
  // quizWordId로 비디오 URL 조회 또는 추론
  const videoUrl = `/videos/quiz/${question.quizWordId}.mp4`;
  
  // UI 업데이트
  showQuestion({
    questionNumber: question.questionNumber,
    title: question.title,
    videoUrl: videoUrl
  });
};

// 3. 다음 문제로 진행
const goToNextQuestion = () => {
  currentQuestionIndex++;
  
  if (currentQuestionIndex < questions.length) {
    setTimeout(() => {
      issueQuestion(questions[currentQuestionIndex]);
    }, 3000);
  } else {
    // 8문제 완료 - 퀴즈 종료 처리
    console.log('모든 문제 완료');
  }
};
```

**백엔드 처리 로직:**
1. 퀴즈 시작 요청 수신
2. 현재 방의 currentRound 값 조회 (1, 2, 3, ...)
3. DB에서 랜덤하게 8개 단어 선택 (QuizWord 테이블)
4. 선택된 단어들을 `questions` 배열로 구성 (quizWordId, title만)
5. 모든 참여자에게 전송
6. GameRoom.currentRound 값은 퀴즈 종료 시 증가

**Error Cases:**
- `403 Forbidden`: `NOT_ROOM_HOST` - 방장 권한 없음
- `400 Bad Request`: `ROOM_MIN_PARTICIPANTS_NOT_MET` - 최소 인원 부족 (2명 미만)
- `400 Bad Request`: `ROOM_ALREADY_STARTED` - 이미 시작된 방
- `404 Not Found`: `WORD_LIST_EMPTY` - 학습 가능한 단어가 없음

---

### 6.4 정답 도전하기 (버튼 클릭)

**[Client → Server]**
```
SEND /app/room/{gameRoomId}/quiz/challenge

{
  "quizWordId": 101,
  "questionNumber": 1
}
```

**Request Spec:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `quizWordId` | `Long` | 퀴즈 단어 ID |
| `questionNumber` | `Integer` | 문제 번호 (1~8) |

**[Server → Room]** (선착순 획득)
```
SEND /topic/room/{gameRoomId}/quiz

{
  "success": true,
  "message": "정답 도전 기회를 획득했습니다.",
  "timestamp": "2025-10-15T14:30:00",
  "data": {
    "eventType": "CHALLENGE_ACQUIRED",
    "userId": 101,
    "nickname": "user456",
    "quizWordId": 101,
    "questionNumber": 1,
    "round": 1,
    "challengeOrder": 1,
    "countdownStart": 3
  }
}
```

**응답 데이터 설명:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `challengeOrder` | `Integer` | 해당 문제에서 몇 번째로 도전하는지 (1, 2, 3, 4) |
| `countdownStart` | `Integer` | 카운트다운 시작 (3초) |

**서버 처리 로직:**
```java
// 각 문제마다 도전 순서를 추적
private final Map<String, AtomicInteger> challengeOrderMap = new ConcurrentHashMap<>();
// Key: "gameRoomId:questionNumber", Value: 도전 순서

@MessageMapping("/room/{gameRoomId}/quiz/challenge")
public void handleChallenge(@DestinationVariable Long gameRoomId,
                             @Payload ChallengeRequest request) {
    
    String key = gameRoomId + ":" + request.getQuestionNumber();
    
    // 도전 순서 증가 (1, 2, 3, 4)
    int challengeOrder = challengeOrderMap
        .computeIfAbsent(key, k -> new AtomicInteger(0))
        .incrementAndGet();
    
    // 최대 4명까지만 허용
    if (challengeOrder > 4) {
        sendError(userId, "QUIZ_MAX_ATTEMPTS_REACHED");
        return;
    }
    
    // CHALLENGE_ACQUIRED 브로드캐스트
    sendChallengeAcquired(gameRoomId, userId, request, challengeOrder);
}
```

**[Server → Client]** (선착순 실패)
```
SEND /user/queue/errors

{
  "timestamp": "2025-10-15T14:30:00",
  "status": 409,
  "error": "QUIZ_BUTTON_ALREADY_TAKEN",
  "detail": "다른 참가자가 먼저 버튼을 눌렀습니다.",
  "path": "/app/room/123/quiz/challenge"
}
```

---

### 6.5 점수 계산 및 랭킹 업데이트

**[Client → Server]**
```
SEND /app/room/{gameRoomId}/quiz/result

{
  "quizWordId": 101,
  "questionNumber": 1,
  "challengeOrder": 1,
  "isCorrect": true
}
```

**Request Spec:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `quizWordId` | `Long` | 퀴즈 단어 ID |
| `questionNumber` | `Integer` | 문제 번호 (1~8) |
| `challengeOrder` | `Integer` | 6.4에서 받은 도전 순서 (1~4) |
| `isCorrect` | `Boolean` | 정답 여부 |

**[Server → Room]** (메모리 기반 점수 관리)
```
SEND /topic/room/{gameRoomId}/quiz

{
  "success": true,
  "message": "정답 판정이 완료되었습니다.",
  "timestamp": "2025-10-15T14:30:00",
  "data": {
    "eventType": "ANSWER_RESULT",
    "userId": 101,
    "nickname": "user456",
    "quizWordId": 101,
    "questionNumber": 1,
    "round": 1,
    "challengeOrder": 1,
    "isCorrect": true,
    "scoreChange": 100,
    "currentScore": 100,
    "ranking": [
      { "userId": 101, "nickname": "user456", "score": 100 },
      { "userId": 100, "nickname": "user123", "score": 0 }
    ]
  }
}
```

**점수 규칙:**
| 도전 순서 (challengeOrder) | 정답 시 | 오답 시 |
|---------------------------|---------|---------|
| 1번째 | +100점 | -50점 |
| 2번째 | +90점 | -50점 |
| 3번째 | +80점 | -50점 |
| 4번째 | +70점 | -50점 |

**백엔드 처리 로직:**
```java
@MessageMapping("/room/{gameRoomId}/quiz/result")
public void handleQuizResult(@DestinationVariable Long gameRoomId,
                              @AuthenticationPrincipal String subject,
                              @Payload QuizResultRequest request) {
    
    Long userId = Long.valueOf(subject);
    
    // 1. challengeOrder 검증 (서버에서 발급한 순서인지 확인)
    validateChallengeOrder(gameRoomId, userId, request.getQuestionNumber(), 
                          request.getChallengeOrder());
    
    // 2. 메모리(HashMap)에서 현재 점수 조회
    Integer currentScore = scoreCache.get(gameRoomId, userId);
    
    // 3. 점수 계산 (도전 순서 기반)
    int scoreChange = calculateScore(request.isCorrect(), request.getChallengeOrder());
    Integer newScore = currentScore + scoreChange;
    
    // 4. 메모리에 업데이트 (1~7번 문제)
    scoreCache.put(gameRoomId, userId, newScore);
    
    // 5. 실시간 랭킹 조회 (메모리에서)
    List<RankingDto> ranking = scoreCache.getRanking(gameRoomId);
    
    // 6. 모든 참가자에게 브로드캐스트
    sendAnswerResult(gameRoomId, userId, request, scoreChange, newScore, ranking);
    
    // 7. 8번 문제 완료 시 DB 저장
    if (request.getQuestionNumber() == 8) {
        saveToDatabase(gameRoomId, userId, newScore);
    }
}

private int calculateScore(boolean isCorrect, int challengeOrder) {
    if (!isCorrect) {
        return -50;  // 오답
    }
    
    // 정답인 경우, 도전 순서에 따라 점수 차등 지급
    return switch(challengeOrder) {
        case 1 -> 100;
        case 2 -> 90;
        case 3 -> 80;
        case 4 -> 70;
        default -> 0;  // 5번째 이상은 점수 없음
    };
}

private void saveToDatabase(Long gameRoomId, Long userId, Integer finalScore) {
    GameRoom room = findRoom(gameRoomId);
    
    GameHistory history = GameHistory.builder()
        .gameRoom(room)
        .participant(findUser(userId))
        .score(finalScore)
        .round(room.getCurrentRound())
        .build();
    
    gameHistoryRepository.save(history);
}
```

**메모리 캐시 구조 (HashMap):**
```java
// ScoreCache.java
private final Map<Long, Map<Long, Integer>> roomScores = new ConcurrentHashMap<>();
// Key: gameRoomId, Value: Map<userId, score>

// 예시:
// roomScores = {
//   123: {
//     101: 100,
//     102: 90,
//     103: -50,
//     104: 0
//   }
// }
```

**프론트엔드 처리:**
```javascript
case 'ANSWER_RESULT':
  console.log(`${data.questionNumber}번 문제 결과:`, 
              data.isCorrect ? '정답' : '오답');
  console.log(`점수 변화: ${data.scoreChange > 0 ? '+' : ''}${data.scoreChange}`);
  console.log(`현재 점수: ${data.currentScore}`);
  
  // 랭킹 UI 업데이트
  updateRanking(data.ranking);
  
  // 다음 문제로 진행
  goToNextQuestion();
  break;
```

**Error Cases:**
- `400 Bad Request`: `QUIZ_ANSWER_TIME_EXPIRED` - 답변 시간 만료
- `400 Bad Request`: `ALREADY_ANSWERED` - 이미 답변한 문제
- `500 Internal Server Error`: `SCORE_UPDATE_FAILED` - 점수 업데이트 실패

---

### 6.6 퀴즈 종료

**[Server → Room]**
```
SEND /topic/room/{gameRoomId}/quiz

{
  "success": true,
  "message": "퀴즈가 종료되었습니다.",
  "timestamp": "2025-10-15T14:30:00",
  "data": {
    "eventType": "QUIZ_FINISHED",
    "completedRound": 1,
    "nextRound": 2,
    "finalRanking": [
      {
        "rank": 1,
        "userId": 101,
        "nickname": "user456",
        "profileImageUrl": "https://...",
        "totalScore": 450
      },
      {
        "rank": 2,
        "userId": 100,
        "nickname": "user123",
        "profileImageUrl": "https://...",
        "totalScore": 380
      }
    ],
    "backToWaitingRoom": true,
    "delaySeconds": 5
  }
}
```

**응답 데이터 설명:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `completedRound` | `Integer` | 방금 완료한 게임 라운드 번호 |
| `nextRound` | `Integer` | 다음 게임 라운드 번호 (completedRound + 1) |
| `finalRanking` | `Array` | 최종 순위 |
| `backToWaitingRoom` | `Boolean` | 대기실 복귀 여부 (항상 true) |
| `delaySeconds` | `Integer` | 대기실 복귀까지 대기 시간 (초) |

**서버 처리 로직:**
```java
@MessageMapping("/room/{gameRoomId}/quiz/finish")
public void finishQuiz(@DestinationVariable Long gameRoomId) {
    
    // 1. 메모리에서 최종 순위 집계
    List<RankingDto> finalRanking = scoreCache.getRanking(gameRoomId);
    
    // 2. 방 정보 조회
    GameRoom room = findRoom(gameRoomId);
    Integer completedRound = room.getCurrentRound();
    
    // 3. **DB에 최종 점수 저장** (game_history 테이블)
    for (RankingDto rank : finalRanking) {
        GameHistory history = GameHistory.builder()
            .gameRoom(room)
            .participant(findUser(rank.getUserId()))
            .score(rank.getTotalScore())
            .round(completedRound)
            .build();
        
        gameHistoryRepository.save(history);
    }
    
    // 4. 메모리 점수 초기화 (다음 게임 준비)
    scoreCache.clear(gameRoomId);
    
    // 5. round 증가 (1 → 2, 2 → 3, ...)
    room.proceedToNextRound();
    
    // 6. 상태를 WAITING으로 변경 (대기실 복귀)
    room.setStatus(GameRoomStatus.WAITING);
    gameRoomRepository.save(room);
    
    // 7. QUIZ_FINISHED 이벤트 발송
    sendQuizFinished(gameRoomId, completedRound, room.getCurrentRound(), finalRanking);
}
```

**DB 저장 내용 (game_history):**
| 컬럼 | 값 | 설명 |
|------|-----|------|
| `game_history_id` | Auto | 기록 ID |
| `game_room_id` | 123 | 방 ID |
| `participant_id` | 101 | 참가자 ID |
| `score` | 450 | 최종 점수 |
| `round` | 1 | 게임 라운드 |
| `created_at` | 2025-10-15 14:30:00 | 기록 시간 |

**프론트엔드 처리:**
```javascript
case 'QUIZ_FINISHED':
  console.log(`${data.completedRound}번째 게임 완료!`);
  console.log(`다음은 ${data.nextRound}번째 게임입니다.`);
  
  // 순위 표시
  showFinalRanking(data.finalRanking);
  
  // 5초 후 대기실로 복귀
  setTimeout(() => {
    showWaitingRoom(); // UI를 대기실로 전환
  }, 5000);
  break;
```

**대기실 복귀 후:**
- 메모리 점수는 초기화됨 (새 게임 준비)
- 방장은 다시 "퀴즈 시작" 버튼 클릭 가능
- 퀴즈 시작 시 `QUIZ_STARTED` 이벤트의 `round` 값은 2가 됨
- 이전 게임의 점수는 DB에 영구 저장됨

---

### 6.7 WebRTC 시그널링

#### Offer
**[Client → Server]**
```
SEND /app/room/{gameRoomId}/webrtc/offer

{
  "targetUserId": 102,
  "sdp": {
    "type": "offer",
    "sdp": "v=0\r\no=- ..."
  }
}
```

**[Server → Target User]**
```
SEND /user/{targetUserId}/queue/webrtc

{
  "success": true,
  "message": "WebRTC Offer를 수신했습니다.",
  "timestamp": "2025-10-15T14:30:00",
  "data": {
    "eventType": "WEBRTC_OFFER",
    "fromUserId": 101,
    "sdp": {
      "type": "offer",
      "sdp": "v=0\r\no=- ..."
    }
  }
}
```

#### Answer
**[Client → Server]**
```
SEND /app/room/{gameRoomId}/webrtc/answer

{
  "targetUserId": 101,
  "sdp": {
    "type": "answer",
    "sdp": "v=0\r\no=- ..."
  }
}
```

**[Server → Target User]**
```
SEND /user/{targetUserId}/queue/webrtc

{
  "success": true,
  "message": "WebRTC Answer를 수신했습니다.",
  "timestamp": "2025-10-15T14:30:00",
  "data": {
    "eventType": "WEBRTC_ANSWER",
    "fromUserId": 102,
    "sdp": {
      "type": "answer",
      "sdp": "v=0\r\no=- ..."
    }
  }
}
```

#### ICE Candidate
**[Client → Server]**
```
SEND /app/room/{gameRoomId}/webrtc/ice-candidate

{
  "targetUserId": 102,
  "candidate": {
    "candidate": "candidate:1 ...",
    "sdpMLineIndex": 0,
    "sdpMid": "0"
  }
}
```

**[Server → Target User]**
```
SEND /user/{targetUserId}/queue/webrtc

{
  "success": true,
  "message": "ICE Candidate를 수신했습니다.",
  "timestamp": "2025-10-15T14:30:00",
  "data": {
    "eventType": "ICE_CANDIDATE",
    "fromUserId": 101,
    "candidate": {
      "candidate": "candidate:1 ...",
      "sdpMLineIndex": 0,
      "sdpMid": "0"
    }
  }
}
```

---

## 7. 에러 처리

### 7.1 에러 전송 방식

**개인 에러 (특정 사용자만):**
```
SEND /user/queue/errors
```

**방 전체 에러 (드물게 사용):**
```
SEND /topic/room/{gameRoomId}/errors
```

### 7.2 주요 에러 코드

#### 인증 에러 (401)
| 코드 | 메시지 | 설명 |
|------|--------|------|
| `UNAUTHORIZED` | 인증되지 않은 사용자입니다 | 로그인 필요 |
| `INVALID_TOKEN` | 유효하지 않은 토큰입니다 | 토큰 재발급 필요 |
| `EXPIRED_TOKEN` | 만료된 토큰입니다 | 토큰 갱신 필요 |

#### 방 관련 에러
| 코드 | 메시지 | Status |
|------|--------|--------|
| `ROOM_NOT_FOUND` | 퀴즈 방을 찾을 수 없습니다 | 404 |
| `ROOM_FULL` | 방 인원이 가득 찼습니다 | 400 |
| `ROOM_ALREADY_STARTED` | 이미 시작된 방입니다 | 400 |
| `ROOM_MIN_PARTICIPANTS_NOT_MET` | 최소 인원이 부족합니다 | 400 |
| `NOT_ROOM_HOST` | 방장 권한이 없습니다 | 403 |
| `PARTICIPANT_ALREADY_IN_ROOM` | 이미 방에 참여 중입니다 | 409 |

#### 퀴즈 관련 에러
| 코드 | 메시지 | Status |
|------|--------|--------|
| `QUIZ_NOT_STARTED` | 퀴즈가 시작되지 않았습니다 | 400 |
| `QUIZ_ALREADY_FINISHED` | 이미 종료된 퀴즈입니다 | 400 |
| `QUIZ_ANSWER_TIME_EXPIRED` | 답변 시간이 만료되었습니다 | 400 |
| `ALREADY_ANSWERED` | 이미 답변한 문제입니다 | 400 |
| `QUIZ_BUTTON_ALREADY_TAKEN` | 다른 참가자가 먼저 버튼을 눌렀습니다 | 409 |

#### AI/좌표 처리 에러
| 코드 | 메시지 | Status | 발생 위치 |
|------|--------|--------|----------|
| `COORDINATE_EXTRACTION_FAILED` | 좌표 추출에 실패했습니다 | 500 | 프론트엔드 |
| `AI_MODEL_ERROR` | AI 모델 처리 중 오류가 발생했습니다 | 500 | FastAPI |
| `AI_MODEL_TIMEOUT` | AI 모델 응답 시간이 초과되었습니다 | 504 | FastAPI |
| `INVALID_COORDINATE_DATA` | 유효하지 않은 좌표 데이터입니다 | 400 | FastAPI |
| `SCORE_UPDATE_FAILED` | 점수 업데이트에 실패했습니다 | 500 | 백엔드 |

**FastAPI 에러 응답 예시:**
```json
{
  "error": "AI_MODEL_ERROR",
  "message": "AI 모델 처리 중 오류가 발생했습니다.",
  "detail": "Model inference failed"
}
```

**프론트엔드 처리:**
```javascript
// FastAPI 에러 처리
try {
  const response = await fetch('https://ai.signbell.com/api/predict', {...});
  
  if (!response.ok) {
    const error = await response.json();
    console.error('FastAPI 에러:', error);
    
    // 사용자에게 알림
    alert(`AI 판정 실패: ${error.message}`);
    
    // 재시도 또는 스킵 처리
    return;
  }
  
  const result = await response.json();
  // 정상 처리...
  
} catch (error) {
  console.error('네트워크 에러:', error);
  alert('AI 서버와 통신할 수 없습니다.');
}
```

#### WebRTC 에러
| 코드 | 메시지 | Status |
|------|--------|--------|
| `WEBRTC_CONNECTION_FAILED` | WebRTC 연결에 실패했습니다 | 500 |
| `SIGNALING_ERROR` | 시그널링 오류가 발생했습니다 | 500 |
| `PEER_CONNECTION_ERROR` | 피어 연결 오류가 발생했습니다 | 500 |
| `NETWORK_DISCONNECTED` | 네트워크 연결이 끊어졌습니다 | 500 |

#### 카메라 에러
| 코드 | 메시지 | Status |
|------|--------|--------|
| `CAMERA_PERMISSION_REQUIRED` | 카메라 권한이 필요합니다 | 403 |
| `CAMERA_PERMISSION_DENIED` | 카메라 권한이 거부되었습니다 | 403 |
| `CAMERA_NOT_AVAILABLE` | 카메라를 사용할 수 없습니다 | 400 |

### 7.3 에러 처리 예시
```javascript
client.subscribe('/user/queue/errors', (message) => {
  const errorResponse = JSON.parse(message.body);
  
  switch (errorResponse.error) {
    case 'UNAUTHORIZED':
      window.location.href = '/login';
      break;
      
    case 'ROOM_FULL':
      toast.error(errorResponse.detail);
      break;
      
    case 'QUIZ_BUTTON_ALREADY_TAKEN':
      showWaitingForNextQuestion();
      break;
      
    case 'VALIDATION_ERROR':
      errorResponse.validationErrors?.forEach(err => {
        showFieldError(err.field, err.message);
      });
      break;
      
    default:
      console.error('Unknown error:', errorResponse);
  }
});
```

---

## 8. 연결 생명주기

### 8.1 연결 상태
| 상태 | 설명 |
|------|------|
| `CONNECTING` | WebSocket 연결 시도 중 |
| `CONNECTED` | STOMP 핸드셰이크 완료, 사용 가능 |
| `DISCONNECTING` | 연결 종료 중 |
| `DISCONNECTED` | 연결 끊김 |

### 8.2 Heartbeat 설정
```javascript
const client = new Client({
  brokerURL: 'ws://localhost:9000/ws',
  heartbeatIncoming: 10000,  // 서버 → 클라이언트 (10초)
  heartbeatOutgoing: 10000,  // 클라이언트 → 서버 (10초)
});
```

**서버 설정:**
```java
// WebSocketConfig.java
@Override
public void configureMessageBroker(MessageBrokerRegistry registry) {
    registry.enableSimpleBroker("/topic", "/queue")
        .setHeartbeatValue(new long[]{10000, 10000});  // [서버→클라이언트, 클라이언트→서버]
}
```

**타임아웃 계산:**
- Heartbeat가 2회 연속 누락되면 연결 해제로 간주
- 실제 타임아웃: heartbeatIncoming * 2 = 20초

### 8.3 연결 해제 감지

**클라이언트 측:**
```javascript
const client = new Client({
  brokerURL: 'ws://localhost:9000/ws',
  
  onWebSocketClose: (event) => {
    console.log('WebSocket 연결 종료:', event.code, event.reason);
    
    // 정상 종료가 아닌 경우
    if (event.code !== 1000) {
      handleUnexpectedDisconnect();
    }
  },
  
  onWebSocketError: (error) => {
    console.error('WebSocket 에러:', error);
  },
  
  onStompError: (frame) => {
    console.error('STOMP 에러:', frame.headers, frame.body);
  }
});

// 비정상 종료 처리
function handleUnexpectedDisconnect() {
  // 사용자에게 알림
  showNotification('연결이 끊어졌습니다. 재연결을 시도합니다...');
  
  // 재연결은 자동으로 시도됨 (reconnectDelay 설정 시)
}
```

**서버 측:**
```java
@EventListener
public void handleSessionConnect(SessionConnectEvent event) {
    StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
    String sessionId = accessor.getSessionId();
    
    log.info("WebSocket 연결: sessionId={}", sessionId);
}

@EventListener
public void handleSessionDisconnect(SessionDisconnectEvent event) {
    StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
    String sessionId = accessor.getSessionId();
    CloseStatus closeStatus = accessor.getCloseStatus();
    
    log.info("WebSocket 연결 해제: sessionId={}, status={}", 
             sessionId, closeStatus);
    
    // 사용자 정보 조회
    Long userId = (Long) accessor.getSessionAttributes().get("userId");
    Long gameRoomId = (Long) accessor.getSessionAttributes().get("gameRoomId");
    
    // 방에 참여 중이었다면 자동 퇴장 처리
    if (userId != null && gameRoomId != null) {
        quizService.handleDisconnect(gameRoomId, userId);
    }
}
```

### 8.4 재연결 전략

**기본 재연결:**
```javascript
const client = new Client({
  brokerURL: 'ws://localhost:9000/ws',
  reconnectDelay: 5000,  // 5초 후 재연결 시도
  
  // 최대 재연결 시도 횟수 (0 = 무제한)
  maxReconnectAttempts: 5,
  
  onConnect: (frame) => {
    console.log('연결 성공');
    
    // 재연결 시 방 상태 복구
    const lastRoomId = sessionStorage.getItem('currentRoomId');
    if (lastRoomId) {
      attemptRejoin(lastRoomId);
    }
  }
});
```

**재입장 로직:**
```javascript
async function attemptRejoin(gameRoomId) {
  try {
    // 방 상태 확인 (REST API)
    const response = await axios.get(`/api/quiz/rooms/${gameRoomId}`);
    
    // WAITING 상태인 경우에만 재입장 허용
    if (response.data.data.status === 'WAITING') {
      client.publish({
        destination: `/app/room/${gameRoomId}/join`,
        body: JSON.stringify({ gameRoomId })
      });
    } else {
      // 이미 시작된 방이면 방 목록으로 이동
      showNotification('게임이 이미 시작되어 재입장할 수 없습니다.');
      navigate('/quiz/rooms');
    }
  } catch (error) {
    // 방이 삭제되었거나 접근 불가
    showNotification('방을 찾을 수 없습니다.');
    navigate('/quiz/rooms');
  }
}
```

**지수 백오프 (선택적):**
```javascript
let reconnectAttempts = 0;

const client = new Client({
  brokerURL: 'ws://localhost:9000/ws',
  
  reconnectDelay: () => {
    reconnectAttempts++;
    // 5초, 10초, 20초, 40초, ... (최대 60초)
    return Math.min(5000 * Math.pow(2, reconnectAttempts - 1), 60000);
  },
  
  onConnect: () => {
    reconnectAttempts = 0;  // 성공 시 리셋
  }
});
```

### 8.5 브라우저 탭 닫기 감지
```javascript
// beforeunload 이벤트로 탭 닫기 감지
window.addEventListener('beforeunload', (event) => {
  if (client && client.connected && currentRoomId) {
    // 방 퇴장 메시지 전송 (비동기)
    client.publish({
      destination: `/app/room/${currentRoomId}/leave`,
      body: JSON.stringify({ gameRoomId: currentRoomId })
    });
    
    // WebSocket 연결 즉시 종료
    client.deactivate();
  }
});

// visibilitychange로 탭 전환 감지 (선택적)
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    console.log('탭이 백그라운드로 전환됨');
  } else {
    console.log('탭이 포어그라운드로 전환됨');
    
    // 연결 상태 확인
    if (client && !client.connected) {
      console.log('연결이 끊어져 재연결 시도');
      client.activate();
    }
  }
});
```

**주의사항:**
- `beforeunload`는 브라우저에 따라 동작이 다를 수 있음
- 모바일에서는 제한적으로 작동
- 서버 측 타임아웃이 주요 방어선

### 8.6 WS 세션 가드 API (단일 탭 강제)
클라이언트는 방 페이지 진입 전에 HTTP 가드 API를 호출하여, 현재 사용자가 이미 활성 WebSocket 세션을 보유 중인지 확인합니다. 활성 세션이 존재하면 새 탭 진입을 차단하고 안내합니다.

- 엔드포인트: `GET /api/ws/session/active`
- 인증: 필요(JWT/세션 쿠키)
- 응답 포맷: `ApiResponse<WsSessionStatusResponse>`

응답 스키마(data):
```json
{
  "active": true,
  "reason": "ACTIVE_SESSION_EXISTS"
}
```

- `active`: 활성 WebSocket 세션 존재 여부(boolean)
- `reason`: "ACTIVE_SESSION_EXISTS" | "NONE"

샘플 성공 응답:
```json
{
  "success": true,
  "message": "WS 세션 활성 여부",
  "timestamp": "2025-10-16T15:45:12.345",
  "data": {
    "active": false,
    "reason": "NONE"
  }
}
```

프론트 권장 흐름:
1) 방 라우트 진입 가드에서 호출 → `active=true`면 진입 차단 및 리다이렉트
2) 가드 통과 시에만 STOMP CONNECT/구독 수행

---

## 9. 시퀀스 다이어그램

### 9.1 방 생성 및 입장 플로우
```
Client                REST API          WebSocket Server
  |                       |                    |
  |--- GET /api/quiz/rooms ->|                 |
  |<-- Room List ---------|                    |
  |                       |                    |
  |--- POST /api/quiz/rooms ->|                |
  |<-- gameRoomId: 123 ---|                    |
  |                       |                    |
  |--- CONNECT /ws ---------------------------->|
  |<-- CONNECTED --------------------------------|
  |                       |                    |
  |--- SUBSCRIBE /topic/room/123/participant -->|
  |--- SUBSCRIBE /topic/room/123/quiz --------->|
  |--- SUBSCRIBE /user/queue/errors ----------->|
  |                       |                    |
  |--- SEND /app/room/123/join ---------------->|
  |<-- ROOM_JOINED (개인) -----------------------|
  |                       |                    |
  |                       | PARTICIPANT_JOINED |
  |<-- (방 전체 broadcast) ----------------------|
```

### 9.2 퀴즈 진행 및 AI 판정 플로우
```
Client (방장)    Backend(Spring)    FastAPI Server    Other Clients
  |                  |                    |                  |
  |--- /app/room/123/quiz/start -------------------------->|
  |                  |                    |                  |
  |                  | (8개 단어 선택)    |                  |
  |                  | (HashMap 초기화)   |                  |
  |                  |                    |                  |
  |<-- QUIZ_STARTED -|--- QUIZ_STARTED ---------------------->|
  |  (round: 1)      |    (round: 1)      |                  |
  |  (8개 문제)      |    (8개 문제)      |                  |
  |                  |                    |                  |
  | (클라이언트가 1번 문제 출제)            |                  |
  |<=========== 문제 1번 표시 ==============================>|
  |                  |                    |                  |
  | [정답 도전하기]   |                    |   [정답 도전하기] |
  |                  |<-- /challenge (먼저!)                 |
  |<-- CHALLENGE_ACQUIRED|-- CHALLENGE_ACQUIRED ------------->|
  |  (challengeOrder:1)  |  (challengeOrder:1)  |            |
  |                  |                    |                  |
  |                  | (3초 카운트다운)    |                  |
  |                  | (5초 동작 수행)    |                  |
  |                  |                    |                  |
  | MediaPipe 좌표 추출 |                 |                  |
  |                  |                    |                  |
  |--- HTTP POST -------------->|          |                  |
  |    (좌표 데이터)             |          |                  |
  |                  |          |  AI 모델 판정              |
  |<-- HTTP Response ------------|          |                  |
  |    (isCorrect)              |          |                  |
  |                  |                    |                  |
  |--- /app/room/123/quiz/result -------->|                  |
  |    (challengeOrder:1, isCorrect)      |                  |
  |                  |                    |                  |
  |                  | (challengeOrder 검증) |               |
  |                  | (HashMap 점수 업데이트) |              |
  |                  | (1등: +100점)      |                  |
  |                  | (실시간 랭킹 조회)   |                |
  |                  |                    |                  |
  |<-- ANSWER_RESULT -|--- ANSWER_RESULT ---------------------->|
  |  (점수, 랭킹)     |    (점수, 랭킹)     |                  |
  |                  |                    |                  |
  | (클라이언트가 2번 문제 출제)            |                  |
  |<=========== 문제 2번 표시 ==============================>|
  |                  |                    |                  |
  | [정답 도전하기]   |<-- /challenge      |   [정답 도전하기] |
  |<-- CHALLENGE_ACQUIRED|-- CHALLENGE_ACQUIRED ------------->|
  |  (challengeOrder:1)  |  (challengeOrder:1)  |            |
  |                  |                    |                  |
  |                  |                    |  [정답 도전하기]  |
  |                  |                    |<-- /challenge    |
  |                  |--- CHALLENGE_ACQUIRED --------------->|
  |                  |                    |  (challengeOrder:2)|
  |                  |                    |                  |
  | (반복... 8번 문제까지)                 |                  |
  |                  |                    |                  |
  | (8번 문제 완료)   |                    |                  |
  |--- /app/room/123/quiz/result -------->|                  |
  |    (challengeOrder, isCorrect)        |                  |
  |                  | (HashMap → DB 저장) |                 |
  |                  | (game_history 기록) |                 |
  |                  | (HashMap 초기화)   |                  |
  |                  | (round 증가: 1→2)  |                  |
  |                  | (status: WAITING)  |                  |
  |<-- QUIZ_FINISHED -|--- QUIZ_FINISHED ---------------------->|
  |  (completedRound:1)|  (completedRound:1)|                |
  |  (nextRound:2)    |    (nextRound:2)   |                  |
  |                  |                    |                  |
  | (5초 후 대기실)   |                    |  (5초 후 대기실) |
```

**핵심 포인트:**
1. ✅ 좌표 데이터는 **FastAPI 서버**로만 전송
2. ✅ AI 판정 결과만 백엔드로 전송
3. ✅ **challengeOrder**를 통해 점수 차등 지급 (1등: 100점, 2등: 90점, 3등: 80점, 4등: 70점)
4. ✅ 1~7번 문제: HashMap에서 점수 관리
5. ✅ 8번 문제: DB(game_history)에 최종 저장
6. ✅ 퀴즈 종료 후 HashMap 초기화

### 9.3 방장 퇴장 시 방 종료 플로우
```
Host (방장)         Server              Other Clients
  |                    |                      |
  |--- /app/room/123/leave ----------------->|
  |                    |                      |
  |                    | (GameRoom.status = FINISHED)
  |                    | (모든 참가자 삭제)     |
  |                    |                      |
  |<-- ROOM_CLOSED ----|---- ROOM_CLOSED ---->|
  |                    |                      |
  | (WebSocket 종료)    |         (WebSocket 종료)
  | (리다이렉트)        |              (리다이렉트)
  ↓                    |                      ↓
/quiz/rooms            |              /quiz/rooms
```

### 9.4 WebRTC 연결 플로우
```
Client A              Server              Client B
  |                      |                      |
  | (방 입장 완료)         |                      |
  |                      |                      |
  |--- createOffer() --->|                      |
  |--- /app/room/123/webrtc/offer ------------>|
  |                      |---- /user/B/queue/webrtc (OFFER) -->
  |                      |                      |
  |                      |                 createAnswer()
  |                      |<--- /app/room/123/webrtc/answer ---
  |<-- /user/A/queue/webrtc (ANSWER) ----------|
  |                      |                      |
  |--- ICE gathering --->|                      |
  |--- /app/room/123/webrtc/ice-candidate ---->|
  |                      |---- /user/B/queue/webrtc (ICE) -->
  |                      |                      |
  |                      |<--- ICE gathering ---|
  |                      |<--- /app/room/123/webrtc/ice-candidate
  |<-- /user/A/queue/webrtc (ICE) -------------|
  |                      |                      |
  |<======== P2P 영상 스트림 연결 완료 =========>|
```

---

## 10. 전체 예제 코드

### 10.1 React + JavaScript + @stomp/stompjs

```javascript
import { Client } from '@stomp/stompjs';
import { useEffect, useState } from 'react';
import axios from 'axios';

function QuizRoom() {
  const [client, setClient] = useState(null);
  const [roomData, setRoomData] = useState(null);
  const [questions, setQuestions] = useState([]);
  const [currentQuestionIndex, setCurrentQuestionIndex] = useState(0);
  const [currentChallengeOrder, setCurrentChallengeOrder] = useState(null);

  // 1. 방 생성 (REST API)
  const createRoom = async (gameTitle) => {
    try {
      const response = await axios.post(
          '/api/quiz/rooms',
          { gameTitle },
          { withCredentials: true }
      );

      const gameRoomId = response.data.data.gameRoomId;
      connectWebSocket(gameRoomId);
    } catch (error) {
      console.error('방 생성 실패:', error);
    }
  };

  // 2. WebSocket 연결 및 방 입장
  const connectWebSocket = (gameRoomId) => {
    const stompClient = new Client({
      brokerURL: 'ws://localhost:9000/ws',

      onConnect: () => {
        console.log('WebSocket 연결 성공');

        // 에러 구독
        stompClient.subscribe('/user/queue/errors', handleError);

        // 방 참여자 업데이트 구독
        stompClient.subscribe(
            `/topic/room/${gameRoomId}/participant`,
            handleParticipantUpdate
        );

        // 퀴즈 이벤트 구독
        stompClient.subscribe(
            `/topic/room/${gameRoomId}/quiz`,
            handleQuizEvent
        );

        // WebRTC 시그널링 구독
        stompClient.subscribe(
            '/user/queue/webrtc',
            handleWebRTCSignaling
        );

        // 방 입장
        stompClient.publish({
          destination: `/app/room/${gameRoomId}/join`,
          body: JSON.stringify({ gameRoomId })
        });
      },

      onStompError: (frame) => {
        console.error('STOMP 에러:', frame);
      }
    });

    stompClient.activate();
    setClient(stompClient);
  };

  // 3. 에러 핸들러
  const handleError = (message) => {
    const error = JSON.parse(message.body);

    switch (error.error) {
      case 'UNAUTHORIZED':
        window.location.href = '/login';
        break;
      case 'ROOM_FULL':
        alert(error.detail);
        break;
      case 'QUIZ_BUTTON_ALREADY_TAKEN':
        console.log('다른 참가자가 먼저 눌렀습니다');
        break;
      default:
        console.error('에러:', error);
    }
  };

  // 4. 참여자 업데이트 핸들러
  const handleParticipantUpdate = (message) => {
    const response = JSON.parse(message.body);

    switch (response.data.eventType) {
      case 'PARTICIPANT_JOINED':
        console.log('새 참가자:', response.data.participant);
        break;

      case 'PARTICIPANT_LEFT':
        console.log('참가자 퇴장:', response.data.userId);
        break;

      case 'ROOM_CLOSED':
        // 방 폐쇄 처리
        handleRoomClosed(response);
        break;
    }
  };

  // 방 폐쇄 핸들러
  const handleRoomClosed = (response) => {
    // 1. WebSocket 연결 종료
    client?.deactivate();

    // 2. 사용자 알림 (optional)
    alert(response.message);

    // 3. 방 목록으로 리다이렉트
    window.location.href = '/quiz/rooms';
  };

  // 5. 퀴즈 이벤트 핸들러
  const handleQuizEvent = (message) => {
    const response = JSON.parse(message.body);

    switch (response.data.eventType) {
      case 'QUIZ_STARTED':
        // 8개 문제 저장
        setQuestions(response.data.questions);
        setCurrentQuestionIndex(0);

        // 3초 후 첫 번째 문제 출제
        setTimeout(() => {
          issueQuestion(response.data.questions[0]);
        }, 3000);
        break;

      case 'CHALLENGE_ACQUIRED':
        console.log(`${response.data.nickname}님 도전!`);

        // 내가 도전권을 얻었다면 challengeOrder 저장
        if (response.data.userId === currentUserId) {
          setCurrentChallengeOrder(response.data.challengeOrder);
        }
        break;

      case 'ANSWER_RESULT':
        console.log('결과:', response.data.isCorrect ? '정답' : '오답');
        console.log(`도전 순서: ${response.data.challengeOrder}등`);

        // challengeOrder 초기화
        setCurrentChallengeOrder(null);

        // 다음 문제로 진행
        goToNextQuestion();
        break;

      case 'QUIZ_FINISHED':
        console.log('퀴즈 종료:', response.data.finalRanking);
        break;
    }
  };

  // 6. 문제 출제 (클라이언트 주도)
  const issueQuestion = (question) => {
    console.log('문제 출제:', question);

    // UI에 문제 표시
    // - question.title: 단어
    // - question.quizWordId: 비디오 URL 추론에 사용
    // - question.questionNumber: 문제 번호
  };

  // 7. 다음 문제로 진행
  const goToNextQuestion = () => {
    const nextIndex = currentQuestionIndex + 1;

    if (nextIndex < questions.length) {
      setCurrentQuestionIndex(nextIndex);

      // 3초 대기 후 다음 문제
      setTimeout(() => {
        issueQuestion(questions[nextIndex]);
      }, 3000);
    } else {
      // 마지막 문제 완료
      console.log('모든 문제 완료');
    }
  };

  // 8. 정답 도전하기
  const handleChallenge = (quizWordId, questionNumber) => {
    if (client && roomData) {
      client.publish({
        destination: `/app/room/${roomData.gameRoomId}/quiz/challenge`,
        body: JSON.stringify({ quizWordId, questionNumber })
      });
    }
  };

  // 9. AI 판정 후 결과 제출
  const submitQuizResult = async (quizWordId, questionNumber, isCorrect) => {
    if (!currentChallengeOrder) {
      console.error('도전 순서가 없습니다. 먼저 정답 도전하기 버튼을 눌러주세요.');
      return;
    }

    if (client && roomData) {
      client.publish({
        destination: `/app/room/${roomData.gameRoomId}/quiz/result`,
        body: JSON.stringify({
          quizWordId,
          questionNumber,
          challengeOrder: currentChallengeOrder,
          isCorrect
        })
      });
    }
  };

  // 10. WebRTC 시그널링 핸들러
  const handleWebRTCSignaling = (message) => {
    const response = JSON.parse(message.body);

    switch (response.data.eventType) {
      case 'WEBRTC_OFFER':
        // Offer 처리
        break;
      case 'WEBRTC_ANSWER':
        // Answer 처리
        break;
      case 'ICE_CANDIDATE':
        // ICE Candidate 처리
        break;
    }
  };

  // 11. 방 퇴장 (자발적)
  const leaveRoom = () => {
    if (!client || !roomData) return;

    // 방장인 경우 확인 모달
    const isHost = roomData.participants.some(
        p => p.isHost && p.userId === currentUserId
    );

    if (isHost) {
      const confirmed = window.confirm(
          '방장이 퇴장하면 방이 종료됩니다. 계속하시겠습니까?'
      );
      if (!confirmed) return;
    }

    // 퇴장 메시지 전송
    client.publish({
      destination: `/app/room/${roomData.gameRoomId}/leave`,
      body: JSON.stringify({ gameRoomId: roomData.gameRoomId })
    });

    // WebSocket 연결 종료
    client.deactivate();

    // 방 목록으로 이동
    window.location.href = '/quiz/rooms';
  };

  // 12. 정리
  useEffect(() => {
    return () => {
      client?.deactivate();
    };
  }, [client]);

  return (
      <div>
        <button onClick={() => createRoom('테스트 방')}>
          방 생성
        </button>
        <button onClick={leaveRoom}>
          방 나가기
        </button>
      </div>
  );
}
```

---

## 11. 제약사항

### 11.1 성능 제한
| 항목 | 제한 | 비고 |
|------|------|------|
| 최대 동시 방 개수 | 100개 | 백엔드 |
| 방당 최대 인원 | 4명 (고정) | 백엔드 |
| 방당 최소 인원 | 2명 | 백엔드 |
| 퀴즈 문제 수 | 8개 (고정) | 백엔드 |
| 정답 도전 시간 | 8초 (3초 카운트다운 + 5초 동작) | 프론트엔드 |
| AI 판정 타임아웃 | 5초 | FastAPI |
| 좌표 데이터 최대 크기 | 5MB | FastAPI |
| HashMap 캐시 TTL | 1시간 | 백엔드 |

### 11.5 브라우저 호환성
- Chrome 90+
- Firefox 88+
- Safari 14+
- Edge 90+

### 11.6 필수 권한
- WebSocket 연결 권한
- 카메라 접근 권한
- 마이크 접근 권한 (옵션)

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|-----------|--------|
| v1.0.0 | 2025-10-15 | 초안 작성 | Claude |
| v1.1.0 | 2025-10-15 | ApiResponse/ErrorResponse 반영 | Claude |
| v1.2.0 | 2025-10-15 | REST API 명세 반영, 필드명 통일 (gameRoomId), 방 리스트 실시간 업데이트 제거 | Claude |
| v1.3.0 | 2025-10-15 | 방장 퇴장 시 방 종료 정책 적용, ROOM_CLOSED 이벤트 추가 | Claude |
| v1.4.0 | 2025-10-15 | 개발 포트 9000번 변경, TypeScript → JavaScript, 퀴즈 시작 시 8개 단어 일괄 전송, 클라이언트 주도 문제 출제 방식, 무한 스크롤 명시 | Claude |
| v1.5.0 | 2025-10-15 | round 개념 변경 (문제번호 → 게임진행횟수), questionNumber 필드 추가, quizWordId + title만 전송 (videoUrl 제거), 퀴즈 종료 시 round 증가 로직 추가 | Claude |
| v1.6.0 | 2025-10-15 | AI 판정 플로우 변경 (프론트→FastAPI→프론트→백엔드), Redis/HashMap 기반 점수 관리 (1~7번), DB 저장 시점 명시 (8번 완료 시), 좌표 데이터는 백엔드로 미전송 | Claude |
| v1.7.0 | 2025-10-15 | videoUrl 관련 설명 제거, 6.4 문제 출제 섹션 삭제, 동작 제출 섹션 삭제, similarity 필드 제거, Redis → HashMap 변경 | Claude |
| v1.8.0 | 2025-10-15 | challengeOrder 필드 추가 (6.4, 6.5), 점수 계산 로직 개선 (도전 순서 기반), 시퀀스 다이어그램 업데이트 | Claude |

---

## 참고 자료
- [STOMP Protocol](https://stomp.github.io/stomp-specification-1.2.html)
- [Spring WebSocket](https://docs.spring.io/spring-framework/reference/web/websocket.html)
- [@stomp/stompjs](https://github.com/stomp-js/stompjs)
- SignBell 퀴즈 REST API 명세서