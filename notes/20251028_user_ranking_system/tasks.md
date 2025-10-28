# Implementation Plan

- [x] 1. 백엔드 DTO 및 Repository 확장

  - UserRankResponse DTO 생성 (rank, nickname, score, profileImage 필드 포함)
  - UserRepository에 상위 8명 조회 메서드 추가 (findTop8ByOrderByTotalScoreDesc)
  - 정적 팩토리 메서드 from(User user, int rank) 구현
  - _Requirements: 2.4, 3.1, 3.2, 3.3, 3.4_

- [x] 2. 백엔드 서비스 계층 구현

  - RankListService 클래스 생성
  - getTopRankings() 메서드 구현 (UserRepository 호출, DTO 변환, 순위 할당)
  - 빈 리스트 처리 로직 추가
  - BusinessException 에러 처리 추가 (데이터베이스 조회 실패 시)
  - @Transactional(readOnly = true) 어노테이션 적용
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6_

- [x] 3. 백엔드 컨트롤러 구현

  - RankController 클래스 생성 (controller/user 패키지)
  - GET /api/users/rank 엔드포인트 구현
  - RankListService 의존성 주입
  - ApiResponse로 응답 래핑
  - 로깅 추가 (@Slf4j)
  - _Requirements: 2.1, 2.2, 2.3, 2.5, 2.6_

- [x] 4. 프론트엔드 API 서비스 구현

  - rankService.js 파일 생성 (services/api 폴더)
  - getRankings() 함수 구현 (apiClient.get('/users/rank') 호출)
  - ApiResponse의 data 필드 추출 로직 추가
  - 에러 처리 및 전파
  - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6_

- [x] 5. 프론트엔드 RankingBoard 컴포넌트 수정

  - 더미 데이터 제거
  - rankService import 및 호출
  - fetchRankings() 함수를 rankService.getRankings()로 변경
  - 에러 처리 강화 (ErrorResponse 처리)
  - 기존 UI 로직 유지 (getRankIcon, getProfileImage)
  - _Requirements: 1.1, 1.3, 1.4, 1.5, 5.1, 5.2, 5.3, 5.4, 5.5, 6.1, 6.2, 6.3, 6.4, 6.5_

- [ ] 6. 통합 테스트 및 검증




  - 백엔드 API 엔드포인트 수동 테스트 (Postman 또는 브라우저)
  - 프론트엔드에서 랭킹 데이터 로드 확인
  - 상위 3명 아이콘 표시 확인
  - 프로필 이미지 null 처리 확인
  - 에러 시나리오 테스트 (데이터베이스 연결 실패, 인증 실패)
  - 로딩 상태 표시 확인
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_
