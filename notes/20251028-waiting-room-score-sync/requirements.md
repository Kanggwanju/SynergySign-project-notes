# Requirements Document

## Introduction

현재 퀴즈 대기방에서 모든 참가자의 점수가 하드코딩된 `0`으로 표시되어 실제 DB의 점수와 동기화되지 않는 문제가 있습니다. 이 기능은 백엔드에서 참가자 정보에 점수를 포함하여 전달하고, 프론트엔드에서 이를 올바르게 표시하는 것을 목표로 합니다.

사용자가 게임을 완료하고 점수를 획득한 후 다시 대기방에 입장했을 때, 자신과 다른 참가자들의 업데이트된 점수가 즉시 반영되어야 하며, 이는 사용자 경험과 데이터 정합성 측면에서 중요합니다.

## Requirements

### Requirement 1: 백엔드에서 참가자 점수 정보 제공

**User Story:** As a 백엔드 시스템, I want 대기방 참가자 목록에 각 참가자의 점수를 포함하여 전달하고 싶다, so that 프론트엔드가 모든 참가자의 점수를 표시할 수 있다

#### Acceptance Criteria

1. WHEN 대기방 입장 API가 호출되면 THEN ParticipantResponse에 totalScore 필드가 포함되어야 한다
2. WHEN 참가자 정보를 생성할 때 THEN User 엔티티의 totalScore 값을 ParticipantResponse에 매핑해야 한다
3. WHEN WebSocket으로 새 참가자 입장 이벤트를 전송할 때 THEN 해당 참가자의 totalScore가 포함되어야 한다
4. WHEN 참가자의 점수가 null이면 THEN 0으로 기본값을 설정해야 한다

### Requirement 2: 프론트엔드에서 참가자 점수 표시

**User Story:** As a 사용자, I want 대기방 화면에서 모든 참가자의 점수를 확인하고 싶다, so that 다른 사람들의 게임 성과를 비교할 수 있다

#### Acceptance Criteria

1. WHEN 대기방 입장 응답을 받으면 THEN 각 참가자의 totalScore를 participant.score에 매핑해야 한다
2. WHEN 새 참가자가 입장하면 THEN 해당 참가자의 totalScore를 participant.score에 매핑해야 한다
3. WHEN 점수 정보가 없으면 THEN 기본값 0을 표시해야 한다
4. WHEN 점수가 표시될 때 THEN 숫자는 포맷 없이 그대로 표시되어야 한다 (예: 1000)

### Requirement 3: WebSocket 이벤트에 점수 포함

**User Story:** As a 시스템, I want WebSocket 이벤트로 참가자 정보를 전송할 때 점수를 포함하고 싶다, so that 실시간으로 참가자 점수가 동기화된다

#### Acceptance Criteria

1. WHEN PARTICIPANT_JOINED 이벤트가 발생하면 THEN 새 참가자의 totalScore가 포함되어야 한다
2. WHEN 대기방 입장 응답이 전송되면 THEN 모든 참가자의 totalScore가 포함되어야 한다
3. WHEN WebSocket 메시지가 전송될 때 THEN totalScore 필드가 누락되지 않아야 한다

### Requirement 4: 데이터 정합성

**User Story:** As a 개발자, I want 참가자 점수가 항상 최신 상태로 유지되기를 원한다, so that 사용자가 정확한 정보를 볼 수 있다

#### Acceptance Criteria

1. WHEN 사용자가 게임을 완료하고 점수를 획득하면 THEN User 엔티티의 totalScore가 업데이트되어야 한다
2. WHEN 업데이트된 사용자가 대기방에 입장하면 THEN 최신 totalScore가 반영되어야 한다
3. WHEN 참가자 목록을 조회할 때 THEN DB의 최신 totalScore 값을 가져와야 한다

### Requirement 5: 에러 처리 및 기본값

**User Story:** As a 시스템, I want 점수 정보가 없거나 오류가 발생해도 대기방이 정상 작동하기를 원한다, so that 사용자 경험이 중단되지 않는다

#### Acceptance Criteria

1. WHEN 참가자의 totalScore가 null이면 THEN 0을 기본값으로 사용해야 한다
2. WHEN User 엔티티를 조회할 수 없으면 THEN 적절한 에러 메시지와 함께 실패해야 한다
3. WHEN 백엔드에서 점수 매핑 중 오류가 발생하면 THEN 로그를 남기고 0을 기본값으로 사용해야 한다
4. WHEN 프론트엔드에서 totalScore 필드가 누락되면 THEN 0을 기본값으로 사용해야 한다
