leakcanary原理，和内存泄漏相关

##leakcanary原理解析
watch一个要销毁的对象，
首先通过provide特性在oncreate的时候进行初始化，通过applicaiton一个注册lifecyclecallback监听activity的生命周期，在ondestory的时候，会生成一个objectWtcher观察者开始观察这个activity，生成一个key通过这个key生成一个弱引用和生成的观察者关联起来，存储在map里面，开始对这个activity进行观察，五秒后移到怀疑队列里面，根据引用队列规则判断是否有内存系列（手动执行gc垃圾回收，如果activity存在内存泄漏refercequene中就不会有这个引用)如果发生内存泄漏，通过haha可达性分析找到这个泄漏的对象，找到hprof文件,转换为一个快照，在这个快照中找到第一个弱引用对象，遍历这个对象，寻找当初创建的key相同的对象，这个对象就是内存溢出的对象

涉及到的原理

refercequene 一般和软引用，弱引用关联
如果对象被垃圾回收期回收了 就会把这个对象对应的引用容器加入到refercequene队列队尾

如果对象被回收了那么在这个队列里面就会出现这个引用的容器，反之就是内存泄漏

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

##4种引用类型
分为4种，用来控制对象的生命周期
强引用 不会被垃圾回收期回收，会抛出oom异常
软引用 在内存足够的情况下 不会去回收 效果和强引用一样 内存不够的时候会去回收
弱引用 生命周期比软引用少很多，在gc回收的时候，不管内存是否足够 都会回收他
虚引用 就是没有引用，垃圾回收期任何时候都会回收他

