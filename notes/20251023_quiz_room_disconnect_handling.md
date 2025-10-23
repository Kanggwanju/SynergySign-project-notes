# 퀴즈 방 연결 해제 및 예외 상황 처리 문서

## 📋 목차

1. [개요](#개요)
2. [현재 구현 상황](#현재-구현-상황)
3. [예외 상황 분류](#예외-상황-분류)
4. [대기방 처리 방안](#대기방-처리-방안)
5. [게임 페이지 처리 방안](#게임-페이지-처리-방안)
6. [구현 우선순위](#구현-우선순위)
7. [상세 구현 가이드](#상세-구현-가이드)

---

## 개요

퀴즈 방 시스템에서 사용자의 비정상적인 연결 해제 상황을 처리하기 위한 종합 문서입니다.

### 목표

- **대기방**: 모든 예외 상황 완벽 처리
- **게임 페이지**: 나가기 버튼만 우선 처리 (추후 확장 용이하게 설계)

### 🔥 중요: 서버 측 자동 처리

**WebSocketEventsListener에서 이미 구현된 기능:**

- WebSocket 연결 끊김 감지 시 자동으로 처리
- **방장 연결 끊김**: DB에서 방 인원 전체 삭제 + 브로드캐스팅
- **참가자 연결 끊김**: DB에서 해당 참가자 삭제 + 브로드캐스팅

**비정상 종료 시 WebSocket 연결은 자동으로 끊어집니다:**

| 상황          | WebSocket 연결 | 서버 감지        |
| ------------- | -------------- | ---------------- |
| 탭 닫기       | ✅ 즉시 끊김   | ✅ 즉시 감지     |
| 브라우저 종료 | ✅ 즉시 끊김   | ✅ 즉시 감지     |
| 새로고침 (F5) | ✅ 즉시 끊김   | ✅ 즉시 감지     |
| 인터넷 끊김   | ✅ 끊김        | ✅ 타임아웃 감지 |
| PC 강제 종료  | ✅ 끊김        | ✅ 타임아웃 감지 |

**동작 원리:**

```
탭 닫기/브라우저 종료/새로고침
→ 브라우저가 모든 네트워크 연결 즉시 종료
→ WebSocket TCP 연결 끊김
→ 서버의 WebSocketEventsListener가 연결 끊김 감지
→ DB 정리 + 브로드캐스팅 자동 실행
```

**따라서 클라이언트는:**

- ✅ 정상 종료 시에만 명시적으로 `leaveRoom` 메시지 전송
- ✅ 비정상 종료는 **WebSocket 연결이 자동으로 끊어지므로** 서버가 자동 처리
- ✅ 클라이언트는 로컬 리소스 정리(Janus, 웹캠)에만 집중

---

## 현재 구현 상황

### ✅ 완료된 기능

- WebSocket 연결 및 방 입장
- Janus WebRTC 연결 (P2P 영상 통신)
- 웹캠 관리 (useWebcam hook)
- 준비 상태 관리
- 참가자 입장/퇴장 이벤트 처리

### ⚠️ 미구현 영역

- 비정상 종료 시 리소스 정리
- 탭 닫기/새로고침 감지
- 네트워크 끊김 감지 및 재연결
- 방장 위임 로직
- 게임 중 연결 해제 처리

---

## 예외 상황 분류

### 1. 정상 종료 (User-Initiated)

| 상황             | 발생 위치           | 우선순위 |
| ---------------- | ------------------- | -------- |
| 나가기 버튼 클릭 | 대기방, 게임 페이지 | 🔴 높음  |
| 뒤로가기 버튼    | 대기방, 게임 페이지 | 🟡 중간  |

### 2. 비정상 종료 (Abnormal Exit)

| 상황                 | 발생 위치           | 우선순위 |
| -------------------- | ------------------- | -------- |
| 브라우저 탭 닫기     | 대기방, 게임 페이지 | 🔴 높음  |
| 브라우저 종료        | 대기방, 게임 페이지 | 🔴 높음  |
| 페이지 새로고침 (F5) | 대기방, 게임 페이지 | 🔴 높음  |

### 3. 네트워크 문제

| 상황                   | 발생 위치           | 우선순위 |
| ---------------------- | ------------------- | -------- |
| WebSocket 연결 끊김    | 대기방, 게임 페이지 | 🔴 높음  |
| Janus WebRTC 연결 끊김 | 대기방, 게임 페이지 | 🟡 중간  |
| 인터넷 연결 끊김       | 대기방, 게임 페이지 | 🔴 높음  |
| 서버 다운              | 대기방, 게임 페이지 | 🟢 낮음  |

### 4. 특수 상황

| 상황                      | 발생 위치           | 우선순위 |
| ------------------------- | ------------------- | -------- |
| 중복 탭 접속              | 대기방 입장 시      | 🔴 높음  |
| URL 강제 접근 (비참여 방) | 대기방 입장 시      | 🔴 높음  |
| 방장 나가기               | 대기방, 게임 페이지 | 🔴 높음  |
| 게임 중 참가자 이탈       | 게임 페이지         | 🟡 중간  |
| 웹캠 권한 취소            | 대기방, 게임 페이지 | 🟢 낮음  |
| 장시간 비활성 (AFK)       | 대기방, 게임 페이지 | 🟢 낮음  |

---

## 대기방 처리 방안

### 🎯 구현 범위

대기방에서는 **모든 예외 상황**을 완벽하게 처리합니다.

### 1. 정상 종료 처리

#### 1.1 나가기 버튼 클릭

```javascript
const handleConfirmExit = async () => {
  try {
    // 1. WebSocket으로 방 나가기 전송 (정상 퇴장 알림)
    // 서버에서 DB 정리 + 다른 참가자에게 브로드캐스팅
    websocketService.leaveRoom(Number(roomId));

    // 2. Janus WebRTC 정리 (클라이언트 리소스)
    cleanupJanus();

    // 3. 웹캠 정리 (클라이언트 리소스)
    stopWebcam();

    // 4. WebSocket 연결 해제
    websocketService.disconnect();

    // 5. 메인 페이지로 이동
    navigate("/main");
  } catch (error) {
    console.error("방 나가기 실패:", error);
    // 에러가 있어도 강제로 메인으로 이동
    // 서버는 WebSocket 연결 끊김을 감지하여 자동 처리
    navigate("/main");
  }
};
```

**💡 동작 흐름:**

1. 클라이언트: `leaveRoom` 메시지 전송
2. 서버: 메시지 수신 → DB 처리 → 브로드캐스팅
3. 클라이언트: 리소스 정리 후 페이지 이동

#### 1.2 뒤로가기 버튼

```javascript
useEffect(() => {
  // 브라우저 뒤로가기 감지
  const handlePopState = (event) => {
    event.preventDefault();
    setShowExitModal(true);
  };

  window.addEventListener("popstate", handlePopState);

  // 뒤로가기 방지 (히스토리 추가)
  window.history.pushState(null, "", window.location.href);

  return () => {
    window.removeEventListener("popstate", handlePopState);
  };
}, []);
```

### 2. 비정상 종료 처리

#### 2.1 탭 닫기 / 브라우저 종료 / 새로고침

**🎯 핵심: 서버가 자동으로 처리하므로 클라이언트는 리소스 정리만!**

```javascript
useEffect(() => {
  const handleBeforeUnload = (event) => {
    // ❌ WebSocket 메시지 전송 불필요!
    // beforeunload는 비동기 작업을 보장하지 않으며,
    // 서버의 WebSocketEventsListener가 연결 끊김을 자동 감지하여 처리

    // ✅ 클라이언트 리소스만 정리
    // 1. Janus 정리
    if (janusRef.current) {
      try {
        janusRef.current.destroy();
      } catch (error) {
        console.error("Janus 정리 실패:", error);
      }
    }

    // 2. 웹캠 정리
    if (stream) {
      stream.getTracks().forEach((track) => track.stop());
    }

    // 3. 브라우저 경고 메시지 (선택사항)
    event.preventDefault();
    event.returnValue = "게임 방에서 나가시겠습니까?";
    return event.returnValue;
  };

  window.addEventListener("beforeunload", handleBeforeUnload);

  return () => {
    window.removeEventListener("beforeunload", handleBeforeUnload);
  };
}, [roomId]);
```

**💡 동작 흐름:**

1. 사용자: 탭 닫기/새로고침
2. 클라이언트: `beforeunload` 이벤트 발생 → 리소스 정리
3. 브라우저: WebSocket 연결 자동 종료
4. 서버: `WebSocketEventsListener`가 연결 끊김 감지
5. 서버: 방장이면 방 전체 삭제, 참가자면 해당 참가자만 삭제
6. 서버: 남은 참가자들에게 브로드캐스팅

**⚠️ 주의사항:**

- `beforeunload`에서 WebSocket 메시지 전송은 **신뢰할 수 없음**
- 서버의 자동 처리에 의존하는 것이 더 안정적
- 클라이언트는 로컬 리소스 정리에만 집중

### 3. 네트워크 문제 처리

#### 3.1 WebSocket 연결 끊김

**🎯 전략: 짧은 시간 재연결 시도 → 실패 시 서버의 자동 처리에 맡김**

```javascript
useEffect(() => {
  const handleWebSocketDisconnect = () => {
    console.error("❌ WebSocket 연결 끊김");

    // 짧은 시간(5초) 내 재연결 시도 (네트워크 일시적 끊김 대응)
    let reconnectAttempts = 0;
    const maxAttempts = 2; // 2회만 시도 (총 4초)

    const reconnectInterval = setInterval(async () => {
      reconnectAttempts++;

      try {
        await websocketService.connect();
        await websocketService.joinRoom(Number(roomId));

        showAlert("연결이 복구되었습니다.", "연결 복구", "success");
        clearInterval(reconnectInterval);
      } catch (error) {
        if (reconnectAttempts >= maxAttempts) {
          clearInterval(reconnectInterval);

          // 재연결 실패 - 서버가 이미 퇴장 처리했을 것
          showAlert(
            "서버와의 연결이 끊어졌습니다. 메인 페이지로 이동합니다.",
            "연결 실패",
            "error"
          );

          setTimeout(() => {
            // 리소스 정리만 수행
            cleanupJanus();
            stopWebcam();
            navigate("/main");
          }, 2000);
        }
      }
    }, 2000);
  };

  websocketService.on("disconnect", handleWebSocketDisconnect);

  return () => {
    websocketService.off("disconnect", handleWebSocketDisconnect);
  };
}, [roomId]);
```

**💡 동작 흐름:**

1. WebSocket 연결 끊김 감지
2. 클라이언트: 2회(4초) 재연결 시도
3. **성공**: 방에 재입장 → 정상 진행
4. **실패**:
    - 서버는 이미 `WebSocketEventsListener`에서 퇴장 처리 완료
    - 클라이언트는 리소스 정리 후 메인으로 이동

**⚠️ 재연결 시 주의사항:**

- 서버에서 이미 퇴장 처리되었을 수 있음
- `joinRoom` 실패 시 방이 없거나 세션이 만료된 상태
- 에러 처리를 통해 적절히 대응 필요

#### 3.2 Janus WebRTC 연결 끊김

```javascript
// Janus 세션 생성 시 에러 핸들러 추가
janusRef.current = new Janus({
  server: JANUS_SERVER,
  success: function () {
    console.log("✅ Janus 서버 연결 성공");
  },
  error: function (error) {
    console.error("❌ Janus 서버 연결 실패:", error);
    showAlert(
      "영상 통화 연결에 실패했습니다. 음성만 사용 가능합니다.",
      "연결 실패",
      "warning"
    );
  },
  destroyed: function () {
    console.log("🔌 Janus 세션 종료");
  },
  // 연결 끊김 감지
  iceState: function (state) {
    if (state === "disconnected" || state === "failed") {
      console.error("❌ Janus ICE 연결 끊김:", state);
      showAlert("영상 연결이 불안정합니다.", "연결 경고", "warning");
    }
  },
});
```

#### 3.3 인터넷 연결 끊김

```javascript
useEffect(() => {
  const handleOnline = () => {
    console.log("✅ 인터넷 연결 복구");
    showAlert("인터넷 연결이 복구되었습니다.", "연결 복구", "success");

    // WebSocket 재연결 시도
    websocketService.connect().then(() => {
      websocketService.joinRoom(Number(roomId));
    });
  };

  const handleOffline = () => {
    console.error("❌ 인터넷 연결 끊김");
    showAlert(
      "인터넷 연결이 끊어졌습니다. 연결을 확인해주세요.",
      "연결 끊김",
      "error"
    );
  };

  window.addEventListener("online", handleOnline);
  window.addEventListener("offline", handleOffline);

  return () => {
    window.removeEventListener("online", handleOnline);
    window.removeEventListener("offline", handleOffline);
  };
}, [roomId]);
```

### 4. 특수 상황 처리

#### 4.1 중복 탭 접속 및 URL 강제 접근 방지

**🎯 백엔드 검증 로직 (이미 구현됨):**

```java
// GameRoomJoinService.java

// 1. 이미 이 방에 참가했는지 확인
boolean alreadyInThisRoom = participantRepository.existsByParticipantAndGameRoom(user, room);
if (alreadyInThisRoom) {
    // 중복 입장 시도 → 현재 방 상태 반환 (에러 아님)
    return JoinRoomResponse.of(room, participantResponses);
}

// 2. 방장이 아닌 경우: 다른 방에 참여 중인지 확인
if (!isHost) {
    boolean alreadyInRoom = participantRepository.existsByParticipantAndGameRoom_StatusIn(
        user, GameRoomStatus.WAITING, GameRoomStatus.IN_PROGRESS
    );
    if (alreadyInRoom) {
        // 다른 방에 참여 중 → PARTICIPANT_ALREADY_IN_ROOM 예외
        throw new BusinessException(ErrorCode.PARTICIPANT_ALREADY_IN_ROOM);
    }
}
```

**💡 예외 케이스 분석:**

| 케이스                                 | 백엔드 처리                                 | 프론트 처리                |
| -------------------------------------- | ------------------------------------------- | -------------------------- |
| **케이스 1**: 같은 방 중복 탭 접근     | ✅ 정상 처리 (방 정보 반환)                 | ✅ 정상 입장               |
| **케이스 2**: 다른 방 참여 중 URL 접근 | ✅ `PARTICIPANT_ALREADY_IN_ROOM` 예외       | ✅ 에러 처리 필요          |
| **케이스 3**: 비참여 방 URL 강제 접근  | ✅ 정상 입장 (새 참가자로 추가)             | ✅ 정상 입장               |
| **케이스 4**: WebSocket 중복 연결 시도 | ✅ `UserSessionRegistry`에서 기존 세션 유지 | ✅ 프론트 세션 체크로 차단 |

**프론트엔드 처리 (이미 구현됨):**

```javascript
// RealTimeQuizSidebar.jsx - 방 카드 클릭 시
const handleRoomClick = async (roomId) => {
  // WebSocket 세션 체크
  try {
    const sessionStatus = await RoomService.checkWsSession();

    if (sessionStatus.active) {
      showAlert(
        "이미 다른 탭에서 게임에 참여 중입니다. 기존 탭을 종료해주세요.",
        "입장 불가",
        "warning"
      );
      return; // 입장 차단
    }

    // 모든 조건 통과 → 입장 허용
    navigate(`/quiz/waiting/${roomId}`);
    onClose();
  } catch (err) {
    console.error("세션 체크 실패:", err);
    showAlert("오류", "입장 확인 중 오류가 발생했습니다.", "error");
  }
};
```

**QuizWaitingRoom 컴포넌트 추가 처리:**

```javascript
// QuizWaitingRoom.jsx - 컴포넌트 마운트 시
useEffect(() => {
  const initWebSocket = async () => {
    try {
      // 1. WebSocket 이벤트 리스너 등록
      websocketService.on("room:join", handleRoomJoin);
      websocketService.on("participant", handleParticipantEvent);
      websocketService.on("error", handleError);

      // 2. WebSocket 연결
      await websocketService.connect();
      console.log("✅ WebSocket 연결 성공!");

      // 3. 방 입장 시도
      setTimeout(() => {
        websocketService.joinRoom(Number(roomId));
        console.log(`🚪 방 ${roomId}에 입장 시도`);
      }, 300);
    } catch (error) {
      console.error("❌ WebSocket 연결 실패:", error);

      // 에러 타입별 처리
      if (error.response?.data?.error === "PARTICIPANT_ALREADY_IN_ROOM") {
        showAlert(
          "이미 다른 방에 참여 중입니다. 기존 방을 먼저 나가주세요.",
          "입장 불가",
          "error"
        );
        setTimeout(() => navigate("/main"), 2000);
      } else {
        showAlert("연결 실패: " + error.message, "오류", "error");
      }
    }
  };

  initWebSocket();
}, [roomId]);

// 에러 핸들러 추가
const handleError = (data) => {
  console.error("📥 에러:", data);

  const errorCode = data.error || data.detail;

  switch (errorCode) {
    case "PARTICIPANT_ALREADY_IN_ROOM":
      showAlert("이미 다른 방에 참여 중입니다.", "입장 불가", "error");
      setTimeout(() => navigate("/main"), 2000);
      break;

    case "ROOM_NOT_FOUND":
      showAlert("방을 찾을 수 없습니다.", "입장 불가", "error");
      setTimeout(() => navigate("/main"), 2000);
      break;

    case "ROOM_ALREADY_STARTED":
      showAlert("이미 게임이 시작된 방입니다.", "입장 불가", "error");
      setTimeout(() => navigate("/main"), 2000);
      break;

    case "ROOM_FULL":
      showAlert("방이 가득 찼습니다.", "입장 불가", "error");
      setTimeout(() => navigate("/main"), 2000);
      break;

    default:
      showAlert(data.detail || "오류가 발생했습니다.", "오류", "error");
  }
};
```

**🔒 보안 및 안정성:**

1. **중복 탭 접속 (같은 방)**

    - 백엔드: 중복 입장 허용 (같은 방이므로 문제없음)
    - 프론트: WebSocket 세션 체크로 차단 (권장)
    - 결과: 안전하게 차단됨

2. **다른 방 참여 중 URL 접근**

    - 백엔드: `PARTICIPANT_ALREADY_IN_ROOM` 예외 발생
    - 프론트: 에러 메시지 표시 후 메인으로 이동
    - 결과: 완벽하게 차단됨

3. **비참여 방 URL 강제 접근**

    - 백엔드: 정상 입장 처리 (새 참가자로 추가)
    - 프론트: 정상 입장
    - 결과: 정상 동작 (문제없음)

4. **WebSocket 중복 연결**
    - 백엔드: `UserSessionRegistry`가 기존 세션 유지
    - 프론트: 세션 체크로 사전 차단
    - 결과: 안전하게 차단됨

#### 4.2 방장 나가기 처리

**🎯 서버 로직:**

- 방장 연결 끊김 → DB에서 방 인원 전체 삭제 → 브로드캐스팅
- 참가자들은 `PARTICIPANT_LEFT` 이벤트에서 `roomClosed: true` 수신

```javascript
// handleParticipantEvent 함수 내부 (이미 구현됨)
case 'PARTICIPANT_LEFT':
  console.log('👋 참가자 퇴장:', eventData.participant);

  // 방장이 나가서 방이 종료된 경우
  if (eventData.roomClosed) {
    showAlert('방장이 나가서 방이 종료되었습니다.', '방 종료', 'info');

    // 리소스 정리
    cleanupJanus();
    stopWebcam();
    websocketService.disconnect();

    setTimeout(() => {
      navigate('/main');
    }, 2000);
    return;
  }

  // 일반 참가자가 나간 경우
  setParticipants(prev =>
    prev.filter(p => p.userId !== eventData.participant.userId)
  );
  setRoomInfo(prev => ({
    ...prev,
    currentParticipants: eventData.currentParticipants
  }));

  // 방장 위임 로직 (서버에서 처리하는 경우)
  // 서버가 방장 위임을 지원한다면 아래 코드 추가
  if (eventData.newHostId) {
    setRoomInfo(prev => ({
      ...prev,
      hostId: eventData.newHostId
    }));

    setParticipants(prev => prev.map(p => ({
      ...p,
      isHost: p.userId === eventData.newHostId
    })));

    if (eventData.newHostId === myUserId) {
      showAlert('당신이 새로운 방장이 되었습니다.', '방장 위임', 'info');
    }
  }
  break;
```

**💡 동작 흐름:**

1. 방장: 연결 끊김 (정상/비정상 모두)
2. 서버: `WebSocketEventsListener`가 감지
3. 서버: DB에서 방 전체 삭제
4. 서버: 남은 참가자들에게 `PARTICIPANT_LEFT` + `roomClosed: true` 브로드캐스팅
5. 참가자들: 알림 표시 → 리소스 정리 → 메인으로 이동

**🔧 방장 위임 기능 추가 시:**

- 서버에서 방장 나가기 시 방을 종료하지 않고 다음 참가자에게 방장 위임
- `newHostId` 필드를 브로드캐스팅에 포함
- 클라이언트는 위 코드로 자동 처리

#### 4.3 웹캠 권한 취소

```javascript
useEffect(() => {
  if (webcamError) {
    showAlert(
      "웹캠 권한이 거부되었습니다. 게임 참여를 위해 웹캠 권한이 필요합니다.",
      "권한 필요",
      "warning"
    );

    // 준비 상태 자동 해제
    if (isReady) {
      websocketService.setReady(Number(roomId), false);
    }
  }
}, [webcamError]);
```

### 5. 리소스 정리 함수

```javascript
// Janus 정리 함수
const cleanupJanus = useCallback(() => {
  console.log("🧹 Janus 리소스 정리 시작");

  // 1. 모든 원격 피드 정리
  Object.values(remoteFeedsRef.current).forEach((feed) => {
    try {
      feed.detach();
    } catch (error) {
      console.error("원격 피드 정리 실패:", error);
    }
  });
  remoteFeedsRef.current = {};

  // 2. 플러그인 핸들 정리
  if (pluginHandleRef.current) {
    try {
      pluginHandleRef.current.detach();
    } catch (error) {
      console.error("플러그인 핸들 정리 실패:", error);
    }
    pluginHandleRef.current = null;
  }

  // 3. Janus 세션 종료
  if (janusRef.current) {
    try {
      janusRef.current.destroy();
    } catch (error) {
      console.error("Janus 세션 종료 실패:", error);
    }
    janusRef.current = null;
  }

  // 4. 상태 초기화
  setRemoteStreams({});
  userIdToFeedIdRef.current = {};
  setIsJanusConnected(false);

  console.log("✅ Janus 리소스 정리 완료");
}, []);

// 컴포넌트 언마운트 시 정리
useEffect(() => {
  return () => {
    console.log("🧹 QuizWaitingRoom 언마운트 - 리소스 정리");

    // ⚠️ 주의: 언마운트는 다양한 상황에서 발생
    // 1. 정상 나가기 (navigate) - leaveRoom 이미 호출됨
    // 2. 게임 시작으로 페이지 전환 - leaveRoom 호출하면 안됨!
    // 3. 비정상 종료 - 서버가 자동 처리

    // 따라서 여기서는 WebSocket 메시지 전송하지 않음
    // 리소스 정리만 수행

    // Janus 정리
    cleanupJanus();

    // 웹캠 정리
    stopWebcam();

    // WebSocket 이벤트 리스너 제거
    websocketService.off("room:join", handleRoomJoin);
    websocketService.off("participant", handleParticipantEvent);
    websocketService.off("error", handleError);
  };
}, [cleanupJanus, stopWebcam]);
```

**💡 중요한 설계 결정:**

- 언마운트 시 `leaveRoom` 호출하지 않음
- 이유: 게임 시작으로 페이지 전환 시에도 언마운트되기 때문
- 정상 나가기는 `handleConfirmExit`에서 명시적으로 처리
- 비정상 종료는 서버의 `WebSocketEventsListener`가 자동 처리

---

## 게임 페이지 처리 방안

### 🎯 구현 범위 (1차)

게임 페이지에서는 **나가기 버튼만** 우선 처리합니다.
나머지 예외 상황은 팀원의 게임 로직 완성 후 2차로 추가합니다.

### 1. 나가기 버튼 처리 (1차 구현)

```javascript
const confirmExit = async () => {
  try {
    // 1. 게임 중 나가기 확인
    const isGameInProgress = gamePhase !== "challenge";

    if (isGameInProgress) {
      // 게임 진행 중이면 추가 경고
      const confirmed = window.confirm(
        "게임이 진행 중입니다. 나가면 패배 처리됩니다. 정말 나가시겠습니까?"
      );

      if (!confirmed) {
        setShowExitModal(false);
        return;
      }
    }

    // 2. WebSocket으로 게임 나가기 전송
    // TODO: 팀원이 게임 나가기 API 구현 후 연동
    // websocketService.leaveGame(Number(roomId));

    // 3. 웹캠 정리
    stopWebcam();

    // 4. WebSocket 연결 유지 (대기방으로 돌아갈 수 있음)
    // 또는 완전히 나가기
    // websocketService.disconnect();

    // 5. 메인 페이지로 이동
    navigate("/main");
  } catch (error) {
    console.error("게임 나가기 실패:", error);
    navigate("/main");
  }
};
```

### 2. 추후 구현 예정 (2차)

팀원의 게임 로직 완성 후 추가할 항목:

```javascript
// TODO: 2차 구현 - 탭 닫기/새로고침 처리
useEffect(() => {
  const handleBeforeUnload = (event) => {
    // 게임 중이면 경고
    if (gamePhase !== "challenge") {
      event.preventDefault();
      event.returnValue = "게임이 진행 중입니다. 나가시겠습니까?";

      // 긴급 퇴장 신호
      // websocketService.leaveGame(Number(roomId), { emergency: true });
    }
  };

  window.addEventListener("beforeunload", handleBeforeUnload);

  return () => {
    window.removeEventListener("beforeunload", handleBeforeUnload);
  };
}, [gamePhase, roomId]);

// TODO: 2차 구현 - WebSocket 연결 끊김 처리
// TODO: 2차 구현 - 다른 참가자 이탈 처리
// TODO: 2차 구현 - 게임 중 방장 나가기 처리
```

---

## 구현 우선순위

### Phase 1: 대기방 완성 (현재)

1. ✅ 나가기 버튼 처리
2. ✅ 탭 닫기/새로고침 감지
3. ✅ WebSocket 연결 끊김 처리
4. ✅ 중복 탭 접속 방지
5. ✅ 방장 나가기 처리
6. ✅ 리소스 정리 함수

### Phase 2: 게임 페이지 기본 (현재)

1. ✅ 나가기 버튼만 처리
2. ⏳ 나머지는 팀원 작업 완료 대기

### Phase 3: 게임 페이지 완성 (추후)

1. ⏳ 탭 닫기/새로고침 처리
2. ⏳ 게임 중 연결 끊김 처리
3. ⏳ 게임 중 참가자 이탈 처리
4. ⏳ 게임 중 방장 나가기 처리

---

## 상세 구현 가이드

### 1. WebSocket 서비스 확장

`websocketService.js`에 추가할 메서드:

```javascript
// 방 나가기 (정상 퇴장 시에만 사용)
leaveRoom(roomId) {
  // 서버에 정상 퇴장 알림
  // 서버는 DB 정리 + 브로드캐스팅 수행
  this.sendMessage(`/app/room/${roomId}/leave`, {
    roomId,
    reason: 'USER_EXIT'
  });
}

// 연결 상태 확인
isConnected() {
  return this.stompClient && this.stompClient.connected;
}

// 재연결
async reconnect() {
  if (this.isConnected()) {
    return;
  }

  await this.disconnect();
  await this.connect();
}
```

**💡 설계 변경:**

- `emergency` 옵션 제거 (불필요)
- 비정상 종료는 서버의 `WebSocketEventsListener`가 자동 처리
- `leaveRoom`은 정상 퇴장(나가기 버튼) 시에만 호출

### 2. 서버 측 처리 현황 및 추가 제안

#### 2.1 ✅ 이미 구현됨: WebSocketEventsListener

```
현재 서버에서 처리 중:
- WebSocket 연결 끊김 자동 감지
- 방장 연결 끊김: DB에서 방 인원 전체 삭제 + 브로드캐스팅
- 참가자 연결 끊김: DB에서 해당 참가자 삭제 + 브로드캐스팅
```

#### 2.2 💡 추가 제안: Heartbeat 메커니즘 (선택사항)

```
목적: 좀비 연결(연결은 유지되지만 응답 없음) 감지

구현 방법:
- 클라이언트: 10초마다 ping 메시지 전송
- 서버: 30초 이상 ping 없으면 연결 끊김으로 간주
- 현재는 WebSocket 자체 타임아웃으로도 충분할 수 있음
```

#### 2.3 💡 추가 제안: 방장 위임 로직 (선택사항)

```
현재: 방장 나가면 방 전체 종료
제안: 방장 나가면 다음 참가자에게 방장 위임

우선순위:
1. 가장 먼저 입장한 참가자
2. 준비 상태인 참가자 우선
3. 레벨/점수가 높은 참가자 우선

브로드캐스팅 메시지에 추가:
{
  eventType: 'PARTICIPANT_LEFT',
  participant: {...},
  roomClosed: false,  // 방 유지
  newHostId: 123,     // 새 방장 ID
  currentParticipants: 3
}
```

### 3. 에러 처리 전략

```javascript
// 에러 타입별 처리
const handleError = (error, context) => {
  const errorType = error.type || "UNKNOWN";

  switch (errorType) {
    case "NETWORK_ERROR":
      // 재연결 시도
      attemptReconnect();
      break;

    case "SESSION_EXPIRED":
      // 로그인 페이지로 이동
      navigate("/");
      break;

    case "ROOM_NOT_FOUND":
      // 메인 페이지로 이동
      navigate("/main");
      break;

    case "PERMISSION_DENIED":
      // 권한 요청
      requestPermission();
      break;

    default:
      // 일반 에러 처리
      showAlert("오류가 발생했습니다.", "오류", "error");
  }
};
```

### 4. 테스트 시나리오

#### 대기방 테스트

```
1. 나가기 버튼 클릭 → 정상 퇴장 확인
2. 브라우저 탭 닫기 → 다른 참가자 화면에서 퇴장 확인
3. F5 새로고침 → 재입장 또는 퇴장 확인
4. 네트워크 끊기 → 재연결 또는 퇴장 확인
5. 방장 나가기 → 새 방장 지정 확인
6. 중복 탭 열기 → 경고 메시지 확인
```

#### 게임 페이지 테스트 (1차)

```
1. 나가기 버튼 클릭 → 정상 퇴장 확인
2. 게임 중 나가기 → 경고 메시지 확인
```

---

## 추가 고려사항

### 1. 사용자 경험 (UX)

- 연결 끊김 시 명확한 피드백 제공
- 재연결 시도 중 로딩 인디케이터 표시
- 에러 메시지는 사용자 친화적으로 작성

### 2. 성능 최적화

- 리소스 정리는 비동기로 처리하되 블로킹 방지
- 메모리 누수 방지 (이벤트 리스너, 타이머 정리)

### 3. 보안

- 클라이언트 측 검증은 보조 수단
- 서버 측에서 모든 상태 변경 검증 필수

### 4. 로깅

- 모든 연결 해제 이벤트 로깅
- 에러 발생 시 컨텍스트 정보 포함

---

## 예외 케이스 종합 분석

### 질문 1: URL 강제 접근 (비참여 방)

**시나리오**: 참여자 A가 참여 중이지 않은 B 방 URL로 직접 접근

| 상황                    | 백엔드 처리                           | 프론트 처리         | 결과           |
| ----------------------- | ------------------------------------- | ------------------- | -------------- |
| A가 어떤 방에도 없음    | ✅ 정상 입장 (새 참가자로 추가)       | ✅ 정상 입장        | ✅ 정상 동작   |
| A가 다른 C 방에 참여 중 | ✅ `PARTICIPANT_ALREADY_IN_ROOM` 예외 | ✅ 에러 처리 + 이동 | ✅ 완벽히 차단 |
| B 방이 존재하지 않음    | ✅ `ROOM_NOT_FOUND` 예외              | ✅ 에러 처리 + 이동 | ✅ 완벽히 차단 |
| B 방이 이미 시작됨      | ✅ `ROOM_ALREADY_STARTED` 예외        | ✅ 에러 처리 + 이동 | ✅ 완벽히 차단 |
| B 방이 가득 참          | ✅ `ROOM_FULL` 예외                   | ✅ 에러 처리 + 이동 | ✅ 완벽히 차단 |

**결론**: ✅ **완벽하게 커버됨** - 백엔드의 `GameRoomJoinService`가 모든 케이스 검증

### 질문 2: 중복 탭 접근 (참여 중인 방)

**시나리오**: 참여자 A가 참여 중인 B 방에 새 탭으로 접근

| 단계 | 상황                       | 백엔드 처리                                              | 프론트 처리                                 | 결과         |
| ---- | -------------------------- | -------------------------------------------------------- | ------------------------------------------- | ------------ |
| 1    | 방 카드 클릭 시            | -                                                        | ✅ `checkWsSession()` 체크 → 차단           | ✅ 사전 차단 |
| 2    | URL 직접 입력 시           | ✅ `alreadyInThisRoom` 체크 → 방 정보 반환 (에러 아님)   | ✅ 정상 입장 (같은 방이므로 문제없음)       | ⚠️ 허용됨    |
| 3    | WebSocket 연결 시도        | ✅ `UserSessionRegistry`가 기존 세션 유지 (새 세션 무시) | ⚠️ 연결 성공 (하지만 기존 세션과 충돌 가능) | ⚠️ 주의 필요 |
| 4    | 기존 탭에서 연결 끊김 감지 | ✅ `WebSocketEventsListener`가 감지 → 퇴장 처리          | ✅ 다른 참가자들에게 퇴장 알림              | ✅ 자동 처리 |

**문제점 분석**:

- 백엔드는 같은 방 중복 입장을 허용 (정상 동작으로 간주)
- 하지만 WebSocket 중복 연결 시 세션 충돌 가능
- 기존 탭과 새 탭 중 어느 것이 활성 세션인지 불명확

**해결 방안**:

```javascript
// QuizWaitingRoom.jsx - 컴포넌트 마운트 시 추가
useEffect(() => {
  const checkDuplicateAccess = async () => {
    try {
      // 1. WebSocket 세션 체크
      const sessionStatus = await RoomService.checkWsSession();

      if (sessionStatus.active) {
        // 이미 다른 탭에서 WebSocket 연결 중
        showAlert(
          "이미 다른 탭에서 이 방에 참여 중입니다. 기존 탭을 사용해주세요.",
          "중복 접속",
          "warning"
        );

        // 2초 후 메인으로 이동
        setTimeout(() => {
          navigate("/main");
        }, 2000);

        return false; // WebSocket 연결 중단
      }

      return true; // WebSocket 연결 진행
    } catch (error) {
      console.error("세션 체크 실패:", error);
      return true; // 에러 시 진행 (서버가 처리)
    }
  };

  // WebSocket 연결 전 체크
  checkDuplicateAccess().then((canProceed) => {
    if (canProceed) {
      initWebSocket();
    }
  });
}, [roomId]);
```

**결론**: ⚠️ **추가 처리 필요** - 프론트에서 WebSocket 연결 전 세션 체크 추가

### 백엔드 개선 제안 (선택사항)

현재 백엔드는 같은 방 중복 입장을 허용하지만, 더 엄격한 검증을 원한다면:

```java
// GameRoomJoinService.java 개선안

// 이미 이 방에 참가했는지 확인
boolean alreadyInThisRoom = participantRepository.existsByParticipantAndGameRoom(user, room);

if (alreadyInThisRoom) {
    // 현재 옵션 1: 중복 입장 허용 (현재 구현)
    return JoinRoomResponse.of(room, participantResponses);

    // 옵션 2: 중복 입장 차단 (더 엄격)
    // WebSocket 세션이 활성화되어 있는지 추가 체크
    if (userSessionRegistry.isActive(userId)) {
        throw new BusinessException(ErrorCode.ALREADY_CONNECTED);
    }
    return JoinRoomResponse.of(room, participantResponses);
}
```

## 결론

이 문서는 퀴즈 방 시스템의 연결 해제 및 예외 상황 처리를 위한 종합 가이드입니다.

**현재 작업:**

- 대기방: 모든 예외 상황 완벽 처리
- 게임 페이지: 나가기 버튼만 처리

**예외 케이스 커버리지:**

| 케이스                        | 백엔드 | 프론트 | 상태           |
| ----------------------------- | ------ | ------ | -------------- |
| URL 강제 접근 (비참여 방)     | ✅     | ✅     | ✅ 완벽히 커버 |
| 다른 방 참여 중 URL 접근      | ✅     | ✅     | ✅ 완벽히 커버 |
| 중복 탭 접근 (같은 방)        | ⚠️     | ⚠️     | ⚠️ 추가 필요   |
| WebSocket 연결 끊김 자동 처리 | ✅     | ✅     | ✅ 완벽히 커버 |
| 방장 나가기 자동 처리         | ✅     | ✅     | ✅ 완벽히 커버 |

**추후 작업:**

1. **즉시**: 중복 탭 접근 시 WebSocket 연결 전 세션 체크 추가
2. **2차**: 게임 페이지 예외 상황 처리 (팀원 작업 완료 후)

이 설계를 따르면 팀원의 작업과 충돌 없이 안정적인 연결 관리 시스템을 구축할 수 있습니다.
