# #11 Add / Edit Logic
## 요약
- AddEditTaskFragment에서 발생하는 1. Add 동작과 2. Edit 동작 처리하기
  - AddEditTaskViewModel
    - invalid input 체크
    - database 처리 (update or insert)
    - Event 전송 
  - AddEditTaskFragment
    - 뷰에서 발생하는 이벤트 처리
    - Event Flow를 collect하여 setFragmentResult-popBackStack (Add, Edit 중 어떤 동작을 했는지 알려주기 위함) 또는 Snackbar
- AddEditTaskFragment가 TasksFragment로 돌아갈때
  - TasksFragment
    - setFragmentResultListener - viewModel의 onAddEditResult
    - Event Flow를 collect하여 Snackbar 띄움
  - TasksViewModel
    - onAddEditResult - result에 따라 Event 전송


## addedittask / AddEditTaskViewModel
- fab 클릭시 호출되는 onSaveClick 
  - taskName이 blank
  - task가 null이면 add task, 아니면 edit task
  ```
  fun onSaveClick() {
    if (taskName.isBlank()) { // show invalid input message
        showInvalidInputMessage("Name cannot be empty") // -> Snackbar
        return
    }
    if (task != null) {
        // update existing task
        val updateTask = task.copy(name = taskName, important = taskImportance)
        updateTask(updateTask)
    } else {
        // add new task
        val newTask = Task(name = taskName, important = taskImportance)
        createTask(newTask)
    }
  }
  ```
- Event를 전송할 Channel과 Event Flow
  ```
  private val addEditTaskEventChannel = Channel<AddEditTaskEvent>()
  val addEditTaskEvent = addEditTaskEventChannel.receiveAsFlow()
  ```
- showInvalidInputMessage : Input이 유효하지 않다는 Event를 전송
  ```
  private fun showInvalidInputMessage(text: String) = viewModelScope.launch {
      addEditTaskEventChannel.send(AddEditTaskEvent.ShowInvalidInputMessage(text))
  }
  ```
- createTask : database에 insert하고 result code와 함께 NavigateBackWithResult Event를 전송
  ```
  private fun createTask(task: Task) = viewModelScope.launch {
      taskDao.insert(task)
      // navigate back
      addEditTaskEventChannel.send(AddEditTaskEvent.NavigateBackWithResult(ADD_TASK_RESULT_OK))
  }
  ```
- updateTask : database를 update하고 result code와 함께 NavigateBackWithResult Event를 전송
  ```
  private fun updateTask(task: Task) = viewModelScope.launch {
      taskDao.update(task)
      // navigate back
      addEditTaskEventChannel.send(AddEditTaskEvent.NavigateBackWithResult(EDIT_TASK_RESULT_OK))
  }
  ```
- Event 클래스 정의
  ```
  sealed class AddEditTaskEvent {
      data class ShowInvalidInputMessage(val msg: String) : AddEditTaskEvent()
      data class NavigateBackWithResult(val result: Int) : AddEditTaskEvent()
  }
  ```
### Result Code는 MainActivity 에 정의
```
const val ADD_TASK_RESULT_OK = Activity.RESULT_FIRST_USER
const val EDIT_TASK_RESULT_OK = Activity.RESULT_FIRST_USER + 1
```

## addedit / AddEditTaskFragment
- onViewCreated의 ```binding.apply{}```
  ```
  // EditText의 text가 바뀔 때마다 viewModel의 taskName을 업데이트. 
  editTextTaskName.addTextChangedListener { 
      viewModel.taskName = it.toString()
  }
  ```
  - **addTextChangedListener** : Adds a TextWatcher to the list of those whose methods are called whenever this TextView's text changes.
  ```
  // checkBox 체크 속성이 바뀔 때마다 viewModel의 taskImportance 업데이트
  checkBoxImportant.setOnCheckedChangeListener{ _, isChecked ->
      viewModel.taskImportance = isChecked
  }
  // fab 클릭시 viewModel의 onSaveClick 호출
  fabSaveTask.setOnClickListener {
      viewModel.onSaveClick()
  }
  ```
- collect Event Flow 
  - setFragmentResult 호출 
  > Sets the given result for the requestKey. This result will be delivered **to a FragmentResultListener** 
  > that is called given to setFragmentResultListener with the **same requestKey**. 
  > If no FragmentResultListener with the same key is set or the Lifecycle associated with the listener 
  > is not at least androidx.lifecycle.Lifecycle.State.STARTED, the result is stored until one becomes available, 
  > or clearFragmentResult is called with the same requestKey.
  > Params:
  > **requestKey** - key used to identify the result
  > **result** - the result to be passed to another fragment or null if you want to clear out any pending result.
- event에 따라 처리
  ```
  viewLifecycleOwner.lifecycleScope.launchWhenStarted {
      viewModel.addEditTaskEvent.collect { event ->
          when(event) {
              is AddEditTaskViewModel.AddEditTaskEvent.NavigateBackWithResult -> {
                  binding.editTextTaskName.clearFocus()
                  setFragmentResult(
                      "add_edit_request", // requestKey: String
                      bundleOf("add_edit_result" to event.result) // result: Bundle
                  )
                  findNavController().popBackStack() // -> TasksFragment
              }
              is AddEditTaskViewModel.AddEditTaskEvent.ShowInvalidInputMessage -> {
                  Snackbar.make(requireView(), event.msg, Snackbar.LENGTH_LONG).show() // "Name cannot be empty"
              }
          }.exhaustive
      }
  }
  ```
  
## tasks / TasksFragment, TasksViewModel
- Fragment - onViewCreated
  - AddEditTaskFragment에서 돌아오면서 FragmentResult를 수신
  - bundle에서 result를 얻어옴(ADD_TASK_RESULT_OK 또는 EDIT_TASK_RESULT_OK)
  - viewModel의 onAddEditResult 호출
  ```
  setFragmentResultListener("add_edit_request") { _, bundle ->
      val result = bundle.getInt("add_edit_result")
      viewModel.onAddEditResult(result)
  }
  ```
- ViewModel 
  - result에따라 다른 toast 메세지를 띄우기 위해 showTaskSavedConfirmationMessage 호출
  ```
  fun onAddEditResult(result: Int) {
      when (result) {
          ADD_TASK_RESULT_OK -> showTaskSavedConfirmationMessage("Task added")
          EDIT_TASK_RESULT_OK -> showTaskSavedConfirmationMessage("Task updated")
      }
  }
  ```
  - TasksEvent.ShowTaskSavedConfirmationMessage 이벤트를 전송
  ```
  private fun showTaskSavedConfirmationMessage(text: String) = viewModelScope.launch {
      tasksEventChannel.send(TasksEvent.ShowTaskSavedConfirmationMessage(text))
  }
  
  sealed class TasksEvent {
      object NavigateToAddTaskScreen : TasksEvent()
      data class NavigateToEditTaskScreen(val task: Task) : TasksEvent()
      data class ShowUndoDeleteTaskMessage(val task: Task) : TasksEvent()
      data class ShowTaskSavedConfirmationMessage(val msg: String) : TasksEvent() //추가
  }
  ```
- Fragment
  - 이벤트 collect -> ShowTaskSavedConfirmationMessage 처리
  ```
  is TasksViewModel.TasksEvent.ShowTaskSavedConfirmationMessage -> {
      Snackbar.make(requireView(), event.msg, Snackbar.LENGTH_LONG).show()
  }
  ```
  
  ## AndroidManifest - 키보드 사용 시 floating button이 가려지는 문제 
  ```
  <activity android:name="com.codinginflow.mvvmtodo.ui.MainActivity"
    android:windowSoftInputMode="adjustResize">
  ```
  
