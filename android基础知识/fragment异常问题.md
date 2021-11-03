fragment 异常

1,getActivity == null  

不要在异步任务后获取activity，因为可能为空

2,Can not perform this action after onSaveInstanceState

不能在onSaveInstanceState之后的生命周期里面commit fragment 
不要在子线程commit fragment  已经走完onSaveInstanceState

3,fragment 重叠

​    设置屏幕的方向，走 onConfigurationChanged 

​	 if(savedInstanceState == null){add fragment }在这个情况下才添加

4，传参问题

不要写有参数的构造函数，因为内存重启自动恢复的时候，会自动调用fragment无参的构造方法，这样就会出现异常情况

5，添加到回退栈 重写back自己处理回退逻辑
   addToBackStack(@Nullable String name) 
   popBackStack()
   name: 
   popBackStack(@Nullable final String name, final int flags)

6，add replace不要混合使用

replace会把容器的view全部清空

add会往容易中叠加添加view

