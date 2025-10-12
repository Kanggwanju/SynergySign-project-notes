# SignBell 실시간 퀴즈 백엔드 API 명세서 (STOMP)

## 문서 정보

- **작성자**: 강관주
- **작성일**: 2025-10-12
- **버전**: v2.0.0 (STOMP 방식)
- **대상**: 실시간 퀴즈 기능 백엔드 API

---

## 목차

1. [개요](#1-개요)
2. [인증 방식](#2-인증-방식)
3. [REST API](#3-rest-api)
4. [WebSocket API (STOMP)](#4-websocket-api-stomp)
5. [DTO 명세](#5-dto-명세)
6. [에러 코드](#6-에러-코드)
7. [시퀀스 다이어그램](#7-시퀀스-다이어그램)
8. [구현 가이드](#8-구현-가이드)

---

## 1. 개요

### 1.1 API 기본 정보

- **Base URL**: `https://api.signbell.com`
- **WebSocket URL**: `wss://api.signbell.com/ws/quiz`
- **Protocol**: REST + STOMP over WebSocket
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

### 2.2 WebSocket (STOMP) 인증

WebSocket 연결 시 핸드셰이크에서 JWT 토큰 검증:

```javascript
const socket = new SockJS('http://localhost:8080/ws/quiz');
const stompClient = Stomp.over(socket);

stompClient.connect(
    {
        // 헤더에 JWT 토큰 포함
        'Authorization': `Bearer ${accessToken}`
    },
    (frame) => {
        console.log('연결 성공:', frame);
    }
);
```

---

## 3. REST API

### 3.1 퀴즈 방 관리

#### 3.1.1 퀴즈 방 생성

- **Endpoint**: `POST /api/quiz/rooms`
- **설명**: 새로운 퀴즈 방을 생성합니다.
- **인증**: **필수**
- **요청**:
  - **Headers**:
    ```
    Authorization: Bearer {access_token}
    Content-Type: application/json
    ```
  - **Body**: `CreateRoomRequest`
  - **JSON 요청 예시**:
    ```json
    {
      "gameTitle": "초보자 환영 방"
    }
    ```
  - **상세 스펙**:

    | 필드 | 타입 | 필수 | 설명 | 검증 규칙 |
    |---|---|---|---|---|
    | `gameTitle` | `String` | ✅ | 방 제목 | 1~50자 |

- **응답 (201 Created)**:
  - **Body**: `ApiResponse<RoomResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "퀴즈 방이 생성되었습니다.",
      "timestamp": "...",
      "data": {
        "gameRoomId": 123
      }
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
    |---|---|---|
    | `gameRoomId` | `Long` | 방 고유 ID |

- **오류**:
  - `401 Unauthorized`: `UNAUTHORIZED` - 인증되지 않은 사용자
  - `401 Unauthorized`: `INVALID_TOKEN` - 유효하지 않은 토큰
  - `409 Conflict`: `PARTICIPANT_ALREADY_IN_ROOM` - 이미 다른 방에 참여 중
  - `400 Bad Request`: `VALIDATION_ERROR` - 입력값 검증 실패

---

#### 3.1.2 퀴즈 방 리스트 조회

- **Endpoint**: `GET /api/quiz/rooms`
- **설명**: 퀴즈 방 목록을 조회합니다. 무한 스크롤 구현을 위해 Slice 방식을 사용합니다. `WAITING` 상태의 퀴즈 방만 조회합니다.
- **인증**: **필수**
- **요청**:
  - **Query Parameters**:

   | 파라미터 | 타입 | 필수 | 기본값 | 설명 |
   |---|---|---|---|---|
   | `page` | `Integer` | ❌ | `0` | 페이지 번호 (0부터 시작) |
   | `size` | `Integer` | ❌ | `20` | 페이지 크기 (1~100) |
    
  - **요청 예시**:
    ```
    GET /api/quiz/rooms?page=0&size=20
    ```

- **응답 (200 OK)**:
  - **Body**: `ApiResponse<RoomListSliceResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "퀴즈 방 목록을 조회했습니다.",
      "timestamp": "2025-10-12T10:30:00",
      "data": {
        "content": [
          {
            "gameRoomId": 123,
            "gameTitle": "초보자 환영 방",
            "hostNickname": "홍길동",
            "currentParticipants": 2,
            "maxParticipants": 4,
            "currentRound": 0,
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
  - **상세 스펙 (`data.content[]`)**:

    | 필드 | 타입 | 설명 |
    |---|---|---|
    | `gameRoomId` | `Long` | 방 고유 ID |
    | `gameTitle` | `String` | 방 제목 |
    | `hostNickname` | `String` | 방장 닉네임 |
    | `currentParticipants` | `Integer` | 현재 인원 |
    | `maxParticipants` | `Integer` | 최대 인원 |
    | `currentRound` | `Integer` | 현재 라운드 |
    | `status` | `String` | 방 상태 (`WAITING`, `IN_PROGRESS`, `FINISHED`) |

  - **Slice 정보 (`data`)**:

    | 필드 | 타입 | 설명 |
    |---|---|---|
    | `content` | `List<RoomListResponse>` | 방 목록 |
    | `hasNext` | `Boolean` | 다음 페이지 존재 여부 (무한 스크롤용) |

- **오류**:
  - `400 Bad Request`: `INVALID_INPUT` - 유효하지 않은 파라미터

---

#### 3.1.3 특정 퀴즈 방 조회

- **Endpoint**: `GET /api/quiz/rooms/{gameRoomId}`
- **설명**: 특정 퀴즈 방의 상세 정보를 조회합니다.
- **인증**: **필수**
- **요청**:
  - **Path Parameters**:

    | 파라미터 | 타입 | 필수 | 설명 |
    |---|---|---|---|
    | `gameRoomId` | `Long` | ✅ | 조회할 방 ID |

  - **요청 예시**:
    ```
    GET /api/quiz/rooms/123
    ```

- **응답 (200 OK)**:
  - **Body**: `ApiResponse<RoomDetailResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "퀴즈 방 정보를 조회했습니다.",
      "timestamp": "2025-10-12T10:30:00",
      "data": {
        "gameRoomId": 123,
        "gameTitle": "초보자 환영 방",
        "host": {
          "userId": 1,
          "nickname": "홍길동"
        },
        "participants": [
          {
            "userId": 1,
            "nickname": "홍길동",
            "totalScore": 2450,
            "isHost": true,
            "isReady": true,
          },
          {
            "userId": 2,
            "nickname": "김영희",
            "totalScore": 1820,
            "isHost": false,
            "isReady": false
          }
        ],
        "currentParticipants": 2,
        "maxParticipants": 4,
        "currentRound": 0,
        "status": "WAITING"
      }
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
    |---|---|---|
    | `gameRoomId` | `Long` | 방 고유 ID |
    | `gameTitle` | `String` | 방 제목 |
    | `host` | `UserSimpleResponse` | 방장 정보 |
    | `participants` | `List<ParticipantResponse>` | 참가자 목록 |
    | `currentParticipants` | `Integer` | 현재 인원 |
    | `maxParticipants` | `Integer` | 최대 인원 |
    | `currentRound` | `Integer` | 현재 라운드 |
    | `status` | `String` | 방 상태 |

  - **참가자 정보 (`participants[]`)**:

    | 필드 | 타입 | 설명 |
    |---|---|---|
    | `userId` | `Long` | 사용자 ID |
    | `nickname` | `Integer` | 닉네임 |
    | `totalScore` | `String` | 게임 히스토리 합산 |
    | `isHost` | `Boolean` | 방장 여부 |
    | `isReady` | `Boolean` | 준비 상태 |

- **오류**:
  - `404 Not Found`: `ROOM_NOT_FOUND` - 퀴즈 방을 찾을 수 없음

---

## 4. WebSocket API (STOMP)

### 4.1 연결 설정

#### 4.1.1 STOMP Endpoint

**Connection URL**: `wss://api.signbell.com/ws/quiz`

#### 4.1.2 STOMP Configuration

```javascript
// 구독(Subscribe) Prefix
/topic   - 방 전체 브로드캐스트
/queue   - 개인 메시지

// 발행(Publish) Prefix
/app     - 서버로 메시지 전송
```

#### 4.1.3 클라이언트 연결 예시

```javascript
import { Client } from '@stomp/stompjs';
import SockJS from 'sockjs-client';

const client = new Client({
    webSocketFactory: () => new SockJS('http://localhost:8080/ws/quiz'),
    
    connectHeaders: {
        'Authorization': `Bearer ${accessToken}`
    },
    
    onConnect: (frame) => {
        console.log('STOMP 연결 성공');
        
        // 방 메시지 구독
        client.subscribe(`/topic/rooms/${roomId}`, (message) => {
            const data = JSON.parse(message.body);
            handleMessage(data);
        });
        
        // 개인 에러 메시지 구독
        client.subscribe('/user/queue/errors', (message) => {
            const error = JSON.parse(message.body);
            handleError(error);
        });
    },
    
    onDisconnect: () => {
        console.log('STOMP 연결 종료');
    }
});

client.activate();
```

---

### 4.2 방 입장/퇴장

#### 4.2.1 방 입장

**Client → Server**:
```
DESTINATION: /app/rooms/{roomId}/join
BODY: (없음)
```

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "user_joined",
  "success": true,
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

**Error Response** (개인에게만):
```
DESTINATION: /user/queue/errors
BODY:
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
- `ROOM_FULL`: 방 인원이 가득 찼음
- `ROOM_ALREADY_STARTED`: 이미 시작된 방
- `PARTICIPANT_ALREADY_IN_ROOM`: 이미 방에 참여 중
- `CAMERA_PERMISSION_REQUIRED`: 카메라 권한 필요

**JavaScript 예시**:
```javascript
// 방 입장
client.publish({
    destination: `/app/rooms/${roomId}/join`
});
```

---

#### 4.2.2 방 퇴장

**Client → Server**:
```
DESTINATION: /app/rooms/{roomId}/leave
BODY: (없음)
```

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "user_left",
  "success": true,
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

**JavaScript 예시**:
```javascript
// 방 퇴장
client.publish({
    destination: `/app/rooms/${roomId}/leave`
});
```

---

#### 4.2.3 방장 변경 알림

방장이 퇴장할 경우 자동으로 다음 참가자가 방장이 됨

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "host_changed",
  "success": true,
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
```
DESTINATION: /app/rooms/{roomId}/ready
BODY:
{
  "isReady": true
}
```

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "user_ready_status",
  "success": true,
  "data": {
    "userId": 2,
    "nickname": "김영희",
    "isReady": true,
    "readyCount": 2,
    "totalParticipants": 3
  }
}
```

**JavaScript 예시**:
```javascript
// 준비 완료
client.publish({
    destination: `/app/rooms/${roomId}/ready`,
    body: JSON.stringify({ isReady: true })
});
```

---

#### 4.3.2 퀴즈 시작 (방장 전용)

**Client → Server**:
```
DESTINATION: /app/rooms/{roomId}/start
BODY: (없음)
```

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "quiz_started",
  "success": true,
  "data": {
    "roomId": 123,
    "round": 1,
    "status": "IN_PROGRESS",
    "totalQuestions": 8,
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

**Error Responses**:
- `NOT_ROOM_HOST`: 방장 권한 없음
- `ROOM_MIN_PARTICIPANTS_NOT_MET`: 최소 인원 부족 (2명 미만)
- `PARTICIPANT_NOT_READY`: 준비되지 않은 참가자 존재

**JavaScript 예시**:
```javascript
// 퀴즈 시작 (방장만)
client.publish({
    destination: `/app/rooms/${roomId}/start`
});
```

---

### 4.4 퀴즈 진행

#### 4.4.1 문제 출제

퀴즈 시작 또는 다음 문제로 진행 시 서버에서 자동 전송

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "question_presented",
  "success": true,
  "data": {
    "questionNumber": 1,
    "totalQuestions": 8,
    "round": 1,
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
```
DESTINATION: /app/rooms/{roomId}/answer-attempt
BODY:
{
  "questionNumber": 1,
  "round": 1
}
```

**Server → All Clients in Room** (선착순 성공):
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "turn_assigned",
  "success": true,
  "data": {
    "answerer": {
      "userId": 2,
      "nickname": "김영희"
    },
    "questionNumber": 1,
    "round": 1
  }
}
```

**Error Response** (개인에게만):
```
DESTINATION: /user/queue/errors
BODY:
{
  "type": "error",
  "success": false,
  "error": {
    "code": "QUIZ_BUTTON_ALREADY_TAKEN",
    "message": "다른 참가자가 먼저 버튼을 눌렀습니다."
  }
}
```

**JavaScript 예시**:
```javascript
// 정답 도전
client.publish({
    destination: `/app/rooms/${roomId}/answer-attempt`,
    body: JSON.stringify({
        questionNumber: 1,
        round: 1
    })
});
```

---

#### 4.4.3 카운트다운 시작

응답자가 선정되면 서버에서 자동으로 전송

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "countdown_start",
  "success": true,
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

카운트다운 종료 후 서버에서 자동 전환

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "perform_sign",
  "success": true,
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
```
DESTINATION: /app/rooms/{roomId}/submit-answer
BODY:
{
  "questionNumber": 1,
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
```

**Server → All Clients in Room** (정답):
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "answer_result",
  "success": true,
  "data": {
    "answerer": {
      "userId": 2,
      "nickname": "김영희"
    },
    "isCorrect": true,
    "similarity": 0.92,
    "scoreChange": 100,
    "currentScore": 100,
    "questionNumber": 1,
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

**JavaScript 예시**:
```javascript
// 좌표 제출
client.publish({
    destination: `/app/rooms/${roomId}/submit-answer`,
    body: JSON.stringify({
        questionNumber: 1,
        round: 1,
        wordId: 5,
        coordinates: {
            landmarks: [...],
            timestamp: new Date().toISOString()
        }
    })
});
```

---

#### 4.4.6 다음 문제 진행

정답/오답 판정 후 서버에서 자동으로 전송 (3초 대기)

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "next_question",
  "success": true,
  "data": {
    "nextQuestion": 2,
    "delayTime": 3000
  }
}
```

그 후 `question_presented` 메시지 전송

---

### 4.5 퀴즈 종료

#### 4.5.1 라운드 완료

8문제 완료 시 서버에서 자동 전송

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "round_completed",
  "success": true,
  "data": {
    "roomId": 123,
    "round": 1,
    "totalQuestions": 8,
    "completedAt": "2025-10-12T10:45:00"
  }
}
```

---

#### 4.5.2 라운드 순위 발표

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "round_ranking",
  "success": true,
  "data": {
    "round": 1,
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
      }
    ]
  }
}
```

---

#### 4.5.3 대기실 복귀

최종 순위 발표 후 서버에서 자동으로 전송 (5초 대기)

**Server → All Clients in Room**:
```
DESTINATION: /topic/rooms/{roomId}
BODY:
{
  "type": "return_to_lobby",
  "success": true,
  "data": {
    "roomId": 123,
    "status": "WAITING",
    "currentRound": 1,
    "delayTime": 5000
  }
}
```

---

### 4.6 WebRTC 시그널링

#### 4.6.1 WebRTC Offer 전송

**Client → Server**:
```
DESTINATION: /app/rooms/{roomId}/webrtc/offer
BODY:
{
  "targetUserId": 2,
  "sdp": "v=0\r\no=- 123456789 2 IN IP4 127.0.0.1\r\n..."
}
```

**Server → Target Client**:
```
DESTINATION: /user/queue/webrtc
BODY:
{
  "type": "webrtc_offer",
  "success": true,
  "data": {
    "fromUserId": 1,
    "sdp": "v=0\r\no=- 123456789 2 IN IP4 127.0.0.1\r\n..."
  }
}
```

**JavaScript 예시**:
```javascript
// WebRTC Offer 전송
client.publish({
    destination: `/app/rooms/${roomId}/webrtc/offer`,
    body: JSON.stringify({
        targetUserId: 2,
        sdp: offer.sdp
    })
});
```

---

#### 4.6.2 WebRTC Answer 전송

**Client → Server**:
```
DESTINATION: /app/rooms/{roomId}/webrtc/answer
BODY:
{
  "targetUserId": 1,
  "sdp": "v=0\r\no=- 987654321 2 IN IP4 127.0.0.1\r\n..."
}
```

**Server → Target Client**:
```
DESTINATION: /user/queue/webrtc
BODY:
{
  "type": "webrtc_answer",
  "success": true,
  "data": {
    "fromUserId": 2,
    "sdp": "v=0\r\no=- 987654321 2 IN IP4 127.0.0.1\r\n..."
  }
}
```

---

#### 4.6.3 ICE Candidate 교환

**Client → Server**:
```
DESTINATION: /app/rooms/{roomId}/webrtc/ice
BODY:
{
  "targetUserId": 2,
  "candidate": {
    "candidate": "candidate:1 1 UDP 2130706431 192.168.1.100 54321 typ host",
    "sdpMid": "0",
    "sdpMLineIndex": 0
  }
}
```

**Server → Target Client**:
```
DESTINATION: /user/queue/webrtc
BODY:
{
  "type": "webrtc_ice_candidate",
  "success": true,
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


## 9. STOMP 엔드포인트 요약표

### 9.1 Client → Server (발행)

| Destination | 설명 | Request Body | 권한 |
|-------------|------|--------------|------|
| `/app/rooms/{roomId}/join` | 방 입장 | 없음 | 인증 필요 |
| `/app/rooms/{roomId}/leave` | 방 퇴장 | 없음 | 인증 필요 |
| `/app/rooms/{roomId}/ready` | 준비 상태 변경 | `{ isReady: boolean }` | 인증 필요 |
| `/app/rooms/{roomId}/start` | 퀴즈 시작 | 없음 | 방장 전용 |
| `/app/rooms/{roomId}/answer-attempt` | 정답 도전 | `{ questionNumber, round }` | 인증 필요 |
| `/app/rooms/{roomId}/submit-answer` | 좌표 제출 | `{ questionNumber, round, wordId, coordinates }` | 응답자만 |
| `/app/rooms/{roomId}/webrtc/offer` | WebRTC Offer | `{ targetUserId, sdp }` | 인증 필요 |
| `/app/rooms/{roomId}/webrtc/answer` | WebRTC Answer | `{ targetUserId, sdp }` | 인증 필요 |
| `/app/rooms/{roomId}/webrtc/ice` | ICE Candidate | `{ targetUserId, candidate }` | 인증 필요 |

---

### 9.2 Server → Client (구독)

| Destination | 설명 | Message Type | 수신자 |
|-------------|------|--------------|--------|
| `/topic/rooms/{roomId}` | 방 전체 메시지 | 다양 | 방 전체 |
| `/user/queue/errors` | 에러 메시지 | `error` | 개인 |
| `/user/queue/webrtc` | WebRTC 시그널링 | `webrtc_*` | 개인 |

---

### 9.3 Message Type 목록

#### 방 전체 브로드캐스트 (`/topic/rooms/{roomId}`)

| type | 설명 | 발생 시점 |
|------|------|----------|
| `user_joined` | 사용자 입장 | 방 입장 성공 시 |
| `user_left` | 사용자 퇴장 | 방 퇴장 시 |
| `host_changed` | 방장 변경 | 방장 퇴장 시 |
| `user_ready_status` | 준비 상태 변경 | 준비 버튼 클릭 시 |
| `quiz_started` | 퀴즈 시작 | 방장이 시작 시 |
| `question_presented` | 문제 출제 | 라운드 시작/다음 문제 시 |
| `turn_assigned` | 응답자 선정 | 선착순 성공 시 |
| `countdown_start` | 카운트다운 시작 | 응답자 선정 후 |
| `perform_sign` | 수어 수행 시작 | 카운트다운 종료 후 |
| `answer_result` | 정답/오답 결과 | AI 검증 완료 후 |
| `next_question` | 다음 문제 진행 | 결과 표시 후 |
| `round_completed` | 라운드 종료 | 8문제 완료 시 |
| `round_ranking` | 라운드 순위 | 라운드 종료 직후 |
| `return_to_lobby` | 대기실 복귀 | 순위 발표 후 |

#### 개인 메시지 (`/user/queue/*`)

| type | destination | 설명 |
|------|-------------|------|
| `error` | `/user/queue/errors` | 에러 발생 시 |
| `webrtc_offer` | `/user/queue/webrtc` | WebRTC Offer 수신 |
| `webrtc_answer` | `/user/queue/webrtc` | WebRTC Answer 수신 |
| `webrtc_ice_candidate` | `/user/queue/webrtc` | ICE Candidate 수신 |

---

## 11. FAQ

### Q1. STOMP와 순수 WebSocket의 차이는?
**A**: STOMP는 WebSocket 위에서 동작하는 메시징 프로토콜로, URL 경로 기반 라우팅, 자동 메시지 변환, 구독/발행 패턴 등을 제공합니다. Spring Boot와의 통합이 쉽고 코드가 더 깔끔해집니다.

### Q2. `/topic`과 `/queue`의 차이는?
**A**:
- `/topic`: 브로드캐스트 (1:N) - 해당 토픽을 구독한 모든 클라이언트가 메시지 수신
- `/queue`: 개인 메시지 (1:1) - 특정 사용자에게만 메시지 전송

### Q3. Principal은 어떻게 주입되나요?
**A**: WebSocket 핸드셰이크 시 JWT 토큰을 검증하고, 사용자 정보를 Principal로 설정합니다. 이후 `@MessageMapping` 메서드에서 `Principal` 파라미터로 자동 주입됩니다.

### Q4. 메시지 순서가 보장되나요?
**A**: 같은 세션 내에서는 메시지 순서가 보장됩니다. 하지만 여러 클라이언트 간의 메시지 순서는 보장되지 않으므로, 중요한 경우 타임스탬프나 시퀀스 번호를 포함해야 합니다.

### Q5. 재연결은 어떻게 처리하나요?
**A**: STOMP 클라이언트는 자동 재연결 기능을 제공합니다. `reconnectDelay` 옵션으로 재연결 간격을 설정할 수 있으며, 재연결 성공 시 구독을 다시 설정해야 합니다.

### Q6. SockJS는 필수인가요?
**A**: 아니오. SockJS는 WebSocket을 지원하지 않는 브라우저를 위한 폴백입니다. 최신 브라우저만 지원한다면 생략 가능합니다.

---

## 부록 A. STOMP 메시지 구조

### 연결 요청 (CONNECT)
```
CONNECT
Authorization: Bearer eyJhbGc...
accept-version: 1.2
heart-beat: 10000,10000

^@
```

### 구독 (SUBSCRIBE)
```
SUBSCRIBE
id: sub-0
destination: /topic/rooms/123

^@
```

### 메시지 전송 (SEND)
```
SEND
destination: /app/rooms/123/join
content-type: application/json

^@
```

### 메시지 수신 (MESSAGE)
```
MESSAGE
subscription: sub-0
message-id: 007
destination: /topic/rooms/123
content-type: application/json

{"type":"user_joined","success":true,"data":{...}}
^@
```

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| v1.0.0 | 2025-10-12 | 순수 WebSocket 방식 초기 작성 | 강관주 |
| v2.0.0 | 2025-10-12 | STOMP over WebSocket 방식으로 전환 | 강관주 |
