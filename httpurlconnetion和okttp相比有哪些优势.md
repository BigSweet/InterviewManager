

##事件分发机制
viewgroup被点击后会触发dispatchTouchEvent，然后里面有一个onInterceptTouchEvent判断是否拦截，如果不拦截就会循环调用子view的dispatchTouchEvent，通过坐标来判断点击的是哪个view，如果点击的是空白区域target==null就会调用super.dispatchTouchEvent(ev)

view的事件分发被点击后先调用dispatchTouchEvent，然后判断是否设置了onTouch事件mOnTouchListener是否为空，按钮是否是可点击的，在调用onTouchEvent，在MotionEvent.ACTION_UP里面调用performClick，如果设置了onclicklistener就会调用onclick事件


##fragment之间传递数据
eventbus
通过特定的方法，拿到父fragment，去调用这个方法
接口回调方式传递数据

##eventbus源码解析
eventbus注册的时候会传入当前的类名，然后通过反射去获取注解了@subscribe的方法名，threadMode，参数，并存储在findState类中.
findState中有方法名，eventtype的数组
发送的时候会把event放入一个队列里面eventQueue，如果这个event注册过的话，就会取出这个队列的第一条event，通过方法的.invoke触发回调
如果接收event的线程和当前的线程不一致的话就会触发线程切换

##glide源码解析
通过用法来解析，glide.with(this).load(url).into(target)
.with传入了一个上下文，glide通过这个上下文，创建了一个requestmanager和一个空的fragment,将fragment添加到activity中，将requestmanager和fragment进行绑定。这样这个requestmanager就拥有生命周期了
.load 只是进行了一些参数的赋值，获取到了一个requestbuilder
.into 获取一个request，这个request就开始执行。回调onLoadStarted方法，确定好尺寸之后，调用onSizeRead,在里面调用engine.load
先去缓存拿数据，拿到的话直接onresourceread,没拿到的话就会一个DecodeJob的runnable,选择不同的datedetch去loaddata，里面通过urlConnection，去获取图片的流decode成一个resource



##git merge rebase的区别
merge会产生一个merge分支 rebase不会产生额外的commit更加干净一点


##livedata
观察者模式构建的一个和生命周期有关系的一个库，可以减少内存泄漏，保证UI状态和数据的统一，不需要手动处理生命周期的变化
一般用到的都是LifecycleBoundObserver，他有一个statechange方法，当生命周期变化后，会通知livedata去更新数据，如果生命周期大于start，就会回调onchange方法，生命周期结束，会移除这个mObserver

##lifecycle
通过lifecycleOwner.getLifecycle().addObserver(this)给presenter添加lifecycle，fragment和activity默认实现了lifecycleowner，在presenter里面注解@OnLifecycleEvent，当生命周期变化后就会回调这个对应的方法

原理
android 9.0ComponentActivity默认实现了LifecycleOwner，lifecycle的一个接口类，在oncreate的时候生成了一个reportfragment,并把这个fragment依赖ComponentActivity，然后再reportfragment生命周期变化的时候，会dispatch lifecycle的event，在handleLifecycleEvent，最后会触发mLifecycleObserver的onStateChanged方法，
然后这个observer里面有一个callbackinfo，里面用一个map存储了所有标记了@lifecycleevent的方法名和event值，在通过invokeCallbacks传进来的这个event找到对应的方法，通过invoke回调出去


##4种引用类型
分为4种，用来控制对象的生命周期
强引用 不会被垃圾回收期回收，会抛出oom异常
软引用 在内存足够的情况下 不会去回收 效果和强引用一样
弱引用 生命周期比软引用少很多，在gc回收的时候，不管内存是否足够 都会回收他
虚引用 就是没有引用，垃圾回收期任何时候都会回收他
refercequene 一般和软引用，弱引用关联
如果软应用被垃圾回收期回收了 就会把这个引用加入到这个队列当中

##leakcanary原理解析
watch一个要销毁的对象，
首先创建一个activitywatch，通过一个lifecyclecallback对象和activity生命周期关联，在ondestory的时候，会生成一个key和弱引用对这个activity进行包装，放入弱引用队列当中，然后会开启一个线程，来分析这个acitvity，会手动执行gc垃圾回收，如果activity存在内存泄漏是不会被回收的，找到hprof文件,转换为一个快照，在这个快照中找到第一个弱引用。遍历这个对象，寻找当初创建的key相同的对象，这个对象就是内存溢出的对象

当发生内存溢出的时候，JVM就会将当前的虚拟机的堆等信息放入hprof文件中


##内存泄漏引起的方式
单例引起的内存泄漏，单例的周期周期是整个应用的生命周期，如果创建单例传入的是activty，会导致单例持有这个activity，导致它无法被回收
解决办法就是将传入的context，去获取application的context，保持生命周期同步

handler引起的内存泄漏： handler做延时处理的时候，没有释放掉这个message，message持有handler引用。handler持有actvity引用，导致无法被回收
解决办法是将handler定义为静态的，不持有activity的引用，或者用弱引用包裹actvity，或者在activity 
ondestory的时候移除handler的message

静态类不持有外部类的引用

线程引起的内存泄漏
线程去做一个耗时任务，这个时候activity关闭了，这个任务还在执行，导致这个actvity无法被回收
解决办法也是将线程定义为静态的

webview引起的内存泄漏
webview去加载很多复杂的页面的时候，会导致内存泄漏，解决办法就是把webview所在的activity放在一条单独的进程当中，在ondestory的时候，去杀掉这条进程

##原子性操作：原子性在一个操作是不可中断的，要么全部执行成功要么全部执行失败

##hashmap的扩容
默认是16大小的tab数组，当数组长度增加到16*负载因子0.75的时候就是大于>12的时候，就会进行扩容 12是筏值threshold=默认大小乘以负载因子
当进行扩容的时候，newCap = oldCap << 1，原始数组大小乘以2变成新数组的大小,同时新的threshold也会扩大俩倍。
根据新的原始数组大小，会创建一个新的空数组，开始转移数据。for循环原始数组，将数据取出，移到新的数组中。
如果其中的item是红黑树，就会走((TreeNode<K,V>)e).split(this, newTab, j, oldCap);方法进行数据转移

##红黑树


rxjava的obser和sub作用域哪一次生效
##kotlin的协程


##gradlew 构建自动化加固
新建java依赖包。新建一个class 继承plugin 注册plugin,通过project创建扩展。拿取username，签名文件


##class的加载过程
加载-》验证-》准备-》解析》初始化-》使用-》卸载
class.forname()会对类进行初始化操作
classloader.loadclass不会对类进行初始化

