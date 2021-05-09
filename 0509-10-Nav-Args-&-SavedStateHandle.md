# #10 Nav Args & SvedStateHandle

## 요약
AddEditTaskFragment, AddEditTaskViewModel 추가
- Task의 편집 및 생성을 위한 fragment와 viewModel
- TasksFragment에서 FAB 버튼으로 navigate
- **SavedStateHandle** 사용
  
# navigation / nav_graph.xml
- destination 추가 : addEditTaskFragment  
  - Arguments 1. task (Task)
    - custom parcelable arguments
    - Nullable (true) & Default Value @null 
    - Add -> null, Edit -> nonNull Task 
  - Argument 2. title (String)
    - actionbar에 표시될 이름으로 설정 ```android:label="{title}"``` 
- action 추가 : ```action_tasksFragment_to_addEditTaskFragment```

## addedittask / AddEditTaskViewModel
- SavedStateHandle : configuration change에도 상태(입력된 텍스트 등)를 보존하기 위한 클래스
> A handle to saved state passed down to ViewModel. You should use SavedStateViewModelFactory if you want to receive this object in ViewModel's constructor.
This is a **key-value map** that will let you **write and retrieve objects to and from the saved state**. 
These values will **persist after the process is killed by the system and remain available via the same object.**
You can read a value from it via **get(String)** or observe it via LiveData returned by **getLiveData(String).**
You can write a value to it via **set(String, Object)** or setting a value to MutableLiveData returned by **getLiveData(String).**
```
class AddEditTaskViewModel @ViewModelInject constructor(
    private val taskDao: TaskDao,
    @Assisted private val state : SavedStateHandle
) : ViewModel() {
```
- read saved "task"
```
    val task = state.get<Task>("task")
```
- read saved "taskName" 
- taskName의 setter 호출 시 SavedStateHandle에도 저장되도록 함
```
    var taskName = state.get<String>("taskName") ?: task?.name ?: ""
        set(value) {
            field = value
            state.set("taskName", value)
        }
```
- read saved "taskImportance"
- taskImportance의 setter 호출 시 SavedStatedHandle에도 저자오디도록 함
```
    var taskImportance = state.get<Boolean>("taskImportance") ?: task?.important ?: false
        set(value) {
            field = value
            state.set("taskImportance", value)
        }

}
```

## addedittask / AddEditTaskFragment.kt
```
@AndroidEntryPoint
class AddEditTaskFragment : Fragment(R.layout.fragment_add_edit_task) {

    private val viewModel: AddEditTaskViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        val binding = FragmentAddEditTaskBinding.bind(view)
```
- viewModel의 taskName, taskImportance, task (SavedSatateHandle에 저장된 값이거나 초기화된 값) 을 뷰에 설정
```
        binding.apply {
            editTextTaskName.setText(viewModel.taskName)
            checkBoxImportant.isChecked = viewModel.taskImportance
            checkBoxImportant.jumpDrawablesToCurrentState() // animation, transition을 생략 (완료된 상태로) 
            textViewDateCreated.isVisible = viewModel.task != null // task가 null이 아닐 때에만 보이도록 함
            textViewDateCreated.text = "Created : ${viewModel.task?.createdDateFormatted}"
        }
    }
}
```

## tasks / TasksFragment.kt, TasksViewModel.kt
- ViewModel의 TasksEvent에 navigation을 위한 이벤트 추가
```
  sealed class TasksEvent {
      object NavigateToAddTaskScreen : TasksEvent()
      data class NavigateToEditTaskScreen(val task: Task) : TasksEvent()
      data class ShowUndoDeleteTaskMessage(val task: Task) : TasksEvent()
  }
```

- FAB 클릭시 navigate 
  - Fragment
    ```
    fabAddTask.setOnClickListener {
        viewModel.onAddNewTaskClick()
    }
    ```
  - ViewModel의 onAddNewTaskClick - tasksEventChannel을 통해 TasksEvent.NavigateToAddTaskScreen 전송
    ```
    fun onAddNewTaskClick() = viewModelScope.launch {
        tasksEventChannel.send(TasksEvent.NavigateToAddTaskScreen)
    }
    ```
- Task 아이템 클릭시 navigate
  - Fragment
    ```
    override fun onItemClick(task: Task) {
        viewModel.onTaskSelected(task)
    }
    ```
  - ViewModel의 onTaskSelected - tasksEventChannel을 통해 TasksEvent.NavigateToEditTaskScreen(선택된 task) 전송
    ```
    fun onTaskSelected(task: Task) = viewModelScope.launch {
        tasksEventChannel.send(TasksEvent.NavigateToEditTaskScreen(task))
    }
    ```
- collect TasksEvents & actions
  - Arguments와 함께 navigate
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
              is TasksViewModel.TasksEvent.NavigateToAddTaskScreen -> {
                  val action = TasksFragmentDirections.actionTasksFragmentToAddEditTaskFragment(null, "New Task")
                  findNavController().navigate(action)
              }
              is TasksViewModel.TasksEvent.NavigateToEditTaskScreen -> {
                  val action = TasksFragmentDirections.actionTasksFragmentToAddEditTaskFragment(event.task, "Edit Task")
                  findNavController().navigate(action)
              }
          }.exhaustive
      }
  }
```
- **Utils.kt** 확장함수 exhaustive 정의 
  ``` 
  val <T> T.exhaustive: T
    get() = this
  ```
  - when 절이 하나의 expression으로 여겨져서 컴파일러가 모든 case에 대해 정의하도록 강제함

## MainActivity.kt
- 클래스 내부 최상단에 lateinit var navController
- onCreate에서 초기화, setupActionBarWithNavController 호출
  ```
    val navHostFragment =
        supportFragmentManager.findFragmentById(R.id.nav_host_fragment) as NavHostFragment
    navController = navHostFragment.findNavController()

    // deal with action bar
    setupActionBarWithNavController(navController)
  ```
- onSupportNavigateUp으로 Up 동작 override - nav_graph.xml에 따른 Up동작이 있을 경우 수행
  ```
  override fun onSupportNavigateUp(): Boolean {
      return navController.navigateUp() || super.onSupportNavigateUp()
  }
  ```
