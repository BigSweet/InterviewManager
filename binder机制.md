binder机制通过aidl通信
android interface define language
安卓接口定义语言

aidl分为proxy，和stub。
proxy发送数据,
stub接收数据
_data  //发送到服务端的数据 
_reply // 服务端返回的数据

client                proxy                binder           stub           service
aidl.method       mremote.transact       ontransact		 this.method  	 触发实现类中的对应方法

系统binder通信的流程
服务端通过addservice将binder实体注册添加到servicemanager中，
客户端获取service的时候，servermanager会在注册的Service列表中查找该服务，返回给client
客户端获取到服务端的binder，转换为服务端的aidl，就能调用服务端相对应的方法了。

自定义service binder通信
定义aidl文件夹，定义aidl方法和数据类
服务端注册service服务，将binder添加到activitymanagerservice中
客户端bindserver。在ActivityManagerService查找相应的Service组件，
最终通过bindservice时传入的connection，将Service内部Binder的句柄传给Client。在通过binder转为为服务端的aidl在去调用方法


binder的memorymap函数，会将内核空间和用户空间对应，用户空间可以直接访问内核空间的数据
 映射原理
 mmap会从当前进程中获取用户态可用的虚拟地址空间
 在内存中分配一块连续的虚拟地址空间，
 把Binder在内核空间的数据直接通过指针地址映射到用户空间，供进程在用户空间使用

 一次拷贝
 binder在目标进程的内核空间直接拷贝，然后目标进程的内核空间和用户空间是有映射的，这就是一次拷贝的原理


