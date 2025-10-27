# Host Exit Game Termination - Test Execution Guide

## Test 5.1: 방장 퇴장 시나리오 테스트

### Test Objective
Verify that when the host leaves during a game, all participants receive proper notifications, resources are cleaned up, and the room is completely terminated.

### Prerequisites
- Backend server running (Spring Boot)
- Frontend development server running (Vite)
- At least 2 browser windows/tabs (1 for host, 1+ for participants)
- Backend logs accessible for verification

### Test Environment Setup

#### 1. Start Backend Server
```bash
cd backend
./gradlew bootRun
```

#### 2. Start Frontend Server
```bash
cd frontend
npm run dev
```

#### 3. Open Browser Developer Tools
- Open Chrome DevTools (F12) in all browser windows
- Navigate to Console tab to monitor logs
- Keep Network tab open to monitor WebSocket connections

### Test Execution Steps

#### Step 1: Create Game Room (Host)
**Browser Window 1 (Host)**

1. Navigate to `http://localhost:5173` (or your frontend URL)
2. Login as host user
3. Navigate to `/main` page
4. Click "Create Room" or equivalent button
5. Configure room settings
6. Create the room

**Expected Results:**
- ✅ Room created successfully
- ✅ Host is in the waiting room
- ✅ Room appears in room list

**Console Verification:**
```
🎮 방 생성 성공
🔌 WebSocket 연결 성공
```

#### Step 2: Join Game Room (Participant)
**Browser Window 2 (Participant)**

1. Navigate to `http://localhost:5173`
2. Login as participant user
3. Navigate to `/main` page
4. Find the created room in room list
5. Click "Join Room"

**Expected Results:**
- ✅ Participant joins successfully
- ✅ Host sees participant in participant list
- ✅ Participant count increases

**Console Verification (Both Windows):**
```
📥 참가자 입장 이벤트 수신
✅ 참가자 추가 완료
```

#### Step 3: Start Game
**Browser Window 1 (Host)**

1. Click "Start Game" button
2. Wait for game to initialize
3. Verify all participants are connected via WebRTC (Janus)
4. Verify game state is active

**Expected Results:**
- ✅ Game starts successfully
- ✅ All participants see game screen
- ✅ WebRTC connections established
- ✅ Webcams are visible (if enabled)

**Console Verification (All Windows):**
```
🎮 게임 시작
🔌 Janus 연결 성공
📹 WebRTC 연결 성공
```

#### Step 4: Host Exits During Game
**Browser Window 1 (Host)**

1. Close the browser tab/window OR
2. Navigate away from the page OR
3. Refresh the page

**Expected Results:**
- ✅ Host disconnects from game
- ✅ Backend detects host disconnection

**Backend Log Verification:**
Look for these log messages in backend console:
```
방장 퇴장 감지 - 방 종료 처리 시작. roomId: {roomId}
✅ 타이머 정리 완료 - roomId: {roomId}
도전 신청 타이머 취소 (if active)
준비 타이머 취소 (if active)
수어 동작 타이머 취소 (if active)
```

#### Step 5: Verify Participant Receives Event
**Browser Window 2 (Participant)**

**Expected Results:**
- ✅ PARTICIPANT_LEFT event received with `roomClosed: true`
- ✅ Recording stops immediately (if active)
- ✅ FastAPI WebSocket disconnects
- ✅ AlertModal appears automatically

**Console Verification:**
```
📥 참가자 퇴장 이벤트 수신: {
  "eventType": "PARTICIPANT_LEFT",
  "roomClosed": true,
  ...
}
🚪 방장 퇴장으로 인한 게임 종료
🚨 방장 퇴장 처리 시작
✅ 녹화 중단 완료 (if recording)
✅ FastAPI 연결 해제 완료
```

#### Step 6: Verify AlertModal Display
**Browser Window 2 (Participant)**

**Expected Results:**
- ✅ AlertModal is visible
- ✅ Modal type is "warning" (yellow/orange styling)
- ✅ Title displays: "게임 종료"
- ✅ Message displays: "방장이 나가서 게임이 종료되었습니다"
- ✅ Confirm button is visible
- ✅ Modal overlay is present
- ✅ NO Toast notification appears

**Visual Verification:**
- Modal should be centered on screen
- Warning icon should be visible
- Text should be clear and readable
- Overlay should dim the background

#### Step 7: Click Confirm Button
**Browser Window 2 (Participant)**

1. Click the "확인" (Confirm) button on AlertModal

**Expected Results:**
- ✅ Resource cleanup begins immediately
- ✅ Remote feeds detached
- ✅ Publisher plugin leaves Janus room
- ✅ Publisher plugin detached
- ✅ Janus connection destroyed
- ✅ Webcam stopped
- ✅ WebSocket disconnected
- ✅ Navigation to `/main` page occurs

**Console Verification:**
```
🧹 리소스 정리 및 메인 이동 시작
🔌 Remote feeds 정리 중...
🔌 Publisher plugin 정리 중...
✅ Janus 방 떠나기 성공
🔌 Janus 연결 종료 중...
✅ 웹캠 정리 완료
✅ WebSocket 연결 해제 완료
✅ 리소스 정리 완료
🏠 메인 페이지로 이동
```

#### Step 8: Verify Main Page Navigation
**Browser Window 2 (Participant)**

**Expected Results:**
- ✅ User is on `/main` page
- ✅ Game state is reset
- ✅ No game-related UI elements visible
- ✅ User can interact with main page normally

#### Step 9: Verify Room Removal
**Browser Window 2 (Participant)**

1. Check the room list on main page

**Expected Results:**
- ✅ The game room is no longer in the room list
- ✅ Room count decreased by 1

**Backend Verification:**
Query the database or check backend logs:
```sql
SELECT * FROM game_room WHERE id = {roomId};
-- Should return FINISHED status or no record
```

#### Step 10: Verify Backend Timer Cancellation
**Backend Console**

**Expected Log Messages:**
```
방장 퇴장 감지 - 방 종료 처리 시작. roomId: {roomId}
✅ 타이머 정리 완료 - roomId: {roomId}
```

If timers were active, you should see:
```
도전 신청 타이머 취소 - roomId: {roomId}
준비 타이머 취소 - roomId: {roomId}, questionNumber: {n}
수어 동작 타이머 취소 - roomId: {roomId}, questionNumber: {n}
```

**Verification:**
- ✅ No timer-related events fire after host exit
- ✅ No timer callbacks execute
- ✅ Timer threads are cancelled

### Test Variations

#### Variation A: Multiple Participants
Repeat the test with 3+ participants to ensure all receive the event and navigate correctly.

#### Variation B: During Different Game States
Test host exit during:
- Waiting for challenge
- Preparation phase
- Sign language recording phase
- Result display phase

#### Variation C: Overlay Click
Instead of clicking confirm button, click the modal overlay to verify it also triggers cleanup.

### Success Criteria

All of the following must be true for the test to pass:

- [x] PARTICIPANT_LEFT event sent to all participants with `roomClosed: true`
- [x] AlertModal displays with correct title, message, and type
- [x] No Toast notification appears
- [x] Recording stops immediately (if active)
- [x] FastAPI WebSocket disconnects immediately
- [x] Confirm button click triggers resource cleanup
- [x] All WebRTC resources cleaned up (feeds, publisher, Janus)
- [x] Webcam stopped
- [x] WebSocket disconnected
- [x] Navigation to `/main` page successful
- [x] Backend timers cancelled (verified in logs)
- [x] Room removed from room list
- [x] Room status is FINISHED or deleted in database

### Failure Scenarios to Check

#### If AlertModal doesn't appear:
- Check console for errors
- Verify `showHostExitModal` state is set to true
- Verify `handleHostExit` is called
- Check if `roomClosed` flag is true in event

#### If navigation doesn't occur:
- Check console for navigation errors
- Verify `cleanupResourcesAndExit` is called
- Check if any resource cleanup step is blocking
- Verify `navigate('/main')` is in finally block

#### If timers continue running:
- Check backend logs for timer cleanup calls
- Verify `timerManager.cleanupRoom(roomId)` is called
- Check if timer keys are correct format
- Verify timer cancellation logic

#### If room remains in list:
- Check room status in database
- Verify `room.closeRoom()` is called
- Check if room list API filters FINISHED rooms
- Verify room deletion logic

### Rollback Plan

If critical issues are found:
1. Document the issue with screenshots and logs
2. Revert recent changes if necessary
3. Review requirements and design documents
4. Fix the issue and re-test

### Test Report Template

```markdown
## Test Execution Report - Task 5.1

**Date:** [Date]
**Tester:** [Name]
**Environment:** [Dev/Staging/Local]

### Test Results

| Step | Expected Result | Actual Result | Status | Notes |
|------|----------------|---------------|--------|-------|
| 1. Create Room | Room created | [Result] | ✅/❌ | |
| 2. Join Room | Participant joined | [Result] | ✅/❌ | |
| 3. Start Game | Game started | [Result] | ✅/❌ | |
| 4. Host Exit | Host disconnected | [Result] | ✅/❌ | |
| 5. Event Received | PARTICIPANT_LEFT with roomClosed:true | [Result] | ✅/❌ | |
| 6. AlertModal Display | Modal shown correctly | [Result] | ✅/❌ | |
| 7. Confirm Click | Resources cleaned up | [Result] | ✅/❌ | |
| 8. Main Navigation | Navigated to /main | [Result] | ✅/❌ | |
| 9. Room Removal | Room removed from list | [Result] | ✅/❌ | |
| 10. Timer Cancellation | Timers cancelled in logs | [Result] | ✅/❌ | |

### Overall Status: ✅ PASS / ❌ FAIL

### Issues Found:
[List any issues]

### Screenshots:
[Attach relevant screenshots]

### Backend Logs:
[Paste relevant log excerpts]

### Recommendations:
[Any suggestions for improvement]
```

## Additional Verification Commands

### Check Backend Logs
```bash
# If using Docker
docker logs [container-name] | grep "방장 퇴장"

# If running locally
# Check console output or log files
tail -f logs/application.log | grep "타이머"
```

### Check Database State
```sql
-- Check room status
SELECT id, status, current_participants, host_id 
FROM game_room 
WHERE id = [room_id];

-- Check participants
SELECT * FROM game_room_participant 
WHERE game_room_id = [room_id];
```

### Monitor WebSocket Connections
Use browser DevTools Network tab:
1. Filter by "WS" (WebSocket)
2. Check connection status
3. Monitor messages sent/received
4. Verify disconnection occurs

## Notes for Testers

- Keep all browser windows visible during testing
- Take screenshots at each critical step
- Save console logs for debugging
- Note any unexpected behavior
- Test on different browsers if possible (Chrome, Firefox, Safari)
- Test with different network conditions if relevant
- Document any edge cases discovered

## Contact

If you encounter issues during testing:
- Check the design document for expected behavior
- Review the requirements document for acceptance criteria
- Consult the implementation code for debugging
- Report issues with detailed logs and reproduction steps
