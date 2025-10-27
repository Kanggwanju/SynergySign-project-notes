# Requirements Document

## Introduction

퀴즈 게임 진행 중 현재 도전자가 준비 시간(PREPARE 타이머) 또는 수어 동작 시간(SIGNING 타이머) 중에 방을 나갔을 때, 다음 도전자가 타이머를 초기화된 상태로 시작할 수 있도록 하는 기능입니다. 현재는 이전 도전자가 사용하던 남은 시간을 그대로 이어받아 불공평한 상황이 발생하고 있습니다.

이 기능은 게임의 공정성을 보장하고, 참가자 퇴장으로 인한 부작용을 최소화하여 원활한 게임 진행을 가능하게 합니다.

## Requirements

### Requirement 1: 도전자 퇴장 시 타이머 상태 감지

**User Story:** As a 게임 시스템, I want 현재 도전자가 퇴장할 때 진행 중인 타이머 상태를 정확히 감지하고 싶다, so that 다음 도전자에게 올바른 타이머 설정을 제공할 수 있다

#### Acceptance Criteria

1. WHEN 현재 도전자(isCurrentPlayer=true)가 PARTICIPANT_LEFT 이벤트를 발생시키면 THEN 시스템은 현재 게임 단계(gamePhase)를 확인해야 한다
2. WHEN 퇴장 시점의 게임 단계가 'myTurn' 또는 'solving'이면 THEN 시스템은 진행 중인 타이머 타입(PREPARE 또는 SIGNING)을 식별해야 한다
3. WHEN 도전자가 준비 시간(PREPARE) 중에 퇴장하면 THEN 시스템은 solvingTimer 값을 기록해야 한다
4. WHEN 도전자가 수어 동작 시간(SIGNING) 중에 퇴장하면 THEN 시스템은 signingTimer 값을 기록해야 한다

### Requirement 2: 백엔드 타이머 재시작 요청

**User Story:** As a 프론트엔드 시스템, I want 도전자 퇴장 시 백엔드에 타이머 재시작을 요청하고 싶다, so that 다음 도전자가 공정한 시간을 받을 수 있다

#### Acceptance Criteria

1. WHEN 현재 도전자가 타이머 진행 중에 퇴장하면 THEN 프론트엔드는 백엔드에 타이머 재시작 요청을 전송해야 한다
2. IF 백엔드가 이미 다음 도전자 정보(nextChallengerId)를 PARTICIPANT_LEFT 이벤트에 포함했다면 THEN 프론트엔드는 타이머 재시작 요청을 즉시 전송해야 한다
3. WHEN 타이머 재시작 요청이 전송되면 THEN 요청에는 roomId, questionNumber, timerType(PREPARE 또는 SIGNING) 정보가 포함되어야 한다
4. IF 타이머 재시작 요청이 실패하면 THEN 시스템은 에러를 로깅하고 사용자에게 알림을 표시해야 한다

### Requirement 3: 백엔드 타이머 재시작 처리

**User Story:** As a 백엔드 시스템, I want 타이머 재시작 요청을 받아 처리하고 싶다, so that 다음 도전자에게 초기화된 타이머를 제공할 수 있다

#### Acceptance Criteria

1. WHEN 백엔드가 타이머 재시작 요청을 받으면 THEN 현재 진행 중인 타이머를 중단해야 한다
2. WHEN 타이머가 중단되면 THEN 해당 타이머 타입(PREPARE 또는 SIGNING)의 초기값으로 재설정해야 한다
3. IF 타이머 타입이 PREPARE이면 THEN 5초로 재설정해야 한다
4. IF 타이머 타입이 SIGNING이면 THEN 5초로 재설정해야 한다
5. WHEN 타이머가 재설정되면 THEN 새로운 타이머를 시작하고 모든 참가자에게 TIMER_UPDATE 이벤트를 브로드캐스트해야 한다
6. WHEN TIMER_UPDATE 이벤트가 전송되면 THEN 이벤트에는 timerType, remainingSeconds(초기값) 정보가 포함되어야 한다

### Requirement 4: 프론트엔드 타이머 UI 동기화

**User Story:** As a 게임 참가자, I want 도전자 교체 시 타이머가 초기화되는 것을 시각적으로 확인하고 싶다, so that 공정한 게임이 진행되고 있음을 알 수 있다

#### Acceptance Criteria

1. WHEN 프론트엔드가 TIMER_UPDATE 이벤트를 수신하면 THEN 해당 타이머 타입의 UI를 업데이트해야 한다
2. IF timerType이 PREPARE이면 THEN solvingTimer 상태를 remainingSeconds 값으로 업데이트해야 한다
3. IF timerType이 SIGNING이면 THEN signingTimer 상태를 remainingSeconds 값으로 업데이트해야 한다
4. WHEN 타이머가 재시작되면 THEN 다음 도전자의 화면에 "준비하세요!" 또는 유사한 알림 메시지를 표시해야 한다
5. WHEN 타이머 UI가 업데이트되면 THEN 변경사항이 즉시 화면에 반영되어야 한다

### Requirement 5: 엣지 케이스 처리

**User Story:** As a 시스템 관리자, I want 예외 상황에서도 안정적으로 동작하는 타이머 재시작 기능을 원한다, so that 게임이 중단되지 않고 계속 진행될 수 있다

#### Acceptance Criteria

1. WHEN 도전자가 도전 신청 단계(challenge)에서 퇴장하면 THEN 타이머 재시작 로직을 실행하지 않아야 한다
2. WHEN 도전자가 결과 표시 단계(result)에서 퇴장하면 THEN 타이머 재시작 로직을 실행하지 않아야 한다
3. IF 다음 도전자가 없으면 THEN 타이머 재시작 요청을 전송하지 않아야 한다
4. WHEN 타이머 재시작 중 네트워크 오류가 발생하면 THEN 시스템은 재시도 로직을 실행해야 한다 (최대 3회)
5. IF 재시도가 모두 실패하면 THEN 에러를 로깅하고 게임을 challenge 단계로 되돌려야 한다
6. WHEN 여러 참가자가 동시에 퇴장하면 THEN 각 퇴장 이벤트를 순차적으로 처리해야 한다

### Requirement 6: 로깅 및 모니터링

**User Story:** As a 개발자, I want 타이머 재시작 과정을 상세히 로깅하고 싶다, so that 문제 발생 시 디버깅이 용이하다

#### Acceptance Criteria

1. WHEN 도전자가 타이머 진행 중에 퇴장하면 THEN 퇴장 시점의 타이머 상태(타입, 남은 시간)를 로깅해야 한다
2. WHEN 타이머 재시작 요청이 전송되면 THEN 요청 내용(roomId, questionNumber, timerType)을 로깅해야 한다
3. WHEN 백엔드가 타이머를 재시작하면 THEN 재시작 성공 여부와 새로운 타이머 값을 로깅해야 한다
4. WHEN 타이머 재시작이 실패하면 THEN 실패 원인과 스택 트레이스를 로깅해야 한다
5. WHEN 프론트엔드가 TIMER_UPDATE 이벤트를 수신하면 THEN 수신한 타이머 값을 로깅해야 한다
