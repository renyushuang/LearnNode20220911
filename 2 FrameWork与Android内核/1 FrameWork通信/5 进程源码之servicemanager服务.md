| 服务启动  service_manager服务    |
| -------------------------------- |
| 服务发现(sevicemanager服务查找)  |
| 服务调度(serivcemanager消息分发) |
| copy_form_user/copy_to_user函数  |
| binder线程池的管理               |

```java
private static class Proxy implements com.example.ipclib.DavidBinderInterface{
//发送数据的
  private android.os.IBinder mRemote; //== BinderPoxy
```



```java
public static abstract class Stub extends android.os.Binder implements com.example.ipclib.DavidBinderInterface
{
  // 接收数据的
  @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
  {
```

以AMS为例

```java
public final class SystemServer {
	public void run(){
    ...
    ActivityTaskManagerService atm = mSystemServiceManager.startService(
            ActivityTaskManagerService.Lifecycle.class).getService();
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
            mSystemServiceManager, atm);
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);


    ...//进行组件的注册了
    mActivityManagerService.setSystemProcess();
    ...
  }
}
```

服务注册：

```java
public void setSystemProcess() {
    try {
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                DUMP_FLAG_PRIORITY_HIGH);
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        ServiceManager.addService("dbinfo", new DbBinder(this));
        if (MONITOR_CPU_USAGE) {
            ServiceManager.addService("cpuinfo", new CpuBinder(this),
                    /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
        }
        ServiceManager.addService("permission", new PermissionController(this));
        ServiceManager.addService("processinfo", new ProcessInfoService(this));
        ServiceManager.addService("cacheinfo", new CacheBinder(this));
```

ServiceManager.addService


```java
public static void addService(String name, IBinder service) {
    try {
        getIServiceManager().addService(name, service, false);
    } catch (RemoteException e) {
        Log.e(TAG, "error in addService", e);
    }
}
```

getIServiceManager()

```java
private static IServiceManager getIServiceManager() {
  if (sServiceManager != null) {
    return sServiceManager;
  }

  // Find the service manager
  sServiceManager = ServiceManagerNative
    .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
  return sServiceManager;
}
```

该处有三个部分

```c++
java >>> BinderInternal.getContextObject()
	c++ >>> ProcessState::getContextObject
		c++	>>>	 b = new BpBinder(handle); 主要是为了实例化BpBinder，BpBinder是为了发送数据的
			c++ >>>javaObjectForIBinder 与Porx进行关联了 返回BinderPoxy
			
			
Binder.allowBlocking(BinderInternal.getContextObject())


ServiceManagerNative
    .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
	java >>> new ServiceManagerProxy(obj) 为mRomte赋值
    
```

发送消息，获取Service

```java
public IBinder getService(String name) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IServiceManager.descriptor);
    data.writeString(name);
    mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
    IBinder binder = reply.readStrongBinder();
    reply.recycle();
    data.recycle();
    return binder;
}
```



```java
public final class BinderProxy implements IBinder {
public native boolean transactNative(int code, Parcel data, Parcel reply,
        int flags) throws RemoteException;
```



```c++
writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
```

binder驱动



