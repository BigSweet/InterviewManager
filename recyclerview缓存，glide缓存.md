recyclerview缓存，glide缓存

##recylerview的缓存
四级缓存主要是通过Recycler这个类

mChangedScrap

 mChangedScrap 表示数据已经改变的 ViewHolder 列表, 存储 notifyXXX 方法时需要改变的 ViewHolder

mAttachedScrap

mAttachedScrap 存储的是当前还在屏幕中的 ViewHolder；按照 id 和 position 来查找 ViewHolder

上面俩个统称为scrap缓存

mCachedViews (最大缓存2个)

刚刚移出屏幕的数据缓存 默认是2个，在来一个的话就会放入recyclerviewpool里面，通过position来寻找holder

可以通过setViewCacheSize改变大小

mViewCacheExtension

开发者可自定义的一层缓存，是虚拟类ViewCacheExtension的一个实例，开发者可实现方法getViewForPositionAndType(Recycler recycler, int position, int type)来实现自己的缓存。

recyclerviewpool (最大缓存5个)

保存mCachedViews存不下的viewholder 会把viewholder的数据全部重置，相当于一个新的viewholder
通过itemtype来寻找holder，会重新走onbindview方法

缓存一般都是先通过position来拿在通过id来拿，最后如果没有缓存的话就创建新的viewholder

当界面最开始出现的时候，没有任何缓存，生成新的holder，并且存入mAttachedScrap，如果屏幕中的holder有数据改变，就会存入mChangedScrap，这个时候界面上滑的话，新进入的holder，根据position和id都没有缓存，会存入mAttachedScrap，滑出去的holder会存入mCachedViews，当滑出去的holder数量大于2的时候，就会存入recyclerviewpool中最多五个，当继续来了新的holder的时候，就会把旧的缓存中的holder移出去



##glide的缓存
内存缓存 弱引用+LruCache 

先从弱引用中拿ActiveResources里面有一个activeEngineResources是一个map存储了key和resource

弱引用缓存存储的是正在使用的图片

如果在弱引用没有拿到在去内存缓存中去拿MemoryCache是一个接口默认实现是LruResourceCache

里面是lrucache算法

在onResourceReleased释放资源的时候会存入内存缓存

在图片解码完毕后onEngineJobComplete会存入弱引用缓存

通过变量acquired大于0来表示正在使用的图片



磁盘缓存

decodejob在run的时候在DataCacheGenerator的startNext方法中获取磁盘缓存的文件

```
cacheFile = helper.getDiskCache().get(originalKey);
```

在buildLoadData的时候获取到的是FileLoader直接通过文件解析为数据回调回去

在decodejob的notifyEncodeAndRelease方法中存入

```
diskCacheProvider
    .getDiskCache()
    .put(key, new DataCacheWriter<>(encoder, toEncode, options));
```

一般缓存的是转换后的图片，用的是diskLruCache，google提供了一个工具类DiskLruCache
disklrucache根据一个日志文件journal记录操作记录
journal文件结构分析
dirty  图片的key  
clean 图片的key 图片的大小size
read  图片的key
remove 图片的key
开始写入缓存的时候会生成dirty,调用commmit方法缓存成功之后，会生成clean记录，如果缓存失败了，调用abort生成一条remove记录
每次调用get方法会生成一条read记录
journal文件中的变量redundantOpCount最大值是2000，当达到这个峰值之后，会把多余的、不必要的记录全部清除掉

##lrucache的原理
LruCache通过LinkedHashMap（一个数组加双向链表）来实现，未尾的数据会在达到最大值后被剔除
linkhashmap通过访问顺序来实现lrucache，还有一个插入顺序
构造方法中初始化创建了一个有头节点的双向链表，重写了entry
put方法也是根据key的hash值和长度做一个与运算，生成一个entry存储，但是它重写了createEntry方法，linkhashmap的entry里面是一个双向链表，调用entry的addBefore方法，把新插入的节点的头指向上一个节点，尾指向head节点
直接遍历双向链表,通过节点next双向链表，一直找到尾节点，after是空head节点这个时候就代表找完了