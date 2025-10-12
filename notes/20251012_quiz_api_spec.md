# SignBell 실시간 퀴즈 백엔드 API 명세서

## 문서 정보

- **작성자**: 강관주
- **작성일**: 2025-10-12
- **버전**: v1.0.0
- **대상**: 실시간 퀴즈 기능 백엔드 API

---

## 목차

1. [개요](#1-개요)
2. [인증 방식](#2-인증-방식)
3. [REST API](#3-rest-api)
4. [WebSocket API](#4-websocket-api)
5. [DTO 명세](#5-dto-명세)
6. [에러 코드](#6-에러-코드)
7. [시퀀스 다이어그램](#7-시퀀스-다이어그램)

---

## 1. 개요

### 1.1 API 기본 정보

- **Base URL**: `https://api.signbell.com`
- **WebSocket URL**: `wss://api.signbell.com/ws/quiz`
- **Protocol**: REST + WebSocket
- **Content-Type**: `application/json`
- **Character Encoding**: `UTF-8`

### 1.2 응답 구조

#### 성공 응답
```json
{
  "success": true,
  "message": "요청이 성공적으로 처리되었습니다",
  "timestamp": "2025-10-12T10:30:00",
  "data": { }
}
```

#### 에러 응답
```json
{
  "timestamp": "2025-10-12T10:30:00",
  "status": 404,
  "error": "ROOM_NOT_FOUND",
  "detail": "퀴즈 방을 찾을 수 없습니다.",
  "path": "/api/quiz/rooms/123"
}
```

---

## 2. 인증 방식

### 2.1 JWT 토큰 인증

- **방식**: Bearer Token
- **Header**: `Authorization: Bearer {access_token}`
- **토큰 위치**: HTTP Cookie에 저장
- **만료 시**: 리프레시 토큰으로 자동 갱신

### 2.2 WebSocket 인증

WebSocket 연결 시 첫 메시지로 인증 수행:

```json
{
  "type": "authenticate",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

---

## 3. REST API

### 3.1 퀴즈 방 관리

#### 3.1.1 퀴즈 방 생성

**Endpoint**: `POST /api/quiz/rooms`

**Request Headers**:
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request Body**:
```json
{
  "gameTitle": "초보자 환영 방"
}
```

**Success Response** (201 Created):
```json
{
  "success": true,
  "message": "퀴즈 방이 생성되었습니다.",
  "timestamp": "2025-10-12T10:30:00",
  "data": {
    "roomId": 123,
    "gameTitle": "초보자 환영 방",
    "host": {
      "userId": 1,
      "nickname": "홍길동",
      "email": "hong@example.com"
    },
    "maxParticipants": 4,
    "currentParticipants": 1,
    "status": "WAITING",
    "createdAt": "2025-10-12T10:30:00"
  }
}
```

**Error Responses**:
- `401 UNAUTHORIZED`: 인증되지 않은 사용자
- `409 PARTICIPANT_ALREADY_IN_ROOM`: 이미 다른 방에 참여 중

---

#### 3.1.2 퀴즈 방 리스트 조회

**Endpoint**: `GET /api/quiz/rooms`

**Query Parameters**:
- `status` (optional): 방 상태 필터 (WAITING, IN_PROGRESS, FINISHED)
- `page` (optional, default: 0): 페이지 번호
- `size` (optional, default: 20): 페이지 크기

**Request Example**:
```
GET /api/quiz/rooms?status=WAITING&page=0&size=20
```

**Success Response** (200 OK):
```json
{
  "success": true,
  "message": "퀴즈 방 목록을 조회했습니다.",
  "timestamp": "2025-10-12T10:30:00",
  "data": {
    "content": [
      {
        "roomId": 123,
        "gameTitle": "초보자 환영 방",
        "hostNickname": "홍길동",
        "currentParticipants": 2,
        "maxParticipants": 4,
        "status": "WAITING",
        "createdAt": "2025-10-12T10:30:00"
      },
      {
        "roomId": 124,
        "gameTitle": "고수들의 방",
        "hostNickname": "김철수",
        "currentParticipants": 3,
        "maxParticipants": 4,
        "status": "WAITING",
        "createdAt": "2025-10-12T10:35:00"
      }
    ],
    "pageable": {
      "pageNumber": 0,
      "pageSize": 20,
      "totalElements": 2,
      "totalPages": 1
    }
  }
}
```

---

#### 3.1.3 특정 퀴즈 방 조회

**Endpoint**: `GET /api/quiz/rooms/{roomId}`

**Path Parameters**:
- `roomId`: 조회할 방 ID

**Success Response** (200 OK):
```json
{
  "success": true,
  "message": "퀴즈 방 정보를 조회했습니다.",
  "timestamp": "2025-10-12T10:30:00",
  "data": {
    "roomId": 123,
    "gameTitle": "초보자 환영 방",
    "host": {
      "userId": 1,
      "nickname": "홍길동"
    },
    "participants": [
      {
        "userId": 1,
        "nickname": "홍길동",
        "isHost": true,
        "isReady": true,
        "joinedAt": "2025-10-12T10:30:00"
      },
      {
        "userId": 2,
        "nickname": "김영희",
        "isHost": false,
        "isReady": false,
        "joinedAt": "2025-10-12T10:32:00"
      }
    ],
    "currentParticipants": 2,
    "maxParticipants": 4,
    "status": "WAITING",
    "createdAt": "2025-10-12T10:30:00"
  }
}
```

**Error Responses**:
- `404 ROOM_NOT_FOUND`: 퀴즈 방을 찾을 수 없음

---

#### 3.1.4 퀴즈 방 삭제

**Endpoint**: `DELETE /api/quiz/rooms/{roomId}`

**Path Parameters**:
- `roomId`: 삭제할 방 ID

**Request Headers**:
```
Authorization: Bearer {access_token}
```

**Success Response** (200 OK):
```json
{
  "success": true,
  "message": "퀴즈 방이 삭제되었습니다.",
  "timestamp": "2025-10-12T10:30:00",
  "data": null
}
```

**Error Responses**:
- `404 ROOM_NOT_FOUND`: 퀴즈 방을 찾을 수 없음
- `403 NOT_ROOM_HOST`: 방장 권한이 없음
- `400 ROOM_ALREADY_STARTED`: 이미 시작된 방은 삭제 불가

---

### 3.2 단어 관리

#### 3.2.1 퀴즈 단어 리스트 조회

**Endpoint**: `GET /api/quiz/words`

**Query Parameters**:
- `category` (optional): 카테고리 필터
- `page` (optional, default: 0): 페이지 번호
- `size` (optional, default: 50): 페이지 크기

**Success Response** (200 OK):
```json
{
  "success": true,
  "message": "퀴즈 단어 목록을 조회했습니다.",
  "timestamp": "2025-10-12T10:30:00",
  "data": {
    "content": [
      {
        "wordId": 1,
        "title": "안녕하세요",
        "videoUrl": "https://cdn.signbell.com/videos/hello.mp4",
        "categoryType": "인사"
      },
      {
        "wordId": 2,
        "title": "감사합니다",
        "videoUrl": "https://cdn.signbell.com/videos/thanks.mp4",
        "categoryType": "인사"
      }
    ],
    "pageable": {
      "pageNumber": 0,
      "pageSize": 50,
      "totalElements": 2,
      "totalPages": 1
    }
  }
}
```

---

#### 3.2.2 랜덤 퀴즈 단어 조회

**Endpoint**: `GET /api/quiz/words/random`

**Query Parameters**:
- `count` (optional, default: 10): 추출할 단어 개수

**Success Response** (200 OK):
```json
{
  "success": true,
  "message": "랜덤 퀴즈 단어를 조회했습니다.",
  "timestamp": "2025-10-12T10:30:00",
  "data": [
    {
      "wordId": 5,
      "title": "사랑해요",
      "videoUrl": "https://cdn.signbell.com/videos/love.mp4",
      "categoryType": "감정"
    },
    {
      "wordId": 12,
      "title": "고마워요",
      "videoUrl": "https://cdn.signbell.com/videos/thanks.mp4",
      "categoryType": "인사"
    }
  ]
}
```

**Error Responses**:
- `404 WORD_LIST_EMPTY`: 학습 가능한 단어가 없음

---

### 3.3 퀴즈 기록

#### 3.3.1 사용자 퀴즈 기록 조회

**Endpoint**: `GET /api/quiz/history`

**Request Headers**:
```
Authorization: Bearer {access_token}
```

**Query Parameters**:
- `page` (optional, default: 0): 페이지 번호
- `size` (optional, default: 20): 페이지 크기

**Success Response** (200 OK):
```json
{
  "success": true,
  "message": "퀴즈 기록을 조회했습니다.",
  "timestamp": "2025-10-12T10:30:00",
  "data": {
    "content": [
      {
        "historyId": 1,
        "roomTitle": "초보자 환영 방",
        "score": 450,
        "round": 5,
        "rank": 1,
        "totalParticipants": 4,
        "createdAt": "2025-10-12T09:00:00"
      },
      {
        "historyId": 2,
        "roomTitle": "고수들의 방",
        "score": 320,
        "round": 5,
        "rank": 2,
        "totalParticipants": 3,
        "createdAt": "2025-10-11T15:30:00"
      }
    ],
    "pageable": {
      "pageNumber": 0,
      "pageSize": 20,
      "totalElements": 2,
      "totalPages": 1
    }
  }
}
```

---

#### 3.3.2 퀴즈 결과 저장

**Endpoint**: `POST /api/quiz/results`

**Request Headers**:
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request Body**:
```json
{
  "roomId": 123,
  "participantScores": [
    {
      "userId": 1,
      "score": 450,
      "round": 5
    },
    {
      "userId": 2,
      "score": 380,
      "round": 5
    }
  ]
}
```

**Success Response** (201 Created):
```json
{
  "success": true,
  "message": "퀴즈 결과가 저장되었습니다.",
  "timestamp": "2025-10-12T10:30:00",
  "data": {
    "savedCount": 2
  }
}
```

**Error Responses**:
- `404 ROOM_NOT_FOUND`: 퀴즈 방을 찾을 수 없음
- `400 INVALID_INPUT`: 유효하지 않은 입력값

---

## 4. WebSocket API

### 4.1 연결 및 인증

#### 4.1.1 WebSocket 연결

**Connection URL**: `wss://api.signbell.com/ws/quiz`

**Client → Server (인증)**:
```json
{
  "type": "authenticate",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Server → Client (인증 성공)**:
```json
{
  "type": "authenticated",
  "success": true,
  "data": {
    "userId": 1,
    "nickname": "홍길동"
  }
}
```

**Server → Client (인증 실패)**:
```json
{
  "type": "error",
  "success": false,
  "error": {
    "code": "INVALID_TOKEN",
    "message": "유효하지 않은 토큰입니다."
  }
}
```

---

### 4.2 방 입장/퇴장

#### 4.2.1 방 입장 요청

**Client → Server**:
```json
{
  "type": "join_room",
  "data": {
    "roomId": 123
  }
}
```

**Server → All Clients in Room (입장 성공)**:
```json
{
  "type": "user_joined",
  "data": {
    "user": {
      "userId": 2,
      "nickname": "김영희",
      "isHost": false,
      "isReady": false
    },
    "room": {
      "roomId": 123,
      "currentParticipants": 2,
      "participants": [
        {
          "userId": 1,
          "nickname": "홍길동",
          "isHost": true,
          "isReady": true
        },
        {
          "userId": 2,
          "nickname": "김영희",
          "isHost": false,
          "isReady": false
        }
      ]
    }
  }
}
```

**Server → Client (입장 실패)**:
```json
{
  "type": "error",
  "success": false,
  "error": {
    "code": "ROOM_FULL",
    "message": "방 인원이 가득 찼습니다."
  }
}
```

**Possible Errors**:
- `ROOM_NOT_FOUND`: 퀴즈 방을 찾을 수 없음
- `ROOM_FULL`: 방 인원이 가득 참
- `ROOM_ALREADY_STARTED`: 이미 시작된 방
- `PARTICIPANT_ALREADY_IN_ROOM`: 이미 방에 참여 중
- `CAMERA_PERMISSION_REQUIRED`: 카메라 권한 필요

---

#### 4.2.2 방 퇴장

**Client → Server**:
```json
{
  "type": "leave_room",
  "data": {
    "roomId": 123
  }
}
```

**Server → All Clients in Room**:
```json
{
  "type": "user_left",
  "data": {
    "user": {
      "userId": 2,
      "nickname": "김영희"
    },
    "room": {
      "roomId": 123,
      "currentParticipants": 1,
      "participants": [
        {
          "userId": 1,
          "nickname": "홍길동",
          "isHost": true,
          "isReady": true
        }
      ]
    }
  }
}
```

---

#### 4.2.3 방장 변경 알림

방장이 퇴장할 경우 자동으로 다음 참가자가 방장이 됨

**Server → All Clients in Room**:
```json
{
  "type": "host_changed",
  "data": {
    "newHost": {
      "userId": 2,
      "nickname": "김영희"
    },
    "room": {
      "roomId": 123,
      "currentParticipants": 3,
      "host": {
        "userId": 2,
        "nickname": "김영희"
      }
    }
  }
}
```

---

### 4.3 대기실

#### 4.3.1 준비 상태 변경

**Client → Server**:
```json
{
  "type": "ready",
  "data": {
    "roomId": 123,
    "isReady": true
  }
}
```

**Server → All Clients in Room**:
```json
{
  "type": "user_ready_status",
  "data": {
    "userId": 2,
    "nickname": "김영희",
    "isReady": true,
    "readyCount": 2,
    "totalParticipants": 3
  }
}
```

---

#### 4.3.2 퀴즈 시작 (방장 전용)

**Client → Server**:
```json
{
  "type": "start_quiz",
  "data": {
    "roomId": 123
  }
}
```

**Server → All Clients in Room**:
```json
{
  "type": "quiz_started",
  "data": {
    "roomId": 123,
    "status": "IN_PROGRESS",
    "totalRounds": 10,
    "participants": [
      {
        "userId": 1,
        "nickname": "홍길동",
        "score": 0
      },
      {
        "userId": 2,
        "nickname": "김영희",
        "score": 0
      }
    ]
  }
}
```

**Possible Errors**:
- `NOT_ROOM_HOST`: 방장 권한 없음
- `ROOM_MIN_PARTICIPANTS_NOT_MET`: 최소 인원 부족 (2명 미만)
- `PARTICIPANT_NOT_READY`: 준비되지 않은 참가자 존재

---

### 4.4 퀴즈 진행

#### 4.4.1 문제 출제

퀴즈 시작 또는 다음 문제로 진행 시 자동 전송

**Server → All Clients in Room**:
```json
{
  "type": "question_presented",
  "data": {
    "round": 1,
    "totalRounds": 10,
    "word": {
      "wordId": 5,
      "title": "사랑해요"
    },
    "timeLimit": 10000
  }
}
```

---

#### 4.4.2 정답 도전 버튼 클릭

**Client → Server**:
```json
{
  "type": "answer_attempt",
  "data": {
    "roomId": 123,
    "round": 1
  }
}
```

**Server → All Clients in Room (선착순 성공)**:
```json
{
  "type": "turn_assigned",
  "data": {
    "answerer": {
      "userId": 2,
      "nickname": "김영희"
    },
    "round": 1
  }
}
```

**Server → Client (실패)**:
```json
{
  "type": "error",
  "success": false,
  "error": {
    "code": "QUIZ_BUTTON_ALREADY_TAKEN",
    "message": "다른 참가자가 먼저 버튼을 눌렀습니다."
  }
}
```

---

#### 4.4.3 카운트다운 시작

응답자가 선정되면 자동으로 카운트다운 시작

**Server → All Clients in Room**:
```json
{
  "type": "countdown_start",
  "data": {
    "answerer": {
      "userId": 2,
      "nickname": "김영희"
    },
    "countdown": 3,
    "preparationTime": 3000
  }
}
```

---

#### 4.4.4 수어 동작 수행 중

카운트다운 종료 후 자동 전환

**Server → All Clients in Room**:
```json
{
  "type": "perform_sign",
  "data": {
    "answerer": {
      "userId": 2,
      "nickname": "김영희"
    },
    "timeLimit": 10000,
    "startTime": "2025-10-12T10:30:00"
  }
}
```

---

#### 4.4.5 좌표 데이터 제출

응답자가 수어 동작을 수행한 후 좌표 데이터 전송

**Client → Server**:
```json
{
  "type": "submit_answer",
  "data": {
    "roomId": 123,
    "round": 1,
    "wordId": 5,
    "coordinates": {
      "landmarks": [
        {"x": 0.5, "y": 0.3, "z": -0.1},
        {"x": 0.52, "y": 0.32, "z": -0.12}
      ],
      "timestamp": "2025-10-12T10:30:05"
    }
  }
}
```

**Server → All Clients in Room (정답)**:
```json
{
  "type": "answer_result",
  "data": {
    "answerer": {
      "userId": 2,
      "nickname": "김영희"
    },
    "isCorrect": true,
    "similarity": 0.92,
    "scoreEarned": 100,
    "currentScore": 100,
    "round": 1,
    "ranking": [
      {
        "userId": 2,
        "nickname": "김영희",
        "score": 100,
        "rank": 1
      },
      {
        "userId": 1,
        "nickname": "홍길동",
        "score": 0,
        "rank": 2
      }
    ]
  }
}
```

**Server → All Clients in Room (오답)**:
```json
{
  "type": "answer_result",
  "data": {
    "answerer": {
      "userId": 2,
      "nickname": "김영희"
    },
    "isCorrect": false,
    "similarity": 0.45,
    "scoreEarned": -50,
    "currentScore": -50,
    "round": 1,
    "ranking": [
      {
        "userId": 1,
        "nickname": "홍길동",
        "score": 0,
        "rank": 1
      },
      {
        "userId": 2,
        "nickname": "김영희",
        "score": -50,
        "rank": 2
      }
    ]
  }
}
```

**Possible Errors**:
- `QUIZ_ANSWER_TIME_EXPIRED`: 답변 시간 만료
- `INVALID_COORDINATE_DATA`: 유효하지 않은 좌표 데이터
- `AI_MODEL_ERROR`: AI 모델 처리 중 오류

---

#### 4.4.6 다음 문제 진행

정답/오답 판정 후 자동으로 다음 문제로 진행 (3초 대기)

**Server → All Clients in Room**:
```json
{
  "type": "next_question",
  "data": {
    "nextRound": 2,
    "delayTime": 3000
  }
}
```

그 후 `question_presented` 이벤트 발생

---

### 4.5 퀴즈 종료

#### 4.5.1 퀴즈 완료

모든 라운드 종료 시 자동 전송

**Server → All Clients in Room**:
```json
{
  "type": "quiz_completed",
  "data": {
    "roomId": 123,
    "totalRounds": 10,
    "completedAt": "2025-10-12T10:45:00"
  }
}
```

---

#### 4.5.2 최종 순위 발표

**Server → All Clients in Room**:
```json
{
  "type": "final_ranking",
  "data": {
    "rankings": [
      {
        "rank": 1,
        "userId": 2,
        "nickname": "김영희",
        "finalScore": 450,
        "correctAnswers": 5,
        "wrongAnswers": 0
      },
      {
        "rank": 2,
        "userId": 1,
        "nickname": "홍길동",
        "finalScore": 380,
        "correctAnswers": 4,
        "wrongAnswers": 1
      },
      {
        "rank": 3,
        "userId": 3,
        "nickname": "박민수",
        "finalScore": 270,
        "correctAnswers": 3,
        "wrongAnswers": 0
      }
    ]
  }
}
```

---

#### 4.5.3 대기실 복귀

최종 순위 발표 후 자동으로 대기실로 복귀 (5초 대기)

**Server → All Clients in Room**:
```json
{
  "type": "return_to_lobby",
  "data": {
    "roomId": 123,
    "status": "WAITING",
    "delayTime": 5000
  }
}
```

---

### 4.6 WebRTC 시그널링

#### 4.6.1 WebRTC Offer 전송

**Client → Server**:
```json
{
  "type": "webrtc_offer",
  "data": {
    "roomId": 123,
    "targetUserId": 2,
    "sdp": "v=0\r\no=- 123456789 2 IN IP4 127.0.0.1\r\n..."
  }
}
```

**Server → Target Client**:
```json
{
  "type": "webrtc_offer",
  "data": {
    "fromUserId": 1,
    "sdp": "v=0\r\no=- 123456789 2 IN IP4 127.0.0.1\r\n..."
  }
}
```

---

#### 4.6.2 WebRTC Answer 전송

**Client → Server**:
```json
{
  "type": "webrtc_answer",
  "data": {
    "roomId": 123,
    "targetUserId": 1,
    "sdp": "v=0\r\no=- 987654321 2 IN IP4 127.0.0.1\r\n..."
  }
}
```

**Server → Target Client**:
```json
{
  "type": "webrtc_answer",
  "data": {
    "fromUserId": 2,
    "sdp": "v=0\r\no=- 987654321 2 IN IP4 127.0.0.1\r\n..."
  }
}
```

---

#### 4.6.3 ICE Candidate 교환

**Client → Server**:
```json
{
  "type": "webrtc_ice_candidate",
  "data": {
    "roomId": 123,
    "targetUserId": 2,
    "candidate": {
      "candidate": "candidate:1 1 UDP 2130706431 192.168.1.100 54321 typ host",
      "sdpMid": "0",
      "sdpMLineIndex": 0
    }
  }
}
```

**Server → Target Client**:
```json
{
  "type": "webrtc_ice_candidate",
  "data": {
    "fromUserId": 1,
    "candidate": {
      "candidate": "candidate:1 1 UDP 2130706431 192.168.1.100 54321 typ host",
      "sdpMid": "0",
      "sdpMLineIndex": 0
    }
  }
}
```

---

## 5. DTO 명세

### 5.1 Request DTOs

#### CreateRoomRequest
```java
public class CreateRoomRequest {
    @NotBlank(message = "방 제목은 필수입니다.")
    @Size(max = 50, message = "방 제목은 50자 이하여야 합니다.")
    private String gameTitle;
}
```

#### JoinRoomRequest
```java
public class JoinRoomRequest {
    @NotNull(message = "방 ID는 필수입니다.")
    private Long roomId;
}
```

#### ReadyRequest
```java
public class ReadyRequest {
    @NotNull(message = "방 ID는 필수입니다.")
    private Long roomId;
    
    @NotNull(message = "준비 상태는 필수입니다.")
    private Boolean isReady;
}
```

#### SubmitAnswerRequest
```java
public class SubmitAnswerRequest {
    @NotNull(message = "방 ID는 필수입니다.")
    private Long roomId;
    
    @NotNull(message = "라운드는 필수입니다.")
    private Integer round;
    
    @NotNull(message = "단어 ID는 필수입니다.")
    private Long wordId;
    
    @NotNull(message = "좌표 데이터는 필수입니다.")
    private CoordinateData coordinates;
}
```

#### CoordinateData
```java
public class CoordinateData {
    @NotNull(message = "랜드마크 데이터는 필수입니다.")
    @Size(min = 1, message = "최소 1개 이상의 좌표가 필요합니다.")
    private List<Landmark> landmarks;
    
    @NotNull(message = "타임스탬프는 필수입니다.")
    private LocalDateTime timestamp;
}
```

#### Landmark
```java
public class Landmark {
    @NotNull(message = "X 좌표는 필수입니다.")
    private Double x;
    
    @NotNull(message = "Y 좌표는 필수입니다.")
    private Double y;
    
    @NotNull(message = "Z 좌표는 필수입니다.")
    private Double z;
}
```

#### SaveQuizResultRequest
```java
public class SaveQuizResultRequest {
    @NotNull(message = "방 ID는 필수입니다.")
    private Long roomId;
    
    @NotNull(message = "참가자 점수 목록은 필수입니다.")
    @Size(min = 2, max = 4, message = "참가자는 2~4명이어야 합니다.")
    private List<ParticipantScore> participantScores;
}
```

#### ParticipantScore
```java
public class ParticipantScore {
    @NotNull(message = "사용자 ID는 필수입니다.")
    private Long userId;
    
    @NotNull(message = "점수는 필수입니다.")
    private Integer score;
    
    @NotNull(message = "라운드는 필수입니다.")
    private Integer round;
}
```

---

### 5.2 Response DTOs

#### RoomResponse
```java
public class RoomResponse {
    private Long roomId;
    private String gameTitle;
    private UserSimpleResponse host;
    private Integer maxParticipants;
    private Integer currentParticipants;
    private GameRoomStatus status;
    private LocalDateTime createdAt;
}
```

#### RoomDetailResponse
```java
public class RoomDetailResponse {
    private Long roomId;
    private String gameTitle;
    private UserSimpleResponse host;
    private List<ParticipantResponse> participants;
    private Integer currentParticipants;
    private Integer maxParticipants;
    private GameRoomStatus status;
    private LocalDateTime createdAt;
}
```

#### RoomListResponse
```java
public class RoomListResponse {
    private Long roomId;
    private String gameTitle;
    private String hostNickname;
    private Integer currentParticipants;
    private Integer maxParticipants;
    private GameRoomStatus status;
    private LocalDateTime createdAt;
}
```

#### UserSimpleResponse
```java
public class UserSimpleResponse {
    private Long userId;
    private String nickname;
    private String email; // optional
}
```

#### ParticipantResponse
```java
public class ParticipantResponse {
    private Long userId;
    private String nickname;
    private Boolean isHost;
    private Boolean isReady;
    private LocalDateTime joinedAt;
}
```

#### WordResponse
```java
public class WordResponse {
    private Long wordId;
    private String title;
    private String videoUrl;
    private String signDescription;
    private String categoryType;
}
```

#### QuizHistoryResponse
```java
public class QuizHistoryResponse {
    private Long historyId;
    private String roomTitle;
    private Integer score;
    private Integer round;
    private Integer rank;
    private Integer totalParticipants;
    private LocalDateTime createdAt;
}
```

#### RankingResponse
```java
public class RankingResponse {
    private Integer rank;
    private Long userId;
    private String nickname;
    private Integer finalScore;
    private Integer correctAnswers;
    private Integer wrongAnswers;
}
```

---

### 5.3 WebSocket Message DTOs

#### WebSocketMessage (Base)
```java
public class WebSocketMessage<T> {
    private String type;
    private Boolean success;
    private T data;
    private ErrorData error;
}
```

#### ErrorData
```java
public class ErrorData {
    private String code;
    private String message;
}
```

#### AuthenticateData
```java
public class AuthenticateData {
    private String token;
}
```

#### JoinRoomData
```java
public class JoinRoomData {
    private Long roomId;
}
```

#### UserJoinedData
```java
public class UserJoinedData {
    private UserSimpleResponse user;
    private RoomStateData room;
}
```

#### RoomStateData
```java
public class RoomStateData {
    private Long roomId;
    private Integer currentParticipants;
    private List<ParticipantResponse> participants;
}
```

#### ReadyStatusData
```java
public class ReadyStatusData {
    private Long userId;
    private String nickname;
    private Boolean isReady;
    private Integer readyCount;
    private Integer totalParticipants;
}
```

#### QuizStartedData
```java
public class QuizStartedData {
    private Long roomId;
    private GameRoomStatus status;
    private Integer totalRounds;
    private List<ParticipantScoreData> participants;
}
```

#### ParticipantScoreData
```java
public class ParticipantScoreData {
    private Long userId;
    private String nickname;
    private Integer score;
}
```

#### QuestionData
```java
public class QuestionData {
    private Integer round;
    private Integer totalRounds;
    private WordSimpleData word;
    private Integer timeLimit; // milliseconds
}
```

#### WordSimpleData
```java
public class WordSimpleData {
    private Long wordId;
    private String title;
}
```

#### TurnAssignedData
```java
public class TurnAssignedData {
    private UserSimpleResponse answerer;
    private Integer round;
}
```

#### CountdownData
```java
public class CountdownData {
    private UserSimpleResponse answerer;
    private Integer countdown; // seconds
    private Integer preparationTime; // milliseconds
}
```

#### PerformSignData
```java
public class PerformSignData {
    private UserSimpleResponse answerer;
    private Integer timeLimit; // milliseconds
    private LocalDateTime startTime;
}
```

#### AnswerResultData
```java
public class AnswerResultData {
    private UserSimpleResponse answerer;
    private Boolean isCorrect;
    private Double similarity;
    private Integer scoreEarned;
    private Integer currentScore;
    private Integer round;
    private List<RankingData> ranking;
}
```

#### RankingData
```java
public class RankingData {
    private Long userId;
    private String nickname;
    private Integer score;
    private Integer rank;
}
```

#### FinalRankingData
```java
public class FinalRankingData {
    private List<RankingResponse> rankings;
}
```

#### WebRTCSignalData
```java
public class WebRTCSignalData {
    private Long roomId;
    private Long targetUserId;
    private Long fromUserId;
    private String sdp;
    private ICECandidateData candidate;
}
```

#### ICECandidateData
```java
public class ICECandidateData {
    private String candidate;
    private String sdpMid;
    private Integer sdpMLineIndex;
}
```

---

## 6. 에러 코드

### 6.1 인증/회원 관련 (401, 403, 404, 409)

| 에러 코드 | HTTP Status | 메시지 | 설명 |
|---------|-------------|--------|------|
| `UNAUTHORIZED` | 401 | 인증되지 않은 사용자입니다. | 토큰이 없거나 유효하지 않음 |
| `INVALID_TOKEN` | 401 | 유효하지 않은 토큰입니다. | 토큰 형식이 잘못됨 |
| `EXPIRED_TOKEN` | 401 | 만료된 토큰입니다. | 토큰 만료 |
| `USER_NOT_FOUND` | 404 | 사용자를 찾을 수 없습니다. | 존재하지 않는 사용자 |

---

### 6.2 퀴즈 방 관련 (400, 404)

| 에러 코드 | HTTP Status | 메시지 | 설명 |
|---------|-------------|--------|------|
| `ROOM_NOT_FOUND` | 404 | 퀴즈 방을 찾을 수 없습니다. | 존재하지 않는 방 |
| `ROOM_FULL` | 400 | 방 인원이 가득 찼습니다. | 최대 인원(4명) 초과 |
| `ROOM_ALREADY_STARTED` | 400 | 이미 시작된 방입니다. | 진행 중인 방에 입장 시도 |
| `ROOM_MIN_PARTICIPANTS_NOT_MET` | 400 | 퀴즈 시작을 위한 최소 인원이 부족합니다. | 2명 미만일 때 시작 시도 |
| `ROOM_MAX_PARTICIPANTS_EXCEEDED` | 400 | 최대 인원을 초과했습니다. | 4명 초과 |
| `INVALID_ROOM_STATUS` | 400 | 유효하지 않은 방 상태입니다. | 잘못된 상태 전환 |
| `ROOM_ALREADY_FINISHED` | 400 | 이미 종료된 방입니다. | 종료된 방 재입장 시도 |

---

### 6.3 참가자 관련 (400, 403, 404, 409)

| 에러 코드 | HTTP Status | 메시지 | 설명 |
|---------|-------------|--------|------|
| `PARTICIPANT_NOT_FOUND` | 404 | 참가자를 찾을 수 없습니다. | 존재하지 않는 참가자 |
| `PARTICIPANT_ALREADY_IN_ROOM` | 409 | 이미 방에 참여 중입니다. | 다른 방에 참여 중 |
| `NOT_ROOM_HOST` | 403 | 방장 권한이 없습니다. | 방장만 가능한 작업 시도 |
| `CAMERA_PERMISSION_REQUIRED` | 403 | 카메라 권한이 필요합니다. | 카메라 권한 없이 입장 시도 |
| `PARTICIPANT_NOT_IN_ROOM` | 400 | 방에 참여하지 않은 사용자입니다. | 방에 없는 사용자의 작업 시도 |
| `PARTICIPANT_NOT_READY` | 400 | 참가자가 준비되지 않았습니다. | 준비 안 된 상태에서 시작 |

---

### 6.4 단어/학습 관련 (400, 404)

| 에러 코드 | HTTP Status | 메시지 | 설명 |
|---------|-------------|--------|------|
| `WORD_NOT_FOUND` | 404 | 단어를 찾을 수 없습니다. | 존재하지 않는 단어 |
| `WORD_LIST_EMPTY` | 404 | 학습 가능한 단어가 없습니다. | 단어 목록이 비어있음 |
| `INVALID_WORD_ID` | 400 | 유효하지 않은 단어 ID입니다. | 잘못된 단어 ID |

---

### 6.5 퀴즈 관련 (400, 409)

| 에러 코드 | HTTP Status | 메시지 | 설명 |
|---------|-------------|--------|------|
| `QUIZ_NOT_STARTED` | 400 | 퀴즈가 시작되지 않았습니다. | 시작 전 답변 시도 |
| `QUIZ_ALREADY_FINISHED` | 400 | 이미 종료된 퀴즈입니다. | 종료된 퀴즈 진행 시도 |
| `QUIZ_ANSWER_TIME_EXPIRED` | 400 | 답변 시간이 만료되었습니다. | 제한 시간 초과 |
| `ALREADY_ANSWERED` | 400 | 이미 답변한 문제입니다. | 중복 답변 시도 |
| `QUIZ_BUTTON_ALREADY_TAKEN` | 409 | 다른 참가자가 먼저 버튼을 눌렀습니다. | 선착순 실패 |
| `INVALID_QUIZ_QUESTION` | 400 | 유효하지 않은 퀴즈 문제입니다. | 잘못된 문제 데이터 |

---

### 6.6 AI/좌표 처리 관련 (400, 500, 504)

| 에러 코드 | HTTP Status | 메시지 | 설명 |
|---------|-------------|--------|------|
| `COORDINATE_EXTRACTION_FAILED` | 500 | 좌표 추출에 실패했습니다. | MediaPipe 처리 실패 |
| `AI_MODEL_ERROR` | 500 | AI 모델 처리 중 오류가 발생했습니다. | 모델 서버 오류 |
| `SIMILARITY_CHECK_FAILED` | 500 | 유사도 검사에 실패했습니다. | 유사도 계산 실패 |
| `INVALID_COORDINATE_DATA` | 400 | 유효하지 않은 좌표 데이터입니다. | 잘못된 좌표 형식 |
| `AI_MODEL_TIMEOUT` | 504 | AI 모델 응답 시간이 초과되었습니다. | 모델 응답 지연 |

---

### 6.7 WebRTC 관련 (500)

| 에러 코드 | HTTP Status | 메시지 | 설명 |
|---------|-------------|--------|------|
| `WEBRTC_CONNECTION_FAILED` | 500 | WebRTC 연결에 실패했습니다. | 연결 실패 |
| `SIGNALING_ERROR` | 500 | 시그널링 오류가 발생했습니다. | 시그널링 처리 오류 |
| `PEER_CONNECTION_ERROR` | 500 | 피어 연결 오류가 발생했습니다. | P2P 연결 오류 |
| `NETWORK_DISCONNECTED` | 500 | 네트워크 연결이 끊어졌습니다. | 네트워크 단절 |

---

### 6.8 공통 에러 (400, 403, 404, 405, 500, 503)

| 에러 코드 | HTTP Status | 메시지 | 설명 |
|---------|-------------|--------|------|
| `BAD_REQUEST` | 400 | 잘못된 요청입니다. | 일반적인 잘못된 요청 |
| `VALIDATION_ERROR` | 400 | 유효성 검사에 실패했습니다. | @Valid 검증 실패 |
| `FORBIDDEN` | 403 | 접근 권한이 없습니다. | 권한 부족 |
| `RESOURCE_NOT_FOUND` | 404 | 요청한 리소스를 찾을 수 없습니다. | 일반적인 404 |
| `METHOD_NOT_ALLOWED` | 405 | 허용되지 않은 메소드입니다. | 잘못된 HTTP 메소드 |
| `INTERNAL_SERVER_ERROR` | 500 | 서버 내부 오류가 발생했습니다. | 예상치 못한 서버 오류 |
| `SERVICE_UNAVAILABLE` | 503 | 서비스를 사용할 수 없습니다. | 서비스 점검 중 |

---

## 7. 시퀀스 다이어그램

### 7.1 방 생성 및 입장 플로우

```
Client A                 Backend                 Database
   |                        |                        |
   |--POST /api/quiz/rooms->|                        |
   |                        |--INSERT game_room----->|
   |                        |--INSERT game_participant->|
   |<---201 Created---------|                        |
   |                        |                        |
Client B                    |                        |
   |--POST /api/quiz/rooms->|                        |
   |(동일 방 입장 시도)       |                        |
   |                        |--SELECT game_room----->|
   |                        |--CHECK current_participants->|
   |<---409 ROOM_FULL-------|                        |
```

---

### 7.2 WebSocket 연결 및 인증 플로우

```
Client                  WebSocket Server           Auth Service
   |                        |                        |
   |--Connect WS----------->|                        |
   |<--Connection Opened----|                        |
   |                        |                        |
   |--authenticate--------->|                        |
   |   {token: "..."}       |--Validate Token------->|
   |                        |<--User Info------------|
   |<--authenticated--------|                        |
   |   {userId, nickname}   |                        |
```

---

### 7.3 퀴즈 진행 전체 플로우

```
Client A(방장)  Client B    Backend        AI Service    Database
    |            |            |                |            |
    |--start_quiz------------>|                |            |
    |            |            |--Check ready-->|            |
    |            |            |--UPDATE status->           |
    |<--quiz_started----------|                |            |
    |            |<-----------|                |            |
    |            |            |                |            |
    |            |            |--Random word-->|            |
    |<--question_presented----|                |            |
    |            |<-----------|                |            |
    |            |            |                |            |
    |            |--answer_attempt------------>|            |
    |<--turn_assigned---------|                |            |
    |            |<-----------|                |            |
    |            |            |                |            |
    |            |<--countdown_start-----------|            |
    |            |<--perform_sign--------------|            |
    |            |            |                |            |
    |            |--submit_answer------------->|            |
    |            |            |--Validate----->|            |
    |            |            |<--Similarity---|            |
    |            |            |--UPDATE score->            |
    |<--answer_result---------|                |            |
    |            |<-----------|                |            |
    |            |            |                |            |
    |<--next_question---------|                |            |
    |            |<-----------|                |            |
    |            |            |                |            |
    ... (반복) ...            |                |            |
    |            |            |                |            |
    |<--quiz_completed--------|                |            |
    |            |<-----------|                |            |
    |<--final_ranking---------|                |            |
    |            |<-----------|                |            |
    |            |            |--INSERT history------------>|
    |<--return_to_lobby-------|                |            |
    |            |<-----------|                |            |
```

---

### 7.4 선착순 버튼 경쟁 상황

```
Client A     Client B     Client C     Backend
    |            |            |            |
    |--answer_attempt-------->|            |
    |            |--answer_attempt-------->|
    |            |            |--answer_attempt->|
    |            |            |            |
    |            |            |    [동시성 제어]
    |            |            |    1. A 먼저 도착
    |            |            |    2. B, C 거부
    |            |            |            |
    |<--turn_assigned---------|            |
    |            |<--error: BUTTON_TAKEN---|
    |            |            |<--error----|
```

---

### 7.5 방장 퇴장 시 방장 승계

```
Client A(방장) Client B   Client C    Backend      Database
    |            |            |            |            |
    |--leave_room------------>|            |            |
    |            |            |            |--SELECT next host->|
    |            |            |            |--UPDATE host_id--->|
    |            |<--host_changed----------|            |
    |            |            |<-----------|            |
    |            |            |            |            |
    |            |<--user_left-------------|            |
    |            |            |<-----------|            |
```

---

## 8. 구현 가이드

### 8.1 WebSocket 상태 관리

#### 서버 측 상태 관리 필요 항목

1. **연결된 사용자 맵**
    - `Map<String, WebSocketSession>` - sessionId → WebSocket Session
    - `Map<Long, String>` - userId → sessionId

2. **방별 참가자 관리**
    - `Map<Long, Set<Long>>` - roomId → Set of userIds
    - `Map<Long, GameState>` - roomId → 현재 게임 상태

3. **게임 상태 관리**
   ```java
   class GameState {
       Long roomId;
       Integer currentRound;
       Long currentAnswerer;
       LocalDateTime questionStartTime;
       Map<Long, Integer> scores;
       List<Long> questionOrder;
   }
   ```

---

### 8.2 동시성 제어

#### 선착순 버튼 처리

```java
@Service
public class QuizService {
    
    private final Map<Long, AtomicBoolean> roomLocks = new ConcurrentHashMap<>();
    
    public boolean tryAnswerAttempt(Long roomId, Long userId) {
        AtomicBoolean lock = roomLocks.computeIfAbsent(roomId, 
            k -> new AtomicBoolean(false));
        
        // CAS(Compare-And-Set) 연산으로 원자적 처리
        boolean success = lock.compareAndSet(false, true);
        
        if (success) {
            // 응답자 설정
            setCurrentAnswerer(roomId, userId);
            return true;
        }
        
        return false;
    }
    
    public void releaseAnswerLock(Long roomId) {
        AtomicBoolean lock = roomLocks.get(roomId);
        if (lock != null) {
            lock.set(false);
        }
    }
}
```

---

### 8.3 타임아웃 처리

#### 답변 시간 초과 처리

```java
@Service
public class QuizTimeoutService {
    
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(10);
    
    public void scheduleAnswerTimeout(Long roomId, Long userId, int timeoutMs) {
        scheduler.schedule(() -> {
            handleAnswerTimeout(roomId, userId);
        }, timeoutMs, TimeUnit.MILLISECONDS);
    }
    
    private void handleAnswerTimeout(Long roomId, Long userId) {
        // 자동 오답 처리
        GameState state = getGameState(roomId);
        if (state.getCurrentAnswerer().equals(userId)) {
            processWrongAnswer(roomId, userId);
            moveToNextQuestion(roomId);
        }
    }
}
```

---

### 8.4 점수 계산 로직

```java
@Service
public class ScoreCalculator {
    
    private static final int SCORE_FIRST = 100;
    private static final int SCORE_SECOND = 90;
    private static final int SCORE_THIRD = 80;
    private static final int SCORE_FOURTH = 70;
    private static final int PENALTY_WRONG = -50;
    
    public int calculateScore(int rank, boolean isCorrect) {
        if (!isCorrect) {
            return PENALTY_WRONG;
        }
        
        return switch (rank) {
            case 1 -> SCORE_FIRST;
            case 2 -> SCORE_SECOND;
            case 3 -> SCORE_THIRD;
            case 4 -> SCORE_FOURTH;
            default -> 0;
        };
    }
}
```

---

### 8.5 AI 모델 연동

#### 좌표 데이터 검증 및 AI 호출

```java
@Service
public class AIValidationService {
    
    @Value("${ai.model.url}")
    private String aiModelUrl;
    
    @Value("${ai.model.timeout}")
    private int timeout;
    
    public AIValidationResponse validateSign(Long wordId, CoordinateData coordinates) {
        try {
            // AI 모델 서버로 HTTP 요청
            RestTemplate restTemplate = new RestTemplate();
            
            AIValidationRequest request = AIValidationRequest.builder()
                .wordId(wordId)
                .coordinates(coordinates)
                .build();
            
            ResponseEntity<AIValidationResponse> response = restTemplate
                .postForEntity(aiModelUrl + "/validate", request, 
                    AIValidationResponse.class);
            
            return response.getBody();
            
        } catch (RestClientException e) {
            log.error("AI 모델 호출 실패", e);
            throw new BusinessException(ErrorCode.AI_MODEL_ERROR);
        }
    }
}
```

---

### 8.6 WebRTC 시그널링 구현

```java
@Component
public class WebRTCSignalingHandler {
    
    public void handleOffer(WebSocketSession session, WebRTCSignalData data) {
        // targetUserId에게 Offer 전달
        WebSocketSession targetSession = getSession(data.getTargetUserId());
        
        if (targetSession != null && targetSession.isOpen()) {
            WebSocketMessage<WebRTCSignalData> message = WebSocketMessage.builder()
                .type("webrtc_offer")
                .data(WebRTCSignalData.builder()
                    .fromUserId(data.getFromUserId())
                    .sdp(data.getSdp())
                    .build())
                .build();
            
            sendMessage(targetSession, message);
        }
    }
    
    public void handleAnswer(WebSocketSession session, WebRTCSignalData data) {
        // targetUserId에게 Answer 전달
        WebSocketSession targetSession = getSession(data.getTargetUserId());
        
        if (targetSession != null && targetSession.isOpen()) {
            WebSocketMessage<WebRTCSignalData> message = WebSocketMessage.builder()
                .type("webrtc_answer")
                .data(WebRTCSignalData.builder()
                    .fromUserId(data.getFromUserId())
                    .sdp(data.getSdp())
                    .build())
                .build();
            
            sendMessage(targetSession, message);
        }
    }
    
    public void handleICECandidate(WebSocketSession session, 
                                   WebRTCSignalData data) {
        // targetUserId에게 ICE Candidate 전달
        WebSocketSession targetSession = getSession(data.getTargetUserId());
        
        if (targetSession != null && targetSession.isOpen()) {
            WebSocketMessage<WebRTCSignalData> message = WebSocketMessage.builder()
                .type("webrtc_ice_candidate")
                .data(WebRTCSignalData.builder()
                    .fromUserId(data.getFromUserId())
                    .candidate(data.getCandidate())
                    .build())
                .build();
            
            sendMessage(targetSession, message);
        }
    }
}
```

---

## 9. 테스트 케이스

### 9.1 방 생성/입장 테스트

#### TC-001: 정상적인 방 생성
- **Given**: 인증된 사용자
- **When**: POST /api/quiz/rooms with valid gameTitle
- **Then**: 201 Created, roomId 반환

#### TC-002: 중복 방 참여 시도
- **Given**: 이미 방에 참여 중인 사용자
- **When**: POST /api/quiz/rooms (새 방 생성 시도)
- **Then**: 409 PARTICIPANT_ALREADY_IN_ROOM

#### TC-003: 방 정원 초과
- **Given**: 4명이 가득 찬 방
- **When**: join_room 시도
- **Then**: error: ROOM_FULL

---

### 9.2 퀴즈 진행 테스트

#### TC-004: 최소 인원 미달 시작 시도
- **Given**: 1명만 있는 방
- **When**: start_quiz 시도
- **Then**: error: ROOM_MIN_PARTICIPANTS_NOT_MET

#### TC-005: 선착순 버튼 경쟁
- **Given**: 문제 출제 후 3명의 사용자
- **When**: 동시에 answer_attempt
- **Then**: 1명만 turn_assigned, 나머지는 QUIZ_BUTTON_ALREADY_TAKEN

#### TC-006: 답변 시간 초과
- **Given**: 응답자로 선정된 사용자
- **When**: 제한 시간(10초) 내 답변 미제출
- **Then**: 자동 오답 처리 후 next_question

---

### 9.3 AI 검증 테스트

#### TC-007: 정상적인 수어 동작 정답
- **Given**: 올바른 좌표 데이터
- **When**: submit_answer with valid coordinates
- **Then**: isCorrect: true, similarity >= 0.7, score += 100

#### TC-008: 수어 동작 오답
- **Given**: 잘못된 좌표 데이터
- **When**: submit_answer with incorrect coordinates
- **Then**: isCorrect: false, similarity < 0.7, score -= 50

#### TC-009: 유효하지 않은 좌표 데이터
- **Given**: 빈 landmarks 배열
- **When**: submit_answer with empty coordinates
- **Then**: error: INVALID_COORDINATE_DATA

---

### 9.4 방장 권한 테스트

#### TC-010: 방장이 아닌 사용자의 시작 시도
- **Given**: 일반 참가자
- **When**: start_quiz 시도
- **Then**: error: NOT_ROOM_HOST

#### TC-011: 방장 퇴장 후 승계
- **Given**: 방장 포함 3명의 방
- **When**: 방장이 leave_room
- **Then**: host_changed 이벤트, 다음 참가자가 방장

#### TC-012: 방장의 방 삭제
- **Given**: 방장
- **When**: DELETE /api/quiz/rooms/{roomId}
- **Then**: 200 OK, 모든 참가자에게 연결 종료

---

### 9.5 WebSocket 연결 테스트

#### TC-013: 유효하지 않은 토큰으로 인증
- **Given**: 잘못된 JWT 토큰
- **When**: authenticate 메시지 전송
- **Then**: error: INVALID_TOKEN, 연결 종료

#### TC-014: 네트워크 단절 후 재연결
- **Given**: 퀴즈 진행 중인 사용자
- **When**: WebSocket 연결 끊김
- **Then**: 자동 재연결 시도, 상태 복구

#### TC-015: 동일 사용자 중복 연결
- **Given**: 이미 연결된 사용자
- **When**: 새로운 WebSocket 연결 시도
- **Then**: 기존 연결 종료, 새 연결 수립

---

## 10. 성능 요구사항

### 10.1 응답 시간

| API 종류 | 목표 응답 시간 | 최대 허용 시간 |
|---------|-------------|-------------|
| REST API (조회) | < 200ms | < 500ms |
| REST API (생성/수정) | < 300ms | < 1000ms |
| WebSocket 메시지 전달 | < 50ms | < 100ms |
| AI 모델 유사도 검증 | < 2000ms | < 3000ms |

---

### 10.2 동시 접속

- **목표**: 동시 접속자 1,000명
- **동시 퀴즈 방**: 최대 250개 (각 방 4명)
- **WebSocket 연결**: 1,000개 유지

---

### 10.3 처리량

- **초당 REST API 요청**: 500 req/s
- **초당 WebSocket 메시지**: 2,000 msg/s
- **AI 모델 검증**: 초당 50건

---

## 11. 보안 고려사항

### 11.1 인증/인가

1. **JWT 토큰 검증**
    - 모든 REST API 요청 시 Bearer 토큰 필수
    - WebSocket 연결 시 첫 메시지로 인증 수행

2. **방장 권한 검증**
    - 퀴즈 시작, 방 삭제 등 방장 전용 기능 호출 시 권한 확인

3. **참가자 검증**
    - 방에 속한 참가자만 해당 방의 WebSocket 메시지 수신

---

### 11.2 입력값 검증

1. **DTO Validation**
    - `@Valid`, `@NotNull`, `@Size` 등 어노테이션 활용
    - 유효하지 않은 입력 시 400 VALIDATION_ERROR 반환

2. **좌표 데이터 검증**
    - 좌표 범위 확인 (x, y, z: -1.0 ~ 1.0)
    - 최소/최대 랜드마크 개수 확인

3. **SQL Injection 방지**
    - JPA/Hibernate 사용으로 Prepared Statement 자동 적용

---

### 11.3 Rate Limiting

1. **API Rate Limit**
    - 사용자당 분당 60회 요청 제한
    - 초과 시 429 Too Many Requests

2. **WebSocket Rate Limit**
    - 초당 100개 메시지 제한
    - 초과 시 연결 일시 차단

---

### 11.4 데이터 보호

1. **개인정보 암호화**
    - 민감 정보(이메일) DB 암호화 저장

2. **HTTPS 필수**
    - 모든 통신 TLS 1.3 사용

3. **CORS 설정**
    - 허용된 도메인만 접근 가능

---

## 12. 모니터링 및 로깅

### 12.1 로깅 전략

#### 로그 레벨

- **ERROR**: 시스템 오류, 예외 발생
- **WARN**: 비정상 동작, 재시도 필요
- **INFO**: 주요 이벤트 (방 생성, 퀴즈 시작/종료)
- **DEBUG**: 상세 처리 과정

#### 로깅 항목

```java
log.info("[QUIZ_START] roomId={}, hostId={}, participants={}", 
    roomId, hostId, participantCount);

log.info("[ANSWER_SUBMIT] roomId={}, userId={}, wordId={}, round={}, similarity={}", 
    roomId, userId, wordId, round, similarity);

log.error("[AI_ERROR] roomId={}, userId={}, error={}", 
    roomId, userId, e.getMessage(), e);
```

---

### 12.2 모니터링 메트릭

#### 시스템 메트릭

- **CPU/메모리 사용률**
- **WebSocket 연결 수**
- **활성 퀴즈 방 수**

#### 비즈니스 메트릭

- **분당 생성되는 방 수**
- **평균 퀴즈 진행 시간**
- **AI 모델 평균 응답 시간**
- **오답률/정답률**

#### 에러 메트릭

- **에러 발생 빈도 (에러 코드별)**
- **AI 모델 실패율**
- **WebSocket 연결 끊김 빈도**

---

## 13. 배포 및 환경 설정

### 13.1 환경별 설정

#### application-dev.yml (개발)
```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/signbell_dev
    username: dev_user
    password: dev_password

websocket:
  max-connections: 100

ai:
  model:
    url: http://localhost:5000
    timeout: 3000
```

#### application-prod.yml (운영)
```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://prod-db.signbell.com:3306/signbell
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5

websocket:
  max-connections: 1000

ai:
  model:
    url: https://ai.signbell.com
    timeout: 3000

logging:
  level:
    root: INFO
    app.signbell: DEBUG
```

---

### 13.2 인프라 구성

#### 권장 서버 스펙

- **API 서버**: 4 Core CPU, 8GB RAM, 2대 (로드 밸런싱)
- **WebSocket 서버**: 4 Core CPU, 8GB RAM, 2대
- **Database**: RDS MySQL 8.0, 4 Core, 16GB RAM
- **AI 모델 서버**: GPU 지원, 8 Core CPU, 16GB RAM

#### 로드 밸런서 설정

- **알고리즘**: Least Connection (WebSocket 고려)
- **Health Check**: GET /health (5초 간격)
- **Sticky Session**: 활성화 (WebSocket 유지)

---

## 14. 참고 문서

### 14.1 관련 기술 문서

- [WebSocket RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)
- [WebRTC Specification](https://www.w3.org/TR/webrtc/)
- [JWT RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)
- [MediaPipe Documentation](https://google.github.io/mediapipe/)

### 14.2 프로젝트 내부 문서

- [SignBell 기능 요구사항 명세서](링크)
- [SignBell 기술 스택 문서](https://github.com/SynergySign/SignBell-Docs/blob/76bac52ce1f1565a8840c33b979cac373289126c/02_Architecture/SignBell%20-%20%EA%B8%B0%EC%88%A0%EC%8A%A4%ED%83%9D.md)
- [SignBell 시스템 아키텍처](https://github.com/SynergySign/SignBell-Docs/blob/76bac52ce1f1565a8840c33b979cac373289126c/02_Architecture/SignBell%20%EC%8B%9C%EC%8A%A4%ED%85%9C%20%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98.md)
- [AI 모델 명세서](링크)
- [프론트엔드 API 연동 가이드](링크)

---

## 15. 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| v1.0.0 | 2025-10-12 | 초기 API 명세서 작성 | 강관주 |

---

## 16. FAQ

### Q1. WebSocket 연결이 끊어지면 어떻게 되나요?
**A**: 클라이언트는 자동으로 재연결을 시도합니다. 퀴즈 진행 중이었다면 현재 라운드 정보를 유지하고, 재연결 시 `room_state_sync` 이벤트를 통해 현재 상태를 동기화합니다.

### Q2. 동시에 여러 명이 버튼을 누르면 어떻게 처리하나요?
**A**: 서버 측에서 `AtomicBoolean`과 CAS(Compare-And-Set) 연산을 사용하여 원자적으로 처리합니다. 가장 먼저 도착한 요청만 성공하고, 나머지는 `QUIZ_BUTTON_ALREADY_TAKEN` 에러를 받습니다.

### Q3. AI 모델이 응답하지 않으면 어떻게 되나요?
**A**: 3초 타임아웃이 설정되어 있습니다. 타임아웃 발생 시 `AI_MODEL_TIMEOUT` 에러를 반환하고, 해당 답변은 자동으로 오답 처리됩니다.

### Q4. 방장이 나가면 방이 사라지나요?
**A**: 아니요. 방장이 나가면 다음 참가자가 자동으로 방장으로 승계됩니다. 모든 참가자가 나가면 방이 자동으로 삭제됩니다.

### Q5. 퀴즈 도중에 나갔다가 다시 들어올 수 있나요?
**A**: MVP 단계에서는 퀴즈 진행 중 재입장이 불가능합니다. 퀴즈가 끝난 후 대기실 상태로 돌아왔을 때만 재입장 가능합니다.

### Q6. 좌표 데이터는 어떤 형식으로 전송하나요?
**A**: MediaPipe에서 추출한 랜드마크 좌표(x, y, z)를 JSON 배열 형식으로 전송합니다. 각 좌표는 -1.0 ~ 1.0 범위의 정규화된 값입니다.

### Q7. 점수 계산은 어떻게 되나요?
**A**: 정답 시 순위별 차등 점수(1위 100점, 2위 90점, 3위 80점, 4위 70점), 오답 시 -50점입니다. 최종 순위는 누적 점수로 결정됩니다.

### Q8. REST API와 WebSocket 중 어느 것을 사용해야 하나요?
**A**:
- **REST API**: 방 생성/조회, 단어 목록, 기록 조회 등 상태 조회/변경
- **WebSocket**: 실시간 게임 진행, 참가자 간 상태 동기화, WebRTC 시그널링

---

## 부록 A. WebSocket 이벤트 요약표

| 이벤트명 | 방향 | 설명 | 트리거 시점 |
|---------|------|------|-----------|
| `authenticate` | C→S | 인증 요청 | WebSocket 연결 직후 |
| `authenticated` | S→C | 인증 성공 | 토큰 검증 성공 시 |
| `join_room` | C→S | 방 입장 요청 | 사용자가 방 입장 시도 |
| `user_joined` | S→All | 사용자 입장 알림 | 방 입장 성공 시 |
| `leave_room` | C→S | 방 퇴장 요청 | 사용자가 방 나가기 |
| `user_left` | S→All | 사용자 퇴장 알림 | 방 퇴장 시 |
| `host_changed` | S→All | 방장 변경 알림 | 방장 퇴장 시 |
| `ready` | C→S | 준비 상태 변경 | 준비 버튼 클릭 |
| `user_ready_status` | S→All | 준비 상태 알림 | 준비 상태 변경 시 |
| `start_quiz` | C→S | 퀴즈 시작 요청 | 방장이 시작 버튼 클릭 |
| `quiz_started` | S→All | 퀴즈 시작 알림 | 퀴즈 시작 시 |
| `question_presented` | S→All | 문제 출제 | 라운드 시작 시 |
| `answer_attempt` | C→S | 답변 도전 | 버튼 클릭 |
| `turn_assigned` | S→All | 응답자 선정 | 선착순 성공 시 |
| `countdown_start` | S→All | 카운트다운 시작 | 응답자 선정 후 |
| `perform_sign` | S→All | 수어 수행 시작 | 카운트다운 종료 후 |
| `submit_answer` | C→S | 좌표 데이터 제출 | 수어 동작 완료 후 |
| `answer_result` | S→All | 정답/오답 결과 | AI 검증 완료 후 |
| `next_question` | S→All | 다음 문제 진행 | 결과 표시 3초 후 |
| `quiz_completed` | S→All | 퀴즈 종료 | 모든 라운드 완료 |
| `final_ranking` | S→All | 최종 순위 발표 | 퀴즈 종료 직후 |
| `return_to_lobby` | S→All | 대기실 복귀 | 순위 발표 5초 후 |
| `webrtc_offer` | C→S, S→C | WebRTC Offer | 피어 연결 시작 |
| `webrtc_answer` | C→S, S→C | WebRTC Answer | Offer 수신 후 |
| `webrtc_ice_candidate` | C→S, S→C | ICE Candidate | ICE 수집 중 |
| `error` | S→C | 에러 발생 | 에러 발생 시 |

**범례**: C=Client, S=Server, All=Room 내 모든 클라이언트

---

## 부록 B. HTTP 상태 코드 요약

| 상태 코드 | 의미 | 사용 예시 |
|----------|------|----------|
| 200 OK | 요청 성공 | GET, DELETE 성공 |
| 201 Created | 리소스 생성 성공 | POST 방 생성 |
| 400 Bad Request | 잘못된 요청 | 유효성 검증 실패 |
| 401 Unauthorized | 인증 실패 | 토큰 없음/만료 |
| 403 Forbidden | 권한 없음 | 방장 권한 부족 |
| 404 Not Found | 리소스 없음 | 방/단어 찾을 수 없음 |
| 409 Conflict | 리소스 충돌 | 중복 방 참여 시도 |
| 500 Internal Server Error | 서버 오류 | 예상치 못한 오류 |
| 503 Service Unavailable | 서비스 이용 불가 | 점검 중 |
| 504 Gateway Timeout | 타임아웃 | AI 모델 응답 지연 |

