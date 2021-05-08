美团热修复robust使用

最近在整理各个热修复的使用方法和原理，第一个研究的是美团的robust
github地址为[robust](https://github.com/Meituan-Dianping/Robust)
基本的使用方法其实github上面有，但是我发现我集成花费了一天的时间，还有是有很多坑的所以才写了这篇文章记录一下

首先是APP目录的build里面增加我红色箭头标记的

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426102638387.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
带箭头的就是新增的
出了增加美团的热修复包外，还增加了权限库和分包库multidex
```
apply plugin: 'com.android.application'
//制作补丁时将这个打开，auto-patch-plugin紧跟着com.android.application
//apply plugin: 'auto-patch-plugin'
apply plugin: 'robust'
	
	
implementation 'com.meituan.robust:robust:0.4.99'
    implementation 'com.github.tbruyelle:rxpermissions:0.12'
    implementation 'io.reactivex.rxjava3:rxandroid:3.0.0'
    implementation 'io.reactivex.rxjava3:rxjava:3.0.5'
    implementation 'com.android.support:multidex:1.0.3'
```
接下来是配置根目录的build
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426102811402.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
## 注意如果是用kotlin开发的话，这里的gradle版本和kotlin版本不能太高，否则会报错
```
第一个错误
java.lang.RuntimeException: java.io.IOException: invalid constant type: 19 at 5
第二个错误
Caused by: org.codehaus.groovy.runtime.typehandling.GroovyCastException: Cannot cast object 'task ':app:packageNjfBetaRelease' property 'resourceFiles'' with class 'org.gradle.api.internal.file.DefaultFilePropertyFactory$DefaultDirectoryVar' to class 'java.io.File'
```


在github的issue里面有解决办法
[issue439](https://github.com/Meituan-Dianping/Robust/issues/439)
   另外一个解决办法就是降低你的gradle版本
   这里是第一个容易出问题的地方

继续看APP的build里面
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426103230313.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
分包记得打开
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426103248408.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
配置好签名，到时候测试的时候最好是打release包带签名的
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426103315369.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
配置签名，打开混淆
混淆文件中记得添加
```
-keepclassmembers class **{
public static com.meituan.robust.ChangeQuickRedirect *;
}
```
接下来在APP的目录下面新建robust.xml文件
里面的具体配置直接copy github里面的
[robust.xml](https://github.com/Meituan-Dianping/Robust/blob/master/app/robust.xml)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426103519516.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
## 注意上面几个地方修改成你自己的

接下来manifest记得配置权限
```
  <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

接下来开始写测试布局
mainActivity
```
lateinit var textView: TextView

    override
    fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        textView = findViewById(R.id.content_tv)

        val file =
            File(externalCacheDir?.absolutePath + File.separator + "robust" + File.separator + "patch ")
        if (!file.exists()) {
            file.mkdirs()
        }
        RxPermissions(this)
            .request(
                Manifest.permission.READ_EXTERNAL_STORAGE,
                Manifest.permission.WRITE_EXTERNAL_STORAGE
            )
            .subscribe { granted ->
            }


    }

    fun getString(): String {
        return "111"
    }

    fun loadPatch(view: View) {
        PatchExecutor(this, PatchManipulateImp(), object : RobustCallBack {
            override fun onPatchListFetched(
                result: Boolean,
                isNet: Boolean,
                patches: MutableList<Patch>?
            ) {
            }

            override fun onPatchFetched(result: Boolean, isNet: Boolean, patch: Patch?) {
                Log.d("swt", "onPatchFetched")
            }

            override fun onPatchApplied(result: Boolean, patch: Patch?) {
                Log.d("swt", "onPatchApplied")

            }

            override fun logNotify(log: String?, where: String?) {
                Log.d("swt", "logNotify")

            }

            override fun exceptionNotify(throwable: Throwable?, where: String?) {
                Log.d("swt", "exceptionNotify")

            }

        }).start()
    }
    
    fun setTV(view: View) {
        textView.text = getString()
    }

```
loadPatch里面有一个PatchExecutor最后记得start
主要看下第二个参数
PatchManipulateImp
```
class PatchManipulateImp : PatchManipulate() {
    override fun fetchPatchList(context: Context): MutableList<Patch> {
        val patch = Patch()
        patch.name = "test"
        patch.tempPath =
            context.externalCacheDir?.absolutePath + File.separator + "robust" + File.separator + "patch"
        patch.localPath =
            context.externalCacheDir?.absolutePath + File.separator + "robust" + File.separator + "patch"
        patch.patchesInfoImplClassFullName = "com.freebrio.robustdemo.PatchesInfoImpl"
        val list = arrayListOf<Patch>()
        list.add(patch)
        return list
    }

    override fun verifyPatch(context: Context?, patch: Patch?): Boolean {
        return true
    }

    override fun ensurePatchExist(patch: Patch?): Boolean {
        return true
    }
}
```
## 这里的temppath和localpath最好都设置一下吧，我没测试localpath不设置会不会有问题了，就都写了，反正temppath要设置的
## 然后下面俩个方法都返回true，后面导入手机的patch文件名字要改成patch_temp.jar
## 划重点
 patch.patchesInfoImplClassFullName = "com.freebrio.robustdemo.PatchesInfoImpl"
 这个包名和robust.xml里面patchPackname的配置要一样
 然后最后的PatchesInfoImpl一定要写死
 这里我们的类名是PatchManipulateImp但是这个地方一定要写死PatchesInfoImpl
 不然会到时候抱错
 ```
 PatchsInfoImpl failed,cause of java.lang.ClassNotFoundException:
 ```
到这里基本的配置就弄完了，开始打release包
```
./gradlew assembleRelease
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426104853185.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
打到成功后如果能出现下面几个文件那就说明前面配置的没什么大问题了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426105415861.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
接下来在APP的目录下面新建robust文件夹把那俩个文件放进去
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426105504496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
把APK安装到手机里面，这个APK就是我们未修复的版本
接下来我们修改mainActivity
在修改的地方添加@modify注释
同事新增一个方法，点击按钮的时候，调用getstring1方法返回222
```

    @Modify
    fun setTV(view: View) {
        textView.text = getString1()
    }

    @Add
    fun getString1(): String {
        return "222"
    }
```
然后把APP的build文件中的
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426105735604.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
这个插件打开
就下来继续执行
 ./gradlew assembleRelease
打包会失败但是没关系出现了下面的这个就可以了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426105844842.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
然后再build目录下面会看到这个文件
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021042610590968.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
将这个文件push到我们的手机中
```
 adb push /Users/yanzhe/android/hotfixAll/robust/app/build/outputs/robust/patch.jar /storage/emulated/0/Android/data/com.freebrio.robustdemo/cache/robust/patch_temp.jar

```
这是我的路径你们修改成自己的路径就可以了
记得push的jar包名字为patch_temp.jar
不要问我为什么
接下来打开我们未修复的APP，点击按钮
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426110511711.gif)
我们第一下点击的时候textview是111，加载补丁之后，在点击就变成了222
同时log里面会有这个提示
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426110624731.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)
就证明这次补丁加载成功了
我的代码地址为
[github](https://github.com/BigSweet/hotFixALL/tree/master/robust)