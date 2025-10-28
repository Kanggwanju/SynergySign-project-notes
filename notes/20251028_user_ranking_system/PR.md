# Pull Request

> 제목: **[Feat]: 메인 페이지 사용자 랭킹 시스템 구현 (#이슈번호)**

## PR 유형 (Type of PR)

- [x] 기능 (Feature)
- [ ] 버그 수정 (Bugfix)
- [ ] 기능 개선 (Enhancement)
- [ ] 리팩토링 (Refactoring)
- [ ] 문서 (Docs)
- [ ] 기타 (Chore)

---

## PR 요약 (PR Summary)

메인 페이지에 사용자의 누적 점수(totalScore)를 기반으로 상위 8명의 랭킹을 표시하는 시스템을 구현했습니다. 백엔드는 Spring Boot 기반의 RESTful API로 랭킹 데이터를 제공하고, 프론트엔드는 React 컴포넌트로 시각적으로 표현합니다. 상위 3명은 트로피/메달 아이콘으로 강조되며, 로딩 및 에러 상태를 처리합니다.

---

## 관련 이슈 (Related Issue)

- Closes #
- Fixes #
- Resolves #
- Relates to #

---

## 변경 사항 (Changes)

### 백엔드 (Backend)

**1. DTO 및 Repository 확장**
- `UserRankResponse` DTO 생성
  - 필드: rank, nickname, score, profileImage
  - 정적 팩토리 메서드 `from(User user, int rank)` 구현
- `UserRepository`에 `findTop8ByOrderByTotalScoreDesc()` 메서드 추가
  - totalScore 기준 내림차순 정렬로 상위 8명 조회

**2. 서비스 계층 구현**
- `RankListService` 클래스 생성
  - `getTopRankings()` 메서드: 랭킹 조회 및 DTO 변환
  - 빈 리스트 처리 로직
  - BusinessException 에러 처리 (데이터베이스 오류 시)
  - `@Transactional(readOnly = true)` 적용

**3. 컨트롤러 구현**
- `RankController` 클래스 생성
  - `GET /api/users/rank` 엔드포인트 제공
  - ApiResponse로 응답 래핑
  - 로깅 추가 (요청/응답 로그)

### 프론트엔드 (Frontend)

**1. API 서비스 모듈**
- `rankService.js` 생성
  - `getRankings()` 함수: `/users/rank` API 호출
  - ApiResponse의 data 필드 추출
  - 에러 전파 처리

**2. RankingBoard 컴포넌트**
- 랭킹 데이터 표시 UI 구현
- 상태 관리: rankings, isLoading, error
- 순위별 아이콘 표시
  - 1위: 금색 트로피 (🏆)
  - 2위: 은색 메달 (🥈)
  - 3위: 동색 어워드 (🥉)
  - 4~8위: 숫자 표시
- 프로필 이미지 처리
  - 이미지 있으면 표시
  - 없으면 닉네임 첫 글자 표시
- 로딩 상태 표시 (스피너 + 메시지)
- 에러 상태 처리 (ErrorResponse 형식 지원)

**3. MainPage 통합**
- RankingBoard 컴포넌트를 메인 페이지 우측 섹션에 배치

---

## 스크린샷 (Screenshots) (Optional)

(랭킹 보드 UI 스크린샷 첨부)

---

## 테스트 방법 (Test Steps)

### 백엔드 테스트

1. 애플리케이션 실행
2. Postman 또는 브라우저에서 `GET http://localhost:8080/api/users/rank` 호출
3. 응답 확인:
   ```json
   {
     "success": true,
     "message": "랭킹 조회에 성공했습니다.",
     "timestamp": "2025-10-28T...",
     "data": [
       {
         "rank": 1,
         "nickname": "사용자1",
         "score": 1250,
         "profileImage": "https://..."
       },
       ...
     ]
   }
   ```

### 프론트엔드 테스트

1. 프론트엔드 개발 서버 실행
2. 메인 페이지 접속
3. 우측 랭킹 보드 확인:
   - 상위 8명의 사용자 표시
   - 1~3위 아이콘 표시 확인
   - 프로필 이미지 또는 기본 표시 확인
   - 점수 포맷팅 확인 (천 단위 콤마)

### 에러 시나리오 테스트

1. 백엔드 서버 중지 후 프론트엔드 접속
2. 에러 메시지 표시 확인: "랭킹 정보를 불러올 수 없습니다."

### 로딩 상태 테스트

1. 네트워크 속도를 느리게 설정 (개발자 도구)
2. 메인 페이지 접속
3. 로딩 스피너 및 "랭킹 불러오는 중..." 메시지 확인

---

## 셀프 체크리스트 (Self-Checklist)

- [x] 제목 규칙을 지켰습니다.
- [ ] 관련 이슈를 연결했습니다.
- [x] 로컬에서 충분히 테스트했습니다.
- [ ] `dev` 브랜치를 Pull하여 최신 코드를 반영했습니다.
- [ ] CI/CD 파이프라인이 통과했습니다.

---

## 리뷰어에게 (To Reviewers)

### 주요 검토 포인트

1. **백엔드 아키텍처**
   - 3-tier 구조 (Controller → Service → Repository) 준수 여부
   - DTO 변환 로직의 적절성
   - 에러 처리 방식 (BusinessException, GlobalExceptionHandler)

2. **API 설계**
   - RESTful API 규칙 준수 여부
   - ApiResponse 래핑 일관성
   - 응답 메시지의 명확성

3. **프론트엔드 구현**
   - 컴포넌트 구조 및 상태 관리
   - 에러 처리 및 사용자 피드백
   - UI/UX (아이콘 표시, 프로필 이미지 처리)

4. **성능 고려사항**
   - 상위 8명만 조회하는 쿼리 최적화
   - `@Transactional(readOnly = true)` 적용

### 참고 사항

- 현재 캐싱은 구현되지 않았습니다. 향후 필요 시 Redis 캐싱 고려 가능합니다.
- totalScore 필드에 인덱스가 있으면 쿼리 성능이 향상됩니다.
- 테스트 코드는 선택적으로 작성되었습니다 (tasks.md의 optional 항목).

### 관련 문서

- 요구사항: `.kiro/specs/user-ranking-system/requirements.md`
- 설계: `.kiro/specs/user-ranking-system/design.md`
- 작업 목록: `.kiro/specs/user-ranking-system/tasks.md`
