## 泛型进阶

```
<T extend String>
指定泛型的类型，一般定义在类后面，成为限定符
```

```
泛型方法

public <E> void add(E e)

泛型类的泛型只影响泛型的普通方法，不影响泛型类中的泛型方法，所以泛型方法可以和泛型的泛型不一直，比如
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

