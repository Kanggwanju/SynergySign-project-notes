# Test Evidence Directory

This directory contains test documentation and evidence for Task 5.1: 방장 퇴장 시나리오 테스트

## 📁 Directory Contents

### Test Documentation (Created)
- ✅ **QUICK-TEST-GUIDE.md** - Quick reference for executing the test (7 minutes)
- ✅ **test-execution-guide.md** (parent directory) - Comprehensive step-by-step guide
- ✅ **automated-test-checklist.md** (parent directory) - Verification commands
- ✅ **TEST-EXECUTION-SUMMARY.md** (parent directory) - Implementation status

### Test Evidence (To Be Collected)
When you execute the test, save these files here:

- [ ] **alertmodal-screenshot.png** - Screenshot of the AlertModal
- [ ] **console-logs.txt** - Browser console logs from participant window
- [ ] **backend-logs.txt** - Backend log excerpts showing timer cancellation
- [ ] **test-5.1-report.md** - Completed test report
- [ ] **screen-recording.mp4** (optional) - Video of full test flow

## 🚀 How to Execute Test

1. **Read QUICK-TEST-GUIDE.md** for fastest execution (7 minutes)
2. **Or read test-execution-guide.md** for detailed instructions
3. **Execute the test** following the steps
4. **Collect evidence** and save in this directory
5. **Fill out test report** using the template

## ✅ Implementation Status

### Code Verification: COMPLETE ✅

**Frontend Implementation:**
- File: `frontend/src/pages/quiz/QuizGamePage.jsx`
- handleHostExit: ✅ Line 342
- cleanupResourcesAndExit: ✅ Implemented
- showHostExitModal state: ✅ Implemented
- AlertModal rendering: ✅ Implemented
- roomClosed flag check: ✅ Lines 483, 496

**Backend Implementation:**
- File: `backend/src/main/java/app/signbell/backend/service/GameRoomLeaveService.java`
- Timer cleanup: ✅ Line 154 (`timerManager.cleanupRoom(roomId)`)
- Cleanup logging: ✅ Line 156
- Error handling: ✅ Try-catch block

### Test Status: READY FOR EXECUTION 🔄

All code is in place. Manual integration test is ready to be executed.

## 📋 Test Checklist

Use this checklist during testing:

### Before Test
- [ ] Backend server running
- [ ] Frontend server running
- [ ] 2 browser windows open
- [ ] DevTools open in both windows
- [ ] Backend logs visible

### During Test
- [ ] Host creates and starts game
- [ ] Participant joins game
- [ ] Host exits (close tab/window)
- [ ] AlertModal appears in participant window
- [ ] Modal shows correct content
- [ ] No Toast notification
- [ ] Click confirm button
- [ ] Navigate to /main

### After Test
- [ ] Console logs show cleanup
- [ ] Backend logs show timer cancellation
- [ ] Room removed from list
- [ ] Screenshot captured
- [ ] Logs saved
- [ ] Report completed

## 📊 Expected Results

### Frontend Console
```
📥 참가자 퇴장 이벤트 수신: { "roomClosed": true, ... }
🚪 방장 퇴장으로 인한 게임 종료
🚨 방장 퇴장 처리 시작
✅ 녹화 중단 완료
✅ FastAPI 연결 해제 완료
🧹 리소스 정리 및 메인 이동 시작
[... cleanup logs ...]
✅ 리소스 정리 완료
🏠 메인 페이지로 이동
```

### Backend Logs
```
방장 퇴장 감지 - 방 종료 처리 시작. roomId: {id}
✅ 타이머 정리 완료 - roomId: {id}
```

### Visual Results
- AlertModal with warning styling
- Title: "게임 종료"
- Message: "방장이 나가서 게임이 종료되었습니다"
- Navigation to /main page
- Room not in room list

## 🎯 Success Criteria

Test passes if ALL of these are true:

1. ✅ PARTICIPANT_LEFT event sent with roomClosed=true
2. ✅ AlertModal displays automatically
3. ✅ Modal shows correct title and message
4. ✅ Modal has warning styling
5. ✅ NO Toast notification appears
6. ✅ Confirm button triggers cleanup
7. ✅ All resources cleaned up (console logs)
8. ✅ Navigate to /main successfully
9. ✅ Backend logs show timer cancellation
10. ✅ Room removed from room list

## 🐛 Troubleshooting

### Issue: AlertModal doesn't appear
**Solution:** Check console for errors, verify roomClosed flag is true

### Issue: Navigation doesn't work
**Solution:** Check if cleanupResourcesAndExit is called, verify finally block

### Issue: Timers not cancelled
**Solution:** Check backend logs for cleanupRoom call

See **automated-test-checklist.md** for detailed troubleshooting.

## 📝 Test Report Template

```markdown
# Test Report - Task 5.1

**Date:** [Date]
**Tester:** [Name]
**Environment:** Local Development
**Status:** ✅ PASS / ❌ FAIL

## Test Execution

### Setup
- Backend: ✅ Running
- Frontend: ✅ Running
- Browsers: ✅ 2 windows ready

### Test Steps
1. Create room: ✅/❌
2. Join room: ✅/❌
3. Start game: ✅/❌
4. Host exit: ✅/❌
5. AlertModal display: ✅/❌
6. Confirm click: ✅/❌
7. Navigation: ✅/❌
8. Cleanup: ✅/❌
9. Timer cancellation: ✅/❌
10. Room removal: ✅/❌

## Results

### What Worked
- [List successful items]

### Issues Found
- [List any issues]

### Evidence Files
- Screenshot: alertmodal-screenshot.png
- Console logs: console-logs.txt
- Backend logs: backend-logs.txt

## Conclusion

[Overall assessment]

## Recommendations

[Any suggestions for improvement]
```

## 📞 Support

If you need help:
1. Check QUICK-TEST-GUIDE.md for quick reference
2. Check test-execution-guide.md for detailed steps
3. Check automated-test-checklist.md for troubleshooting
4. Review design.md and requirements.md for expected behavior

## ⏭️ Next Steps

After completing Task 5.1:
1. Save all evidence in this directory
2. Complete test report
3. Mark task as complete
4. Proceed to Task 5.2: AlertModal 오버레이 클릭 테스트

---

**Task 5.1 Status: READY FOR MANUAL EXECUTION** 🚀

All documentation is complete. Execute the test and collect evidence!
