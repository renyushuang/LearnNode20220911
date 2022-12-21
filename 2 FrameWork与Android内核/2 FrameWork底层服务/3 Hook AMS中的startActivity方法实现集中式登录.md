Hook点

## Hook住启动的部分

```java
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}

private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                return am;
            }
        };
```

拿到系统的对象

```java
        Class  ActivityManagerClz = Class.forName("android.app.ActivityManager");
        Field IActivityManagerSingletonFiled = ActivityManagerClz.getDeclaredField("IActivityManagerSingleton");
        IActivityManagerSingletonFiled.setAccessible(true);
        Object IActivityManagerSingletonObj = IActivityManagerSingletonFiled.get(null);
```

反射返回值Singleton

```java
public abstract class Singleton<T> {
    private T mInstance;

    protected abstract T create();

    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}
```

```java
 Class SingletonClz = Class.forName("android.util.Singleton");
//        反射代码  mInstance
        Field mInstanceField = SingletonClz.getDeclaredField("mInstance");
        mInstanceField.setAccessible(true);
        Object IActivityManagerObj=  mInstanceField.get(IActivityManagerSingletonObj);
        IActivityManagerObj.hashCode();
```

动态代理hook住startActiviity

```java
//        动态代理
        Class IActivityManagerClz = Class.forName("android.app.IActivityManager");
        Object proxyIActivityManager =Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                new Class[]{IActivityManagerClz}, new AmsInvocationHandler(IActivityManagerObj));
        mInstanceField.setAccessible(true);
        mInstanceField.set(IActivityManagerSingletonObj,proxyIActivityManager);
```



替换掉要启动的Activity

```java
    class AmsInvocationHandler implements InvocationHandler {
        private Object iActivityManagerObject;
        public AmsInvocationHandler(Object iActivityManagerObject) {
            this.iActivityManagerObject = iActivityManagerObject;
        }
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {


            if ("startActivity".contains(method.getName())) {

                Intent intent = null;
                int index = 0;
                for (int i = 0; i < args.length; i++) {
                    Object arg = args[i];
                    if (arg instanceof Intent) {
                        intent = (Intent) args[i]; // 原意图，过不了安检
                        index = i;
                        break;
                    }
                }
                Intent proxyIntent = new Intent();
                ComponentName componentName = new ComponentName(context, proxyActivity);
                proxyIntent.setComponent(componentName);
//                rxjava   ---> 架构师  解耦观念 隐藏到里面
                proxyIntent.putExtra("oldIntent", intent);
                args[index] = proxyIntent;
            }
            Log.i("david", "invoke: "+method.getName());
            return method.invoke(iActivityManagerObject, args);
        }
    }
```

## Hook ActivityThread mhandler

```java
    public void hookSystemHandler() throws  Exception {

        Class ActivityThreadClz = Class.forName("android.app.ActivityThread");
        Field field = ActivityThreadClz.getDeclaredField("sCurrentActivityThread");
        field.setAccessible(true);
        Object ActivityThreadObj =field.get(null);
        ActivityThreadObj.hashCode();

        Field mHField = ActivityThreadClz.getDeclaredField("mH");
        mHField.setAccessible(true);
        Handler mHObj = (Handler) mHField.get(ActivityThreadObj);//ok，当前的mH拿到了

        Field mCallbackField = Handler.class.getDeclaredField("mCallback");
        mCallbackField.setAccessible(true);
        ProxyHandlerCallback proxyMHCallback = new ProxyHandlerCallback();
        mCallbackField.set(mHObj, proxyMHCallback);

    }
    private class ProxyHandlerCallback implements Handler.Callback {
        private int EXECUTE_TRANSACTION = 159;
        @Override
        public boolean handleMessage(Message msg) {

            Log.i("david", "handleMessage: "+msg.what);

            if (msg.what == 159) {

                Log.i("david", "---->: "+msg.obj.getClass().toString());
                try {
                    Class ClientTransactionClz = Class.forName("android.app.servertransaction.ClientTransaction");
                    if (!ClientTransactionClz.isInstance(msg.obj)) return false;

                    Class LaunchActivityItemClz = Class.forName("android.app.servertransaction.LaunchActivityItem");

                    Field mActivityCallbacksField = ClientTransactionClz.getDeclaredField("mActivityCallbacks");//ClientTransaction的成员
//设值可访问
                    mActivityCallbacksField.setAccessible(true);
                    Object mActivityCallbacksObj = mActivityCallbacksField.get(msg.obj);
                    List list = (List) mActivityCallbacksObj;
                    if (list.size() == 0) return false;
                    Object LaunchActivityItemObj = list.get(0);
                    if (!LaunchActivityItemClz.isInstance(LaunchActivityItemObj)) return false;

//                    startActivity  一定是
                    Field mIntentField = LaunchActivityItemClz.getDeclaredField("mIntent");
                    mIntentField.setAccessible(true);
                    Intent mIntent = (Intent) mIntentField.get(LaunchActivityItemObj);
                    Intent realIntent = mIntent.getParcelableExtra("oldIntent");
                    if (realIntent != null) {
//                        SecondActivity
//                        登录判断
                        SharedPreferences share = context.getSharedPreferences("david",
                                Context.MODE_PRIVATE);
                        if (share.getBoolean("login", false)) {
                            mIntent.setComponent(realIntent.getComponent());
                        }else {
                            ComponentName componentName = new ComponentName(context, LoginActivity.class);
                            mIntent.putExtra("extraIntent", realIntent.getComponent()
                                    .getClassName());
                            mIntent.setComponent(componentName);
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }


            }
            return false;
        }
    }
```