# Implementation Plan

- [x] 1. QuizTimerManager 컴포넌트 생성

  - QuizTimerManager 클래스 생성 및 기본 구조 구현
  - ScheduledExecutorService 초기화 (스레드 풀 크기 100)
  - activeTimers Map 선언 (ConcurrentHashMap)
  - buildKey 메서드 구현 (타이머 키 생성)
  - _Requirements: 1.1, 1.2, 1.3, 1.4_

- [x] 2. QuizTimerManager 타이머 스케줄링 기능 구현

  - scheduleTimer 메서드 구현
  - 기존 타이머 자동 취소 로직 추가
  - ScheduledFuture 리스트 관리
  - 타이머 스케줄링 로깅 추가
  - _Requirements: 2.1, 2.2, 2.3, 6.2_

- [x] 3. QuizTimerManager 타이머 취소 기능 구현

  - cancelTimer 메서드 구현
  - 개별 ScheduledFuture 취소 로직
  - 취소 성공/실패 카운팅
  - cancelAllTimersForQuestion 메서드 구현
  - 타이머 취소 로깅 추가 (성공/실패 건수 포함)
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 6.3_

- [x] 4. QuizStateCache에 현재 문제 번호 추적 기능 추가

  - GameRoomState에 currentQuestionNumber 필드 추가
  - getCurrentQuestionNumber 메서드 구현
  - setCurrentQuestionNumber 메서드 구현
  - _Requirements: 1.1, 1.2_

- [x] 5. QuizService에 타이머 메시지 검증 로직 추가

  - sendTimerUpdate private 메서드 생성
  - 현재 도전자 유효성 검증 로직 구현
  - 도전자 변경 시 메시지 전송 스킵 로직
  - 타이머 메시지 전송 실패 처리
  - _Requirements: 4.1, 4.2, 4.3, 4.5, 6.5_

- [x] 6. QuizService의 startPrepareTimer 메서드 리팩토링

  - 기존 CompletableFuture 방식 제거
  - QuizTimerManager를 사용한 타이머 스케줄링으로 변경
  - sendTimerUpdate 메서드 호출로 변경
  - 5초 후 startSigningTimer 호출 로직 유지
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6_

- [x] 7. QuizService의 startSigningTimer 메서드 리팩토링

  - 기존 CompletableFuture 방식 제거
  - QuizTimerManager를 사용한 타이머 스케줄링으로 변경
  - sendTimerUpdate 메서드 호출로 변경
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6_

- [x] 8. QuizService에 참가자 퇴장 시 타이머 재시작 로직 구현

  - handleParticipantLeft 메서드 수정
  - 현재 문제 번호 조회 로직 추가
  - 현재 도전자 확인 로직 추가
  - 도전자 퇴장 시 타이머 취소 로직 추가
  - 다음 도전자 확인 및 차례 넘김 로직 추가
  - 다음 도전자에게 새 타이머 시작 로직 추가
  - 다음 도전자 없을 시 다음 문제로 이동 로직 추가
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.1, 2.2, 2.3, 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 5.1, 5.2, 5.3, 6.1_

- [x] 9. QuizService의 게임 시작 시 현재 문제 번호 설정

  - startGame 메서드에 currentQuestionNumber 설정 추가
  - 첫 번째 문제 시작 시 questionNumber=1 설정
  - _Requirements: 1.1, 1.2_

- [x] 10. QuizService의 다음 문제 이동 시 현재 문제 번호 업데이트

  - moveToNextQuestion 메서드에 currentQuestionNumber 업데이트 추가
  - 이전 문제의 타이머 정리 로직 추가
  - _Requirements: 1.1, 1.2_

- [x] 11. QuizService의 게임 종료 시 타이머 정리

  - endGame 메서드에 모든 타이머 취소 로직 추가
  - QuizTimerManager.cleanupRoom 호출
  - _Requirements: 5.1, 5.2_

- [x] 12. QuizTimerManager에 방 정리 메서드 추가

  - cleanupRoom 메서드 구현
  - 특정 방의 모든 타이머 제거 로직
  - 정리 완료 로깅 추가
  - _Requirements: 5.1, 5.2, 6.3_

- [x] 13. 동시 퇴장 처리를 위한 동기화 추가

  - QuizService에 participantLeftLock 객체 추가
  - handleParticipantLeft 메서드에 synchronized 블록 추가
  - _Requirements: 5.6_

- [x] 14. 엣지 케이스 처리 로직 추가

  - 도전 신청 단계(challenge)에서 퇴장 시 타이머 재시작 스킵
  - 결과 표시 단계(result)에서 퇴장 시 타이머 재시작 스킵
  - 다음 도전자 없을 시 타이머 재시작 스킵
  - _Requirements: 5.1, 5.2, 5.3_

- [ ]\* 15. QuizTimerManager 단위 테스트 작성

  - scheduleTimer 테스트
  - cancelTimer 테스트
  - cancelAllTimersForQuestion 테스트
  - cleanupRoom 테스트
  - _Requirements: 모든 요구사항_

- [ ]\* 16. QuizService 타이머 재시작 단위 테스트 작성

  - handleParticipantLeft_CurrentChallenger 테스트
  - handleParticipantLeft_NotCurrentChallenger 테스트
  - handleParticipantLeft_NoNextChallenger 테스트
  - _Requirements: 모든 요구사항_

- [ ]\* 17. 통합 테스트 작성

  - 정상 시나리오 테스트 (퇴장 없음)
  - 준비 시간 중 퇴장 테스트
  - 수어 시간 중 퇴장 테스트
  - 마지막 도전자 퇴장 테스트
  - 동시 퇴장 테스트
  - _Requirements: 모든 요구사항_

- [x] 18. 로깅 개선

  - 타이머 시작 로그 추가 (INFO 레벨)
  - 타이머 취소 로그 추가 (INFO 레벨)
  - 타이머 전송 실패 로그 추가 (ERROR 레벨)
  - 도전자 퇴장 로그 추가 (INFO 레벨)
  - 타이머 상태 로그 추가 (DEBUG 레벨)
  - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

- [x] 19. 코드 정리 및 문서화

  - 기존 CompletableFuture 방식 코드 제거 확인
  - JavaDoc 주석 추가 (작성자: 강관주 (Kanggwanju))
  - 메서드 시그니처 정리
  - 불필요한 import 제거
  - _Requirements: 모든 요구사항_
