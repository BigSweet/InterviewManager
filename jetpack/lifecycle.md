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
