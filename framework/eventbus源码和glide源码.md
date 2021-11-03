eventbus源码和glide源码

## eventbus源码解析
eventbus注册的时候会传入当前的类名，然后通过反射去获取注解了@subscribe的方法名，threadMode，参数，并存储在findState类中.
findState中有方法名，eventtype的数组
发送的时候会把event放入一个队列里面eventQueue，如果这个event注册过的话，就会取出这个队列的第一条event，通过方法的.invoke触发回调
如果接收event的线程和当前的线程不一致的话就会触发线程切换

eventbus详细源码在csdn

https://blog.csdn.net/qq_15527709/article/details/100982138?spm=1001.2014.3001.5502

## glide源码解析
通过用法来解析，glide.with(this).load(url).into(target)
.with传入了一个上下文，glide通过这个上下文，创建了一个requestmanager和一个空的fragment,将fragment添加到activity中，将requestmanager和fragment进行绑定。这样这个requestmanager就拥有生命周期了
.load 只是进行了一些参数的赋值，获取到了一个requestbuilder
.into 获取一个request，这个request就开始执行。回调onLoadStarted方法，确定好尺寸之后，调用onSizeRead,在里面调用engine.load
先去缓存拿数据，拿到的话直接onresourceread,没拿到的话就会一个DecodeJob的runnable,选择不同的datedetch去loaddata，里面通过urlConnection，去获取图片的流decode成一个resource

glide详细源码在csdn

https://blog.csdn.net/qq_15527709/article/details/114528807?spm=1001.2014.3001.5502