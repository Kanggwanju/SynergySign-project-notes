# Feature

> 제목: **[Feat]: 메인 페이지 사용자 랭킹 시스템 구현**

## 제안 배경 (Background)

SignBell 애플리케이션의 사용자들이 자신의 게임 성과를 다른 사용자들과 비교하고, 경쟁심을 유발하여 더 많은 참여를 유도하기 위해 랭킹 시스템이 필요합니다. 현재는 사용자의 누적 점수(totalScore)가 데이터베이스에 저장되고 있지만, 이를 시각적으로 표현하고 순위를 제공하는 기능이 없어 사용자들이 자신의 위치를 파악하기 어렵습니다.

랭킹 시스템을 통해:
- 사용자들의 게임 참여 동기를 부여할 수 있습니다
- 상위 랭커들을 시각적으로 강조하여 성취감을 제공할 수 있습니다
- 커뮤니티 내 경쟁 문화를 조성할 수 있습니다

---

## 제안할 기능 (Proposed Feature)

메인 페이지에 **상위 8명의 사용자 랭킹 보드**를 표시하는 기능을 구현합니다.

**사용자 경험:**
- 메인 페이지 접속 시 자동으로 랭킹 정보가 로드됩니다
- 각 랭커의 순위, 닉네임, 누적 점수, 프로필 이미지가 표시됩니다
- 1~3위는 트로피/메달 아이콘으로 시각적으로 강조됩니다
- 4~8위는 숫자로 순위가 표시됩니다
- 프로필 이미지가 없는 경우 닉네임 첫 글자가 표시됩니다
- 로딩 중에는 로딩 인디케이터가 표시됩니다
- 에러 발생 시 사용자 친화적인 에러 메시지가 표시됩니다

---

## 주요 작업 목록 (To-Do List)

### 백엔드 작업
- [x] 작업 1: UserRankResponse DTO 생성 (rank, nickname, score, profileImage 필드)
- [x] 작업 2: UserRepository에 findTop8ByOrderByTotalScoreDesc() 메서드 추가
- [ ] 작업 3: RankListService 구현 (랭킹 조회 비즈니스 로직)
- [ ] 작업 4: RankController 구현 (GET /api/users/rank 엔드포인트)
- [ ] 작업 5: 에러 처리 로직 구현 (BusinessException, ErrorCode)

### 프론트엔드 작업
- [ ] 작업 6: rankService 모듈 구현 (API 호출 로직)
- [ ] 작업 7: RankingBoard 컴포넌트 구현 (UI 렌더링)
- [ ] 작업 8: 순위별 아이콘 표시 로직 구현 (1~3위 트로피/메달)
- [ ] 작업 9: 프로필 이미지 처리 로직 구현 (기본 표시 포함)
- [ ] 작업 10: 로딩 및 에러 상태 처리 구현
- [ ] 작업 11: MainPage에 RankingBoard 컴포넌트 통합

### 테스트 작업
- [ ] 작업 12: 백엔드 단위 테스트 작성 (Service, Repository)
- [ ] 작업 13: API 통합 테스트 작성
- [ ] 작업 14: 프론트엔드 컴포넌트 테스트 작성

---

## 기술적인 구현 방안 (Implementation Details)

### 백엔드 (Spring Boot)

**1. DTO 구조**
```java
public class UserRankResponse {
    private Integer rank;        // 순위 (1~8)
    private String nickname;     // 닉네임
    private Long score;          // 누적 점수
    private String profileImage; // 프로필 이미지 URL (nullable)
}
```

**2. API 엔드포인트**
- **URL**: `GET /api/users/rank`
- **인증**: JWT 토큰 기반 인증 (기존 시스템 활용)
- **응답 형식**: `ApiResponse<List<UserRankResponse>>`
- **에러 응답**: `ErrorResponse` (GlobalExceptionHandler 처리)

**3. 데이터 조회**
- Spring Data JPA의 쿼리 메서드 활용: `findTop8ByOrderByTotalScoreDesc()`
- totalScore 필드 기준 내림차순 정렬
- 상위 8명만 조회하여 성능 최적화

**4. 계층 구조**
```
RankController → RankListService → UserRepository → User Entity
```

### 프론트엔드 (React)

**1. API 서비스 모듈**
```javascript
// services/rankService.js
export const getRankings = async () => {
  const response = await apiClient.get('/api/users/rank');
  return response.data.data; // ApiResponse에서 data 추출
};
```

**2. 컴포넌트 구조**
```
MainPage
  └── RankingBoard
       └── RankItem (각 랭커 표시)
```

**3. 상태 관리**
- `useState`로 랭킹 데이터, 로딩 상태, 에러 상태 관리
- `useEffect`로 컴포넌트 마운트 시 자동 데이터 로드

**4. UI 구현**
- 1위: 금색 트로피 아이콘 (🏆)
- 2위: 은색 메달 아이콘 (🥈)
- 3위: 동색 어워드 아이콘 (🥉)
- 4~8위: 숫자 표시
- 프로필 이미지: 원형 표시, 없으면 닉네임 첫 글자 + 배경색

**5. 에러 처리**
- try-catch로 API 호출 에러 처리
- 사용자 친화적인 에러 메시지 표시
- 재시도 옵션 제공 (선택적)

---

## 스크린샷 또는 와이어프레임 (Screenshots or Wireframes)

```
┌─────────────────────────────────────┐
│         사용자 랭킹 보드            │
├─────────────────────────────────────┤
│ 🏆 1위  [프로필]  닉네임1   1,250점 │
│ 🥈 2위  [프로필]  닉네임2   1,100점 │
│ 🥉 3위  [프로필]  닉네임3     980점 │
│  4위   [프로필]  닉네임4     850점 │
│  5위   [프로필]  닉네임5     720점 │
│  6위   [프로필]  닉네임6     650점 │
│  7위   [프로필]  닉네임7     580점 │
│  8위   [프로필]  닉네임8     520점 │
└─────────────────────────────────────┘
```

---

## 참고 자료 (References)

### 관련 문서
- `.kiro/specs/user-ranking-system/requirements.md` - 상세 요구사항 문서
- `.kiro/specs/user-ranking-system/design.md` - 설계 문서
- `.kiro/specs/user-ranking-system/tasks.md` - 구현 작업 목록

### 기존 코드베이스 참고
- `User.java` - totalScore 필드 구조
- `UserRepository.java` - 기존 쿼리 메서드 패턴
- `UserProfileResponse.java` - DTO 패턴 참고
- `apiClient` - 기존 HTTP 클라이언트 활용
- `ApiResponse` - 공통 응답 래퍼 구조
- `GlobalExceptionHandler` - 에러 처리 패턴

### 기술 스택
- **백엔드**: Spring Boot, Spring Data JPA, Lombok
- **프론트엔드**: React, Axios
- **데이터베이스**: MySQL (totalScore BIGINT 타입)
- **인증**: JWT 토큰 기반 인증

### 성능 고려사항
- 상위 8명만 조회하여 쿼리 성능 최적화
- 인덱스 활용: totalScore 필드에 인덱스 추가 권장
- 캐싱 전략: 향후 Redis 캐싱 고려 가능 (현재 단계에서는 미구현)
