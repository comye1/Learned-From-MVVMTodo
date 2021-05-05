# #7 Saving Data with Jetpack DataStore
## 요약
- Tasks 정렬 기준과 필터 옵션을 저장하려고 함
- Jetpack DataStore를 통해 저장할 수 있음!
  - DataStore provides a safe and durable way to **store small amounts of data**, such as **preferences** and **application state**.

## data / PreferencesManager.kt
- DataStore에 저장하기 위한 클래스
  - ```enum class SortOrder { BY_NAME, BY_DATE }```
  - ```data class FilterPreferences(val sortOrder: SortOrder, val hideCompleted: Boolean)```
```
@Singleton
class PreferencesManager @Inject constructor(@ApplicationContext context: Context){

    private val dataStore = context.createDataStore("user_preferences") // "user_preferences"라는 이름의 DataStore 생성

    val preferencesFlow = dataStore.data // 저장된 데이터를 Flow로 받아옴 
        .catch { exception -> 
            if (exception is IOException){
                Log.e(TAG, "Error reading preferences", exception)
                emit(emptyPreferences())
            } else {
                throw exception
            }
        } // 예외 처리
        .map { preferences ->
            val sortOrder = SortOrder.valueOf(
                preferences[PreferencesKeys.SORT_ORDER] ?: SortOrder.BY_DATE.name //elvis 연산자(?:) 로 저장된 값이 없는 경우 디폴트 값 지정
            ) // 저장된 값으로부터 SortOrder enum 상수 값을 얻어옴
            val hideCompleted = preferences[PreferencesKeys.HIDE_COMPLETED] ?: false
            FilterPreferences(sortOrder, hideCompleted)
        } // map으로 Flow of preferences -> Flow of FilterPreferences 변환하기 

    // dataStore에서 sortOrder 값 편집
    suspend fun updateSortOrder(sortOrder: SortOrder) {
        dataStore.edit { preferences ->
            preferences[PreferencesKeys.SORT_ORDER] = sortOrder.name
        }
    }

    // dataStore에서 hideCompleted 값 편집 
    suspend fun updateHideCompleted(hideCompleted: Boolean) {
        dataStore.edit { preferences ->
            preferences[PreferencesKeys.HIDE_COMPLETED] = hideCompleted
        }
    }

    // Preferences 접근, 관리를 위한 키 값 (PreferencesKeys.SORT_ORDER, PreferencesKeys.HIDE_COMPLETED)
    private object PreferencesKeys {
        val SORT_ORDER = preferencesKey<String>("sort_order")
        val HIDE_COMPLETED = preferencesKey<Boolean>("hide_completed")
    }
}
```
- 클래스 인스턴스가 생성될 때 (Singleton) DataStore에서 Preferences를 읽어와 가지고 있으며, update~ 함수 호출 시 그 값들을 수정하는 역할!!

## tasks / TasksViewModel.kt
- 생성자에 preferencesManger 주입, 여기에서 인스턴스를 사용
- preferencesFlow를 읽어와 tasksFlow 가져오기(getTasks 호출)
  - ```
    val preferencesFlow = preferencesManager.preferencesFlow

    private val tasksFlow = combine(
        searchQuery,
        preferencesFlow
    ) { query, filterPreferences ->
        Pair(query, filterPreferences)
    }.flatMapLatest { (query, filterPreferences) ->
     taskDao.getTasks(query, filterPreferences.sortOrder, filterPreferences.hideCompleted)
    // switch to another flow with new searchQuery
    }
    ```
- SortOrder, HideCompleted 변경 시 코루틴에서 Preferences 저장(수정)
  - ```
    fun onSortOrderSelected(sortOrder: SortOrder) = viewModelScope.launch {
        preferencesManager.updateSortOrder(sortOrder)
    }

    fun onHideCompletedClick(hideCompleted: Boolean) = viewModelScope.launch {
        preferencesManager.updateHideCompleted(hideCompleted)
    }

    ```
  - 뷰모델 생성 시 PreferenceManager 인스턴스를 통해 Preferences를 읽어 그에 맞게 Tasks를 읽어 오며 뷰(Fragment)에서 값들을 변경하려 할 때 update 함수를 호출하는 역할!!

## tasks / TasksFragment.kt
- hide_complete 체크박스의 isChecked를 설정하기 위해 뷰모델에서 preferencesFlow 값을 읽어옴
  - ```
    // read current state
    viewLifecycleOwner.lifecycleScope.launch {
        menu.findItem(R.id.action_hide_completed_tasks).isChecked =
            viewModel.preferencesFlow.first().hideCompleted //read single value
    }
    ```
- Options Item 선택 시 뷰모델에 이를 알려줌 (onOptionsItemSelected)
  - ```
    R.id.action_sort_by_name -> {
       viewModel.onSortOrderSelected(SortOrder.BY_NAME)
       true
    }
    R.id.action_sort_by_date_created -> {
        viewModel.onSortOrderSelected(SortOrder.BY_DATE)
        true
    }
    R.id.action_hide_completed_tasks -> {
        item.isChecked = !item.isChecked
        viewModel.onHideCompletedClick(item.isChecked)
        true
    }
    ```

    
