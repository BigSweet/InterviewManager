## hashmap的扩容

默认是16大小的tab数组，当数组长度增加到16*负载因子0.75的时候就是大于>12的时候，就会进行扩容 12是筏值threshold=默认大小乘以负载因子
当进行扩容的时候，newCap = oldCap << 1，原始数组大小乘以2变成新数组的大小,同时新的threshold也会扩大俩倍。
根据新的原始数组大小，会创建一个新的空数组，开始转移数据。for循环原始数组，将数据取出，移到新的数组中。
如果其中的item是红黑树，就会走((TreeNode<K,V>)e).split(this, newTab, j, oldCap);方法进行数据转移



## hashmap原理

hashmap是一个数组加一条单向链表
put方法通过key的hash值和hashmap的长度减1进行与计算 h & (length-1)得到下标，将key和value封装成一个entry放入table数组中
为什么hashmap的长度是2的幂次方 2的幂次方是偶数 偶数-1等于奇数，能确保二进制最后一位为1，进行与运算的时候就可以分布均匀，
否则如果是奇数的话，奇数-1=偶数，二进制后面是0，进行与运算一直是0，一直是偶数，浪费了所有的奇数位
get方法，根据key的hash值，和key的值，遍历node节点，在node中寻找寻找hash值相同，且key也相同的node，返回node的value值
为什么hashmap是无序的，因为他是先遍历table在去遍历链表