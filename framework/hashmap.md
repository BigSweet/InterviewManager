## hashmap的扩容

默认是16大小的tab数组，当数组长度增加到16*负载因子0.75的时候就是大于>12的时候，就会进行扩容 12是筏值threshold=默认大小乘以负载因子
当进行扩容的时候，newCap = oldCap << 1，原始数组大小乘以2变成新数组的大小,同时新的threshold也会扩大俩倍。
根据新的原始数组大小，会创建一个新的空数组，开始转移数据。for循环原始数组，将数据取出，移到新的数组中。
如果其中的item是红黑树，就会走((TreeNode<K,V>)e).split(this, newTab, j, oldCap);方法进行数据转移



## hashmap原理

hashmap是一个数组加一条单向链表
put方法通过key的hash值和hashmap的长度减1进行与计算 h & (length-1)得到下标，将key和value封装成一个entry放入table数组中
为什么hashmap的长度是2的幂次方 2的幂次方是偶数 偶数-1等于奇数，能确保二进制最后一位为1，进行与运算的时候就可以分布均匀

如最开始hashmap的长度是16

   0000 0000  0001  0000 

 16-1 = 15 

0000 0000 0000 1111

后面4位都是1111 那么进行与运算的时候 就能确保我们的下标一定是在0-15之间

否则如果是奇数的话，奇数-1=偶数，二进制后面是0，进行与运算一直是0，一直是偶数，浪费了所有的奇数位

get方法，根据key的hash值，和key的值，遍历node节点，在node中寻找寻找hash值相同，且key也相同的node，返回node的value值
为什么hashmap是无序的，因为他是先遍历table在去遍历链表

Hashmap 1.7是头插法

1.8是尾插法

hashmap是通过二次hash来减少hash碰撞

hashmap 1.8 当链表的长度>8且数组长度大于64node就会被转换为树节点（红黑树）

红黑树是黑平衡二叉树

hashmap使用红黑树的意义是为了树不退化成链表（通过右旋来平衡）



## councurrenthashmap

1.7和1.8的区别

### 数据结构方面

1.7:

数组（Segment） + 数组（HashEntry） + 链表（HashEntry节点）

1.8:

数组+链表+红黑树



### 线程安全方面

1.7

ReentrantLock+Segment+HashEntry

分段锁 对整个桶数组进行了分割分段(Segment)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。

1.8

synchronized+CAS+HashEntry+红黑树

使用的是优化的synchronized 关键字同步代码块 和 cas操作了维护并发



### put方面

1.7

Segment的继承体系可以看出，Segment实现了ReentrantLock,也就带有锁的功能，当执行put操作时，会进行第一次key的hash来定位Segment的位置，如果该Segment还没有初始化，即通过CAS操作进行赋值，然后进行第二次hash操作，找到相应的HashEntry的位置，这里会利用继承过来的锁的特性，在将数据插入指定的HashEntry位置时（链表的尾端），会通过继承ReentrantLock的tryLock（）方法尝试去获取锁，如果获取成功就直接插入相应的位置，如果已经有线程获取该Segment的锁，那当前线程会以自旋的方式去继续的调用tryLock（）方法去获取锁，超过指定次数就挂起，等待唤醒

1.8

如果没有初始化就先调用initTable（）方法来进行初始化过程
如果没有hash冲突就直接CAS插入
如果还在进行扩容操作就先进行扩容
如果存在hash冲突，就加锁来保证线程安全，这里有两种情况，一种是链表形式就直接遍历到尾端插入，一种是红黑树就按照红黑树结构插入，
最后一个如果Hash冲突时会形成Node链表，在链表长度超过8，Node数组超过64时会将链表结构转换为红黑树的结构，break再一次进入循环
如果添加成功就调用addCount（）方法统计size，并且检查是否需要扩容
