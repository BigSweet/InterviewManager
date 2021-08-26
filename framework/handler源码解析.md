# handler源码解析
handler主要涉及到message，messagequene，loop，threadlocal
发送一条message会进入messagequene中，然后通过loop循环取出这个message，然后回调handlermessage，如果handler是在主线程调用的话就会回调到主线程

## handler线程切换是怎么实现的
handler创建的时候会生成对应的looper和messagequene



handler的looper中有一个sThreadLocal用来获取looper，sThreadLocal的get和set方法是获取当前线程中的ThreadLocalMap，looper作为value。threadlocal作为key存储起来。



如果handler是在主线程new出来的，那获取到的looper就是mainlooper，handler.sendmessage。message中的target就是主线程的handler。在dispatchmessage的时候回调的就是主线程



如果handler是在子线程中新建的，可以通过threadlocal获取主线程的looper在去调用handlermessage达到线程切换的目的



```
 public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```



looper中的mainlooper是静态的，所有的looper公用

```
 private static Looper sMainLooper;
```


handler.send  messagequene.enquenemessage 将消息放入队列中
loop.loop会触发messagequene.next  将消息从队列中取出
loop.prepare之后线程自己在调用loop.loop开启循环
loop.prepare 中新建了一个loop并且放入threadlocal中 一个threadlocal中只有一个loop
每个消息都有一个when，存放的是当前时间+你要延迟的时间 ，消息队列中的message是根据时间来排列的

消息队列中的阻塞和唤醒机制
messagequene.的next方法中
当队列中没有消息的时候，nextPollTimeoutMillis为-1，将mBlocked设置为true,触发nativePollOnce在C++层进行线程休眠让出cpu 阻塞

messagequene.的enqueueMessage方法中
如果mBlocked为true，会将needWake = mBlocked ，当needwake为true，就会触发nativeWake，唤醒cpu

为什么创建一个message使用obtain而不是new message
obtain 会复用pool中的message，减少内存空间的申请
message在取出来之后会调用recycleUnchecked，将message存储在pool中

如果是在子线程调用loop.loop 在线程执行完成之后，需要手动调用handler.quit()  防止内存溢出

threadlocal 主要是get set方法
set方法，传入一个value（T）,获取当前的线程，获取线程的ThreadLocalMap,将value和当前的threadlocal存入threadlocalmap中
get方法 获取当前线程，获取当前线程的threadlocalmap，根据当前的threadlocal取出map中的entry，返回entry的value值
threadlocal有一个唯一的一个hash值，这个值是根据一个原子操作getandadd方法获得

在handler中ThreadLocalMap存储的是threadlocal和looper