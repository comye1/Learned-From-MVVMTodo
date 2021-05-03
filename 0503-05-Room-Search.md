# #5 Room Search

## menu_fragment_tasks.xml
- showAsAction 속성 사용
	- Search, Sort -> "always" 보여줌 
	- Hide, Delete -> "never" overflow 표시
- actionViewClass 속성
	- 검색을 할 수 있도록 "androidx.appcompat.widget.SearchView"로 설정
	>  An  _action view_  is an action that provides rich functionality within the app bar. For example, **a search action view allows the user to type their search text in the app bar, without having to change activities or fragments.**
		
## TasksFragment.kt
- onCreateOptionsMenu
	- inflate menu
	- deal with searchItem and searchView
	- searchView.**onQueryTextChanged()** - viewModel.searchQuery를 업데이트
- onOptionsItemSelected
- onViewCreated { setHasOptionsMenu

## util / ViewExt.kt
- extension functions
- SearchView.onQueryTextChanged
	- implement onQueryTextListener object
	```
	    override fun onQueryTextSubmit(query: String?): Boolean {
	        return true
	    }

	    override fun onQueryTextChange(newText: String?): Boolean {
	        listener(newText.orEmpty())
	        return true
	    }
    ```
- **inline** - overhead를 줄임 
- **crossline** 
	> Crossinline
The  `crossinline`  marker is used to mark lambdas that mustn’t allow non-local returns, especially when such lambda is passed to another execution context such as a higher order function that is not inlined, a local object or a nested function. In other words, you won’t be able to do a  `return`  in such lambdas. https://medium.com/android-news/inline-noinline-crossinline-what-do-they-mean-b13f48e113c2
## data / TaskDao.kt
- getTasks
```
@Query("SELECT * FROM task_table WHERE name LIKE '%' || :searchQuery || '%' ORDER BY important DESC")
fun getTasks(searchQuery: String): Flow<List<Task>>
```
- '%' || :searchQuery || '%'  ==> '%searchQuery%' searchQuery 문자열 앞뒤에 어떤 것이든 붙어 있거나 없는 경우 => 어떤 tasks 안에 searchQuery가 들어있기만 하면 SELECT
## TasksViewModel.kt
```
val searchQuery = MutableStateFlow("")

private val tasksFlow = searchQuery.flatMapLatest {
 taskDao.getTasks(it)   // switch to another flow with new searchQuery
}

val tasks = tasksFlow.asLiveData()
```
- searchQuery 값이 바뀌면 새로운 searchQuery로 taskDao.getTasks를 호출하여 tasksFlow가 바뀌고, tasksFlow가 바뀔 때마다 tasks 또한 그 값으로 업데이트됨
- **flatMapLatest** 
	> Returns a flow that switches to a new flow produced by transform function every time the original flow emits a value. When the original flow emits a new value, the previous flow produced by transform block is cancelled.
	
