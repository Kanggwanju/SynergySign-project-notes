# 수어 학습 API 명세서 (Sign Education APIs)

## 4. 수어 학습 콘텐츠 (Sign Education Contents)

### 4.1. 카테고리 목록 조회

- **Endpoint**: `GET /api/sign-edu/categories`
- **설명**: 등록된 모든 수어 카테고리(태그) 목록을 중복 없이 조회합니다.
- **인증**: **필수**
- **요청**:
  - 별도의 파라미터 없음 (인증 토큰에서 사용자 식별)
- **응답 (200 OK)**:
  - **Body**: `List<String>`
  - **JSON 응답 예시**:
    ```json
    [
      "감정",
      "교통",
      "날씨",
      "동물",
      "삶",
      "식물",
      "인사"
    ]
    ```
  - **상세 스펙**:
    - 카테고리 문자열 배열을 반환합니다.
    - 카테고리는 오름차순(가나다순)으로 정렬됩니다.
    - 중복된 카테고리는 제거됩니다.
- **오류**:
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)

---

### 4.2. 수어 단어 목록 조회

- **Endpoint**: `GET /api/sign-edu`
- **설명**: 모든 수어 단어 또는 특정 카테고리의 수어 단어 목록을 페이징하여 조회합니다. 키워드 검색 기능을 지원합니다.
- **인증**: **필수**
- **요청**:
  - **Query Parameters**:
    | 이름 | 타입 | 필수 | 기본값 | 설명 |
    |---|---|---|---|---|
    | `category` | `String` | 선택 | - | 조회할 카테고리명 (미입력 또는 "전체" 입력 시 전체 조회) |
    | `keyword` | `String` | 선택 | - | 검색할 단어명 (부분 일치 검색) |
    | `page` | `Integer` | 선택 | `0` | 페이지 번호 (0부터 시작) |
    | `size` | `Integer` | 선택 | `20` | 페이지당 항목 수 |
    | `sort` | `String` | 선택 | `title` | 정렬 기준 필드 |

- **검색 우선순위**:
  - `keyword` 파라미터가 있는 경우, 다음 우선순위로 정렬됩니다:
    1. **완전 일치**: 단어명이 키워드와 정확히 일치
    2. **앞부분 일치**: 단어명이 키워드로 시작
    3. **부분 일치**: 단어명에 키워드가 포함
  - 같은 우선순위 내에서는 단어명 가나다순(오름차순)으로 정렬됩니다.
  - `keyword`가 없는 경우, 기본 정렬은 `title`(단어명) 오름차순입니다.

- **파라미터 우선순위**:
  - `keyword` > `category` > 전체 조회
  - `keyword`가 있으면 `category`는 무시됩니다.

- **응답 (200 OK)**:
  - **Body**: `Page<SignSimpleResponseDto>`
  - **JSON 응답 예시**:
    ```json
    {
      "content": [
        {
          "signId": 1,
          "wordName": "감사합니다"
        },
        {
          "signId": 2,
          "wordName": "미안합니다"
        },
        {
          "signId": 3,
          "wordName": "사랑해요"
        }
      ],
      "pageable": {
        "sort": {
          "sorted": true,
          "unsorted": false,
          "empty": false
        },
        "pageNumber": 0,
        "pageSize": 20,
        "offset": 0,
        "paged": true,
        "unpaged": false
      },
      "totalPages": 5,
      "totalElements": 92,
      "last": false,
      "size": 20,
      "number": 0,
      "sort": {
        "sorted": true,
        "unsorted": false,
        "empty": false
      },
      "numberOfElements": 20,
      "first": true,
      "empty": false
    }
    ```
  - **상세 스펙 (`content[]`)**:

    | 필드 | 타입 | 설명 |
          |---|---|---|
    | `signId` | `Long` | 수어 단어 고유 ID |
    | `wordName` | `String` | 수어 단어명 |

  - **페이징 정보**:

    | 필드 | 타입 | 설명 |
          |---|---|---|
    | `content` | `Array` | 현재 페이지의 수어 단어 목록 |
    | `pageable` | `Object` | 페이징 요청 정보 |
    | `totalPages` | `Integer` | 전체 페이지 수 |
    | `totalElements` | `Long` | 전체 항목 수 |
    | `size` | `Integer` | 페이지당 항목 수 |
    | `number` | `Integer` | 현재 페이지 번호 (0부터 시작) |
    | `first` | `Boolean` | 첫 페이지 여부 |
    | `last` | `Boolean` | 마지막 페이지 여부 |
    | `empty` | `Boolean` | 빈 페이지 여부 |

- **요청 예시**:
  - 전체 조회: `GET /api/sign-edu?page=0&size=20`
  - 카테고리별 조회: `GET /api/sign-edu?category=삶&page=0&size=20`
  - 키워드 검색: `GET /api/sign-edu?keyword=감사&page=0&size=20`
  - 키워드 검색 (카테고리는 무시됨): `GET /api/sign-edu?keyword=감사&category=삶&page=0`

- **참고**:
  - 존재하지 않는 카테고리로 조회 시 빈 페이지(`content: []`)를 반환합니다.
  - `keyword` 검색 시 부분 일치(LIKE '%keyword%')로 검색되며, 결과는 관련성 순으로 정렬됩니다.
  - `category`에 "전체"를 입력하거나 빈 값을 보내면 전체 조회가 됩니다.
  - 키워드 검색 결과는 QueryDSL의 CaseBuilder를 사용하여 최적화된 정렬을 제공합니다.

- **오류**:
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)

---

### 4.3. 수어 단어 상세 조회

- **Endpoint**: `GET /api/sign-edu/{signId}`
- **설명**: 특정 수어 단어의 상세 정보를 조회합니다. 단어명, 설명, 영상 URL, 카테고리 정보를 포함합니다.
- **인증**: **필수**
- **요청**:
  - **Path Parameter**:
    | 이름 | 타입 | 필수 | 설명 |
    |---|---|---|---|
    | `signId` | `Long` | 필수 | 조회할 수어 단어의 고유 ID |

- **응답 (200 OK)**:
  - **Body**: `SignDetailResponseDto`
  - **JSON 응답 예시**:
    ```json
    {
      "signId": 1,
      "wordName": "감사합니다",
      "description": "고마움을 표현하는 수어입니다. 두 손을 가슴 앞에서 모으고 살짝 숙입니다.",
      "videoUrl": "https://example.com/videos/sign_001.mp4",
      "tag": "인사"
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
          |---|---|---|
    | `signId` | `Long` | 수어 단어 고유 ID |
    | `wordName` | `String` | 수어 단어명 |
    | `description` | `String` | 수어 동작에 대한 상세 설명 (TEXT 타입, UTF-8) |
    | `videoUrl` | `String` | 수어 동작 영상 URL (최대 1024자) |
    | `tag` | `String` | 카테고리/태그 (최대 100자) |

- **참고**:
  - `description` 필드는 TEXT 타입으로 긴 설명을 지원합니다.
  - 영상 URL은 외부 CDN 또는 스토리지 서비스의 링크입니다.

- **오류**:
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
  - `404 Not Found`: `WORD_NOT_FOUND` (해당 ID의 수어 단어를 찾을 수 없음)

---

## 공통 사항 (Common Notes)

### 인증 방식
- **쿠키 기반 인증**: 인증 토큰이 쿠키에 저장되어 자동으로 전송됩니다
- 인증이 필요한 API는 쿠키에서 토큰을 추출하여 사용자를 식별합니다
- 모든 수어 학습 API는 인증이 필수입니다
- **요청 시 주의사항**:
  - 브라우저에서 자동으로 쿠키가 전송되므로 별도의 헤더 설정 불필요
  - CORS 환경에서는 `credentials: 'include'` 옵션 필요 (fetch API 사용 시)

### 페이징 정보
- Spring Data JPA의 `Page` 인터페이스를 사용합니다.
- 기본 페이지 크기: 20개
- 페이지 번호는 0부터 시작합니다.
- 정렬 기준은 `sort` 파라미터로 지정할 수 있습니다 (예: `sort=title,asc`)

### 검색 기능
- **키워드 검색**: QueryDSL을 사용한 동적 쿼리로 구현
- **정렬 우선순위**: 완전 일치 > 앞부분 일치 > 부분 일치
- **성능 최적화**: CaseBuilder를 사용한 단일 쿼리로 정렬 처리

### 에러 코드
자세한 에러 코드는 `ErrorCode.java`를 참조하십시오. 주요 에러 코드:
- `UNAUTHORIZED`: 인증되지 않은 사용자
- `WORD_NOT_FOUND`: 수어 단어를 찾을 수 없음
- `INVALID_WORD_ID`: 유효하지 않은 단어 ID
- `DATABASE_ERROR`: 데이터베이스 오류

### 데이터베이스 인덱스
- `learningStatus` 컬럼에 인덱스가 설정되어 있습니다.
- 대용량 데이터 조회 시 성능 최적화를 위해 인덱스를 활용합니다.

### 문자 인코딩
- `signDescription` 필드는 UTF-8 (utf8mb4_unicode_ci)로 저장됩니다.
- 이모지 및 특수 문자를 포함한 모든 유니코드 문자를 지원합니다.