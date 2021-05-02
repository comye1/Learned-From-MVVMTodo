# #4 ListAdapter with View Binding 
## 꿀팁
- TextView에 취소선 긋기 : textView.paint.isStrikeThruText 속성
- 마지막 파라미터가 람다식이면 밖으로 빼기 (몸체처럼)
	```
	 viewModel.tasks.observe(viewLifecycleOwner) {
	     taskAdapter.submitList(it) 
	 }
	```

## task / TasksAdapter.kt
- extends ListAdapter - DiffUtil 사용
- class TasksViewHolder
	- bind
- parent : Fragment or Activity
- class DiffCallback : DiffUtil.ItemCallback\<Task\>
	- Contents - compare old Task and new Task with "==" (benefit of using data class)
	- Items - compare id
	
## tasks / TasksViewModel
- get Tasks as LiveData 
- LiveData : lifecycle-aware

## tasks / TasksFragment.kt
1. fun onCreateView (super)
	-  Fragment 생성자에 전달된 레이아웃을 inflate -> view
 
1. fun onViewCreated (override)
	- 바인딩에 view를 부착(bind)

	- instantiate TaskAdapter
	- binding의 recyclerViewTasks
		- bind adapter
		- layoutManager 설정
		- setHasFixedSize(true)
	 - observe viewModel's tasks
		 - call taskAdapter.submitList(it)
		 - and DiffUtil will take care of the changed list
