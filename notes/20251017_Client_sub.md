## 클라이언트가 구독해야 하는 채널

### 1. **개인 메시지 - 방 입장 성공 응답**
```javascript
stompClient.subscribe('/user/queue/room', (message) => {
  const response = JSON.parse(message.body);
  // response.data는 JoinRoomResponse
  // - roomId, title, status
  // - participants (참가자 목록)
  // - currentParticipants (현재 참가자 수)
  // - maxParticipants (최대 참가자 수)
  
  console.log('방 입장 성공:', response.data);
  // 방 화면 렌더링
  renderRoomScreen(response.data);
});
```

### 2. **브로드캐스트 - 참가자 입장/퇴장 이벤트**
```javascript
stompClient.subscribe(`/topic/room/${roomId}/participant`, (message) => {
  const response = JSON.parse(message.body);
  const eventData = response.data; // ParticipantEventResponse
  
  switch(eventData.eventType) {
    case 'PARTICIPANT_JOINED':
      // 새 참가자 입장
      console.log('새 참가자:', eventData.participant);
      addParticipantToUI(eventData.participant);
      updateParticipantCount(eventData.currentParticipants);
      break;
      
    case 'PARTICIPANT_LEFT':
      // 참가자 퇴장
      console.log('참가자 퇴장:', eventData.participant);
      removeParticipantFromUI(eventData.participant);
      updateParticipantCount(eventData.currentParticipants);
      break;
      
    case 'ROOM_CLOSED':
      // 방 종료 (방장 퇴장)
      alert('방장이 나가서 방이 종료되었습니다.');
      navigateToRoomList();
      break;
  }
});
```

### 3. **개인 메시지 - 에러 알림**
```javascript
stompClient.subscribe('/user/queue/errors', (message) => {
  const error = JSON.parse(message.body);
  // error는 ErrorResponse
  // - timestamp, status, error, detail, path
  
  console.error('에러 발생:', error);
  alert(`오류: ${error.detail}`);
});
```

### 4. **개인 메시지 - 방 강제 종료 알림** (방장 퇴장 시)
```javascript
stompClient.subscribe('/user/queue/room-closed', (message) => {
  const response = JSON.parse(message.body);
  
  alert(response.message); // "방장이 나가서 방이 종료되었습니다."
  stompClient.deactivate();
  navigateToRoomList();
});
```

## 전체 구독 예제 코드

```javascript
let stompClient = null;
let roomSubscriptions = [];

// WebSocket 연결 및 방 입장
function connectAndJoinRoom(roomId) {
  stompClient = new Client({
    brokerURL: 'ws://localhost:8080/ws',
    reconnectDelay: 5000,
    heartbeatIncoming: 4000,
    heartbeatOutgoing: 4000,
  });

  stompClient.onConnect = () => {
    console.log('WebSocket 연결 성공');
    
    // 1. 개인 메시지 구독 (방 입장 성공 응답)
    const roomSub = stompClient.subscribe('/user/queue/room', (message) => {
      const response = JSON.parse(message.body);
      console.log('방 입장 성공:', response.data);
      renderRoomScreen(response.data);
    });
    roomSubscriptions.push(roomSub);
    
    // 2. 에러 메시지 구독
    const errorSub = stompClient.subscribe('/user/queue/errors', (message) => {
      const error = JSON.parse(message.body);
      console.error('에러:', error);
      alert(`오류: ${error.detail}`);
    });
    roomSubscriptions.push(errorSub);
    
    // 3. 방 강제 종료 알림 구독
    const closedSub = stompClient.subscribe('/user/queue/room-closed', (message) => {
      const response = JSON.parse(message.body);
      alert(response.message);
      leaveRoom();
    });
    roomSubscriptions.push(closedSub);
    
    // 4. 참가자 이벤트 구독 (입장/퇴장)
    const participantSub = stompClient.subscribe(
      `/topic/room/${roomId}/participant`, 
      (message) => {
        const response = JSON.parse(message.body);
        handleParticipantEvent(response.data);
      }
    );
    roomSubscriptions.push(participantSub);
    
    // 5. 방 입장 요청 전송
    stompClient.publish({
      destination: `/app/room/${roomId}/join`,
      body: JSON.stringify({})
    });
  };

  stompClient.activate();
}

// 참가자 이벤트 처리
function handleParticipantEvent(eventData) {
  switch(eventData.eventType) {
    case 'PARTICIPANT_JOINED':
      addParticipantToUI(eventData.participant);
      updateParticipantCount(eventData.currentParticipants);
      showNotification(`${eventData.participant.nickname}님이 입장했습니다.`);
      break;
      
    case 'PARTICIPANT_LEFT':
      removeParticipantFromUI(eventData.participant);
      updateParticipantCount(eventData.currentParticipants);
      showNotification(`${eventData.participant.nickname}님이 퇴장했습니다.`);
      break;
      
    case 'ROOM_CLOSED':
      alert('방장이 나가서 방이 종료되었습니다.');
      leaveRoom();
      break;
  }
}

// 방 나가기
function leaveRoom() {
  // 구독 해제
  roomSubscriptions.forEach(sub => sub.unsubscribe());
  roomSubscriptions = [];
  
  // WebSocket 연결 종료
  if (stompClient) {
    stompClient.deactivate();
  }
  
  // 방 목록으로 이동
  navigateToRoomList();
}
```

## 정리: 구독해야 하는 4개 채널

| 채널 | 타입 | 용도 | 메시지 타입 |
|------|------|------|------------|
| `/user/queue/room` | 개인 | 방 입장 성공 응답 | `JoinRoomResponse` |
| `/user/queue/errors` | 개인 | 에러 알림 | `ErrorResponse` |
| `/user/queue/room-closed` | 개인 | 방 강제 종료 알림 | `ApiResponse<null>` |
| `/topic/room/{roomId}/participant` | 브로드캐스트 | 참가자 입장/퇴장 이벤트 | `ParticipantEventResponse` |

