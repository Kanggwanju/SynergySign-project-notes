# Design Document

## Overview

사용자 랭킹 시스템은 SignBell 애플리케이션에서 사용자의 누적 점수(totalScore)를 기반으로 상위 8명의 랭킹을 조회하고 표시하는 기능입니다. 이 시스템은 백엔드 RESTful API와 프론트엔드 React 컴포넌트로 구성되며, 다음과 같은 특징을 가집니다:

- **백엔드**: Spring Boot 기반의 3-tier 아키텍처 (Controller → Service → Repository)
- **프론트엔드**: React 기반의 컴포넌트 구조 (Component → Service → API Client)
- **통신**: RESTful API (JSON 형식, HTTP GET)
- **에러 처리**: 공통 에러 응답 체계 (ErrorResponse, BusinessException)
- **인증**: JWT 기반 인증 (apiClient의 자동 토큰 처리)

## Architecture

### System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Frontend (React)                        │
├─────────────────────────────────────────────────────────────┤
│  MainPage.jsx                                                │
│    └─> RankingBoard.jsx                                      │
│          └─> rankService.js                                  │
│                └─> apiClient.js (axios)                      │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ HTTP GET /api/users/rank
                            │ (JWT Token in Cookie)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Backend (Spring Boot)                      │
├─────────────────────────────────────────────────────────────┤
│  RankController                                              │
│    └─> RankListService                                       │
│          └─> UserRepository (JPA)                            │
│                └─> MySQL Database (user table)               │
├─────────────────────────────────────────────────────────────┤
│  GlobalExceptionHandler (AOP)                                │
│    └─> ErrorResponse                                         │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **사용자 접속**: 사용자가 메인 페이지에 접속
2. **자동 요청**: RankingBoard 컴포넌트가 useEffect로 랭킹 데이터 요청
3. **API 호출**: rankService가 apiClient를 통해 GET /api/users/rank 호출
4. **인증 처리**: apiClient가 HTTP-Only 쿠키의 JWT 토큰을 자동으로 포함
5. **컨트롤러 처리**: RankController가 요청을 받아 RankListService 호출
6. **비즈니스 로직**: RankListService가 UserRepository를 통해 데이터 조회
7. **데이터 변환**: User 엔티티를 UserRankResponse DTO로 변환
8. **응답 래핑**: ApiResponse로 응답 데이터 래핑
9. **프론트엔드 처리**: RankingBoard가 데이터를 받아 UI 렌더링

## Components and Interfaces

### Backend Components

#### 1. RankController

**위치**: `backend/src/main/java/app/signbell/backend/controller/user/RankController.java`

**책임**:
- 랭킹 조회 API 엔드포인트 제공
- 요청 검증 및 응답 반환
- 에러 처리 (GlobalExceptionHandler에 위임)

**주요 메서드**:
```java
@GetMapping("/rank")
public ResponseEntity<ApiResponse<List<UserRankResponse>>> getRankings()
```

**의존성**:
- RankListService (비즈니스 로직 처리)

#### 2. RankListService

**위치**: `backend/src/main/java/app/signbell/backend/service/RankListService.java`

**책임**:
- 랭킹 데이터 조회 비즈니스 로직
- User 엔티티를 UserRankResponse DTO로 변환
- 에러 발생 시 BusinessException 발생

**주요 메서드**:
```java
public List<UserRankResponse> getTopRankings()
```

**의존성**:
- UserRepository (데이터 접근)

#### 3. UserRankResponse

**위치**: `backend/src/main/java/app/signbell/backend/dto/response/userData/UserRankResponse.java`

**필드**:
```java
private Integer rank;           // 순위 (1~8)
private String nickname;         // 닉네임
private Long score;             // 누적 점수
private String profileImage;    // 프로필 이미지 URL (nullable)
```

**정적 팩토리 메서드**:
```java
public static UserRankResponse from(User user, int rank)
```

### Frontend Components

#### 1. RankingBoard Component

**위치**: `frontend/src/components/main/RankingBoard.jsx`

**책임**:
- 랭킹 데이터 표시
- 로딩 상태 관리
- 에러 상태 처리
- 상위 3명 시각적 강조

**상태 관리**:
```javascript
const [rankings, setRankings] = useState([]);
const [isLoading, setIsLoading] = useState(true);
const [error, setError] = useState(null);
```

**주요 함수**:
```javascript
const fetchRankings = async () => {
  // rankService를 통해 데이터 조회
}

const getRankIcon = (rank) => {
  // 1~3위 아이콘 반환
}

const getProfileImage = (user) => {
  // 프로필 이미지 또는 기본 표시 반환
}
```

#### 2. rankService

**위치**: `frontend/src/services/api/rankService.js`

**책임**:
- 랭킹 API 호출
- ApiResponse 형식 처리
- 에러 전파

**주요 함수**:
```javascript
export const getRankings = async () => {
  const response = await apiClient.get('/users/rank');
  return response.data.data; // ApiResponse의 data 필드 추출
}
```

## Data Models

### Database Schema

**Table**: `user`

```sql
CREATE TABLE user (
  user_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  nickname VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE,
  profile_image_url VARCHAR(500),
  provider VARCHAR(50) NOT NULL,
  provider_id VARCHAR(255) NOT NULL,
  required_agree BOOLEAN NOT NULL DEFAULT FALSE,
  optional_agree BOOLEAN NOT NULL DEFAULT FALSE,
  total_score BIGINT NOT NULL DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_provider_id (provider, provider_id)
);

-- 랭킹 조회를 위한 인덱스
CREATE INDEX idx_total_score ON user(total_score DESC);
```

### API Request/Response

#### Request

```http
GET /api/users/rank HTTP/1.1
Host: localhost:8080
Cookie: accessToken=<JWT_TOKEN>
```

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "랭킹 조회에 성공했습니다.",
  "timestamp": "2025-10-28T14:30:00",
  "data": [
    {
      "rank": 1,
      "nickname": "수어마스터",
      "score": 9850,
      "profileImage": "https://example.com/profile1.jpg"
    },
    {
      "rank": 2,
      "nickname": "손말이",
      "score": 8920,
      "profileImage": null
    },
    ...
  ]
}
```

#### Error Response (500 Internal Server Error)

```json
{
  "timestamp": "2025-10-28T14:30:00",
  "status": 500,
  "error": "DATABASE_ERROR",
  "detail": "데이터베이스 오류가 발생했습니다.",
  "path": "/api/users/rank"
}
```

### DTO Mapping

**User Entity → UserRankResponse DTO**

```java
User {
  id: 1L,
  nickname: "수어마스터",
  totalScore: 9850L,
  profileImageUrl: "https://example.com/profile1.jpg"
}
↓
UserRankResponse {
  rank: 1,
  nickname: "수어마스터",
  score: 9850,
  profileImage: "https://example.com/profile1.jpg"
}
```

## Error Handling

### Backend Error Handling

#### 1. BusinessException 발생 시나리오

- 데이터베이스 연결 실패
- 예상치 못한 데이터 형식
- 리포지토리 조회 실패

#### 2. GlobalExceptionHandler 처리

```java
@ExceptionHandler(BusinessException.class)
public ResponseEntity<?> handleBusinessException(
    BusinessException e, 
    HttpServletRequest request
) {
    ErrorResponse errorResponse = ErrorResponse.builder()
        .timestamp(LocalDateTime.now())
        .detail(e.getMessage())
        .path(request.getRequestURI())
        .status(e.getErrorCode().getStatus())
        .error(e.getErrorCode().getCode())
        .build();
    
    return ResponseEntity
        .status(e.getErrorCode().getStatus())
        .body(errorResponse);
}
```

#### 3. ErrorCode 정의

랭킹 시스템에서 사용할 에러 코드:
- `DATABASE_ERROR`: 데이터베이스 오류 (500)
- `INTERNAL_SERVER_ERROR`: 서버 내부 오류 (500)
- `UNAUTHORIZED`: 인증되지 않은 사용자 (401)

### Frontend Error Handling

#### 1. API 호출 에러 처리

```javascript
const fetchRankings = async () => {
  try {
    setIsLoading(true);
    setError(null);
    
    const data = await rankService.getRankings();
    setRankings(data);
  } catch (err) {
    console.error('랭킹 데이터 로드 실패:', err);
    setError('랭킹 정보를 불러올 수 없습니다.');
  } finally {
    setIsLoading(false);
  }
};
```

#### 2. UI 상태별 렌더링

- **로딩 중**: 스피너 + "랭킹 불러오는 중..." 메시지
- **에러 발생**: 에러 메시지 표시
- **성공**: 랭킹 리스트 표시

## Testing Strategy

### Backend Testing

#### 1. Unit Tests (Optional)

**RankListService 테스트**:
- 정상적인 랭킹 조회
- 빈 결과 처리
- 8명 제한 검증
- DTO 변환 검증

**테스트 도구**: JUnit 5, Mockito

#### 2. Integration Tests (Optional)

**RankController 테스트**:
- API 엔드포인트 호출
- ApiResponse 형식 검증
- 에러 응답 검증

**테스트 도구**: Spring Boot Test, MockMvc

### Frontend Testing

#### 1. Component Tests (Optional)

**RankingBoard 테스트**:
- 랭킹 데이터 렌더링
- 로딩 상태 표시
- 에러 상태 표시
- 상위 3명 아이콘 표시

**테스트 도구**: Jest, React Testing Library

#### 2. Service Tests (Optional)

**rankService 테스트**:
- API 호출 성공
- 에러 처리

**테스트 도구**: Jest, axios-mock-adapter

## Implementation Notes

### Backend Implementation

1. **RankController 작성**:
   - `@RestController`, `@RequestMapping("/api/users")` 어노테이션 사용
   - `@RequiredArgsConstructor`로 의존성 주입
   - `@Slf4j`로 로깅 추가

2. **RankListService 작성**:
   - `@Service` 어노테이션 사용
   - `@Transactional(readOnly = true)`로 읽기 전용 트랜잭션 설정
   - UserRepository의 커스텀 쿼리 메서드 활용

3. **UserRepository 확장**:
   - `findTop8ByOrderByTotalScoreDesc()` 메서드 추가
   - Spring Data JPA의 쿼리 메서드 네이밍 규칙 활용

4. **UserRankResponse DTO 작성**:
   - `@Getter`, `@Builder` 어노테이션 사용
   - `from(User user, int rank)` 정적 팩토리 메서드 제공

### Frontend Implementation

1. **rankService 작성**:
   - apiClient import
   - async/await 패턴 사용
   - 에러 전파

2. **RankingBoard 수정**:
   - 더미 데이터 제거
   - rankService 호출로 대체
   - 에러 처리 강화

3. **스타일링**:
   - 기존 SCSS 모듈 유지
   - 상위 3명 강조 스타일 적용

## Performance Considerations

### Database Optimization

1. **인덱스 활용**:
   - `total_score` 필드에 내림차순 인덱스 생성
   - 쿼리 성능 향상

2. **LIMIT 절 사용**:
   - 상위 8명만 조회하여 데이터 전송량 최소화

### Caching Strategy

- 현재 버전에서는 캐싱을 사용하지 않음
- 매 요청마다 데이터베이스에서 직접 조회
- 향후 필요 시 애플리케이션 레벨 캐싱 고려 가능

### Frontend Optimization

1. **불필요한 리렌더링 방지**:
   - React.memo 사용 고려
   - useMemo, useCallback 활용

2. **이미지 최적화**:
   - 프로필 이미지 lazy loading
   - 이미지 크기 최적화

## Security Considerations

1. **인증 검증**:
   - JWT 토큰 검증 (Spring Security)
   - 인증되지 않은 사용자 접근 차단

2. **데이터 노출 제한**:
   - 상위 8명만 공개
   - 민감한 정보 (email, providerId) 제외

3. **SQL Injection 방지**:
   - JPA 사용으로 자동 방어
   - 파라미터 바인딩

4. **XSS 방지**:
   - React의 자동 이스케이핑
   - 사용자 입력 검증

## Deployment Considerations

1. **환경 변수**:
   - 데이터베이스 연결 정보
   - JWT 시크릿 키

2. **로깅**:
   - 랭킹 조회 요청 로그
   - 에러 발생 로그

3. **모니터링**:
   - API 응답 시간 모니터링
   - 에러율 모니터링
