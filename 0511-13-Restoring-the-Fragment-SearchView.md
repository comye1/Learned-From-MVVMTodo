# #13 Restoring the Fragment SearchView
## 요약
- SearchView가 configuration change 발생 시 보존되지 않음
- Fragment가 다시 만들어지기 때문에!


## tasks / TasksFragment
- searchView 선언 이동
  - onCreateOptionsMenu -> TasksFragment 내부 최상위 ```private lateinit var searchView: SearchView```
- viewModel에 SaveStateHandle로 저장된 searchQuery가 존재하는(!= null) 경우 
  - expandActionView : 검색 창 확장
  - setQuery(pendingQuery, false) : 검색어를 pendingQuery로 설정, query를 submit하지 않음
```
searchView = searchItem.actionView as SearchView

val pendingQuery = viewModel.searchQuery.value
if(pendingQuery != null && pendingQuery.isNotEmpty()){
    searchItem.expandActionView()
    searchView.setQuery(pendingQuery, false)
}
```
- onDestroyView에서 searchView Query 리스너 해제
  - view가 destroy된 후에 불필요하게 동작하지 않도록
```
override fun onDestroyView() {
    super.onDestroyView()
    searchView.setOnQueryTextListener(null)
}
```
