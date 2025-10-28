# Requirements Document

## Introduction

사용자 랭킹 시스템은 SignBell 애플리케이션의 메인 페이지에서 전체 사용자의 점수 기반 순위를 표시하는 기능입니다. 이 시스템은 사용자의 누적 점수(totalScore)를 기준으로 상위 랭커들을 조회하고, 프론트엔드에서 시각적으로 표현합니다. 백엔드는 RESTful API를 통해 랭킹 데이터를 제공하며, 프론트엔드는 이를 실시간으로 조회하여 사용자에게 보여줍니다.

## Glossary

- **RankingSystem**: 사용자의 누적 점수를 기반으로 순위를 계산하고 제공하는 시스템
- **RankController**: 랭킹 조회 API 요청을 처리하는 Spring Boot 컨트롤러
- **RankListService**: 랭킹 데이터를 조회하고 비즈니스 로직을 처리하는 서비스 계층
- **UserRankResponse**: 랭킹 정보를 프론트엔드에 전달하기 위한 응답 DTO
- **ApiResponse**: 모든 API 응답에 사용되는 공통 응답 래퍼 클래스 (success, message, timestamp, data 필드 포함)
- **ErrorResponse**: 에러 발생 시 사용되는 공통 에러 응답 클래스 (timestamp, status, error, detail, path 필드 포함)
- **BusinessException**: 비즈니스 로직 에러를 처리하기 위한 커스텀 예외 클래스
- **ErrorCode**: 애플리케이션 전역 에러 코드를 정의한 enum 클래스
- **RankingBoard**: 프론트엔드에서 랭킹 정보를 표시하는 React 컴포넌트
- **rankService**: 프론트엔드에서 랭킹 API를 호출하는 서비스 모듈
- **totalScore**: User 엔티티의 누적 점수 필드 (BIGINT 타입)
- **apiClient**: axios 기반의 HTTP 클라이언트 (인증 토큰 자동 처리)

## Requirements

### Requirement 1

**User Story:** 사용자로서, 메인 페이지에서 전체 사용자의 랭킹을 확인하고 싶습니다. 그래야 내 순위와 다른 사용자들의 성과를 비교할 수 있습니다.

#### Acceptance Criteria

1. WHEN 사용자가 메인 페이지에 접속하면, THE RankingBoard SHALL 백엔드로부터 랭킹 데이터를 자동으로 요청한다
2. THE RankingSystem SHALL 사용자의 totalScore를 기준으로 내림차순 정렬된 랭킹 목록을 반환한다
3. THE RankingBoard SHALL 각 사용자의 순위, 닉네임, 점수, 프로필 이미지를 표시한다
4. WHILE 랭킹 데이터를 로딩하는 동안, THE RankingBoard SHALL 로딩 상태 표시를 사용자에게 보여준다
5. IF 랭킹 데이터 조회에 실패하면, THEN THE RankingBoard SHALL 에러 메시지를 사용자에게 표시한다

### Requirement 2

**User Story:** 개발자로서, 랭킹 조회 API를 RESTful 방식으로 구현하고 싶습니다. 그래야 프론트엔드에서 일관된 방식으로 데이터를 요청할 수 있습니다.

#### Acceptance Criteria

1. THE RankController SHALL GET 메서드로 "/api/users/rank" 엔드포인트를 제공한다
2. THE RankController SHALL 요청 DTO 없이 랭킹 데이터를 조회한다
3. THE RankController SHALL ApiResponse 타입으로 UserRankResponse 리스트를 래핑하여 반환한다
4. THE UserRankResponse SHALL rank, nickname, score, profileImage 필드를 포함한다
5. THE RankController SHALL RankListService를 의존성 주입받아 비즈니스 로직을 처리한다
6. IF 에러가 발생하면, THEN THE RankController SHALL GlobalExceptionHandler를 통해 ErrorResponse 형식으로 에러를 반환한다

### Requirement 3

**User Story:** 개발자로서, 랭킹 조회 비즈니스 로직을 서비스 계층에서 처리하고 싶습니다. 그래야 컨트롤러와 데이터 접근 로직을 분리할 수 있습니다.

#### Acceptance Criteria

1. THE RankListService SHALL UserRepository를 사용하여 사용자 데이터를 조회한다
2. THE RankListService SHALL totalScore 필드를 기준으로 내림차순 정렬을 수행한다
3. THE RankListService SHALL 조회된 User 엔티티를 UserRankResponse DTO로 변환한다
4. THE RankListService SHALL 상위 8명의 사용자만 반환하도록 제한한다
5. IF 조회된 사용자가 없으면, THEN THE RankListService SHALL 빈 리스트를 반환한다
6. IF 데이터베이스 조회 중 에러가 발생하면, THEN THE RankListService SHALL BusinessException을 발생시킨다

### Requirement 4

**User Story:** 개발자로서, 프론트엔드에서 랭킹 API를 호출하는 서비스 모듈을 작성하고 싶습니다. 그래야 컴포넌트에서 API 호출 로직을 재사용할 수 있습니다.

#### Acceptance Criteria

1. THE rankService SHALL apiClient를 사용하여 "/api/users/rank" 엔드포인트를 호출한다
2. THE rankService SHALL GET 메서드로 랭킹 데이터를 요청한다
3. THE rankService SHALL ApiResponse 형식의 응답에서 data 필드를 추출하여 반환한다
4. THE rankService SHALL apiClient의 인증 토큰 자동 처리 기능을 활용한다
5. IF API 호출이 실패하면, THEN THE rankService SHALL 에러를 호출자에게 전파한다
6. IF 서버가 ErrorResponse를 반환하면, THEN THE rankService SHALL 에러 정보를 포함하여 예외를 발생시킨다

### Requirement 5

**User Story:** 사용자로서, 랭킹 보드에서 상위 3명의 사용자를 시각적으로 구분하고 싶습니다. 그래야 최상위 랭커들을 쉽게 인식할 수 있습니다.

#### Acceptance Criteria

1. WHEN 사용자의 순위가 1위이면, THE RankingBoard SHALL 금색 트로피 아이콘을 표시한다
2. WHEN 사용자의 순위가 2위이면, THE RankingBoard SHALL 은색 메달 아이콘을 표시한다
3. WHEN 사용자의 순위가 3위이면, THE RankingBoard SHALL 동색 어워드 아이콘을 표시한다
4. WHEN 사용자의 순위가 4위 이하이면, THE RankingBoard SHALL 숫자로 순위를 표시한다
5. THE RankingBoard SHALL 상위 3명의 랭킹 아이템에 시각적 강조 스타일을 적용한다

### Requirement 6

**User Story:** 사용자로서, 프로필 이미지가 없는 사용자의 경우 기본 표시를 보고 싶습니다. 그래야 모든 사용자를 일관되게 식별할 수 있습니다.

#### Acceptance Criteria

1. WHEN 사용자의 profileImage가 null이면, THE RankingBoard SHALL 닉네임의 첫 글자를 표시한다
2. THE RankingBoard SHALL 기본 프로필 표시에 배경색을 적용한다
3. WHEN 사용자의 profileImage가 존재하면, THE RankingBoard SHALL 해당 이미지를 표시한다
4. THE RankingBoard SHALL 프로필 이미지를 원형으로 표시한다
5. THE RankingBoard SHALL 모든 프로필 표시의 크기를 일관되게 유지한다
