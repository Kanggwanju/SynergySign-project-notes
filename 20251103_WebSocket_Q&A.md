# SignBell 발표 Q&A 가이드

## 📋 구현 기능 요약
- **WebSocket API**: 연결(Connect), 준비(Ready), 해제(Disconnect)
- **REST API**: 방 생성, 방 상세정보 조회, 방 리스트 목록 조회

---

## 🎯 핵심 Q&A

### 1. 기술 선택

**Q1. WebSocket을 선택한 이유는?**
- 실시간 양방향 통신이 필요한 퀴즈 게임 특성
- 4명의 참가자가 동시에 상호작용하므로 낮은 지연시간 필요
- STOMP 프로토콜로 pub/sub 패턴 구현

**Q2. REST API와 WebSocket API를 어떻게 구분했나요?**
- REST API: 방 생성, 조회 등 일회성 작업
- WebSocket API: 실시간 게임 플로우 (입장, 준비, 게임 진행)

---

### 2. WebSocket 세션 관리

**Q3. WebSocket 인증은 어떻게 처리했나요?**
- JWT 토큰을 HttpOnly Secure 쿠키에 저장
- WebSocket 연결 시 쿠키에서 자동으로 JWT 추출 및 검증
- XSS 공격 방지 (HttpOnly), CSRF 방지 (SameSite=Lax)

**Q4. 중복 연결을 어떻게 방지했나요?**
- 사용자당 하나의 WebSocket 연결만 허용하는 단일 세션 정책
- `GET /api/ws/session/active` API로 활성 세션 확인
- 새로운 연결 시도 시 기존 세션이 있으면 차단
- **목적**: 게임 중인 사용자의 연결이 끊기지 않도록 보호

**Q5. 연결이 끊어지면 어떻게 처리하나요?**
- `SessionDisconnectEvent` 리스너로 자동 감지
- 참여 중인 게임방에서 자동 퇴장
- 같은 방 참가자들에게 퇴장 알림 브로드캐스트
- 방장이 나간 경우 방 자동 종료

**Q6. 스테일 세션 문제는 어떻게 해결했나요?**
- Heartbeat(4초 간격)로 연결 상태 지속 확인
- 12초 동안 응답 없으면 자동으로 세션 정리
- finally 블록에서 세션 제거 보장

---

### 3. REST API - 방 관리

**Q7. 방 생성 시 방장이 자동으로 참가자로 등록되는 이유는?**
- 방장도 게임 참가자이므로 일관성 있는 데이터 구조 유지
- 별도 입장 프로세스 없이 방 생성 즉시 게임 준비 가능

**Q8. 이미 다른 방에 참여 중인 사용자가 새 방을 만들면?**
- `409 Conflict - PARTICIPANT_ALREADY_IN_ROOM` 에러 반환
- 사용자는 한 번에 하나의 방에만 참여 가능

**Q9. 방 목록 조회에 Slice 페이징을 사용한 이유는?**
- 무한 스크롤 UI 지원을 위해
- 전체 개수 카운트 없이 `hasNext`로 다음 페이지 존재 여부만 확인
- 성능 최적화 (COUNT 쿼리 제거)

**Q10. FINISHED 상태의 방은 왜 목록에 안 보이나요?**
- 사용자는 입장 가능한 방만 필요 (WAITING/IN_PROGRESS만 표시)
- 종료된 방은 히스토리용으로만 보관

---

### 4. 준비 상태 관리

**Q11. 방장의 준비 상태는 어떻게 처리하나요?**
- 방장의 준비 상태는 자동으로 true로 설정 (v1.1.0)
- 방장은 게임 시작 권한이 있으므로 항상 준비된 상태로 간주

**Q12. 모든 참가자가 준비되면 자동으로 게임이 시작되나요?**
- 아니오, `allReady: true` 플래그만 전송
- 방장이 명시적으로 시작 버튼을 눌러야 게임 시작

---

### 5. 에러 처리

**Q13. WebSocket 에러는 어떻게 처리하나요?**
- 개인 에러는 `/user/{userId}/queue/errors` 토픽으로 전송
- 일관된 ErrorResponse 형식 (status, error, detail, path)
- 프론트엔드에서 에러 토픽 구독하여 사용자에게 알림

**Q14. 방이 가득 찬 상태에서 입장 시도하면?**
- `ROOM_FULL` 에러를 개인 에러 큐로 전송
- 최대 4명 제한을 서버에서 검증

---

### 6. 성능 최적화

**Q15. N+1 쿼리 문제는 어떻게 해결했나요?**
- `JOIN FETCH`로 연관 엔티티를 한 번에 조회
- 방 조회 시 참가자 정보를 Eager Loading으로 가져옴

**Q16. 동시 접속자가 많을 때 부하는?**
- 방당 최대 4명으로 제한
- Heartbeat로 유휴 연결 자동 정리
- 필요 시 Redis Pub/Sub으로 수평 확장 가능

---

### 7. 보안

**Q17. JWT를 쿠키에 저장하는 이유는?**
- JavaScript 접근 불가 (HttpOnly) → XSS 공격 방지
- HTTPS에서만 전송 (Secure) → 중간자 공격 방지
- SameSite=Lax → CSRF 공격 방지

**Q18. 토큰 만료 시 어떻게 처리하나요?**
- 액세스 토큰(15분) 만료 시 리프레시 토큰(7일)으로 자동 갱신
- 리프레시 토큰도 만료 시 로그인 페이지로 리다이렉트

---

### 8. 단일 세션 정책 상세

**Q19. 단일 세션 정책이란?**
- 한 사용자당 하나의 WebSocket 연결만 허용
- First-Come, First-Served: 먼저 연결된 세션을 보호하고 새 연결 차단
- **핵심 목적**: 게임 중인 사용자가 실수로 새 탭을 열어도 기존 게임이 끊기지 않도록 보호

**Q20. 어떻게 구현했나요?**
```
SingleSessionChannelInterceptor
  → CONNECT 시 기존 세션 확인
  → UserSessionRegistry에서 userId로 세션 검색
  → 기존 세션 있으면 DUPLICATE_SESSION 에러 + 연결 차단
  → 없으면 새 세션 등록
```

**Q21. 다중 세션 대신 단일 세션을 선택한 이유는?**
- 데이터 일관성: 사용자는 항상 하나의 방에만 존재
- 게임 경험 보호: 진행 중인 게임이 끊기지 않음
- 리소스 효율성: 서버 부하 66% 절감 (평균 3탭 가정)
- 로직 단순화: 탭 간 상태 동기화 불필요

**Q22. 단점은 없나요?**
- 여러 탭에서 동시 작업 불가
- 하지만 게임 중에는 하나의 탭에만 집중하므로 실제 불편함은 적음
- 명확한 에러 메시지로 사용자에게 상황 안내

**Q23. UserSessionRegistry는 무엇인가요?**
- ConcurrentHashMap으로 userId → sessionId 매핑 관리
- Thread-safe하게 동시 접속 처리
- 메서드: bind(등록), hasOtherActiveSession(중복 확인), unbind(제거)

**Q24. 스테일 세션이란?**
- 브라우저 강제 종료 등으로 DISCONNECT 메시지가 안 온 경우
- UserSessionRegistry에 죽은 세션이 남아 재접속 차단하는 문제
- Heartbeat와 finally 블록으로 해결

**Q25. 보안 측면의 장점은?**
- 세션 하이재킹 시도 시 정상 사용자의 세션이 보호됨
- 공격자의 연결 시도가 로그에 기록되어 즉시 탐지 가능
- 계정 공유 방지 (동시 접속 불가)

---

### 9. 프론트엔드 통합

**Q26. 프론트엔드에서 WebSocket을 어떻게 연결하나요?**
- `@stomp/stompjs`, `sockjs-client` 라이브러리 사용
- WebSocketService 클래스로 연결 로직 캡슐화
- React useEffect로 연결 생명주기 관리

**Q27. 구독해야 할 필수 토픽은?**
```
개인: /user/queue/room (입장 정보)
      /user/queue/errors (에러)
      /user/queue/room-closed (방 종료)
방:   /topic/room/{roomId}/participant (참가자 변경)
게임: /topic/room/{roomId}/quiz/* (게임 진행)
```

---

### 10. 테스트

**Q28. WebSocket API는 어떻게 테스트했나요?**
- @SpringBootTest + WebSocketStompClient로 통합 테스트
- Postman WebSocket 기능으로 수동 테스트
- 프론트엔드와 통합 테스트

---

## 💡 발표 팁

### 강조할 포인트
1. **실시간 양방향 통신**: WebSocket + STOMP 선택 이유
2. **First-Come 단일 세션**: 게임 중인 사용자 보호
3. **보안**: HttpOnly 쿠키 + 세션 하이재킹 탐지
4. **성능**: N+1 해결, Slice 페이징, 리소스 66% 절감

### 데모 시나리오
1. 방 생성 → 목록 조회 → 방 입장 → 준비 → 게임 시작
2. **단일 세션 시연**:
    - 탭1에서 게임 중
    - 탭2에서 연결 시도 → "이미 접속 중입니다" 메시지
    - 탭1은 계속 진행 ✓
3. 한 명 연결 해제 → 자동 퇴장 알림
4. 방장 나가면 → 방 종료 알림

### 답변 요령
- 기술 용어는 간단히 설명
- "왜 그렇게 했는가"에 중점
- 한계점도 솔직히 말하되 개선 방향 제시
- **단일 세션의 핵심**: 게임 경험 보호