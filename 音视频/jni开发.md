
jni开发
java层调用C++层

首先创建java类的头文件
javah命令行

在创建想对应的.C文件实现头文件中的方法。

将.C文件和.H文件通过ndk打包或者通过cmakelists打包成so包，
java类去加载这个so文件，就可以实现调用C文件中的方法的效果



### java和C层数据传递

c层传递数据给java

```
新建一个jbytearray  bytearray
通过下面的这个方法，将C层的buffer传递给bytearray
env->SetByteArrayRegion(env,array,0.byteSize,buffer)
最后return bytearray
java调用这个方法就可以得到一个byte数组
```

java传递数组给C层

```
java层定义一个方法
void nativeSetArray(byte[] array)
C层实现这个方法
新建一个array数组
unsigned  char CArray[size]
通过getByteArrayRegion(env,buffer,0,len,array)这个方法将array的数据拷贝到CArray中

```

string类型

java传给C

```java
public native String sumStr(String strA,String strB); //java
const char* aString = (*pJNIEnv)->GetStringUTFChars(pJNIEnv, a, 0);  //c
```

c传给java

```
//处理字符串追加  
JNIEXPORT jstring JNICALL Java_cn_itcast_ndk3_DataProvider_sayHelloInC  
  (JNIEnv * env, jobject obj, jstring str){  
    char* newstr = "native string";   
    return (*env)->NewStringUTF(env, newstr);  
}
```



### java和C方法相互调用

c++层调用java层。
静态方法：
获取路径获取jclass 
通过方法签名和方法名获取methodID
最后通过指针调用callStaticVoidMethod(jclass,methodId,newdata)
释放指针

实例方法:
获取路径获取jclass 
通过方法签名和方法名获取methodID
通过jclass <"init">,方法签名获取构造方法的constructmethodId
创建类的实例NewObject(jclass,constructmethodId,null)获取实例
通过指针env->CallVoidMethod(实例,methodID,newdata)
释放指针

静态变量:
获取路径获取jclass 
通过jclass，静态变量名字，方法签名获取filedID
通过指针->SetStaticObjectField(jclass,filedId,“想要修改成的值newdata")
释放指针

成员变量:
获取路径获取jclass 
通过jclass，静态变量名字，方法签名获取filedID
通过jclass <"init">,方法签名获取构造方法的constructmethodId
创建类的实例NewObject(jclass,constructmethodId,null)获取实例
通过指针env->SetObejctFiled(实例,filedID,newdata)
释放指针

### 总结

```
通过FindClass获取jclass
如果是方法就获取GetStaticMethodID
静态方法CallStaticVoidMethod
实例方法 NewObject创建对象的实例在CallVoidMethod
如果是修改成员变量就GetFieldID在NewObject获取实例SetObejctFiled设置
如果是静态变量就不需要获取实例直接SetStaticObjectField
```




当ndk出现崩溃的时候，可以使用ndk名命令反编译logcat查看崩溃信息 so必须是带有符号表的so文件 obj下面的so文件就是带有符号表的so文件
adb logcat | ndk-stack -sym so文件路径

