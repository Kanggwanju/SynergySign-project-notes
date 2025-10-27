# Requirements Document

## Introduction

현재 퀴즈 게임에서 방장이 게임 중 퇴장하면 백엔드 타이머가 계속 실행되고, 방이 WAITING 상태로 돌아가 빈 대기방이 남는 문제가 있습니다. 이 기능은 방장이 게임 중 퇴장할 때 모든 참가자에게 알림을 표시하고, 게임 리소스를 정리한 후 메인 페이지로 안전하게 이동시키는 것을 목표로 합니다.

## Requirements

### Requirement 1: 방장 퇴장 감지 및 브로드캐스팅

**User Story:** 게임 참가자로서, 방장이 게임 중 퇴장하면 즉시 알림을 받아 게임이 종료됨을 인지하고 싶습니다.

#### Acceptance Criteria

1. WHEN 방장이 게임 중 퇴장하면 THEN 백엔드는 모든 참가자에게 `PARTICIPANT_LEFT` 이벤트를 브로드캐스트해야 합니다
2. WHEN `PARTICIPANT_LEFT` 이벤트가 수신되고 `roomClosed` 플래그가 true이면 THEN 프론트엔드는 방장 퇴장으로 인한 게임 종료임을 인지해야 합니다
3. WHEN 방장 퇴장 이벤트가 감지되면 THEN 현재 진행 중인 모든 타이머(도전 신청, 준비, 수어 동작)가 즉시 중단되어야 합니다
4. WHEN 방장 퇴장 이벤트가 감지되면 THEN FastAPI WebSocket 연결이 즉시 해제되어야 합니다
5. WHEN 방장 퇴장 이벤트가 감지되면 THEN 진행 중인 프레임 캡처 및 녹화가 즉시 중단되어야 합니다

### Requirement 2: 방장 퇴장 알림 모달 표시

**User Story:** 게임 참가자로서, 방장이 퇴장했을 때 명확한 알림 모달을 통해 게임 종료 사유를 이해하고 싶습니다.

#### Acceptance Criteria

1. WHEN 방장 퇴장이 감지되면 THEN AlertModal 컴포넌트가 자동으로 표시되어야 합니다
2. WHEN AlertModal이 표시되면 THEN 모달 타입은 'warning'이어야 합니다
3. WHEN AlertModal이 표시되면 THEN 제목은 "게임 종료"여야 합니다
4. WHEN AlertModal이 표시되면 THEN 메시지는 "방장이 나가서 게임이 종료되었습니다"여야 합니다
5. WHEN AlertModal이 표시되면 THEN 모달 뒤의 오버레이는 클릭 가능해야 합니다
6. WHEN AlertModal이 표시되면 THEN 확인 버튼이 표시되어야 합니다

### Requirement 3: 리소스 정리 및 메인 페이지 이동

**User Story:** 게임 참가자로서, 방장 퇴장 알림 모달에서 확인 버튼을 누르면 모든 게임 리소스가 정리되고 메인 페이지로 안전하게 이동하고 싶습니다.

#### Acceptance Criteria

1. WHEN 사용자가 AlertModal의 확인 버튼을 클릭하면 THEN 모든 WebRTC 연결(Janus)이 정리되어야 합니다
2. WHEN 사용자가 AlertModal의 오버레이를 클릭하면 THEN 모든 WebRTC 연결(Janus)이 정리되어야 합니다
3. WHEN 리소스 정리가 시작되면 THEN Remote feeds가 모두 detach되어야 합니다
4. WHEN 리소스 정리가 시작되면 THEN Publisher plugin이 Janus 방에서 leave하고 detach되어야 합니다
5. WHEN 리소스 정리가 시작되면 THEN Janus 연결이 destroy되어야 합니다
6. WHEN 리소스 정리가 시작되면 THEN 웹캠이 중지되어야 합니다
7. WHEN 리소스 정리가 시작되면 THEN FastAPI WebSocket 연결이 해제되어야 합니다
8. WHEN 모든 리소스 정리가 완료되면 THEN 사용자는 `/main` 경로로 이동해야 합니다
9. WHEN 메인 페이지로 이동하면 THEN 게임 관련 모든 상태가 초기화되어야 합니다

### Requirement 4: 백엔드 타이머 및 방 상태 관리

**User Story:** 시스템 관리자로서, 방장이 퇴장하면 백엔드의 모든 타이머가 중단되고 방이 완전히 종료되어 빈 대기방이 남지 않기를 원합니다.

#### Acceptance Criteria

1. WHEN 방장이 게임 중 퇴장하면 THEN 백엔드의 모든 활성 타이머(도전 신청, 준비, 수어 동작)가 즉시 취소되어야 합니다
2. WHEN 방장이 게임 중 퇴장하면 THEN 방 상태가 WAITING으로 돌아가지 않고 완전히 종료되어야 합니다
3. WHEN 방장이 게임 중 퇴장하면 THEN 방 정보가 데이터베이스에서 삭제되거나 비활성화되어야 합니다
4. WHEN 방장이 게임 중 퇴장하면 THEN 남은 참가자들의 WebSocket 연결이 정리되어야 합니다
5. WHEN 방이 종료되면 THEN 방 목록에서 해당 방이 제거되어야 합니다

### Requirement 5: 에러 처리 및 예외 상황

**User Story:** 게임 참가자로서, 방장 퇴장 처리 중 오류가 발생해도 안전하게 메인 페이지로 이동하고 싶습니다.

#### Acceptance Criteria

1. WHEN 리소스 정리 중 오류가 발생하면 THEN 오류를 콘솔에 로깅하고 계속 진행해야 합니다
2. WHEN WebRTC 연결 해제 중 오류가 발생하면 THEN 다른 리소스 정리는 계속 진행되어야 합니다
3. WHEN FastAPI 연결 해제 중 오류가 발생하면 THEN 다른 리소스 정리는 계속 진행되어야 합니다
4. WHEN 메인 페이지 이동 중 오류가 발생하면 THEN 사용자에게 에러 토스트를 표시하고 강제로 이동해야 합니다
5. WHEN 방장 퇴장 이벤트 처리 중 예외가 발생하면 THEN 기본 정리 로직이 실행되어야 합니다

### Requirement 6: 기존 Toast 알림 제거

**User Story:** 게임 참가자로서, 방장 퇴장 시 Toast 알림 대신 명확한 모달을 통해 알림을 받고 싶습니다.

#### Acceptance Criteria

1. WHEN 방장 퇴장 이벤트가 처리되면 THEN 기존 Toast 알림("방장이 나가서 게임이 종료되었습니다")이 표시되지 않아야 합니다
2. WHEN AlertModal이 표시되면 THEN Toast 알림은 표시되지 않아야 합니다
3. WHEN 방장 퇴장 처리가 완료되면 THEN AlertModal만 표시되어야 합니다
