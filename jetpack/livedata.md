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

