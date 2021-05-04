# #6 Combining multiple Flows
검색어, 정렬 기준, 필터를 한 번에 처리하기

## tasks / TasksViewModel.kt
- 기존에 있던 searchQuery
- \+sortOrder
  - ``` enum class SortOrder { BY_NAME, BY_DATE }```
- \+hideCompleted
- 기존에는 searchQuery만 관찰해 새로운 Tasks Flow를 가져왔다면
- 이제는 searchQuery, sortOrder, hideComplete를 모두 관찰해 새로운 Tasks Flow를 가져오도록!
  - 여러 Flow들을 결합 -> combine, Triple 사용
  - **combine** https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/combine.html
  ```
  @JvmName("flowCombine") fun <T1, T2, R> Flow<T1>.combine(
      flow: Flow<T2>,
      transform: suspend (a: T1, b: T2) -> R
  ): Flow<R> (source)
  Returns a Flow whose values are generated with transform function by combining the most recently emitted values by each flow.
  ```
  ```
  fun <T1, T2, T3, R> combine(
    flow: Flow<T1>,
    flow2: Flow<T2>,
    flow3: Flow<T3>,
    transform: suspend (T1, T2, T3) -> R
  ): Flow<R> (source)
  ``` 
  ```
  private val tasksFlow = combine(
    searchQuery, // flow
    sortOrder, // flow2
    hideCompleted // flow3
  ) { query, sortOrder, hideCompleted ->
    Triple(query, sortOrder, hideCompleted) //transform, into a Flow of Triple object
  }.flatMapLatest { (query, sortOrder, hideCompleted) ->
  taskDao.getTasks(query, sortOrder, hideCompleted)   // switch to another flow with new searchQuery
  }
  ```


## task / TasksFragment.kt
- 옵션메뉴 선택 시 뷰모델의 sortOrder 값 변경
- onOptionsItemSelected 에서 처리

## data / TaskDao.kt
- TasksViewModel에서 ```taskDao.getTasks(query, sortOrder, hideCompleted)```로 호출

```
fun getTasks(query: String, sortOrder: SortOrder, hideCompleted: Boolean): Flow<List<Task>> =
    when(sortOrder) {
        SortOrder.BY_DATE -> getTasksSortedByDateCreated(query, hideCompleted)
        SortOrder.BY_NAME -> getTasksSortedByName(query, hideCompleted)
    }

@Query("SELECT * FROM task_table WHERE (completed != :hideCompleted OR completed = 0) AND name LIKE '%' || :searchQuery || '%' ORDER BY important DESC, name")
fun getTasksSortedByName(searchQuery: String, hideCompleted: Boolean): Flow<List<Task>>

@Query("SELECT * FROM task_table WHERE (completed != :hideCompleted OR completed = 0) AND name LIKE '%' || :searchQuery || '%' ORDER BY important DESC, created")
fun getTasksSortedByDateCreated(searchQuery: String, hideCompleted: Boolean): Flow<List<Task>>
```
- Query 문 이해하기 : ```(completed != :hideCompleted OR completed = 0)``` 
  - hideCompleted가 true(1)일 때 -> completed != 1 OR completed = 0 -> completed = 0 (false)인 Task만 -> 완료되지 않은 Task만!
  - hideCompleted가 false(0)일 때 -> completed != 0 OR completed = 0 -> completed가 0 또는 1인 Task -> 모든 Task
