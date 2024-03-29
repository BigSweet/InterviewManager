android基础问题

## server的生命周期
startService -> oncreate->onstartcommand->ondestory
bindservice->oncreate->onbind->onunbind->ondestory

### 四种启动模式

Standard 标准模式

```
Android创建Activity时的默认模式，假设没有为Activity设置启动模式的话，默觉得标准模式。每次启动一个Activity都会又一次创建一个新的实例入栈，无论这个实例是否存在。
```

 SingleTask 栈内复用模式

```
若须要创建的Activity已经处于栈中时，此时不会创建新的Activity，而是将存在栈中的Activity上面的其他Activity所有销毁，使它成为栈顶。
```

 SingleTop 栈顶复用模式

```
分两种处理情况：须要创建的Activity已经处于栈顶时，此时会直接复用栈顶的Activity。不会再创建新的Activity；若须要创建的Activity不处于栈顶，此时会又一次创建一个新的Activity入栈，同Standard模式一样。
```

SingleInstance 单实例模式

```
具有此模式的Activity仅仅能单独位于一个任务栈中
```



### A页面跳转到B页面

当A跳转到B的时候，A先执行onPause，然后居然是B再执行onCreate -> onStart -> onResume，最后才执行A的onStop

当B按下返回键，B先执行onPause，然后居然是A再执行onRestart -> onStart -> onResume，最后才是B执行onStop  -> onDestroy

## fragment之间传递数据
eventbus
通过父activity传递数据
接口回调方式传递数据

viewmodel传递数据



## 让自定义view滑动
scrollTo与scrollBy方法可以实现滑动，也是通过修改坐标
或者通过scroll类原理基本一致

## listview滑动卡顿优化   
用viewholder来封装itemview减少findview的操作，滑动的时候不要加载图片，等滑动结束在加载

## webview加载优化 
js文件放在本地实现优化

## 屏幕适配方案
主要由三种适配方案，一种是通过多种dimen文件多个dp比如通过最小宽度文件实现适配
一种是自定义view，自定义一些layout，在获取屏幕的宽高，和设计稿的宽度做一个对比来打到设配
一种是今日头条的适配方案，通过修改系统的dpi值达到适配，也是获取当前设备的宽度和高度来和设计稿做一个对比

## android各版本兼容
6.0 权限适配,移除支持http,低电耗模式



7.0 应用之间共享文件，私有目录将被限制访问 不能通过file:// URI的形式 使用fileprovider要用content:// URI

多窗口支持,画中画，APK signature scheme v2



8.0 通知变化了很多 要加channelid groupid等给通知进行分类 透明的activity不允许设置方向  安装APK，不允许安装来源不明的应用，需要授权，悬浮窗权限的变更



9.0 多摄像头支持，Android 9 引入了 [`AnimatedImageDrawable`](https://developer.android.com/reference/android/graphics/drawable/AnimatedImageDrawable) 类，用于绘制和显示 GIF 和 WebP 动画图像



10.0可折叠设备,5G 网络,深色主题



11.0 Android 11 提供了一些 API 以支持瀑布屏，无线调试，ADB 增量 APK 安装

## 自定义view的俩种方式
一种是自定义viewgroup，只需要重写，onmeasure，和onlayou。子view可以是自定义的view也可以是实例化一个布局
一种是自定义view 继承view 重写onmesure ，ondraw
在ondraw里面画三角形，原型，或者实现你需要的功能，然后还可以自定义属性，通过attrs获取到这个属性

## 序列化Serializable和Parcelable的区别
序列化后的对象可以在网络上进行传输，也可以存储到本地。
Parcelable android 自带  效率高 但是代码会多一点 把对象进行了分解，分解成了intent支持的类型
Serializable java自带 代码量少 但是效率较低   通过反射实现 创建了很多临时变量
存储在磁盘上的话还是优先选择Serializable

## 横竖屏切换生命周期的变化
如果设置了configChanges="orientation|keyboardHidden|screenSize"
那么横竖屏切换的时候只会调用onConfigurationChanged
如果只设置了android:configChanges="orientation"或者没有设置
那么横竖屏切换的生命周期为
首先进入应用
onCreate->onResume
这个时候旋转屏幕
onPause->onStop->onSaveInstanceState->onDestroy->onCreate->onRestoreInstanceState->onResume

## webview和js的交互方式 jsbrige
调用js的方法通过webview.load方法，
js调用java方法，通过定义@JavascriptInterface，统一处理，url的跳转通过shouldOverrideUrlLoading统一处理

## 线程池怎么关闭
shundown  当前正在运行中的不会关闭，
shundowNow 当前正在运行中的也会关闭

## 静态变量和成员变量的区别
静态变量属于类，成员变量属于对象
静态变量位于方法区中的静态区，成员变量存储于堆内存
静态变量可以通过类名调用，也可以通过对象名调用，成员变量只能通过对象名调用
静态变量随着类的加载而加载，成员变量随着对象的创建而存在

## 数组和arraylist的区别
数组的长度是固定的，类型安全的
arraylist的长度不固定，可以动态扩容，类型不安全
arraylist内部封装了一个object的数组。添加，修改数据都会频繁的装箱和拆箱。装箱就是把int转换成object。拆箱就是把object转换成int

## arraylist的扩容
调用add方法，如果是空数组就会扩容到10 ，如果不是空数组，就size+1，如果size+1（size是当前数据数量）> 数组的长度就需要扩容
扩容最大值是integer.maxvalue - 8，扩容大小是数组的长度右移1位+数组的长度，调用数组的copyof方法

## volatile和synchronized的区别
执行控制 内存可见
执行控制的目的是控制代码执行（顺序）及是否可以并发执行。
内存可见控制的是线程执行结果在内存中对其它线程的可见性。根据Java内存模型的实现，线程在具体执行时，会先拷贝主存数据到线程本地（CPU缓存），操作完成后再把结果从线程本地刷到主存。

volatile本质是在告诉jvm当前变量在CPU的高速缓存中的值是不确定的，需要从主存中读取； synchronized则是锁定当前变量，只有当前线程可以访问该变量，其他线程被阻塞住。
volatile仅能使用在变量级别；synchronized则可以使用在变量、方法、和类级别的
volatile仅能实现变量的修改可见性，不能保证原子性；而synchronized则可以保证变量的修改可见性和原子性
volatile不会造成线程的阻塞；synchronized可能会造成线程的阻塞

volatile关键字解决的是内存可见性的问题，会使得所有对volatile变量的读写都会直接刷到主存

volatile这个关键字的其中一个重要作用就是解决可见性问题，即保证当一个线程修改了某个变量之后，该变量对于另外一个线程是立即可见的。

当一个变量被声明成volatile之后，任何一个线程对它进行修改，都会让所有其他CPU高速缓存中的值过期，这样其他线程就必须去内存中重新获取最新的值，也就解决了可见性的问题。

volatile关键字还有另外一个重要的作用，就是禁止指令重排

CPU在执行代码时，其实并不一定会严格按照我们编写的顺序去执行，而是可能会考虑一些效率方面的原因，对那些先后顺序无关紧要的代码进行重新排序，这个操作就被称为指令重排。

##原子性操作：原子性在一个操作是不可中断的，要么全部执行成功要么全部执行失败



## anr相关知识
如果机器上出现了anr，系统会生成一个traces.txt的文件放在/data/anr下
traceview，会显示方法的耗时操作和调用次数
一般用debug。startMethodTracingSampling（）和Debug.stopMethodTracing()，会生成一个trace文件，直接拖到android studio里面就能看到性能方便的一些图，包括方法的调用次数，方法的耗时



## wait await sleep区别
都会让出cpu 
sleep 线程不会释放对象锁 过了指定时间之后就会自动运行
wait 会释放对象锁 ，等待notify才继续执行
await 会释放对象锁 通过signal才继续执行

## git merge rebase的区别
merge会产生一个merge分支 rebase不会产生额外的commit更加干净一点

## 换肤操作
首先是拿到皮肤包的resource，根据这个皮肤包的路径通过反射获取AssetManager，就可以获取到皮肤包resource。

在通过给layoutinfalte设置factory2，接管oncreateview的这个过程，然后再factory的oncreateview中可以拿到xml里面所有的控件和属性，然后可以用一个skikenable是否为true来标记这个view是不是需要换肤，拿到这个标记为true的view，存储起来在一个skinitem里面

换肤库的baseactivity继承一个接口，在生命周期attch的时候，skinmanager会把activity作为一个observer保存起来，deattch的时候移除掉，在触发换肤操作的时候，循环这个observer（就是activity），触发他们的update方法，再出发自定义的基础控件的apply方法，在里面进行换肤操作

自定义view，首先需要些自己需要修改的属性的attrs，在apply方法里面写需要进行的一个操作，然后再AttrFactory里面注册自己的属性名字，调用base里面dynamicAddSkinEnableView方法就可以了



## java8的新特性
扩展方法 给接口添加一个非抽象的方法实现（添加一个默认方法用default关键字标记）
Lambda表达式
jodatime时间库
可以使用多重注解
lambda对对象的字段和静态变量都可以读写
lambda可以访问局部变量
使用::关键字来传递方法或者构造函数引用

## nestscrollview和recyclerview嵌套引起的缓存问题
监听globallistener，获取scrollview的高度给recyclerview
nestscrollview的测量模式是UNSPECIFIED,recyclerview在onmeasure的时候，如果是UNSPECIFIED会实例化所有的child