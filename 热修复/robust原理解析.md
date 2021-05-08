robust原理解析

打基础包的时候，每个类都会被添加

public static ChangeQuickRedirect u;静态成员变量 

查看这个类的源码发现这个是一个接口

```
public interface ChangeQuickRedirect {
    Object accessDispatch(String methodName, Object[] paramArrayOfObject);

    boolean isSupport(String methodName, Object[] paramArrayOfObject);
}
```

反编译基本包能看到每个方法当中都被添加了这个判断

![image-20210427152138942](/Users/yanzhe/android/知识整理/image/image-20210427152138942.png)



这个类的代码为

```
 public static PatchProxyResult proxy(Object[
 
 ] paramsArray, Object current, ChangeQuickRedirect changeQuickRedirect, boolean isStatic, int methodNumber, Class[] paramsClassTypes, Class returnType) {
        PatchProxyResult patchProxyResult = new PatchProxyResult();
        if (PatchProxy.isSupport(paramsArray, current, changeQuickRedirect, isStatic, methodNumber, paramsClassTypes, returnType)) {
            patchProxyResult.isSupported = true;
            patchProxyResult.result = PatchProxy.accessDispatch(paramsArray, current, changeQuickRedirect, isStatic, methodNumber, paramsClassTypes, returnType);
        }
        return patchProxyResult;
    }

    public static boolean isSupport(Object[] paramsArray, Object current, ChangeQuickRedirect changeQuickRedirect, boolean isStatic, int methodNumber, Class[] paramsClassTypes, Class returnType) {
        //Robust补丁优先执行，其他功能靠后
        if (changeQuickRedirect == null) {
            //不执行补丁，轮询其他监听者
            if (registerExtensionList == null || registerExtensionList.isEmpty()) {
                return false;
            }
            for (RobustExtension robustExtension : registerExtensionList) {
                if (robustExtension.isSupport(new RobustArguments(paramsArray, current, isStatic, methodNumber, paramsClassTypes, returnType))) {
                    robustExtensionThreadLocal.set(robustExtension);
                    return true;
                }
            }
            return false;
        }
        String classMethod = getClassMethod(isStatic, methodNumber);
        if (TextUtils.isEmpty(classMethod)) {
            return false;
        }
        Object[] objects = getObjects(paramsArray, current, isStatic);
        try {
            return changeQuickRedirect.isSupport(classMethod, objects);
        } catch (Throwable t) {
            return false;
        }
    }
 
```

如果PatchProxy.isSupport返回true

就执行PatchProxy.accessDispatch

查看源码

```
public static Object accessDispatch(Object[] paramsArray, Object current, ChangeQuickRedirect changeQuickRedirect, boolean isStatic, int methodNumber, Class[] paramsClassTypes, Class returnType) {

        if (changeQuickRedirect == null) {
            RobustExtension robustExtension = robustExtensionThreadLocal.get();
            robustExtensionThreadLocal.remove();
            if (robustExtension != null) {
                notify(robustExtension.describeSelfFunction());
                return robustExtension.accessDispatch(new RobustArguments(paramsArray, current, isStatic, methodNumber, paramsClassTypes, returnType));
            }
            return null;
        }
        String classMethod = getClassMethod(isStatic, methodNumber);
        if (TextUtils.isEmpty(classMethod)) {
            return null;
        }
        notify(Constants.PATCH_EXECUTE);
        Object[] objects = getObjects(paramsArray, current, isStatic);
        return changeQuickRedirect.accessDispatch(classMethod, objects);
    }
```

最后调用的就是changeQuickRedirect的accessDispatch

那实际调用的就是这个接口的实现类的方法

接下来看补丁文件

PatchesInfoImpl

根据我的上篇使用文章，我反编译出patchesinfoimpl的代码为

```
public class PatchesInfoImpl implements PatchesInfo {
  public List getPatchedClassesInfo() {
    ArrayList<PatchedClassInfo> arrayList = new ArrayList();
    arrayList.add(new PatchedClassInfo("com.freebrio.robustdemo.MainActivity", "com.freebrio.robustdemo.MainActivityPatchControl"));
    EnhancedRobustUtils.isThrowable = false;
    return arrayList;
  }
}
```

里面列出了我修改的类和对应的修改方法

MainActivity ->MainActivityPatchControl

打开这个类

```
package com.freebrio.robustdemo;

import android.view.View;
import com.meituan.robust.ChangeQuickRedirect;
import java.util.Map;
import java.util.WeakHashMap;

public class MainActivityPatchControl implements ChangeQuickRedirect {
  public static final String MATCH_ALL_PARAMETER = "(\\w*\\.)*\\w*";
  
  private static final Map<Object, Object> keyToValueRelation = new WeakHashMap<Object, Object>();
  
  private static Object fixObj(Object paramObject) {
    Object object = paramObject;
    if (paramObject instanceof Byte) {
      boolean bool;
      if (((Byte)paramObject).byteValue() != 0) {
        bool = true;
      } else {
        bool = false;
      } 
      object = new Boolean(bool);
    } 
    return object;
  }
  
  public Object accessDispatch(String paramString, Object[] paramArrayOfObject) {
    try {
      MainActivityPatch mainActivityPatch;
      if (paramString.split(":")[2].equals("false")) {
        if (keyToValueRelation.get(paramArrayOfObject[paramArrayOfObject.length - 1]) == null) {
          mainActivityPatch = new MainActivityPatch(paramArrayOfObject[paramArrayOfObject.length - 1]);
          keyToValueRelation.put(paramArrayOfObject[paramArrayOfObject.length - 1], null);
        } else {
          mainActivityPatch = (MainActivityPatch)keyToValueRelation.get(paramArrayOfObject[paramArrayOfObject.length - 1]);
        } 
      } else {
        mainActivityPatch = new MainActivityPatch(null);
      } 
      if ("5".equals(paramString.split(":")[3])) {
        mainActivityPatch.setTV((View)paramArrayOfObject[0]);
        return null;
      } 
    } catch (Throwable throwable) {
      throwable.printStackTrace();
      return null;
    } 
    return null;
  }
  
  public Object getRealParameter(Object paramObject) {
    Object object = paramObject;
    if (paramObject instanceof MainActivity)
      object = new MainActivityPatch(paramObject); 
    return object;
  }
  
  public boolean isSupport(String paramString, Object[] paramArrayOfObject) {
    paramString = paramString.split(":")[3];
    return ":5:".contains(":" + paramString + ":");
  }
}

```

他实现的就是ChangeQuickRedirect

也就是说如果ChangeQuickRedirect不等于null那就会执行MainActivityPatchControl中的accessDispatch

可以看到这一行关键代码

```
if ("5".equals(paramString.split(":")[3])) {
        mainActivityPatch.setTV((View)paramArrayOfObject[0]);
        return null;
      } 
```

在mainactivitypatch的settv方法中

```
public final void setTV(View paramView) {
    MainActivityPatch mainActivityPatch;
    MainActivity mainActivity1;
    CharSequence charSequence;
    MainActivity mainActivity2;
    new IntrinsicsInLinePatch(getRealParameter(new Object[] { null })[0]);
    EnhancedRobustUtils.invokeReflectStaticMethod("checkParameterIsNotNull", IntrinsicsInLinePatch.class, getRealParameter(new Object[] { paramView, "view" }, ), new Class[] { Object.class, String.class });
    if (this instanceof MainActivityPatch) {
      mainActivity1 = this.originClass;
    } else {
      mainActivityPatch = this;
    } 
    TextView textView2 = (TextView)EnhancedRobustUtils.getFieldValue("q", mainActivityPatch, MainActivity.class);
    if (textView2 == null) {
      new IntrinsicsInLinePatch(getRealParameter(new Object[] { null })[0]);
      EnhancedRobustUtils.invokeReflectStaticMethod("throwUninitializedPropertyAccessException", IntrinsicsInLinePatch.class, getRealParameter(new Object[] { "textView" }, ), new Class[] { String.class });
    } 
    String str = (String)EnhancedRobustUtils.invokeReflectMethod("getString1", new MainActivityInLinePatch(getRealParameter(new Object[] { this }, )[0]), getRealParameter(new Object[0]), null, null);
    if (str == this) {
      mainActivity1 = ((MainActivityPatch)str).originClass;
    } else {
      charSequence = (CharSequence)mainActivity1;
    } 
    TextView textView1 = textView2;
    if (textView2 == this)
      mainActivity2 = ((MainActivityPatch)textView2).originClass; 
    EnhancedRobustUtils.invokeReflectMethod("setText", mainActivity2, getRealParameter(new Object[] { charSequence }, ), new Class[] { CharSequence.class }, TextView.class);
  }
```



可以看到这一行关键代码

```
String str = (String)EnhancedRobustUtils.invokeReflectMethod("getString1", new MainActivityInLinePatch(getRealParameter(new Object[] { this }, )[0]), getRealParameter(new Object[0]), null, null);
```

最后调用的是我修改后的getString1方法，达到了修复的目的

![image-20210427145541550](/Users/yanzhe/android/知识整理/image/image-20210427145541550.png)

接下来看一下加载过程

```
new PatchExecutor(getApplicationContext(), new PatchManipulateImp(),  new Callback()).start();
```

在PatchExecutor的patch方法中

```

ClassLoader classLoader = null;

        try {
            File dexOutputDir = getPatchCacheDirPath(context, patch.getName() + patch.getMd5());
            classLoader = new DexClassLoader(patch.getTempPath(), dexOutputDir.getAbsolutePath(),
                    null, PatchExecutor.class.getClassLoader());
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        if (null == classLoader) {
            return false;
        }
        
        
try {
            Log.d("robust", "patch patch_info_name:" + patch.getPatchesInfoImplClassFullName());
            patchesInfoClass = classLoader.loadClass(patch.getPatchesInfoImplClassFullName());
            patchesInfo = (PatchesInfo) patchesInfoClass.newInstance();
        } catch (Throwable t) {
            Log.e("robust", "patch failed 188 ", t);
        }
        
        
         Field[] fields = sourceClass.getDeclaredFields();
                Log.d("robust", "oldClass :" + sourceClass + "     fields " + fields.length);
                Field changeQuickRedirectField = null;
                for (Field field : fields) {
                    if (TextUtils.equals(field.getType().getCanonicalName(), ChangeQuickRedirect.class.getCanonicalName()) && TextUtils.equals(field.getDeclaringClass().getCanonicalName(), sourceClass.getCanonicalName())) {
                        changeQuickRedirectField = field;
                        break;
                    }
                }
```

创建了DexClassLoader，根据dexclassloader去加载dex文件，

```
classLoader.loadClass(patch.getPatchesInfoImplClassFullName())
```

getPatchesInfoImplClassFullName就是PatchesInfoImpl类，里面记录了修改的类的信息

加载完类之后

changeQuickRedirectField = field;将这个静态变量赋值接下来就走上面的步骤了



插件代码

https://github.com/Meituan-Dianping/Robust/tree/master/gradle-plugin

用AsmInsertImpl和JavaAssistInsertImpl

插桩核心代码

createInsertCode

位于https://github.com/Meituan-Dianping/Robust/blob/master/gradle-plugin/src/main/groovy/robust/gradle/plugin/asm/RobustAsmUtils.java

类中

