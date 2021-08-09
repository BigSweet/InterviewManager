准备好java文件，res文件和一个manifest，和一个空的build文件夹 

1,使用aapt2对资源进行编译生成二进制压缩包zip包（.flat后缀）

```
命令 aapt2 compile -o build/res.zip --dir res
```

2,使⽤ aapt2 ⼯具将资源整合生成一个apk文件和R.java文件(生成成功后将R.java文件放到java代码文件下面)

```
aapt2 link build/res.zip -I $ANDROID_HOME/platforms/android-
29/android.jar -- java build --manifest AndroidManifest.xml -o
build/app-debug.apk
```

3,使⽤ javac ⼯具编译 Java ⽂件生成class 字节码⽂件

```
javac -d build -cp $ANDROID_HOME/platforms/android-
28/android.jar com/*/.java
```

4,使⽤ dex ⼯具编译 class 代码生成dex文件

```
d8 --output build/ --lib $ANDROID_HOME/platforms/android-
28/android.jar build/com/example/application/*.class
```

5,使⽤ zip 命令合并代码⽂件和资源⽂件生成未签名的apk文件

```
zip -j build/app-debug.apk build/classes.dex
```

6,使⽤ apksigner ⼯具对 apk 签名,生成可以安装的签名的apk

```
apksigner sign -ks ~/.android/debug.keystore build/appdebug.
apk
```

