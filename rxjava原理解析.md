rxjava原理解析
##rxjava原理解析
基于观察者模式的一个异步操作库，
观察者向被观察者注册，当被观察者有变化的时候，触发观察者去notify，这些具体的实现，由他们对应的子类去实现
3个重要的东西，观察者，被观察者，事件，订阅。线程切换。操作符。
首先创建一个被观察者，一个观察者，被观察者订阅观察者，发动一个事件。可以通过操作符对这个事件进行转换，最后在通过线程切换在不同的线程去做一些事情。
map操作符，针对事件的变换的操作符，比如讲map转换为bitmap
flatmap操作符，是针对被观察者变换的操作符，比如讲string的被观察者变成bitmap类型的被观察者
subscribeon 创建了一个新的subscribe，传入了source。在通过传入的scheduler去creatework
一般是scheduler.io对应IoScheduler创建了一个newthreadwork通过线程池去执行。
因为subscribeon 每次创建新的类传入的source都是我们创建的single，最终在subscribeActual里面触发的都是

source.subscribe发送到下游

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
背压策略有很多种，包括只取最新的，或者设置缓存区无限大，或者直接抛出异常，或者丢弃超出缓存区的事件





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