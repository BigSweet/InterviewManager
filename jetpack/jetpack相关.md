jetpack相关

## livedata

观察者模式构建的一个和生命周期有关系的一个库，可以减少内存泄漏，保证UI状态和数据的统一，不需要手动处理生命周期的变化
一般用到的都是LifecycleBoundObserver，他有一个statechange方法，当生命周期变化后，会通知livedata去更新数据，如果生命周期大于start，就会回调onchange方法，生命周期结束，会移除这个mObserver

相关代码如下

livedata添加监听的时候会生成一个LifecycleBoundObserver他继承了LifecycleEventObserver当生命周期变化后，statechange方法会回调

```
@MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
    }
```

当statechange执行后,如果是destory生命周期，自动移除监听

```
@Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            activeStateChanged(shouldBeActive());
        }
```

判断当前生命周期是否在STARTED之后

```
 @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }
```



## lifecycle

通过lifecycleOwner.getLifecycle().addObserver(this)给presenter添加lifecycle，fragment和activity默认实现了lifecycleowner，在presenter里面注解@OnLifecycleEvent，当生命周期变化后就会回调这个对应的方法

原理
android 9.0ComponentActivity默认实现了LifecycleOwner，lifecycle的一个接口类，在oncreate的时候生成了一个reportfragment,并把这个fragment依赖ComponentActivity，然后再reportfragment生命周期变化的时候，会dispatch lifecycle的event，在handleLifecycleEvent，最后会触发mLifecycleObserver的onStateChanged方法，
然后这个observer里面有一个callbackinfo，里面用一个map存储了所有标记了@lifecycleevent的方法名和event值，在通过invokeCallbacks传进来的这个event找到对应的方法，通过invoke回调出去

相关代码如下

```
@Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mSavedStateRegistryController.performRestore(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);
        if (mContentLayoutId != 0) {
            setContentView(mContentLayoutId);
        }
    }
```

ReportFragment中,每一个生命周期触发的时候，都会分发

```
@Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

```

触发所有的mLifecycleObserver的statechange方法

```
void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
```

标记了注解@lifecycleevent的方法回调流程



```
LifecycleRegistry类中
 @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        //每次都会new一个ObserverWithState
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    }
    
    static class ObserverWithState {
        State mState;
        LifecycleEventObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
            mState = initialState;
        }

    }
    
主要是这个方法
Lifecycling.lifecycleEventObserver(observer);

@NonNull
    static LifecycleEventObserver lifecycleEventObserver(Object object) {   
        int type = getObserverConstructorType(klass); 
    }
    
    private static int getObserverConstructorType(Class<?> klass) {
        int type = resolveObserverCallbackType(klass);
        return type;
    }
    
    private static int resolveObserverCallbackType(Class<?> klass) { 
        boolean hasLifecycleMethods = ClassesInfoCache.sInstance.hasLifecycleMethods(klass);
    }
    
    boolean hasLifecycleMethods(Class<?> klass) {
        Method[] methods = getDeclaredMethods(klass);
        for (Method method : methods) {
            OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
            if (annotation != null) {
                createInfo(klass, methods);
                return true;
            }
        }
    }
    
    
     private CallbackInfo createInfo(Class<?> klass, @Nullable Method[] declaredMethods) {
     
        CallbackInfo info = new CallbackInfo(handlerToEvent);
        mCallbackMap.put(klass, info);
        mHasLifecycleMethods.put(klass, hasLifecycleMethods);
        return info;
    }
    
    createinfo会将所有标记了注解的方法存在mCallbackMap中
    当ReflectiveGenericLifecycleObserver的statechange触发的时候会触发这些callback的方法
    
    
    
```

## viewmodel

viewmodel一般和livedata结合使用，viewmodel是一个可以感知fragment生命周期的，用来做数据存储的一个库

解决网络请求，异步操作带来的内存泄漏问题，fragment传递数据不方便的问题，解决屏幕旋转导致的数据销毁问题。

原理解析
ViewModelProviders.of方法创建AndroidViewModelFactory，ViewModelStore根据这俩个参数创建ViewModelProvider，在调用get方法获取viewmodel实例
如果model是继承androidviewmodel就会通过类的.getConstructor获取这个类的实例对象，然后存入mViewModelStore中

mViewModelStore中使用map来存储，key是model的类名

如果不是继承androidviewmodel就会通过类的modelClass.newInstance(),创建实例化对象。

创建好这个model之后，就可以在里面进行，网络请求，创建livedata回调数据。
在界面摧毁的时候，viewmodel的clear方法会执行，系统类ComponentActivity添加了LifecycleEventObserver在ondestory的监听会触发getViewModelStore().clear()方法



## room数据库

建库@database 继承room的database

```
@Database(entities = [Student::class], version = 1)
abstract class StudentDataBase : RoomDatabase() {
    abstract fun studentDao(): StudentDao
}
```

建表@enetry

```
@Entity
class Student() {

    @PrimaryKey
    var id: Int = 0


    @ColumnInfo(name = "name")
    var name: String = ""

    @ColumnInfo(name = "age")
    var age: Int = 0

    override fun toString(): String {
        return "Student(id=$id, name='$name', age='$age')"
    }

}
```

建操作层dao 里面有insert delete update等操作

```
@Dao
interface StudentDao {

    @Insert
    fun insert(vararg student: Student)


    @Delete
    fun delete(student: Student)

    @Query("select * from Student")
    fun getAll():List<Student>
}
```

最后通过线程操作数据库

```
Thread {
            val dao = Room.databaseBuilder(applicationContext, StudentDataBase::class.java, "test").build()
            dao.studentDao().insert(student)
            Log.d("swt", dao.studentDao().getAll().toString())
        }.start()
```



多张表查询通过主表的主键和外表的外键相关联

数据库升级需要addMigrations

```
val dao = Room.databaseBuilder(applicationContext, StudentDataBase::class.java, "test")
                .addMigrations()
                .build()
```

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

## workmanager

[WorkManager](https://developer.android.google.cn/reference/androidx/work/WorkManager) 是一个 API，使您可以轻松调度那些即使在退出应用或重启设备时仍应运行的

功能:

**添加工作约束**

**强大的调度**

**灵活的重试政策**

**链接多个任务**

**无缝集成RxJava和协程**

实例代码

```
var continuation = workManager
                .beginUniqueWork(
                        IMAGE_MANIPULATION_WORK_NAME,
                        ExistingWorkPolicy.REPLACE,
                        OneTimeWorkRequest.from(CleanupWorker::class.java)
                )
               
val blurBuilder = OneTimeWorkRequestBuilder<BlurWorker>()
 //worker之间传递数据
blurBuilder.setInputData(createInputDataForUri())
continuation = continuation.then(blurBuilder.build())
  //添加约束
val constraints = Constraints.Builder()
                .setRequiresCharging(true)
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
val save = OneTimeWorkRequestBuilder<SaveImageToFileWorker>()
                .setConstraints(constraints)
                .addTag(TAG_OUTPUT)
                .build()
continuation = continuation.then(save)
continuation.enqueue()
```



## navigation



