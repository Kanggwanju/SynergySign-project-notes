# Automated Test Checklist - Task 5.1

## Quick Verification Script

This document provides quick checks you can perform to verify the implementation.

## Frontend Verification Checklist

### 1. Check QuizGamePage Implementation

```bash
# Search for handleHostExit function
grep -n "handleHostExit" frontend/src/pages/QuizGamePage.jsx

# Search for showHostExitModal state
grep -n "showHostExitModal" frontend/src/pages/QuizGamePage.jsx

# Search for cleanupResourcesAndExit function
grep -n "cleanupResourcesAndExit" frontend/src/pages/QuizGamePage.jsx

# Search for AlertModal with host exit message
grep -n "방장이 나가서 게임이 종료되었습니다" frontend/src/pages/QuizGamePage.jsx
```

**Expected Output:**
- ✅ handleHostExit function exists
- ✅ showHostExitModal state exists
- ✅ cleanupResourcesAndExit function exists
- ✅ AlertModal with correct message exists

### 2. Verify Event Handler Logic

```bash
# Check if handleParticipantLeft checks roomClosed flag
grep -A 10 "handleParticipantLeft" frontend/src/pages/QuizGamePage.jsx | grep "roomClosed"
```

**Expected Output:**
- ✅ roomClosed flag is checked in handleParticipantLeft

### 3. Verify Toast Removal

```bash
# Search for old toast notification (should not exist)
grep -n "방장이 나가서 게임이 종료되었습니다" frontend/src/pages/QuizGamePage.jsx | grep "showToast"
```

**Expected Output:**
- ✅ No showToast call with this message (should return empty or only AlertModal)

## Backend Verification Checklist

### 1. Check Timer Cleanup Implementation

```bash
# Search for timer cleanup in GameRoomLeaveService
grep -n "timerManager.cleanupRoom" backend/src/main/java/com/example/backend/service/GameRoomLeaveService.java

# Search for timer cleanup logs
grep -n "타이머 정리 완료" backend/src/main/java/com/example/backend/service/GameRoomLeaveService.java
```

**Expected Output:**
- ✅ timerManager.cleanupRoom is called in handleHostLeave
- ✅ Log message for timer cleanup exists

### 2. Check ParticipantLeftEvent Structure

```bash
# Search for roomClosed field in ParticipantLeftEvent
find backend/src -name "*ParticipantLeftEvent*" -o -name "*ParticipantEventResponse*" | xargs grep -l "roomClosed"
```

**Expected Output:**
- ✅ roomClosed field exists in event class

### 3. Verify Room Closure Logic

```bash
# Search for room.closeRoom() call
grep -n "closeRoom()" backend/src/main/java/com/example/backend/service/GameRoomLeaveService.java
```

**Expected Output:**
- ✅ closeRoom() is called in handleHostLeave

## Runtime Verification

### Start Backend and Check Logs

```bash
cd backend
./gradlew bootRun 2>&1 | tee backend-test.log
```

Then in another terminal, monitor for specific log patterns:

```bash
# Monitor timer-related logs
tail -f backend-test.log | grep -E "(타이머|timer)"

# Monitor room closure logs
tail -f backend-test.log | grep -E "(방장 퇴장|방 종료)"
```

### Start Frontend and Check Console

```bash
cd frontend
npm run dev
```

Open browser console and set up filters:
- Filter for: `방장`, `AlertModal`, `리소스 정리`

## Integration Test Execution

### Minimal Test Flow

1. **Setup (2 minutes)**
   - Start backend: `cd backend && ./gradlew bootRun`
   - Start frontend: `cd frontend && npm run dev`
   - Open 2 browser windows

2. **Create & Join (1 minute)**
   - Window 1: Create room as host
   - Window 2: Join room as participant

3. **Start Game (1 minute)**
   - Window 1: Start game
   - Verify both see game screen

4. **Test Host Exit (2 minutes)**
   - Window 1: Close tab/window
   - Window 2: Watch for AlertModal
   - Window 2: Click confirm button
   - Verify navigation to /main

5. **Verify Cleanup (1 minute)**
   - Check backend logs for timer cancellation
   - Check room list (room should be gone)
   - Check Window 2 console for cleanup logs

**Total Time: ~7 minutes**

## Quick Smoke Test

If you just want to verify the basic flow works:

```bash
# 1. Check files exist
ls -la frontend/src/pages/QuizGamePage.jsx
ls -la backend/src/main/java/com/example/backend/service/GameRoomLeaveService.java

# 2. Check key implementations
grep -c "handleHostExit" frontend/src/pages/QuizGamePage.jsx
# Should return: 3 or more (definition, call, reference)

grep -c "cleanupRoom" backend/src/main/java/com/example/backend/service/GameRoomLeaveService.java
# Should return: 1 or more

# 3. Run frontend build to check for errors
cd frontend
npm run build
# Should complete without errors

# 4. Run backend tests (if any exist)
cd backend
./gradlew test --tests "*GameRoomLeaveService*"
```

## Verification Checklist Summary

Before marking task as complete, verify:

- [ ] Frontend code changes are present
  - [ ] handleHostExit function exists
  - [ ] cleanupResourcesAndExit function exists
  - [ ] showHostExitModal state exists
  - [ ] AlertModal is rendered with correct props
  - [ ] handleParticipantLeft checks roomClosed flag
  - [ ] Toast notification removed

- [ ] Backend code changes are present
  - [ ] timerManager.cleanupRoom() called in handleHostLeave
  - [ ] Timer cleanup logs added
  - [ ] roomClosed field exists in event
  - [ ] Room closure logic verified

- [ ] Manual testing completed
  - [ ] Host exit triggers AlertModal
  - [ ] AlertModal shows correct message
  - [ ] Confirm button navigates to /main
  - [ ] Resources cleaned up (check console)
  - [ ] Backend timers cancelled (check logs)
  - [ ] Room removed from list

- [ ] Edge cases tested
  - [ ] Multiple participants receive event
  - [ ] Overlay click also works
  - [ ] Error during cleanup still navigates
  - [ ] Normal participant exit still works

## Test Evidence Collection

Create a folder for test evidence:

```bash
mkdir -p .kiro/specs/host-exit-game-termination/test-evidence
```

Collect:
1. Screenshots of AlertModal
2. Browser console logs (export as .txt)
3. Backend logs (save relevant excerpts)
4. Screen recording of full test flow (optional)

```bash
# Save browser console
# In browser console: right-click > Save as... > console-logs.txt

# Save backend logs
grep -A 5 -B 5 "방장 퇴장" backend-test.log > test-evidence/backend-host-exit-logs.txt

# Save timer cancellation logs
grep "타이머" backend-test.log > test-evidence/backend-timer-logs.txt
```

## Troubleshooting Common Issues

### Issue: AlertModal doesn't appear

**Check:**
```bash
# Verify handleHostExit is called
grep -A 20 "const handleHostExit" frontend/src/pages/QuizGamePage.jsx

# Verify setShowHostExitModal is called
grep "setShowHostExitModal(true)" frontend/src/pages/QuizGamePage.jsx
```

**Debug in browser:**
```javascript
// In browser console during test
console.log('showHostExitModal:', showHostExitModal);
```

### Issue: Navigation doesn't work

**Check:**
```bash
# Verify navigate is called in finally block
grep -A 30 "cleanupResourcesAndExit" frontend/src/pages/QuizGamePage.jsx | grep -A 5 "finally"
```

### Issue: Timers not cancelled

**Check:**
```bash
# Verify timer cleanup is called
grep -B 5 -A 10 "cleanupRoom" backend/src/main/java/com/example/backend/service/GameRoomLeaveService.java
```

**Check backend logs:**
```bash
grep -i "timer" backend-test.log | tail -20
```

## Performance Verification

### Check Resource Cleanup Time

Add timing logs to measure cleanup performance:

```javascript
// In browser console before test
performance.mark('cleanup-start');

// After cleanup completes
performance.mark('cleanup-end');
performance.measure('cleanup-duration', 'cleanup-start', 'cleanup-end');
console.log(performance.getEntriesByName('cleanup-duration')[0].duration);
```

**Expected:** < 2000ms (2 seconds)

### Check Backend Response Time

```bash
# Monitor backend response time
grep "방장 퇴장 감지" backend-test.log
grep "타이머 정리 완료" backend-test.log
# Time difference should be < 100ms
```

## Final Verification Command

Run this comprehensive check:

```bash
#!/bin/bash
echo "=== Host Exit Feature Verification ==="
echo ""

echo "1. Checking Frontend Implementation..."
if grep -q "handleHostExit" frontend/src/pages/QuizGamePage.jsx; then
    echo "✅ handleHostExit function found"
else
    echo "❌ handleHostExit function NOT found"
fi

if grep -q "showHostExitModal" frontend/src/pages/QuizGamePage.jsx; then
    echo "✅ showHostExitModal state found"
else
    echo "❌ showHostExitModal state NOT found"
fi

if grep -q "cleanupResourcesAndExit" frontend/src/pages/QuizGamePage.jsx; then
    echo "✅ cleanupResourcesAndExit function found"
else
    echo "❌ cleanupResourcesAndExit function NOT found"
fi

echo ""
echo "2. Checking Backend Implementation..."
if grep -q "cleanupRoom" backend/src/main/java/com/example/backend/service/GameRoomLeaveService.java; then
    echo "✅ Timer cleanup found"
else
    echo "❌ Timer cleanup NOT found"
fi

echo ""
echo "3. Build Verification..."
cd frontend
if npm run build > /dev/null 2>&1; then
    echo "✅ Frontend builds successfully"
else
    echo "❌ Frontend build failed"
fi

echo ""
echo "=== Verification Complete ==="
echo "Proceed with manual integration testing"
```

Save this as `verify-implementation.sh` and run:

```bash
chmod +x verify-implementation.sh
./verify-implementation.sh
```
