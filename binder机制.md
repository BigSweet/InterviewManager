

进程与进程之间是相互隔离的，每个进程都有自己的用户空间。

他们通过内核空间进行数据交互。



传统ipc通信方式

首先系统调用copy_from_user将数据从用户空间1copy到内核缓存区（第一次copy）

在将内核缓存区的数据通过copy_to_user将数据从内核空间copy到用户空间（第二次copy）

binder通信机制

通过内存映射将进程2的数据和内核空间的一块区域的数据进行映射

当进程1通过copy_from_user将数据拷贝到内核空间的时候，由于内核空间的数据和进程2用户空间的数据存在映射关系，直接就能刷新









binder机制通过aidl通信
android interface define language
安卓接口定义语言

aidl分为proxy，和stub。
客户端获取aidl实例（proxy），proxy用来发送数据,
服务端实现aidl实例(stub),stub用来接收数据
_data  //发送到服务端的数据 
_reply // 服务端返回的数据

client                proxy                binder           stub           service
aidl.method       mremote.transact       ontransact		 this.method  	 触发实现类中的对应方法



![image-20210419140404511](/Users/yanzhe/android/知识整理/image/image-20210419140404511.png)



系统binder通信的流程

进程通信的时候的几种状态

服务端通过addservice将binder实体注册添加到servicemanager中，
客户端获取service的时候，servermanager会在注册的Service列表中查找该服务，返回给client
客户端获取到服务端的binder，转换为服务端的aidl，就能调用服务端相对应的方法了。

系统binder通信流程

![image-20210419161556703](/Users/yanzhe/android/知识整理/image/image-20210419161556703.png)

```
bindService
 @Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        return mBase.bindService(service, conn, flags);
    }
    mBase是context
    contextimpl是context的实现类
    
    

    @Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, Process.myUserHandle());
    }
    
     private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
            UserHandle user) {
             int res = ActivityManagerNative.getDefault().bindService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, getOpPackageName(), user.getIdentifier());
            }
    
    activitymanagernative
    static public IActivityManager getDefault() {
        return gDefault.get();
    }

IActivityManager是一个接口相当于aidlinterface

 private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
    
    public abstract class ActivityManagerNative extends Binder implements IActivityManager
{

    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }
}
    出现了ActivityManagerProxy，aidl发送数据用的
    ActivityManagerNative继承了binder实现了aidl接口，所以ActivityManagerNative是stub
    
    在ActivityManagerProxy中找到bindService如下
    public int bindService(IApplicationThread caller, IBinder token,
            Intent service, String resolvedType, IServiceConnection connection,
            int flags,  String callingPackage, int userId) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeStrongBinder(token);
        service.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(connection.asBinder());
        data.writeInt(flags);
        data.writeString(callingPackage);
        data.writeInt(userId);
        mRemote.transact(BIND_SERVICE_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        data.recycle();
        reply.recycle();
        return res;
    }
    
    通过mremote调用stub中的bing_service
    如下
     case BIND_SERVICE_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            IBinder token = data.readStrongBinder();
            Intent service = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            b = data.readStrongBinder();
            int fl = data.readInt();
            String callingPackage = data.readString();
            int userId = data.readInt();
            IServiceConnection conn = IServiceConnection.Stub.asInterface(b);
            int res = bindService(app, token, service, resolvedType, conn, fl,
                    callingPackage, userId);
            reply.writeNoException();
            reply.writeInt(res);
            return true;
        }
        ???
   接着调用到activitymanagerservice中的bind
    public int bindService(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags, String callingPackage,
            int userId) throws TransactionTooLargeException {
 
        synchronized(this) {
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, callingPackage, userId);
        }
    }
   
   bindServiceLocked方法中主要看下面俩个方法
   bringUpServiceLocked
   requestServiceBindingLocked
        
```



![image-20210419144237469](/Users/yanzhe/android/知识整理/image/image-20210419144237469.png)



首先看第一种和第二种情况


          在bringUpServiceLocked方法中主要如下
    
    	//第二种情况
       if (app != null && app.thread != null) {
                    try {
                        app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
                        realStartServiceLocked(r, app, execInFg);


​            
       realStartServiceLocked方法下面调用这个
       app.thread.scheduleCreateService
       app.thread是activitythread里面的applicationthread
       
       applicationthread里面的方法如下
         public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
    
            sendMessage(H.CREATE_SERVICE, s);
        }
        通过handler发送消息
        
         case CREATE_SERVICE:
                        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceCreate");
                        handleCreateService((CreateServiceData)msg.obj);
                        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                        break;


​                        
     private void handleCreateService(CreateServiceData data) {
            
            unscheduleGcIdler();
    
            LoadedApk packageInfo = getPackageInfoNoCheck(
                    data.info.applicationInfo, data.compatInfo);
            Service service = null;
            try {
                java.lang.ClassLoader cl = packageInfo.getClassLoader();
                service = (Service) cl.loadClass(data.info.name).newInstance();
            } catch (Exception e) {
                if (!mInstrumentation.onException(service, e)) {
                    throw new RuntimeException(
                        "Unable to instantiate service " + data.info.name
                        + ": " + e.toString(), e);
                }
            }
    
            try {
                if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
    
                ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
                context.setOuterContext(service);
    
                Application app = packageInfo.makeApplication(false, mInstrumentation);
                service.attach(context, this, data.info.name, data.token, app,
                        ActivityManagerNative.getDefault());
                service.onCreate();
                mServices.put(data.token, service);
                try {
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                } catch (RemoteException e) {
                    // nothing to do.
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(service, e)) {
                    throw new RuntimeException(
                        "Unable to create service " + data.info.name
                        + ": " + e.toString(), e);
                }
            }
        }
        
        通过反射创建了service
        
        接下来看第一种情况如果app等于null
          if (app == null) {
                if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                        "service", r.name, false, isolated, false)) == null) {
                    String msg = "Unable to launch app "
                            + r.appInfo.packageName + "/"
                            + r.appInfo.uid + " for service "
                            + r.intent.getIntent() + ": process is bad";
                    Slog.w(TAG, msg);
                    bringDownServiceLocked(r);
                    return msg;
                }
                if (isolated) {
                    r.isolatedProc = app;
                }
            }


​            
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
            创建了进程



接下来看requestServiceBindingLocked

```java
 r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.repProcState);
                        
 public final void scheduleBindService(IBinder token, Intent intent,
                boolean rebind, int processState) {
            updateProcessState(processState, false);
            BindServiceData s = new BindServiceData();
            s.token = token;
            s.intent = intent;
            s.rebind = rebind;

            if (DEBUG_SERVICE)
                Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                        + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
            sendMessage(H.BIND_SERVICE, s);
        }
        handler发送BIND_SERVICE
          
          private void handleBindService(BindServiceData data)
      {  
								if (!data.rebind) {
                        IBinder binder = s.onBind(data.intent);
                        ActivityManagerNative.getDefault().publishService(
                                data.token, data.intent, binder);
                    } else {
                        s.onRebind(data.intent);
                        ActivityManagerNative.getDefault().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
        }
      
调用onbind方法
  如果已经绑定了就调用s.onRebind(data.intent); //第四种情况
  publishService调用ams的publishService方法
 最后触发 c.conn.connected(r.name, service);
                        
 final IServiceConnection conn;  // The client connection.

在bindservice的时候可以知道
 	IServiceConnection sd;
        if (conn == null) {
            throw new IllegalArgumentException("connection is null");
        }
        if (mPackageInfo != null) {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),
                    mMainThread.getHandler(), flags);
        } else {
            throw new RuntimeException("Not supported in system context");
        }
IServiceConnection 等于 mPackageInfo.getServiceDispatcher -》sd.getIServiceConnection();
 private final ServiceDispatcher.InnerConnection mIServiceConnection;

 private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }

            public void connected(ComponentName name, IBinder service) throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    sd.connected(name, service);
                }
            }
        }

最后触发的是这个地方的connected方法
  
   public void connected(ComponentName name, IBinder service) {
            if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0));
            } else {
                doConnected(name, service);
            }
        }

public void doConnected(ComponentName name, IBinder service) {
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;
            // If there was an old service, it is not disconnected.
            if (old != null) {
                mConnection.onServiceDisconnected(name);
            }
            // If there is a new service, it is now connected.
            if (service != null) {
                mConnection.onServiceConnected(name, service);
            }
        }

 最后触发了这个connecetion的serviceconncet方法
   
   
    private void bindService() {
        Intent intent = new Intent();
        intent.setComponent(new ComponentName("com.xx.leo_service", "com.xx.leo_service.LeoAidlService"));
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }


    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.e(TAG, "onServiceConnected: success");
            iLeoAidl = ILeoAidl.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.e(TAG, "onServiceDisconnected: success");
            iLeoAidl = null;
        }
    };
所以bindservice触发了serviceconnected方法

```





自定义service binder通信
定义aidl文件夹，定义aidl方法和数据类

```
// ILeoAidl.aidl
package com.xx.leo_service;

// Declare any non-default types here with import statements

import com.xx.leo_service.Person;

interface ILeoAidl {
    void addPerson(in Person person);

    List<Person> getPersonList();
}

```

```
// Person.aidl
package com.xx.leo_service;

// Declare any non-default types here with import statements

parcelable Person;

```



服务端注册service服务，将binder添加到activitymanagerservice中

```
package com.xx.leo_service;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.os.RemoteException;
import android.support.annotation.Nullable;
import android.util.Log;

import java.util.ArrayList;
import java.util.List;

public class LeoAidlService extends Service {

    private ArrayList<Person> persons;

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        persons = new ArrayList<>();
        Log.e("LeoAidlService", "success onBind");
        return iBinder;
    }

    private IBinder iBinder = new ILeoAidl.Stub() {
        @Override
        public void addPerson(Person person) throws RemoteException {
            persons.add(person);
        }

        @Override
        public List<Person> getPersonList() throws RemoteException {
            return persons;
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        Log.e("LeoAidlService", "onCreate: success");
    }
}
```

在activity中启动这个server

客户端bindserver。在ActivityManagerService查找相应的Service组件，
最终通过bindservice时传入的connection，将Service内部Binder的句柄传给Client。在通过binder转为为服务端的aidl在去调用方法

客户端代码

```
package com.xx.leo_client;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.IBinder;
import android.os.RemoteException;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;

import com.xx.leo_service.ILeoAidl;
import com.xx.leo_service.Person;

import java.util.List;

public class MainActivity extends AppCompatActivity {
    private final static String TAG = "MainActivity";

    private ILeoAidl iLeoAidl;

    private Button btn;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initView();
        bindService();
    }

    private void initView() {
        btn = (Button) findViewById(R.id.but_click);
        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                try {
                    iLeoAidl.addPerson(new Person("leo", 3));
                    List<Person> persons = iLeoAidl.getPersonList();
                    Log.e(TAG, persons.toString());
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        });
    }

    private void bindService() {
        Intent intent = new Intent();
        intent.setComponent(new ComponentName("com.xx.leo_service", "com.xx.leo_service.LeoAidlService"));
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }


    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.e(TAG, "onServiceConnected: success");
            iLeoAidl = ILeoAidl.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.e(TAG, "onServiceDisconnected: success");
            iLeoAidl = null;
        }
    };

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(connection);
    }
}
```



binder的memorymap函数，会将内核空间和用户空间对应，用户空间可以直接访问内核空间的数据
 映射原理
 mmap会从当前进程中获取用户态可用的虚拟地址空间
 在内存中分配一块连续的虚拟地址空间，
 把Binder在内核空间的数据直接通过指针地址映射到用户空间，供进程在用户空间使用

 一次拷贝
 binder在目标进程的内核空间直接拷贝，然后目标进程的内核空间和用户空间是有映射的，这就是一次拷贝的原理


