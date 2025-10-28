# 사용자 관련 API 명세서 (User APIs)

## 1. 사용자 정보 조회 (User Information)

### 1.1. 내 정보 조회

- **Endpoint**: `GET /api/users/me`
- **설명**: 현재 로그인한 사용자의 전체 정보를 조회합니다. 프로필 정보뿐만 아니라 인증 관련 정보와 누적 점수를 포함합니다.
- **인증**: **필수**
- **요청**:
  - 별도의 파라미터 없음 (인증 토큰에서 사용자 식별)
- **응답 (200 OK)**:
  - **Body**: `ApiResponse<UserInfoResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "사용자의 정보가 성공적으로 조회되었습니다.",
      "timestamp": "2025-10-29T12:00:00.123Z",
      "data": {
        "userId": 1,
        "nickname": "수어마스터",
        "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
        "providerId": "123456789",
        "provider": "KAKAO",
        "email": "user@example.com",
        "requiredAgree": true,
        "optionalAgree": true,
        "totalScore": 1500
      }
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
          |---|---|---|
    | `userId` | `Long` | 사용자 고유 ID |
    | `nickname` | `String` | 닉네임 |
    | `profileImageUrl` | `String` | 프로필 이미지 URL |
    | `providerId` | `String` | OAuth 제공자의 사용자 ID |
    | `provider` | `String` | 로그인 제공자 (KAKAO, GOOGLE 등) |
    | `email` | `String` | 이메일 주소 |
    | `requiredAgree` | `Boolean` | 필수 약관 동의 여부 |
    | `optionalAgree` | `Boolean` | 선택 약관 동의 여부 (학습 데이터 수집 동의) |
    | `totalScore` | `Long` | 누적 점수 |
- **오류**:
  - `400 Bad Request`: 잘못된 사용자 ID 형식
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
  - `404 Not Found`: `USER_NOT_FOUND` (해당 사용자 없음)
  - `500 Internal Server Error`: 서버 내부 오류

---

## 2. 사용자 랭킹 (User Ranking)

### 2.1. 랭킹 조회

- **Endpoint**: `GET /api/users/rank`
- **설명**: 누적 점수(totalScore) 기준 상위 8명의 사용자 랭킹을 조회합니다.
- **인증**: **필수**
- **요청**:
  - 별도의 파라미터 없음
- **응답 (200 OK)**:
  - **Body**: `ApiResponse<List<UserRankResponse>>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "랭킹 조회에 성공했습니다.",
      "timestamp": "2025-10-29T12:01:00.456Z",
      "data": [
        {
          "rank": 1,
          "nickname": "랭킹1위",
          "score": 5000,
          "profileImage": "http://example.com/profile1.jpg"
        },
        {
          "rank": 2,
          "nickname": "랭킹2위",
          "score": 4500,
          "profileImage": "http://example.com/profile2.jpg"
        }
      ]
    }
    ```
  - **상세 스펙 (`data` 배열의 각 요소)**:

    | 필드 | 타입 | 설명 |
          |---|---|---|
    | `rank` | `Integer` | 사용자 순위 (1~8) |
    | `nickname` | `String` | 닉네임 |
    | `score` | `Long` | 누적 점수 |
    | `profileImage` | `String` | 프로필 이미지 URL (nullable) |
- **오류**:
  - `500 Internal Server Error`: `DATABASE_ERROR` (데이터베이스 조회 실패)

---

## 3. 마이페이지 - 프로필 관리 (My Page - Profile Management)

### 3.1. 내 프로필 조회

- **Endpoint**: `GET /api/my-page/users/{userId}/profile`
- **설명**: 현재 로그인한 사용자의 프로필 정보를 조회합니다. `userId`는 현재 인증된 사용자의 ID여야 합니다.
- **인증**: **필수**
- **요청**:
  - **Path Parameter**:
    | 이름 | 타입 | 필수 | 설명 |
    |---|---|---|---|
    | `userId` | `Long` | 필수 | 조회할 사용자 고유 ID (현재 로그인된 사용자) |
- **응답 (200 OK)**:
  - **Body**: `ApiResponse<UserProfileResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "프로필 조회 성공",
      "timestamp": "2025-10-29T12:02:00.789Z",
      "data": {
        "nickname": "수어마스터",
        "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
        "optionalAgree": true
      }
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
          |---|---|---|
    | `nickname` | `String` | 닉네임 |
    | `profileImageUrl` | `String` | 프로필 이미지 URL |
    | `optionalAgree` | `Boolean` | 선택 약관 동의 여부 (학습 데이터 수집 동의) |
- **오류**:
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
  - `403 Forbidden`: `FORBIDDEN` (요청한 userId와 로그인 사용자 불일치)
  - `404 Not Found`: `USER_NOT_FOUND` (해당 ID의 사용자 없음)

### 3.2. 내 프로필 수정

- **Endpoint**: `PATCH /api/my-page/users/{userId}/profile`
- **설명**: 현재 로그인한 사용자의 프로필 정보(닉네임, 선택 약관 동의)를 수정합니다. `userId`는 현재 인증된 사용자의 ID여야 합니다.
- **인증**: **필수**
- **요청**:
  - **Path Parameter**:
    | 이름 | 타입 | 필수 | 설명 |
    |---|---|---|---|
    | `userId` | `Long` | 필수 | 수정할 사용자 고유 ID (현재 로그인된 사용자) |
  - **Body (`application/json`)**:
    ```json
    {
      "nickname": "변경된닉네임",
      "optionalAgree": true
    }
    ```
  - **상세 스펙 (Request Body)**:

    | 필드 | 타입 | 필수 | 제약사항 | 설명 |
          |---|---|---|---|---|
    | `nickname` | `String` | 필수 | 1~20자, 공백 불가 | 변경할 닉네임 |
    | `optionalAgree` | `Boolean` | 선택 | - | 선택 약관 동의 여부 (학습 데이터 수집 동의) |
- **응답 (200 OK)**:
  - **Body**: `ApiResponse<UserProfileResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "프로필 수정 성공",
      "timestamp": "2025-10-29T12:03:00.123Z",
      "data": {
        "nickname": "변경된닉네임",
        "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
        "optionalAgree": true
      }
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
          |---|---|---|
    | `nickname` | `String` | 수정된 닉네임 |
    | `profileImageUrl` | `String` | 프로필 이미지 URL |
    | `optionalAgree` | `Boolean` | 수정된 선택 약관 동의 여부 |
- **오류**:
  - `400 Bad Request`: `VALIDATION_ERROR` (닉네임 길이 제약 위반 등 유효성 검사 실패)
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
  - `403 Forbidden`: `FORBIDDEN` (요청한 userId와 로그인 사용자 불일치)
  - `404 Not Found`: `USER_NOT_FOUND` (해당 ID의 사용자 없음)
  - `409 Conflict`: `NICKNAME_ALREADY_EXISTS` (이미 사용 중인 닉네임)

### 3.3. 닉네임 변경

- **Endpoint**: `PATCH /api/my-page/users/{userId}/nickname`
- **설명**: 현재 로그인한 사용자의 닉네임만 변경합니다. 프로필 전체 수정(3.2)과 달리 닉네임만 부분 업데이트합니다.
- **인증**: **필수**
- **요청**:
  - **Path Parameter**:
    | 이름 | 타입 | 필수 | 설명 |
    |---|---|---|---|
    | `userId` | `Long` | 필수 | 수정할 사용자 고유 ID (현재 로그인된 사용자) |
  - **Body (`application/json`)**:
    ```json
    {
      "nickname": "새로운닉네임"
    }
    ```
  - **상세 스펙 (Request Body)**:

    | 필드 | 타입 | 필수 | 제약사항 | 설명 |
          |---|---|---|---|---|
    | `nickname` | `String` | 필수 | 1~10자, 공백 불가 | 변경할 닉네임 |
- **응답 (200 OK)**:
  - **Body**: `ApiResponse<UserProfileResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "닉네임 변경 성공",
      "timestamp": "2025-10-29T12:04:00.456Z",
      "data": {
        "nickname": "새로운닉네임",
        "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
        "optionalAgree": true
      }
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
          |---|---|---|
    | `nickname` | `String` | 변경된 닉네임 |
    | `profileImageUrl` | `String` | 프로필 이미지 URL |
    | `optionalAgree` | `Boolean` | 선택 약관 동의 여부 |
- **오류**:
  - `400 Bad Request`: `VALIDATION_ERROR` (닉네임 길이 제약 위반 등 유효성 검사 실패)
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
  - `403 Forbidden`: `FORBIDDEN` (요청한 userId와 로그인 사용자 불일치)
  - `404 Not Found`: `USER_NOT_FOUND` (해당 ID의 사용자 없음)
  - `409 Conflict`: `NICKNAME_ALREADY_EXISTS` (이미 사용 중인 닉네임)

### 3.4. 약관 동의 처리

- **Endpoint**: `PATCH /api/my-page/users/{userId}/terms-agreement`
- **설명**: 사용자의 약관 동의를 처리합니다. 필수 약관(requiredAgree)은 자동으로 true로 설정되며, 선택 약관(optionalAgree)은 요청값에 따라 설정됩니다.
- **인증**: **필수**
- **요청**:
  - **Path Parameter**:
    | 이름 | 타입 | 필수 | 설명 |
    |---|---|---|---|
    | `userId` | `Long` | 필수 | 약관 동의를 처리할 사용자 고유 ID (현재 로그인된 사용자) |
  - **Body (`application/json`)**:
    ```json
    {
      "optionalAgree": true
    }
    ```
  - **상세 스펙 (Request Body)**:

    | 필드 | 타입 | 필수 | 설명 |
          |---|---|---|---|
    | `optionalAgree` | `Boolean` | 선택 | 선택 약관 동의 여부 (학습 데이터 수집 동의). null인 경우 기존 값 유지 |
- **응답 (200 OK)**:
  - **Body**: `ApiResponse<UserProfileResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "약관 동의 처리 성공",
      "timestamp": "2025-10-29T12:05:00.789Z",
      "data": {
        "nickname": "수어마스터",
        "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
        "optionalAgree": true
      }
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
          |---|---|---|
    | `nickname` | `String` | 닉네임 |
    | `profileImageUrl` | `String` | 프로필 이미지 URL |
    | `optionalAgree` | `Boolean` | 업데이트된 선택 약관 동의 여부 |
- **오류**:
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
  - `403 Forbidden`: `FORBIDDEN` (요청한 userId와 로그인 사용자 불일치)
  - `404 Not Found`: `USER_NOT_FOUND` (해당 ID의 사용자 없음)

---

## 공통 사항 (Common Notes)

### 응답 포맷
모든 API는 `ApiResponse<T>` 형식으로 응답합니다:
```json
{
  "success": true/false,
  "message": "응답 메시지",
  "timestamp": "ISO 8601 형식의 응답 시간",
  "data": "실제 데이터 (타입은 API마다 다름)"
}
```

### 인증 방식
- 인증이 필요한 API는 `@AuthenticationPrincipal`을 통해 사용자 식별
- 인증 토큰에서 추출된 `subject` 값을 사용자 ID로 사용
- Path Parameter의 `userId`와 인증된 사용자 ID의 일치 여부를 검증

### 에러 코드
자세한 에러 코드는 `ErrorCode.java`를 참조하십시오. 주요 에러 코드:
- `UNAUTHORIZED`: 인증되지 않은 사용자
- `FORBIDDEN`: 접근 권한 없음
- `USER_NOT_FOUND`: 사용자를 찾을 수 없음
- `VALIDATION_ERROR`: 유효성 검사 실패
- `NICKNAME_ALREADY_EXISTS`: 이미 사용 중인 닉네임
- `DATABASE_ERROR`: 데이터베이스 오류