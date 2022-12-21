| ActivityThread源码分析                      |
| ------------------------------------------- |
| AMS与ActivityThread通信原理                 |
| Activity启动机制                            |
| Hook AMS中的startActivity方法实现集中式登录 |

假设没有AMS、PMS要启动一个Activity要经历哪些步骤

1.遍历data/app目录

2.解压APK资源

3.dom解析Android.xml

4.定位到主Activity

5.class加载到内存

6.实例化Activity



PMS把123给做了，AMS把456给做了



### 分析PMS是如何解析包的

```java
private AndroidPackage scanPackageLI(File scanFile, int parseFlags, int scanFlags,
        long currentTime, UserHandle user) throws PackageManagerException {
    if (DEBUG_INSTALL) Slog.d(TAG, "Parsing: " + scanFile);

    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
    final ParsedPackage parsedPackage;
    try (PackageParser2 pp = new PackageParser2(mSeparateProcesses, mOnlyCore, mMetrics, null,
            mPackageParserCallback)) {
       	//进行解析
        parsedPackage = pp.parsePackage(scanFile, parseFlags, false);
    } catch (PackageParserException e) {
        throw PackageManagerException.from(e);
    } finally {
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }

    // Static shared libraries have synthetic package names
    if (parsedPackage.isStaticSharedLibrary()) {
        renameStaticSharedLibraryPackage(parsedPackage);
    }

    return addForInitLI(parsedPackage, parseFlags, scanFlags, currentTime, user);
}
```

pp.parsePackage开始解析apk文件，解析完成后，放入到Package的缓存中

```java
public final static class Package implements Parcelable {
    @UnsupportedAppUsage
    public final ArrayList<Permission> permissions = new ArrayList<Permission>(0);
    @UnsupportedAppUsage
    public final ArrayList<PermissionGroup> permissionGroups = new ArrayList<PermissionGroup>(0);
    @UnsupportedAppUsage
    public final ArrayList<Activity> activities = new ArrayList<Activity>(0);
    @UnsupportedAppUsage
    public final ArrayList<Activity> receivers = new ArrayList<Activity>(0);
    @UnsupportedAppUsage
    public final ArrayList<Provider> providers = new ArrayList<Provider>(0);
    @UnsupportedAppUsage
    public final ArrayList<Service> services = new ArrayList<Service>(0);
```

先解析Android XML 

```java
private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
        throws PackageParserException {
    try {
      	// 打开androidMiniFest
        parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);
        final Resources res = new Resources(assets, mMetrics, null);

        final String[] outError = new String[1];
        final Package pkg = parseBaseApk(apkPath, res, parser, flags, outError);
        }
```

### Actvity启动流程

三步

Actvity到AMS

AMS到PMS

AMS到ApplicationThread

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}
```

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,s
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
```

```java
int result = ActivityTaskManager.getService().startActivity(whoThread,
        who.getBasePackageName(), who.getAttributionTag(), intent,
        intent.resolveTypeIfNeeded(who.getContentResolver()), token,
        target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
```

这里分为两部分ActivityTaskManager.getService()和startActivity。先获取binder

```java
public static IActivityTaskManager getService() {
    return IActivityTaskManagerSingleton.get();
}

@UnsupportedAppUsage(trackingBug = 129726065)
private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
        new Singleton<IActivityTaskManager>() {
            @Override
            protected IActivityTaskManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                return IActivityTaskManager.Stub.asInterface(b);
            }
        };
```

startActivity调用就会调用到服务端ams，通过PMS获取activity的信息

```java
void resolveActivity(ActivityStackSupervisor supervisor) {
    resolveInfo = supervisor.resolveIntent(intent, resolvedType, userId,
            0 /* matchFlags */,
            computeResolveFilterUid(callingUid, realCallingUid, filterCallingUid));
```

查找进程、查找栈，继续进行启动

```java
mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
        request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
        restrictedBgActivity, intentGrants);

// 启动模式
 computeLaunchingTaskFlags();
```

插入栈顶后调用

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    if (mInResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }

    boolean result = false;
    try {
        // Protect against recursion.
        mInResumeTopActivity = true;
        result = resumeTopActivityInnerLocked(prev, options);
        final ActivityRecord next = topRunningActivity(true /* focusableOnly */);
        if (next == null || !next.canTurnScreenOn()) {
            checkReadyForSleep();
        }
    } finally {
        mInResumeTopActivity = false;
    }

    return result;
}
```

```java
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);

    boolean knownToBeDead = false;
    if (wpc != null && wpc.hasThread()) {
        try {
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
        knownToBeDead = true;
    }

    r.notifyUnknownVisibilityLaunchedForKeyguardTransition();

    final boolean isTop = andResume && r.isTopRunningActivity();
    mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
}
```

## ActivityThread

```
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    if (r.profilerInfo != null) {
        mProfiler.setProfiler(r.profilerInfo);
        mProfiler.startProfiling();
```

```java
activity = mInstrumentation.newActivity(
        cl, component.getClassName(), r.intent);
```

通过反射创建activity