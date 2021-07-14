## viewmodel

viewmodel一般和livedata结合使用，viewmodel是一个可以感知fragment生命周期的，用来做数据存储的一个库

解决网络请求，异步操作带来的内存泄漏问题，fragment传递数据不方便的问题，解决屏幕旋转导致的数据销毁问题。

原理解析
ViewModelProviders.of方法创建AndroidViewModelFactory，ViewModelStore根据这俩个参数创建ViewModelProvider，在调用get方
法获取viewmodel实例

```
@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);

    if (modelClass.isInstance(viewModel)) {
        //noinspection unchecked
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
    } else {
        viewModel = (mFactory).create(modelClass);
    }
    mViewModelStore.put(key, viewModel);
    //noinspection unchecked
    return (T) viewModel;
}
```

通过ViewModelStore来做缓存

```
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```



如果model是继承androidviewmodel就会通过类的.getConstructor获取这个类的实例对象，然后存入mViewModelStore中

mViewModelStore中使用map来存储，key是model的类名

如果不是继承androidviewmodel就会通过类的modelClass.newInstance(),创建实例化对象。

创建好这个model之后，就可以在里面进行，网络请求，创建livedata回调数据。
在界面摧毁的时候，viewmodel的clear方法会执行，系统类ComponentActivity添加了LifecycleEventObserver在ondestory的监听会触发getViewModelStore().clear()方法

```
 getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_DESTROY) {
                    if (!isChangingConfigurations()) {
                        getViewModelStore().clear();
                    }
                }
            }
        });
```

获取viewmodel的方法

1,

```
val viewModel by lazy { ViewModelProviders.of(this).get(FriendViewModel::class.java) }
```

2,

```
private val viewModel: PlantListViewModel by viewModels {
        Injector.providePlantListViewModelFactory(requireContext())
}
这个是一个扩展方法
@MainThread
inline fun <reified VM : ViewModel> Fragment.viewModels(
    noinline ownerProducer: () -> ViewModelStoreOwner = { this },
    noinline factoryProducer: (() -> Factory)? = null
) = createViewModelLazy(VM::class, { ownerProducer().viewModelStore }, factoryProducer)
```

