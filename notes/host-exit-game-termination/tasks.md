# Implementation Plan

- [x] 1. Frontend - QuizGamePage 컴포넌트에 방장 퇴장 처리 기능 추가

  - QuizGamePage.jsx에 방장 퇴장 알림 모달 상태와 핸들러 함수들을 추가합니다
  - _Requirements: 1.1, 1.2, 1.3, 1.5, 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 5.1, 5.2, 5.3, 5.4, 5.5, 6.1, 6.2, 6.3_

  - [x] 1.1 방장 퇴장 알림 모달 상태 추가

    - `showHostExitModal` state를 추가하여 AlertModal 표시 여부를 관리합니다
    - _Requirements: 2.1_

  - [x] 1.2 handleHostExit 함수 구현

    - 방장 퇴장 시 즉시 실행되는 핸들러 함수를 작성합니다
    - 진행 중인 녹화를 즉시 중단합니다 (isRecordingRef, recordingRef)
    - FastAPI WebSocket 연결을 즉시 해제합니다
    - 대기 상태를 초기화합니다 (setIsWaitingResult)
    - AlertModal을 표시합니다 (setShowHostExitModal)
    - _Requirements: 1.3, 1.4, 1.5, 2.1, 2.2, 2.3, 2.4, 6.1, 6.2_

  - [x] 1.3 cleanupResourcesAndExit 함수 구현

    - AlertModal 확인 시 실행되는 리소스 정리 함수를 작성합니다
    - Remote feeds를 모두 detach합니다
    - Publisher plugin을 Janus 방에서 leave하고 detach합니다
    - Janus 연결을 destroy합니다
    - 웹캠을 중지합니다
    - WebSocket 연결을 해제합니다
    - 메인 페이지로 이동합니다
    - 각 단계를 try-catch로 감싸서 오류 발생 시에도 계속 진행되도록 합니다
    - finally 블록에서 반드시 메인 페이지로 이동하도록 합니다
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 5.1, 5.2, 5.3, 5.4_

  - [x] 1.4 handleParticipantLeft 함수 수정

    - 기존 handleParticipantLeft 함수에 roomClosed 플래그 확인 로직을 추가합니다
    - roomClosed가 true이면 handleHostExit를 호출하고 return합니다
    - roomClosed가 false이면 기존 로직을 실행합니다
    - 기존 Toast 알림 코드를 제거합니다
    - _Requirements: 1.1, 1.2, 6.1, 6.3_

  - [x] 1.5 AlertModal 렌더링 추가

    - QuizGamePage 컴포넌트의 return 문에 AlertModal을 추가합니다
    - isOpen={showHostExitModal}로 설정합니다
    - onClose={cleanupResourcesAndExit}로 설정합니다
    - title="게임 종료"로 설정합니다
    - message="방장이 나가서 게임이 종료되었습니다"로 설정합니다
    - type="warning"으로 설정합니다
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6_

- [x] 2. Backend - 타이머 취소 로직 구현

  - 방장 퇴장 시 QuizTimerManager를 통해 모든 활성 타이머를 즉시 취소하는 로직을 구현합니다
  - _Requirements: 4.1, 5.1_

  - [x] 2.1 GameRoomLeaveService의 handleHostLeave 메서드에 타이머 정리 추가

    - handleHostLeave 메서드에서 방 종료 전에 QuizTimerManager.cleanupRoom(roomId)를 호출합니다
    - cleanupRoom은 해당 방의 모든 타이머(PREPARE, SIGNING)를 자동으로 취소합니다
    - 타이머 정리 후 로그를 출력합니다
    - try-catch로 감싸서 타이머 정리 실패 시에도 방 종료는 계속 진행되도록 합니다
    - _Requirements: 4.1, 5.1_

- [x] 3. Backend - 방 완전 종료 로직 검증 및 개선

  - 방장 퇴장 시 방이 WAITING 상태로 돌아가지 않고 완전히 종료되는지 확인하고 필요시 개선합니다
  - _Requirements: 4.2, 4.3, 4.4, 4.5, 5.2_

  - [x] 3.1 GameRoomLeaveService의 handleHostLeave 메서드 검증

    - handleHostLeave 메서드가 이미 방 종료 로직을 구현하고 있는지 확인합니다
    - room.closeRoom() 메서드가 방 상태를 적절히 변경하는지 확인합니다
    - quizStateCache.clearRoomState(roomId)가 호출되는지 확인합니다
    - 필요시 추가 정리 로직을 구현합니다
    - _Requirements: 4.2, 4.3, 5.2_

  - [x] 3.2 GameRoom.closeRoom() 메서드 확인

    - GameRoom 엔티티의 closeRoom() 메서드가 존재하는지 확인합니다
    - 방 상태를 FINISHED 또는 적절한 종료 상태로 변경하는지 확인합니다
    - 참가자 수를 0으로 초기화하는지 확인합니다
    - 필요시 메서드를 수정하거나 추가합니다
    - _Requirements: 4.2, 4.3_

  - [x] 3.3 남은 참가자 WebSocket 연결 정리 확인

    - handleHostLeave 메서드에서 sessionService.cleanupMultipleSessions가 호출되는지 확인합니다
    - 이미 구현되어 있으므로 추가 작업이 필요 없는지 확인합니다
    - _Requirements: 4.4_

- [x] 4. Backend - PARTICIPANT_LEFT 이벤트에 roomClosed 플래그 추가

  - PARTICIPANT_LEFT 이벤트에 roomClosed 플래그를 추가하여 방장 퇴장임을 명시합니다
  - _Requirements: 1.1, 1.2_

  - [x] 4.1 ParticipantLeftEvent 클래스 수정

    - ParticipantLeftEvent 클래스에 Boolean roomClosed 필드를 추가합니다
    - @Builder 어노테이션이 있으면 자동으로 빌더에 포함됩니다
    - _Requirements: 1.1_

  - [x] 4.2 PARTICIPANT_LEFT 이벤트 생성 시 roomClosed 설정

    - 방장 퇴장 처리 로직에서 PARTICIPANT_LEFT 이벤트를 생성할 때 roomClosed를 true로 설정합니다
    - 일반 참가자 퇴장 시에는 roomClosed를 false로 설정합니다
    - _Requirements: 1.1, 1.2_

- [x] 5. 통합 테스트 및 검증

  - 방장 퇴장 시나리오를 테스트하고 모든 요구사항이 충족되는지 검증합니다
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 4.1, 4.2, 4.3, 4.4, 4.5, 5.1, 5.2, 5.3, 5.4, 5.5, 6.1, 6.2, 6.3_

  - [x] 5.1 방장 퇴장 시나리오 테스트

    - 방장이 게임 중 퇴장합니다
    - 모든 참가자에게 PARTICIPANT_LEFT 이벤트가 전송되는지 확인합니다
    - 참가자들에게 AlertModal이 표시되는지 확인합니다
    - AlertModal의 확인 버튼을 클릭하면 메인 페이지로 이동하는지 확인합니다
    - 백엔드 로그에서 타이머 취소 메시지를 확인합니다
    - 방 목록에서 해당 방이 제거되었는지 확인합니다
    - _Requirements: 1.1, 1.2, 1.3, 2.1, 2.2, 2.3, 2.4, 3.8, 4.1, 4.5_

  - [x] 5.2 AlertModal 오버레이 클릭 테스트

    - 방장이 게임 중 퇴장합니다
    - AlertModal이 표시되면 오버레이를 클릭합니다
    - 메인 페이지로 이동하는지 확인합니다
    - _Requirements: 2.5, 3.2, 3.8_

  - [ ] 5.3 일반 참가자 퇴장 시나리오 테스트

    - 일반 참가자가 게임 중 퇴장합니다
    - roomClosed가 false인지 확인합니다
    - 기존 로직이 정상 작동하는지 확인합니다
    - 게임이 계속 진행되는지 확인합니다
    - _Requirements: 1.2_

  - [ ] 5.4 리소스 정리 오류 시나리오 테스트

    - 리소스 정리 중 의도적으로 오류를 발생시킵니다 (예: Janus가 이미 destroy된 상태)

    - 오류가 발생해도 메인 페이지로 이동하는지 확인합니다
    - 콘솔에 오류 로그가 출력되는지 확인합니다
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

  - [ ] 5.5 백엔드 타이머 취소 확인

    - 방장이 게임 중 퇴장합니다
    - 백엔드 로그에서 "도전 신청 타이머 취소", "준비 타이머 취소", "수어 동작 타이머 취소" 메시지를 확인합니다
    - 타이머가 더 이상 실행되지 않는지 확인합니다
    - _Requirements: 4.1_

  - [ ] 5.6 Toast 알림 제거 확인
    - 방장이 게임 중 퇴장합니다
    - Toast 알림이 표시되지 않는지 확인합니다
    - AlertModal만 표시되는지 확인합니다
    - _Requirements: 6.1, 6.2, 6.3_
