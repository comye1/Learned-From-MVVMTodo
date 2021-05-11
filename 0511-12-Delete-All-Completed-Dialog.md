# #12 Delete-All-Completed Dialog
## 요약
- TaskDao에 완료된 작업 삭제 메소드 추가
- DeleteAllCompletedDialogFragment : 대화상자 fragment -> DialogFragment 상속
- delete-all-complete 메뉴 아이템 클릭 -> DeleteAllCompletedDialogFragment로 navigate
- 확인시 삭제 : DeleteAllCompletedViewModel에서 처리 (TaskDao 메소드 호출)

## data / TaskDao
- completed가 true인 Task들을 삭제하는 메소드 deleteCompletedTasks()
```
@Query("DELETE FROM task_table WHERE completed = 1")
suspend fun deleteCompletedTasks()
```

## deleteallcompleted / DeleteAllCompletedDialogFragment
- DialogFragment() 상속
- onCreateDialog() 오버라이드 : create AlertDialog
- positive button 클릭 리스너 구현
```
@AndroidEntryPoint
class DeleteAllCompletedDialogFragment : DialogFragment() {

    private val viewModel: DeleteAllCompletedViewModel by viewModels()

    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog =
        AlertDialog.Builder(requireContext())
            .setTitle("Confirm deletion")
            .setMessage("Do you really want to delete all completed tasks?")
            .setNegativeButton("Cancel", null)
            .setPositiveButton("Yes"){ _, _ ->
                viewModel.onConfirmClick()
            }
            .create()
}
```
- DialogFragment()는 back key event 또는 button click event 발생 시 disappear!!

## deleteallcompleted / DeleteAllCompletedViewModel
- inject TaskDao, CoroutineScope
- onConfirmClick() -> coroutine으로 taskDao 메소드 호출
```
class DeleteAllCompletedViewModel @ViewModelInject constructor(
    private val taskDao: TaskDao,
    @ApplicationScope private val applicationScope: CoroutineScope
) : ViewModel() {

    fun onConfirmClick() = applicationScope.launch {
        taskDao.deleteCompletedTasks()
    }
}
```

## navigation / nav_graph
- DeleteAllCompletedDialogFragment를 destination에 추가
  - Global action으로 추가


## tasks / TasksViewModel
- TasksEvent 객체 추가
  ```
  sealed class TasksEvent {
      ...
      object NavigateToDeleteAllCompletedScreen : TasksEvent()
  }
  ```
- onDeleteAllCompletedClick() 
  - coroutine에서 tasksEventChannel을 통해 이벤트 전송
  ```
  fun onDeleteAllCompletedClick() = viewModelScope.launch {
      tasksEventChannel.send(TasksEvent.NavigateToDeleteAllCompletedScreen)
  }
  ```
  
## tasks / TasksFragment
- onOptionsItemSelected 메뉴 클릭 시 viewModel에 호출
  ```
  R.id.action_delete_all_complete_tasks -> {
      viewModel.onDeleteAllCompletedClick()
      true
  }
  ```
- viewModel로부터 tasksEvent를 받아 DeleteAllCompletedDialogFragment로 navigate
```
viewLifecycleOwner.lifecycleScope.launchWhenStarted {
    viewModel.tasksEvent.collect { event ->
        when(event) {
            ///(생략)///
            TasksViewModel.TasksEvent.NavigateToDeleteAllCompletedScreen -> {
                val action = TasksFragmentDirections.actionGlobalDeleteAllCompletedDialogFragment()
                findNavController().navigate(action)
            }
        }.exhaustive
    }
}
```
