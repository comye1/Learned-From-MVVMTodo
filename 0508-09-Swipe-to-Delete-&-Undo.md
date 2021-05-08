# #9 Swipte to Delete & Undo
## 요약
- 리사이클러뷰 아이템 swipe시 delete action
- delete 후 snackbar 노출 -> undo action
- Channel 

## tasks / TasksFragment.kt
- binding.apply 블록에서 ItemTouchHelper 생성(callback 구현) 및 recyclerView에 부착
  - ItemTouchHelper.SimpleCallback에서 onMove와 onSwiped를 override
  - onSwiped에서 현재 task를 얻어 **viewModel의 onTaskSwiped를 호출**
- public ItemTouchHelper(@NonNull ItemTouchHelper.Callback callback)
> Creates an ItemTouchHelper that will work with the given Callback.
You can **attach ItemTouchHelper to a RecyclerView via attachToRecyclerView(RecyclerView)**. Upon attaching, it will add an item decoration, an onItemTouchListener and a Child attach / detach listener to the RecyclerView.
Params:
**callback** – The Callback which controls the behavior of this touch helper.
  - ItemTouchHelper.SimpleCallback 
  > Creates a Callback for the given **drag and swipe** allowance. 
  > These values serve as defaults and if you want to customize behavior per ViewHolder, 
  > you can override getSwipeDirs(RecyclerView, RecyclerView.ViewHolder) 
  > and / or getDragDirs(RecyclerView, RecyclerView.ViewHolder).
```
    ItemTouchHelper(object : ItemTouchHelper.SimpleCallback(0,
    ItemTouchHelper.LEFT or ItemTouchHelper.RIGHT) {
        override fun onMove(
            recyclerView: RecyclerView,
            viewHolder: RecyclerView.ViewHolder,
            target: RecyclerView.ViewHolder
        ): Boolean {
            return false
        }

        override fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int) {
            val task = taskAdapter.currentList[viewHolder.adapterPosition]
            viewModel.onTaskSwiped(task)
        }
    }).attachToRecyclerView(recyclerViewTasks)
```

## tasks / TasksViewModel.kt
- onTaskSwiped, onUndoDeleteClick
```
  fun onTaskSwiped(task: Task) = viewModelScope.launch {
      taskDao.delete(task)
  }
  
  fun onUndoDeleteClick(task: Task) = viewModelScope.launch {
      taskDao.insert(task)
  }
```

- Snackbar 노출 및 이벤트 처리
  - TasksEvent를 상속한 ShowUndoDeleteTaskMessage 클래스 
  - Channel<T> 
  - receiveAsFlow()
    - Represents the given receive channel as a hot flow and **receives from the channel in fan-out fashion every time this flow is collected**. 
    One element will be emitted to one collector only.
```
    private val tasksEventChannel = Channel<TasksEvent>()
    val tasksEvent = tasksEventChannel.receiveAsFlow()
    
    fun onTaskSwiped(task: Task) = viewModelScope.launch {
        taskDao.delete(task)
        tasksEventChannel.send(TasksEvent.ShowUndoDeleteTaskMessage(task)) //swipe된 task의 TasksEvent를 send
    }
    
    sealed class TasksEvent {
      data class ShowUndoDeleteTaskMessage(val task: Task) : TasksEvent()
    }
```

## tasks / TasksFragment.kt
```
  viewLifecycleOwner.lifecycleScope.launchWhenStarted {
      viewModel.tasksEvent.collect { event ->
          when(event) {
              is TasksViewModel.TasksEvent.ShowUndoDeleteTaskMessage -> {
                  Snackbar.make(requireView(), "Task deleted", Snackbar.LENGTH_LONG)
                      .setAction("UNDO") {
                          viewModel.onUndoDeleteClick(event.task)
                      }.show()
              }
          }
      }
  }
```
- launchedWhenStarted 
> Launches and runs the given block when the Lifecycle controlling this LifecycleCoroutineScope is at least in Lifecycle.State.STARTED state.
The returned Job will be cancelled when the Lifecycle is destroyed. 
- viewModel의 tasksEvent를 collect - **collects the given flow with a provided action**
- collect된 tasksEvent가 ShowUndoDeleteTaskMessage일 때 Snackbar 띄움 
  - setAction 으로 UNDO 구현, viewModel의 onUndoDeleteClick 호출 
