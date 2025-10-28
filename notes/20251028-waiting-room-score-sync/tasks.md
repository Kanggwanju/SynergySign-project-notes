# Implementation Plan

## Overview

이 구현 계획은 대기방에서 모든 참가자의 점수를 DB와 동기화하여 표시하는 기능을 단계별로 구현합니다. 백엔드에서 ParticipantResponse DTO에 totalScore 필드를 추가하고, 프론트엔드에서 이를 받아 UI에 표시합니다.

## Tasks

- [x] 1. 백엔드 ParticipantResponse DTO에 totalScore 필드 추가

  - `backend/src/main/java/app/signbell/backend/dto/response/ParticipantResponse.java` 파일 수정
  - `totalScore` 필드를 Long 타입으로 추가
  - `from()` 메서드에서 `participant.getParticipant().getTotalScore()` 매핑
  - null 체크 후 기본값 0L 설정
  - _Requirements: 1.1, 1.2, 1.3, 1.4_

- [ ] 2. 프론트엔드 useWaitingRoom 훅 수정

  - [x] 2.1 setInitialParticipants 함수 수정

    - `frontend/src/hooks/useWaitingRoom.js` 파일 수정
    - `p.totalScore ?? 0`을 사용하여 score 설정
    - myUserId, isWebcamOn 파라미터는 유지하되 myTotalScore 파라미터 제거
    - _Requirements: 2.1, 2.2, 2.3_

  - [x] 2.2 addParticipant 함수 수정

    - `frontend/src/hooks/useWaitingRoom.js` 파일 수정
    - `participant.totalScore ?? 0`을 사용하여 score 설정
    - myUserId, isWebcamOn 파라미터는 유지하되 myTotalScore 파라미터 제거
    - _Requirements: 2.1, 2.2, 2.3_

- [ ] 3. 프론트엔드 QuizWaitingRoom 컴포넌트 수정

  - [x] 3.1 handleRoomJoin 함수 수정


    - `frontend/src/pages/quiz/QuizWaitingRoom.jsx` 파일 수정
    - `setInitialParticipants` 호출 시 user.totalScore 파라미터 제거
    - 기존: `setInitialParticipants(roomData.participants, myUserId, isWebcamOn, user?.totalScore)`
    - 수정: `setInitialParticipants(roomData.participants, myUserId, isWebcamOn)`
    - _Requirements: 2.1, 2.2_

  - [x] 3.2 handleParticipantEvent 함수 수정





    - `frontend/src/pages/quiz/QuizWaitingRoom.jsx` 파일 수정
    - `addParticipant` 호출 시 user.totalScore 파라미터 제거
    - 기존: `addParticipant(newUser, myUserId, isWebcamOn, user?.totalScore)`
    - 수정: `addParticipant(newUser, myUserId, isWebcamOn)`
    - _Requirements: 2.1, 2.2_

- [ ]\* 4. 백엔드 코드 검증 및 테스트

  - [ ]\* 4.1 ParticipantResponse DTO 단위 테스트

    - totalScore가 null일 때 0L로 설정되는지 확인
    - totalScore가 정상 값일 때 올바르게 매핑되는지 확인
    - _Requirements: 1.4, 5.1, 5.3_

  - [ ]\* 4.2 GameRoomJoinService 통합 테스트
    - 대기방 입장 시 모든 참가자의 totalScore가 응답에 포함되는지 확인
    - N+1 쿼리가 발생하지 않는지 확인 (쿼리 로그 확인)
    - _Requirements: 1.1, 1.2, 4.3_

- [ ]\* 5. 프론트엔드 코드 검증 및 테스트

  - [ ]\* 5.1 useWaitingRoom 훅 단위 테스트

    - setInitialParticipants에서 totalScore가 올바르게 매핑되는지 확인
    - addParticipant에서 totalScore가 올바르게 매핑되는지 확인
    - totalScore가 undefined일 때 0으로 fallback 되는지 확인
    - _Requirements: 2.1, 2.2, 2.3, 5.4_

  - [ ]\* 5.2 QuizWaitingRoom 컴포넌트 통합 테스트
    - 대기방 입장 시 모든 참가자의 점수가 표시되는지 확인
    - 새 참가자 입장 시 점수가 표시되는지 확인
    - _Requirements: 2.1, 2.2, 2.4_

- [ ]\* 6. 엔드투엔드 테스트

  - [ ]\* 6.1 신규 사용자 대기방 입장 테스트

    - totalScore가 0인 사용자가 입장했을 때 "0점" 표시 확인
    - _Requirements: 2.3, 5.1_

  - [ ]\* 6.2 게임 완료 후 대기방 복귀 테스트

    - 게임 완료 후 점수 획득
    - 대기방 복귀 시 업데이트된 점수 표시 확인
    - _Requirements: 4.1, 4.2, 4.3_

  - [ ]\* 6.3 여러 참가자 점수 표시 테스트

    - 서로 다른 점수를 가진 여러 참가자가 입장
    - 각 참가자의 점수가 올바르게 표시되는지 확인
    - _Requirements: 2.1, 2.2, 2.4_

  - [ ]\* 6.4 WebSocket 이벤트 테스트
    - 새 참가자 입장 시 PARTICIPANT_JOINED 이벤트에 totalScore 포함 확인
    - 기존 참가자들에게 새 참가자의 점수가 표시되는지 확인
    - _Requirements: 3.1, 3.2, 3.3_

## Notes

- 백엔드 변경은 DTO 수정만으로 완료되며, 서비스 로직 변경 불필요
- 프론트엔드 변경은 최소한으로 유지 (파라미터 제거 및 필드 매핑 변경)
- N+1 문제는 이미 해결되어 있음 (JOIN FETCH 적용)
- 게임 완료 후 점수 업데이트는 이미 구현되어 있음 (QuizService.endGame)
- 테스트 작업은 선택적이며, 수동 테스트로 대체 가능
