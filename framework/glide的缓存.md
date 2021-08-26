

## glide的缓存

内存缓存 弱引用+LruCache 

先从弱引用中拿ActiveResources里面有一个activeEngineResources是一个map存储了key和resource

弱引用缓存存储的是正在使用的图片

如果在弱引用没有拿到在去内存缓存中去拿MemoryCache是一个接口默认实现是LruResourceCache

里面是lrucache算法

在onResourceReleased释放资源的时候会存入内存缓存

在图片解码完毕后onEngineJobComplete会存入弱引用缓存

通过变量acquired大于0来表示正在使用的图片



### 示例

如果从内存中拿到了缓存，会将这个缓存从内存中移除，放在弱引用中，同时acquired计数器加1，代表这个图片正在被使用

当图片被释放了之后

这里首先会将缓存图片从activeResources中移除，然后再将它put到LruResourceCache当中。这样也就实现了正在使用中的图片使用弱引用来进行缓存，不在使用中的图片使用LruCache来进行缓存的功能。



## 磁盘缓存

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

## lrucache的原理

LruCache通过LinkedHashMap（一个数组加双向链表）来实现，未尾的数据会在达到最大值后被剔除
linkhashmap通过访问顺序来实现lrucache，还有一个插入顺序

访问顺序为当调用get方法的时候,会将该节点移到链表的尾部,实现了优先淘汰最近最少使用的元素

构造方法中初始化创建了一个有头节点的双向链表，重写了entry
put方法也是根据key的hash值和长度做一个与运算，生成一个entry存储，

但是新写了一个类继承hashmap的node类，linkhashmap的entry里面是一个双向链表

```
  static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
        LinkedHashMapEntry<K,V> before, after;
        LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

linkhashmap的newnode方法为

```
  Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMapEntry<K,V> p =
            new LinkedHashMapEntry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
```

调用entry的addBefore方法，把新插入的节点的头指向上一个节点，尾指向head节点
直接遍历双向链表,通过节点next双向链表，一直找到尾节点，after是空head节点这个时候就代表找完了



glide原理解析csdn链接：https://blog.csdn.net/qq_15527709/article/details/114528807?spm=1001.2014.3001.5501



glide添加进度监听流程

1，注册一个appglidemoudle，将okhttp的loader添加进去，同时给okhttp添加一个拦截器

```
@GlideModule
public class OkHttpLibraryGlideModule extends AppGlideModule {
    @Override
    public void registerComponents(Context context, Glide glide, Registry registry) {
        //添加拦截器到Glide
        OkHttpClient.Builder builder = new OkHttpClient.Builder();
        builder.addInterceptor(new ProgressInterceptor());
        OkHttpClient okHttpClient = builder.build();

        //原来的是  new OkHttpUrlLoader.Factory()；
        registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory(okHttpClient));
    }

    //完全禁用清单解析
    @Override
    public boolean isManifestParsingEnabled() {
        return false;
    }
}
```

2,拦截器拦截response

```
 @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Response response = chain.proceed(request);
        String url = request.url().toString();
        ResponseBody body = response.body();
        Response newResponse = response.newBuilder().body(new ProgressResponseBody(url, body)).build();
        return newResponse;
    }
```

3,在source里面读取进度，并且回调出去

```
private class ProgressSource extends ForwardingSource {

        long totalBytesRead = 0;

        int currentProgress;

        ProgressSource(Source source) {
            super(source);
        }

        @Override
        public long read(Buffer sink, long byteCount) throws IOException {
            long bytesRead = super.read(sink, byteCount);
            long fullLength = responseBody.contentLength();
            if (bytesRead == -1) {
                totalBytesRead = fullLength;
            } else {
                totalBytesRead += bytesRead;
            }
            int progress = (int) (100f * totalBytesRead / fullLength);
            Log.d(TAG, "download progress is " + progress);
            if (listener != null && progress != currentProgress) {
                listener.onProgress(progress);
            }
            if (listener != null && totalBytesRead == fullLength) {
                listener = null;
            }

            currentProgress = progress;
            return bytesRead;
        }
    }
```

