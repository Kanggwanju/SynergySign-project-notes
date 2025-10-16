# 트러블 슈팅: WebSocket에서 예외가 클라이언트에 전달되지 않는 문제 (20251015)

## 📌 문제 상황 (Problem)

### 발생한 문제
WebSocket을 통해 게임 방 입장 기능을 구현하던 중, 비즈니스 예외가 발생했을 때 클라이언트가 에러 메시지를 받지 못하는 문제가 발생했습니다.

### 재현 방법
1. 이미 가득 찬 게임 방(4명)에 입장 시도
2. 서버 로그에는 `ROOM_FULL` 예외가 정상적으로 발생
3. **문제**: 클라이언트는 아무런 응답을 받지 못하고 대기 상태 유지

### 기대 동작 vs 실제 동작

| 구분 | 기대 동작 | 실제 동작 |
|------|----------|----------|
| 서버 | 예외 발생 → 에러 응답 전송 | 예외 발생 → 로그만 출력 |
| 클라이언트 | 에러 메시지 수신 → 알림 표시 | ❌ 아무것도 받지 못함 |

---

## 🔍 원인 분석 (Root Cause)

### 1. 초기 구현 (잘못된 방식)

REST API의 예외 처리 방식을 그대로 WebSocket에 적용했습니다.
```java
@Controller
public class GameRoomJoinController {
    
    @MessageMapping("/room/{gameRoomId}/join")
    public void joinRoom(...) {
        try {
            Long userId = Long.valueOf(subject);
            gameRoomJoinService.joinRoom(userId, gameRoomId);
            
        } catch (BusinessException e) {
            // ❌ 잘못된 방식: REST API처럼 그냥 throw
            throw e;  // 클라이언트에게 전달되지 않음!
        }
    }
}
```

**문제점:**
- REST API에서는 `@ControllerAdvice`가 예외를 자동으로 잡아서 HTTP 응답으로 변환
- **WebSocket에서는 `@ControllerAdvice`가 작동하지 않음**
- 예외가 던져져도 클라이언트는 아무것도 받지 못함

---

### 2. REST API vs WebSocket 예외 처리 차이

#### REST API의 예외 처리 흐름
```
[REST API 예외 처리 - 자동화됨 ✅]

클라이언트 요청
    ↓
Controller (예외 발생)
    ↓
throw new BusinessException(ErrorCode.ROOM_FULL)
    ↓
@ControllerAdvice (자동으로 catch)
    ↓
ErrorResponse 자동 생성
    ↓
HTTP 응답 (status: 400, body: ErrorResponse)
    ↓
클라이언트 수신 ✅
```

**예시 코드:**
```java
@RestController
@RequestMapping("/api/quiz/rooms")
public class CreateRoomController {
    
    @PostMapping
    public ResponseEntity<ApiResponse<CreateRoomResponse>> createRoom(...) {
        try {
            // 비즈니스 로직
            return ResponseEntity.ok(response);
            
        } catch (NumberFormatException e) {
            // ✅ 그냥 던지기만 하면 됨!
            throw new BusinessException(ErrorCode.UNAUTHORIZED);
        }
    }
}

// 자동으로 처리하는 @ControllerAdvice
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        ErrorResponse error = ErrorResponse.builder()
            .status(e.getErrorCode().getStatus())
            .error(e.getErrorCode().getCode())
            .detail(e.getMessage())
            .build();
        
        return ResponseEntity.status(e.getErrorCode().getStatus()).body(error);
    }
}
```

---

#### WebSocket의 예외 처리 흐름
```
[WebSocket 예외 처리 - 수동 처리 필요]

클라이언트 메시지
    ↓
@MessageMapping Handler (예외 발생)
    ↓
throw new BusinessException(ErrorCode.ROOM_FULL)
    ↓
❌ @ControllerAdvice가 작동하지 않음!
    ↓
예외는 서버 로그에만 남음
    ↓
클라이언트는 아무것도 받지 못함 ❌
```

**왜 @ControllerAdvice가 작동하지 않을까?**

1. **프로토콜 차이**
    - REST API: HTTP 요청-응답 (1:1 매칭)
    - WebSocket: 양방향 메시지 스트림 (N:M 관계)

2. **응답 대상의 불명확성**
    - REST: 요청한 클라이언트에게만 응답 (명확함)
    - WebSocket: 누구에게 보낼지 불명확
        - 요청자 개인? (`/user/queue/errors`)
        - 방 전체? (`/topic/room/{id}/errors`)
        - 특정 그룹? (`/topic/...`)

3. **Spring의 설계 철학**
    - `@ControllerAdvice`는 HTTP 요청-응답 모델에 최적화
    - WebSocket은 개발자가 명시적으로 메시지 라우팅을 제어해야 함

---

## ✅ 해결 방법 (Solution)

### 최종 구현 (올바른 방식)

WebSocket에서는 예외를 **직접 catch하고, 직접 메시지를 전송**해야 합니다.
```java
@Controller
@RequiredArgsConstructor
@Slf4j
public class GameRoomJoinController {

    private final GameRoomJoinService gameRoomJoinService;
    private final SimpMessagingTemplate messagingTemplate;

    @MessageMapping("/room/{gameRoomId}/join")
    public void joinRoom(
            @DestinationVariable Long gameRoomId,
            @Payload JoinRoomRequest request,
            @AuthenticationPrincipal String subject
    ) {
        try {
            Long userId = Long.valueOf(subject);
            log.info("Join room request - userId: {}, gameRoomId: {}", userId, gameRoomId);

            // 1. 방 입장 처리
            JoinRoomResponse response = gameRoomJoinService.joinRoom(userId, gameRoomId);

            // 2. 성공 응답 전송
            messagingTemplate.convertAndSendToUser(
                    subject,
                    "/queue/room",
                    ApiResponse.success("방에 입장했습니다.", response)
            );

            // 3. 브로드캐스트
            // ...

        } catch (BusinessException e) {
            // ✅ 올바른 방식: 직접 catch하고 직접 전송
            log.warn("Business exception during join room - errorCode: {}, message: {}",
                    e.getErrorCode().getCode(), e.getMessage());
            sendError(subject, e.getErrorCode());

        } catch (NumberFormatException e) {
            log.error("Invalid user ID format - subject: {}", subject);
            sendError(subject, ErrorCode.INVALID_INPUT);

        } catch (Exception e) {
            log.error("Unexpected error during join room", e);
            sendError(subject, ErrorCode.INTERNAL_SERVER_ERROR);
        }
    }

    /**
     * 에러 메시지를 특정 사용자에게 전송하는 메서드
     */
    private void sendError(String userId, ErrorCode errorCode) {
        ErrorResponse error = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(errorCode.getStatus())
                .error(errorCode.getCode())
                .detail(errorCode.getMessage())
                .path("/app/room/join")
                .build();

        // ✅ 핵심: messagingTemplate로 직접 전송
        messagingTemplate.convertAndSendToUser(
                userId,
                "/queue/errors",
                error
        );

        log.warn("Error sent to user {} - code: {}, message: {}",
                userId, errorCode.getCode(), errorCode.getMessage());
    }
}
```

---

### 주요 변경 사항

#### Before (잘못된 방식)
```java
catch (BusinessException e) {
    throw e;  // ❌ 클라이언트에게 전달되지 않음
}
```

#### After (올바른 방식)
```java
catch (BusinessException e) {
    sendError(subject, e.getErrorCode());  // ✅ 직접 전송
}

private void sendError(String userId, ErrorCode errorCode) {
    ErrorResponse error = ErrorResponse.builder()
        .timestamp(LocalDateTime.now())
        .status(errorCode.getStatus())
        .error(errorCode.getCode())
        .detail(errorCode.getMessage())
        .path("/app/room/join")
        .build();
    
    messagingTemplate.convertAndSendToUser(
        userId,
        "/queue/errors",
        error
    );
}
```

---

### 클라이언트 코드 (에러 수신)
```javascript
// 에러 메시지 구독
client.subscribe('/user/queue/errors', (message) => {
  const error = JSON.parse(message.body);
  
  console.error('에러 발생:', error);
  
  // 사용자에게 알림 표시
  alert(`[${error.error}] ${error.detail}`);
  
  // 에러 코드별 처리
  switch (error.error) {
    case 'ROOM_FULL':
      navigate('/quiz/rooms'); // 방 목록으로 이동
      break;
    case 'ROOM_ALREADY_STARTED':
      alert('이미 시작된 방입니다.');
      break;
    case 'PARTICIPANT_ALREADY_IN_ROOM':
      alert('이미 다른 방에 참여 중입니다.');
      break;
    default:
      console.error('알 수 없는 에러:', error);
  }
});
```

---

## 핵심 정리 (Key Takeaways)

### 1. REST API vs WebSocket 예외 처리 비교표

| 항목 | REST API | WebSocket |
|------|----------|-----------|
| **프로토콜** | HTTP (요청-응답) | WebSocket (양방향 메시지) |
| **예외 처리** | `@ControllerAdvice` 자동 처리 | **수동 처리 필요** |
| **에러 전송** | HTTP 응답으로 자동 반환 | **명시적으로 전송** |
| **코드 방식** | `throw new Exception()` | **`catch` + `messagingTemplate.send()`** |
| **응답 대상** | 요청자에게 자동 전달 | **개발자가 명시 (개인/그룹/전체)** |

### 2. WebSocket 예외 처리 체크리스트

- [ ] **모든 예외를 try-catch로 잡기**
    - `BusinessException`
    - `NumberFormatException`
    - `Exception` (예상치 못한 에러)

- [ ] **에러 응답 객체 직접 생성**
    - `ErrorResponse.builder()` 사용
    - `timestamp`, `status`, `error`, `detail`, `path` 포함

- [ ] **messagingTemplate로 직접 전송**
    - 개인: `convertAndSendToUser(userId, "/queue/errors", error)`
    - 그룹: `convertAndSend("/topic/...", error)`

- [ ] **적절한 로그 남기기**
    - `log.warn()` - 비즈니스 예외
    - `log.error()` - 예상치 못한 에러

### 3. 주의사항

**WebSocket에서 절대 하지 말아야 할 것:**
```java
// ❌ 이렇게 하면 클라이언트는 아무것도 받지 못함!
@MessageMapping("/room/{id}/join")
public void joinRoom(...) {
    throw new BusinessException(ErrorCode.ROOM_FULL);
}
```

✅ **WebSocket에서 반드시 해야 할 것:**
```java
// ✅ catch + 직접 전송
@MessageMapping("/room/{id}/join")
public void joinRoom(...) {
    try {
        // 비즈니스 로직
    } catch (BusinessException e) {
        sendError(subject, e.getErrorCode());
    }
}
```

---

## 📚 참고 자료 (References)

### 공식 문서
- [Spring WebSocket Documentation](https://docs.spring.io/spring-framework/reference/web/websocket.html)
- [STOMP Protocol Specification](https://stomp.github.io/stomp-specification-1.2.html)
- [Spring Messaging Exception Handling](https://docs.spring.io/spring-framework/reference/web/websocket/stomp/handle-exceptions.html)

### 관련 이슈
- [Stack Overflow: @ControllerAdvice not working with WebSocket](https://stackoverflow.com/questions/...)
- [GitHub Issue: Exception handling in @MessageMapping](https://github.com/spring-projects/spring-framework/issues/...)


---

## 💬 결론

WebSocket은 REST API와 다른 통신 모델을 가지고 있기 때문에, **예외 처리도 다르게 접근**해야 합니다.

**핵심은 3가지:**
1. `@ControllerAdvice`는 WebSocket에서 작동하지 않음
2. 모든 예외를 `try-catch`로 직접 처리
3. `SimpMessagingTemplate`로 에러 메시지를 명시적으로 전송

이러한 차이를 이해하고 적절히 처리하면, 클라이언트가 항상 정확한 에러 메시지를 받을 수 있습니다.
