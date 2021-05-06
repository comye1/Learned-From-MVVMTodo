# #8 Updating Checked Tasks

## 요약
- 체크박스 클릭 시 Task의 completed 속성을 바꾸도록 클릭리스너 설정 & 이벤트 처리하기

## tasks / TasksAdapter.kt
- 클래스 내부에 interface 정의 OnItemClickListener
  - fun onItemClick
  - fun onCheckBoxClick

- TasksAdapter 생성자 파라미터에 추가
- TasksViewHolder의 init 블록에서 아이템과 체크박스에 클릭리스너 설정
  -  TasksAdapter 클래스의 getItem(protected method) 접근 위해 inner class로 설정
  -  ```
      root.setOnClickListener {
        val position = adapterPosition
        if (position != RecyclerView.NO_POSITION) { // 유효한 position일 때
            val task = getItem(position) 
            listener.onItemClick(task) // TasksAdatper의 생성자로 전달된 listener의 onItemClick
        }
      }
     ```
    
  - ```
      checkBoxCompleted.setOnClickListener {
        val position = adapterPosition
        if (position != RecyclerView.NO_POSITION) {
            val task = getItem(position)
            listener.onCheckBoxClick(task, checkBoxCompleted.isChecked)
        }
      }
    ``` 
    
## tasks / TasksFragment.kt
- TasksAdapter.OnItemClickListener를 상속
  - onItemClick 구현 ``` viewModel.onTaskSelected(task)```
  - onCheckBoxClick 구현 ```viewModel.onTaskCheckedChanged(task, isChecked)```
- TasksAdapter 생성시 자기 자신(this)을 전달 ```val taskAdapter = TasksAdapter(this)```
- 클릭 이벤트 발생 시 viewModel의 함수를 호출해 이벤트를 처리하기 때문에 자기 자신이 클릭 리스너의 기능을 함 --> interface의 구현!!

## tasks / TasksViewModel.kt
- 클릭 이벤트를 처리 : 비즈니스 로직을 수행!!
  - viewModelScope에서 taskDao의 suspend function update를 호출 (coroutines) 
  - task를 copy 후 completed 속성만 변경하여 update
```
  fun onTaskSelected(task: Task) {
    // ToDo
  }

  fun onTaskCheckedChanged(task: Task, isChecked: Boolean) = viewModelScope.launch {
      taskDao.update(task.copy(completed = isChecked)) // other properties are copied
  }
```
