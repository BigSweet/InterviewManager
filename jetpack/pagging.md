## paging

分页库

基本使用方式

1,创建数据源（网络，数据库，本地测试）

```
class StudentDataSource : PositionalDataSource<Student>() {
    override fun loadInitial(params: LoadInitialParams, callback: LoadInitialCallback<Student>) {
        callback.onResult(getStudents(0, PAGE_SIZE), 0, 1000)
    }

    override fun loadRange(params: LoadRangeParams, callback: LoadRangeCallback<Student>) {
        callback.onResult(getStudents(params.startPosition, params.loadSize))
    }


    fun getStudents(startPosition: Int, pageSize: Int): List<Student> {
        var list = arrayListOf<Student>()
        for (i in startPosition until pageSize + startPosition) {
            val student = Student()
            student.age = i
            student.name = "robat" + i
            list.add(student)
        }
        return list
    }
}
```

2,创建数据源仓库

```
class StudentDataSourceFactory : DataSource.Factory<Int, Student>() {
    override fun create(): DataSource<Int, Student> {
        return StudentDataSource()
    }
}
```

3,创建pagelist

```
class StudentViewModel : ViewModel() {

    var liveData: LiveData<PagedList<Student>>

    init {
        val factory = StudentDataSourceFactory()
        liveData = LivePagedListBuilder<Int, Student>(factory, PAGE_SIZE).build()
    }

}
```

最后在mainactivity中讲得到的数据源设置给recyclerview

```
var model = ViewModelProvider(this, ViewModelProvider.NewInstanceFactory()).get(StudentViewModel::class.java)
        model.liveData.observe(this, Observer {
            adapter.submitList(it)
        })
        rv.layoutManager = LinearLayoutManager(this)
        rv.adapter = adapter
```

流程源码分析
当viewmodel创建的时候会执行

```
LivePagedListBuilder<Int, Student>(factory, PAGE_SIZE).build()
```

在LivePagedListBuilder的create方法中

```
compute()执行跟踪走到
new TiledPagedList<>的create方法中
创建new TiledPagedList<>
最后执行datasource的loadInitial获得初始化的数据
并且
在ComputableLiveData中会把compute得到的Initial发送出去
       mLiveData.postValue(value);             
那么在activity中的observer中会接收到这个数据
设置给adapter
model.liveData.observe(this, Observer {
            adapter.submitList(it)
        })
跟踪adapter.submitlist
会将数据赋值给AsyncPagedListDiffer中的
if (mPagedList == null && mSnapshot == null) {
            // fast simple first insert
            mPagedList = pagedList;
            pagedList.addWeakCallback(null, mPagedListCallback);

            // dispatch update callback after updating mPagedList/mSnapshot
            mUpdateCallback.onInserted(0, pagedList.size());

            onCurrentListChanged(null, pagedList, commitCallback);
            return;
        }
mPagedList = pagedList;
在getitem的时候根据position拿到list中的数据
会再 mUpdateCallback.onInserted(0, pagedList.size());中触发adapter的刷新操作


```

当界面滑动触发刷新操作的时候
会触发
adapter的getitem操作

```
mPagedList.loadAround(index);->
```

跟踪

```
mPagedList.loadAround->loadAroundInternal(index);->子类tiledpagedlist
->mStorage.allocatePlaceholders->当index到达设置的pagesize的时候会触发 callback.onPagePlaceholderInserted(pageIndex);
->子类titlepagelist-> mDataSource.dispatchLoadRange->
loadRange(new LoadRangeParams(startPosition, count), callback);
去调用我们实现的加载更多的数据
```
