# 퀴즈 REST API 명세서

### 퀴즈 방 관리

#### 퀴즈 방 생성

- **Endpoint**: `POST /api/quiz/rooms`
- **설명**: 새로운 퀴즈 방을 생성합니다.
- **인증**: **필수** (쿠키 기반 인증)
- **요청**:
    - **Headers**:
      ```
      Content-Type: application/json
      Cookie: access_token={access_token}
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
      | `gameTitle` | `String` | ✅ | 방 제목 | `@NotBlank`, `@Size(min=1, max=50)` |

- **응답 (200 OK)**:
    - **Body**: `ApiResponse<CreateRoomResponse>`
    - **JSON 응답 예시**:
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
    - **상세 스펙 (`data`)**:

      | 필드 | 타입 | 설명 |
          |---|---|---|
      | `gameRoomId` | `Long` | 생성된 방 고유 ID |

- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` - 인증되지 않은 사용자 (쿠키 없음)
    - `401 Unauthorized`: `INVALID_TOKEN` - 유효하지 않은 액세스 토큰
    - `401 Unauthorized`: `EXPIRED_TOKEN` - 만료된 액세스 토큰
    - `409 Conflict`: `PARTICIPANT_ALREADY_IN_ROOM` - 이미 다른 방에 참여 중 (WAITING 또는 IN_PROGRESS 상태)
    - `400 Bad Request`: `VALIDATION_ERROR` - 입력값 검증 실패
        - `gameTitle`이 공백인 경우: "방 제목은 필수입니다."
        - `gameTitle`이 1~50자를 벗어난 경우: "방 제목은 1~50자여야 합니다."

---

#### 퀴즈 방 리스트 조회

- **Endpoint**: `GET /api/quiz/rooms`
- **설명**: 퀴즈 방 목록을 조회합니다. 무한 스크롤 구현을 위해 Slice 방식을 사용합니다. `WAITING` 상태의 퀴즈 방만 조회하며, 생성일 기준 최신순으로 정렬됩니다.
- **인증**: **필수** (쿠키 기반 인증)
- **요청**:
    - **Headers**:
      ```
      Cookie: access_token={access_token}
      ```
    - **Query Parameters**:

  | 파라미터 | 타입 | 필수 | 기본값 | 설명 | 검증 |
     |---|---|---|---|---|---|
  | `page` | `Integer` | ❌ | `0` | 페이지 번호 (0부터 시작) | 0 이상 |
  | `size` | `Integer` | ❌ | `10` | 페이지 크기 | 1~100 |

    - **요청 예시**:
      ```
      GET /api/quiz/rooms?page=0&size=10
      Cookie: access_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
      ```

- **응답 (200 OK)**:
    - **Body**: `ApiResponse<RoomListSliceResponse>`
    - **JSON 응답 예시**:
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
    - **상세 스펙 (`data.roomList[]`)**:

      | 필드 | 타입 | 설명 |
          |---|---|---|
      | `gameRoomId` | `Long` | 방 고유 ID |
      | `gameTitle` | `String` | 방 제목 (1~50자) |
      | `hostNickname` | `String` | 방장 닉네임 |
      | `currentParticipants` | `Integer` | 현재 인원 (기본값: 1) |
      | `maxParticipants` | `Integer` | 최대 인원 (고정값: 4) |
      | `currentRound` | `Integer` | 현재 라운드 (기본값: 1) |
      | `status` | `String` | 방 상태 (항상 `WAITING`) |

    - **Slice 정보 (`data`)**:

      | 필드 | 타입 | 설명 |
          |---|---|---|
      | `roomList` | `List<RoomListResponse>` | 방 목록 (최대 `size`개) |
      | `hasNext` | `Boolean` | 다음 페이지 존재 여부 (무한 스크롤용) |

- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` - 인증되지 않은 사용자 (쿠키 없음)
    - `401 Unauthorized`: `INVALID_TOKEN` - 유효하지 않은 액세스 토큰
    - `401 Unauthorized`: `EXPIRED_TOKEN` - 만료된 액세스 토큰
    - `400 Bad Request`: `INVALID_INPUT` - 유효하지 않은 파라미터
        - `page`가 음수인 경우
        - `size`가 1 미만 또는 100 초과인 경우 (자동으로 10으로 조정됨)

---

### 공통 사항

#### 인증 방식
- **쿠키 기반 인증**: 모든 API 요청 시 `access_token` 쿠키가 자동으로 전송됩니다.
- 쿠키가 없거나 유효하지 않은 경우 `401 Unauthorized` 오류가 발생합니다.
- 프론트엔드에서는 쿠키를 자동으로 관리하므로 별도의 헤더 설정이 필요하지 않습니다.

#### 페이징 처리
- **Slice 방식**: 전체 페이지 수나 전체 데이터 개수를 제공하지 않아 성능이 우수합니다.
- **무한 스크롤**: `hasNext`가 `true`인 경우 다음 페이지를 요청할 수 있습니다.
- **정렬**: 방 목록은 생성일 기준 최신순(`createdAt DESC`)으로 정렬됩니다.

#### 성능 최적화
- **N+1 문제 해결**: QueryDSL의 `fetch join`을 사용하여 방장 정보를 한 번에 조회합니다.
- **읽기 전용 트랜잭션**: 조회 API는 `@Transactional(readOnly = true)`를 사용합니다.

#### 에러 응답 형식
```json
{
  "success": false,
  "message": "오류 메시지",
  "timestamp": "2025-10-13T10:30:00",
  "data": null
}
```