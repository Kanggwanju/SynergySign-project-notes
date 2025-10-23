

1. 에러 처리 일관성: handleRoomSearchSubmit에서는 세션 체크 실패 시 showAlert를 사용하는데, handleCreateRoomSubmit에서는 setCreateRoomError를 사용하고 있어요. 통일하는 게 좋을 것 같아요.

2. 세션 체크 중복 코드: 세 곳(handleRoomClick, handleRoomSearchSubmit, handleCreateRoomSubmit)에서 동일한 세션 체크 로직이 반복되고 있어요. 별도 함수로 분리하면 유지보수가 편할 거예요.

3. 방 생성 후 목록 갱신: 방 생성 후 대기방으로 이동하는데, 사용자가 뒤로 돌아왔을 때 새로 만든 방이 목록에 보이도록 fetchRoomList를 호출하면 좋을 것 같아요.

4. 로딩 상태 중복 방지: createRoomLoading 상태가 true일 때 모달의 제출 버튼을 비활성화하거나, 중복 클릭을 방지하는 로직이 있으면 좋겠어요.

5. 에러 메시지 초기화 타이밍: 모달을 닫을 때 setCreateRoomError(null)을 하고 있는데, 모달을 다시 열 때도 초기화해주면 이전 에러가 남아있지 않아요.

6. 방 생성 시에도 세션 검증을 추가