## 内存模型

程序计数器：用来记录当前线程执行的位置，线程私有，和线程生命周期一致

虚拟机栈：线程私有，jvm基于虚拟机栈，dvm基于寄存器，每个方法执行的时候都会在虚拟机中创建一个栈帧

每个栈帧包括局部变量表，操作数栈，动态连接，返回地址。

本地方法栈：为虚拟机使用到的 Native 方法服务

堆：所有的线程都可以访问，垃圾回收器工作的区域，存放对象实例

方法区：线程共享，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等

![image-20201026173135257](/Users/yanzhe/android/知识整理/image/image-20201026173135257.png)







## 泛型进阶

```
<T extend String>
指定泛型的类型，一般定义在类后面，成为限定符
```

```
泛型方法

public <E> void add(E e)

泛型类的泛型只影响泛型的普通方法，不影响泛型类中的泛型方法，所以泛型方法可以和泛型的泛型不一致，比如
```

```
public class Genenic<T>{

		//影响普通方法
		public T add(){}
		
		泛型E可以被识别
		public <E> add(E e){}

}


```



```
通配符
<? extend parent> 向上匹配至parent类  parent类的父类不支持  用于安全的得到数据 得到parent类
一般放在方法参数使用的时候 
<? super child> 向下匹配至child类 child类的子类不支持  用于安全的写入数据 得到child和child的子类

```



```
public class WildChar{

    public static void print(GenericType<Fruit> p){
        System.out.println(p.getData().getColor());
    }

    public static void use(){
       GenericType<Fruit> a = new GenericType<>();
        print(a);
       GenericType<Orange> b = new GenericType<>();
        //print(b);
    }


    public static void print2(GenericType<? extends Fruit> p){
        System.out.println(p.getData().getColor());
    }

    public static void use2(){
        GenericType<Fruit> a = new GenericType<>();
        print2(a);
        GenericType<Orange> b = new GenericType<>();
        print2(b);
        //print2(new GenericType<Food>());
        GenericType<? extends Fruit> c =  new GenericType<>();

        Apple apple =  new Apple();
        Fruit fruit = new Fruit();
        //c.setData(apple);
        //c.setData(fruit);
        Fruit x = c.getData();
    }

    public static void printSuper(GenericType<? super Apple> p){
        System.out.println(p.getData());
    }

    public static void useSuper(){
        GenericType<Fruit> fruitGenericType = new GenericType<>();
        GenericType<Apple> appleGenericType = new GenericType<>();
        GenericType<HongFuShi> hongFuShiGenericType = new GenericType<>();
        GenericType<Orange> orangeGenericType = new GenericType<>();
        printSuper(fruitGenericType);
        printSuper(appleGenericType);
//        printSuper(hongFuShiGenericType);
//        printSuper(orangeGenericType);


        //表示GenericType的类型参数的下界是Apple
        GenericType<? super Apple> x = new GenericType<>();
        x.setData(new Apple());
        x.setData(new HongFuShi());
        //x.setData(new Fruit());
        Object data = x.getData();

    }

}
```





通过反射来获取类，可以修改类的私有方法和属性，可以获取这个类的方法，属性，注解等

动态代理proxy invacationhandle

给一个类设置动态代理后，当这个类调用方法的时候，会触发invocationhandler的invoke方法。

在invoke中调用 method.invoke 触发方法本来的操作，通过可以在前后加入想加入的功能代码





多线程

```
 //直接新建threadl
        val thread = Thread {

        }
        //新建runnable
        Thread(Runnable {

        }).start()

        //使用callable。可以有返回值
        val callAble = Callable<String> {
            "test"
        }
        val futureTask = FutureTask<String>(callAble)
        Thread(futureTask).start()
        futureTask.get() //获取返回值
```

线程中断方法，结合

```
val thread = object : Thread() {
            override fun run() {
                super.run()
                while (!this.isInterrupted) {
                    println("do some thing")
                }
            }
        }
        thread.start()
        Thread.sleep(50)
        thread.interrupt()//通过这个方法来打断线程
```

并行和并发

并行是同时几个线程能一起运行

并发是指在某一个时间段里面运行了多少个线程

![image-20200612175059240](/Users/yanzhe/android/知识整理/image/image-20200612175059240.png)

Synchronized

内置锁

可以分为对象锁和类锁

对象锁

```
 Object object = new Object();

    synchronized void test() {

    }

    void test1() {
        synchronized (this) {

        }
    }

    void test2() {
        synchronized (object) {

        }
    }
```

类锁 

```
synchronized static void test() {

    }
```

锁住静态变量

yield,sleep不会释放锁

wait会释放锁

wait通过notify唤醒

wait和notify是配合synchronized一起使用

显式锁lock  获取锁和释放锁都可以控制，可以尝试获取锁

隐式锁 synchronized 无法控制获取锁的这个过程

公平和非公平锁 

如果在时间上，先对锁进行获取的请求一定先被满足，那么这个锁是公平的，反之，是不公平的。公平的获取锁，也就是等待时间最长的线程最优先获取锁，也可以说锁获取是顺序的

synchronized是非公平锁 

```
Lock lock = new ReentrantLock();
        try{
            lock.lock();
        }finally {
            lock.unlock();
        }
```

范式用法

可重入锁：可以重复获取同一种锁

如果想在lock里面实现wait可以使用

```
 lock.newCondition().await();
 lock.newCondition().notify();
```



线程池含义解析

```
 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) 
```

第一个参数核心线程数

线程池执行任务的时候创建的线程数量。

当数量大于核心线程数的时候，就开始存入BlockingQueue（阻塞队列）中。

当队列也存满的时候，会根据maximumPoolSize的大小重新创建新的线程，

当maximumPoolSize也满了的时候，执行RejectedExecutionHandler（拒绝策略）

keepAliveTime阻塞队列中线程的存活时间，超过这个时间还没有被取出，就会被抛弃

unit存活时间的单位

4种拒绝策略，

1，直接抛出异常。（默认策略）

2，抛弃队列头部的数据

3，抛弃队列尾部的数据

4，让当前线程自己处理



asynctask 里面有俩个线程池，一个用来维持串行的队列，一个用来执行异步线程

handler执行异步消息的通知，和线程间的切换



悲观锁  每次对数据进行操作的时候，都认为有线程正在修改， synchronized`和`ReentrantLock 就是悲观锁

乐观锁 atomicintger 原子性操作 CAS检查（比较和替换只有当oldvalue和newvalue一致的时候才会替换）

饿汉式  直接new出来

懒汉式 锁住之后，在new出来



序列化

```
继承 Serializable 需要添加SerializableUid 内存存在大量的反射，会有内存碎片
继承Parcelable 效率更高，android推荐

```

Android里面为什么要设计出Bundle而不是直接用Map结构

Bundle内部是由ArrayMap实现的，ArrayMap的内部实现是两个数组，一个int数组是存储对象数据对应下标，一个对象数组保存key和value，内部使用二分法对key进行排序，所以在添加、删除、查找数据的时候，都会使用二分法查找，只适合于小数据量操作，如果在数据量比较大的情况下，那么它的性能将退化。而HashMap内部则是数组+链表结构，所以在数据量较少的时候，HashMap的Entry Array比ArrayMap占用更多的内存。因为使用Bundle的场景大多数为小数据量，我没见过在两个Activity之间传递10个以上数据的场景，所以相比之下，在这种情况下使用ArrayMap保存数据，在操作速度和内存占用上都具有优势，因此使用Bundle来传递数据，可以保证更快的速度和更少的内存占用。
另外一个原因，则是在Android中如果使用Intent来携带数据的话，需要数据是基本类型或者是可序列化类型，HashMap使用Serializable进行序列化，而Bundle则是使用Parcelable进行序列化。而在Android平台中，更推荐使用Parcelable实现序列化，虽然写法复杂，但是开销更小，所以为了更加快速的进行数据的序列化和反序列化，系统封装了Bundle类，方便我们进行数据的传输。





io字节流

```
DataOutputStream dataOutputStream = new DataOutputStream(new BufferedOutputStream(new FileOutputStream(new File(""))));
```

装饰者模式

outputStream作为基类

FileOutputStream对文件的操作

BufferedOutputStream对字节流的操作

DataOutputStream 对特定类型的操作

```
DataInputStream dataInputStream = new DataInputStream(new BufferedInputStream(new FileInputStream(new File(""))));
```

inputStream作为基类

与上面的一一对应



io字符流 字符有行的概念

BufferedWriter 对字符的操作

OutputStreamWriter 建立字节和字符联系的类







```
RandomAccessFile

seek(long pos) 定位到文件的某一个点
setLength(long newLength) 设置一个指定大小的空文件
```





打包流程

aapt生成R文件

java文件生成class文件

class文件转换为dex文件

资源文件和dex文件打包成apk



加固流程  加密之后的dex无法被虚拟机识别，所以用一个aar文件作为壳子

新建一个android moudle build成一个aar文件 作为壳文件

将旧的apk解压到一个指定文件目录B，找到dex文件，转换为byte数组。对byte数组进行加密之后，在放回dex文件

将aar文件转换为dex文件，将dex文件转换为byte，在B目录下面新建一个class.dex文件，将AAR的byte数组写入class.dex

新建一个apk-unsigned.apk文件，将B目录压缩到这个文件中

新建一个apk-signed.apk,将apk-unsigned.apk进行签名转为apk-signed.apk



脱壳

在Application的attachBaseContext进行脱壳

获取apkFile目录解压到app缓存目录，对dex文件进行解密，将解密后的dex文件通过反射获取dexelementlist，将解密后的dex文件放入elementList中，通过dexclassloader加载到程序中