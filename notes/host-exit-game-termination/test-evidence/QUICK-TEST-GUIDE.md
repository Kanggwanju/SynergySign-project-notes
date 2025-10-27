# Quick Test Guide - Task 5.1

## 🚀 Quick Start (7 minutes)

### 1. Start Servers (2 min)

**Terminal 1 - Backend:**
```bash
cd backend
./gradlew bootRun
```

**Terminal 2 - Frontend:**
```bash
cd frontend
npm run dev
```

### 2. Setup Test (2 min)

**Browser Window 1 (Host):**
1. Open `http://localhost:5173`
2. Login as host
3. Create a game room
4. Start the game

**Browser Window 2 (Participant):**
1. Open `http://localhost:5173` (new window/tab)
2. Login as participant
3. Join the host's room
4. Wait for game to start

**Both Windows:**
- Open DevTools (F12)
- Go to Console tab

### 3. Execute Test (1 min)

**Window 1 (Host):**
- Close the tab/window (or refresh page)

**Window 2 (Participant) - Watch for:**
1. ✅ AlertModal appears
2. ✅ Title: "게임 종료"
3. ✅ Message: "방장이 나가서 게임이 종료되었습니다"
4. ✅ Warning icon (yellow/orange)
5. ✅ NO Toast notification

**Click "확인" button**

### 4. Verify Results (2 min)

**Window 2 Console - Should show:**
```
📥 참가자 퇴장 이벤트 수신: { "roomClosed": true, ... }
🚪 방장 퇴장으로 인한 게임 종료
🚨 방장 퇴장 처리 시작
✅ 녹화 중단 완료
✅ FastAPI 연결 해제 완료
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

**Backend Logs - Should show:**
```
방장 퇴장 감지 - 방 종료 처리 시작. roomId: {id}
✅ 타이머 정리 완료 - roomId: {id}
```

**Window 2 - Should be on:**
- `/main` page
- Room list should NOT show the closed room

---

## ✅ Pass Criteria

Check all that apply:

- [ ] AlertModal appeared automatically
- [ ] Modal showed correct title and message
- [ ] Modal had warning styling
- [ ] NO Toast notification appeared
- [ ] Confirm button worked
- [ ] Navigated to /main page
- [ ] Console showed cleanup logs
- [ ] Backend showed timer cancellation
- [ ] Room removed from list

**If all checked: TEST PASSED ✅**

---

## 📸 Collect Evidence

1. **Screenshot AlertModal** (before clicking confirm)
2. **Save console logs:**
   - Right-click in console
   - "Save as..." → `console-logs.txt`
3. **Copy backend logs:**
   - Find lines with "방장 퇴장" and "타이머"
   - Save to `backend-logs.txt`

---

## 🐛 Common Issues

### AlertModal doesn't appear
- Check console for errors
- Verify roomClosed is true in event
- Check if handleHostExit was called

### Navigation doesn't work
- Check console for navigation errors
- Verify cleanupResourcesAndExit was called
- Check if any cleanup step failed

### Timers still running
- Check backend logs for cleanup call
- Verify timerManager.cleanupRoom was executed

---

## 📝 Report Results

Use this template:

```markdown
## Test 5.1 Results

**Date:** [Today's date]
**Tester:** [Your name]
**Status:** ✅ PASS / ❌ FAIL

### What Worked:
- [List successful items]

### Issues Found:
- [List any issues]

### Evidence:
- Screenshot: [filename]
- Console logs: [filename]
- Backend logs: [filename]
```

Save as: `test-5.1-report.md`

---

## 🎯 Next Steps

After completing this test:

1. Save all evidence files
2. Fill out test report
3. Mark task 5.1 as complete
4. Proceed to task 5.2

---

## 📚 Full Documentation

For detailed instructions, see:
- `test-execution-guide.md` - Complete step-by-step guide
- `automated-test-checklist.md` - Verification commands
- `TEST-EXECUTION-SUMMARY.md` - Implementation status

---

## ⏱️ Time Estimate

- Setup: 2 min
- Execution: 1 min
- Verification: 2 min
- Documentation: 2 min
- **Total: ~7 minutes**

---

Good luck with testing! 🚀
