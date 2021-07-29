rxjava原理解析

## rxjava原理解析

基于观察者模式的一个异步操作库，
观察者向被观察者注册，当被观察者有变化的时候，触发观察者去notify，这些具体的实现，由他们对应的子类去实现
3个重要的东西，观察者，被观察者，事件，订阅。线程切换。操作符。
首先创建一个被观察者，一个观察者，被观察者订阅观察者，发动一个事件。可以通过操作符对这个事件进行转换，最后在通过线程切换在不同的线程去做一些事情。

map操作符，针对事件的变换的操作符，比如讲map转换为bitmap
flatmap操作符，是针对被观察者变换的操作符，比如讲string的被观察者变成bitmap类型的被观察者

```
Observable.just()
                .map(new Function<String, String>() {
                    @Override
                    public String apply(String s) throws Exception {

                })
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String path) throws Exception {

                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(Throwable throwable) throws Exception {
                       
                    }
                });

```

Observable.just返回的是ObservableJust 

.map返回的是ObservableMap

```
public static <T> Observable<T> just(T item) {
        ObjectHelper.requireNonNull(item, "item is null");
        return RxJavaPlugins.onAssembly(new ObservableJust<T>(item));
    }
    
    public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }
```

看下最后的subscribe

```
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
            Action onComplete, Consumer<? super Disposable> onSubscribe) {
        LambdaObserver<T> ls = new LambdaObserver<T>(onNext, onError, onComplete, onSubscribe);
        subscribe(ls);
        return ls;
    }
    
    public final void subscribe(Observer<? super T> observer) {
            subscribeActual(observer);
    }
```

记住最后一个Observer是LambdaObserver

触发的是ObservableMap的subscribeActual

首先看下ObservableMap的构造方法

source 是在ObservableMap创建的时候传入的this,而this就是上一个Observable也就是ObservableJust

```
public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
    super(source);
    this.function = function;
}

@Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
```

source是ObservableJust也就是上一级Observable的subscribe

看下ObservableJust的subscribeActual

```
 @Override
    protected void subscribeActual(Observer<? super T> observer) {
        ScalarDisposable<T> sd = new ScalarDisposable<T>(observer, value);
        observer.onSubscribe(sd);
        sd.run();
    }
```

触发了observer的onSubscribe方法，observer是构造方法中传进来的，subscribeActual是source.subscribe触发的，所以这里的observer就是MapObserver

在mapObserver的onnext方法中

```
 public void onNext(T t) {
         
            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
            downstream.onNext(v);
        }
```

除了出发mapper.apply(t)之外，还调用了downstream.onNext(v);

downstream是在构造方法传进来的

```
 MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);//在这里
            this.mapper = mapper;
        }
  public BasicFuseableObserver(Observer<? super R> downstream) {
        this.downstream = downstream;
    }
```

也就是source.subscribe(new MapObserver<T, U>(t, function));这个t

这个t是subscribeActual的参数

```
@Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
```

所以这个subscribeActual里面的Observer是最后的LambdaObserver

整个流程结束



SubscribeOn只生效第一次，影响的是上游

observeon 可以调用多次  ，影响的是下游，最后一次生效

```
SubscribeOn只生效第一次
subscribeOn(AndroidSchedulers.mainThread())
.subscribeOn(Schedulers.io())

subscribe触发之后，会从下往上调用上一个SubscribeOn的subscribeActual方法，所以最终触发的是第一个SubscribeOn


    observeOn 可以调用多次 最后一次生效
    
     Worker worker = scheduler.createWorker();

        if (s instanceof ConditionalSubscriber) {
            source.subscribe(new ObserveOnConditionalSubscriber<T>(
                    (ConditionalSubscriber<? super T>) s, worker, delayError, prefetch));
        } else {
            source.subscribe(new ObserveOnSubscriber<T>(s, worker, delayError, prefetch));
        }
        
        每次都是创建一个新的work，创建一个新的task去执行
```



##rxjava背压
在异步订阅中，由于被观察者发送数据和观察者接收数据的速度不匹配，导致无法及时响应，内存溢出。就引入了背压这个概念
使用flowable观察者模型，发送的事件会进入一个缓存区，根据观察者的需求，响应式的拿取数据
flowable会手动控制观察者和被观察者的速度 通过一个request方法取数据，被观察者也可以通过emiter.request来获取观察者需要的数据数量（只能在同步订阅中使用）
同步订阅就是被观察者发送一个数据，观察者处理完成之后才会继续处理下一个数据

```
//Observable的实现类ObservableCreate
//被观察者ObservableOnSubscribe  source
//观察者Consumer  
//subscribe触发了实现类的subscribeActual
//subscribeActual触发了source的subscribe和LambdaObserver的onSubscribe方法
// LambdaObserver的onsubscribe中触发了onSubscribe.accept(this);
//这里的 onSubscribe=Consumer
Observable.create(ObservableOnSubscribe<String> {
    it.onNext("nihao")
}).subscribe(Consumer {

})
```



背压策略有很多种，包括只取最新的，或者设置缓存区无限大，或者直接抛出异常，或者丢弃超出缓存区的事件

背压模式

```
public enum BackpressureStrategy {
    /**
     * OnNext events are written without any buffering or dropping.
     * Downstream has to deal with any overflow.
     * <p>Useful when one applies one of the custom-parameter onBackpressureXXX operators.
     */
    MISSING,
    /**
     * Signals a MissingBackpressureException in case the downstream can't keep up.
     */
    ERROR,
    /**
     * Buffers <em>all</em> onNext values until the downstream consumes it.
     */
    BUFFER,
    /**
     * Drops the most recent onNext value if the downstream can't keep up.
     */
    DROP,
    /**
     * Keeps only the latest onNext value, overwriting any previous value if the
     * downstream can't keep up.
     */
    LATEST
}
```

