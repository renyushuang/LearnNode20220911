| init进程与Zygote进程              |
| --------------------------------- |
| Zygote启动流程之 APP进程的启动    |
| Zygote的Socket通信模式            |
| APP启动过程分析之 APP进程的启动   |
| APP启动过程分析之 APP主线程的启动 |

1 init进程作用是什么

2 小米 华为 这种系统服务怎么做

3 Zygote进程最原始的进程是什么进程

​    (或者Zygote进程由来)

4 Zygote为什么需要用到Socket通信而不是Binder

5 每个App都会将系统的资源 ，系统的类都加载一遍吗

6 PMS 是干什么的，你是怎么理解PMS

7  为什么会有AMS AMS的作用

8  为什么一个Activity需要声明在AndroidManifest.xml中

 9 AMS如何管理Activity，探探AMS的执行原理



###  init进程作用是什么

手机启动

fastboost——》Linux kernal——》init——〉zygote

init进程目的：启动关键服务、守护关键服务

/system/core/init/init.cpp 加载解析init.rc

```c++
587int main(int argc, char** argv) {
684    Parser& parser = Parser::GetInstance();
685    parser.AddSectionParser("service",std::make_unique<ServiceParser>());
686    parser.AddSectionParser("on", std::make_unique<ActionParser>());
687    parser.AddSectionParser("import", std::make_unique<ImportParser>());
688    parser.ParseConfig("/init.rc");
749}
```

/system/core/rootdir/init.rc 就是一个服务的配置文件，包含了zygote的配置文件

```
# Copyright (C) 2012 The Android Open Source Project
2#
3# IMPORTANT: Do not create world writable files or directories.
4# This is a common source of Android security bugs.
5#
6
7import /init.environ.rc
8import /init.usb.rc
9import /init.${ro.hardware}.rc
10import /init.usb.configfs.rc
11import /init.${ro.zygote}.rc
```

/system/core/rootdir/init.zygote64.rc zygote配置文件

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
2    class main
3    socket zygote stream 660 root system
4    onrestart write /sys/android_power/request_state wake
5    onrestart write /sys/power/state on
6    onrestart restart audioserver
7    onrestart restart cameraserver
8    onrestart restart media
9    onrestart restart netd
10    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks
```

### 2 小米 华为 这种系统服务怎么做

修改init.rc增加自己的服务

### 3 Zygote进程最原始的进程是什么进程

​    (或者Zygote进程由来)

/frameworks/base/cmds/app_process/

```c
int main(int argc, char* const argv[])
188{	
  		
272    while (i < argc) {
273        const char* arg = argv[i++];
274        if (strcmp(arg, "--zygote") == 0) {
  						 // 解析参数，如果包含--zygote 则将app_process名字修改为ZYGOTE_NICE_NAME
275            zygote = true;
276            niceName = ZYGOTE_NICE_NAME;
277        } else if (strcmp(arg, "--start-system-server") == 0) {
  						// 解析参数，如果包含--start-system-server 则启动SystemServer服务
278            startSystemServer = true;
279        } else if (strcmp(arg, "--application") == 0) {
280            application = true;
281        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
282            niceName.setTo(arg + 12);
283        } else if (strncmp(arg, "--", 2) != 0) {
284            className.setTo(arg);
285            break;
286        } else {
287            --i;
288            break;
289        }
290    }
  
  
339    if (!niceName.isEmpty()) {
340        runtime.setArgv0(niceName.string(), true /* setProcName */);
341    }
342
343    if (zygote) {
344        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
345    } else if (className) {
346        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
347    } else {
348        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
349        app_usage();
350        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
351    }
  	
352}
```

### 4 Zygote为什么需要用到Socket通信而不是Binder

开启进程肯定是一对多的，所以只有binder和socket符合



创建进程是一个较慢的行为，binder效率太高了，会将任务都堆积到Zygote，反而会降低效率

```java
private LocalServerSocket mZygoteSocket;
// 坚挺创建进程消息的
ZygoteServer(boolean isPrimaryZygote) {
    mUsapPoolEventFD = Zygote.getUsapPoolEventFD();

    if (isPrimaryZygote) {
        mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.PRIMARY_SOCKET_NAME);
        mUsapPoolSocket =
                Zygote.createManagedSocketFromInitSocket(
                        Zygote.USAP_POOL_PRIMARY_SOCKET_NAME);
    } else {
        mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.SECONDARY_SOCKET_NAME);
        mUsapPoolSocket =
                Zygote.createManagedSocketFromInitSocket(
                        Zygote.USAP_POOL_SECONDARY_SOCKET_NAME);
    }

    mUsapPoolSupported = true;
    fetchUsapPoolPolicyProps();
}
```

为什么要有zygote，如果没有zygote那么就需要app自己创建进程，那么所有app都能开进程，不好管理

### 5 每个App都会将系统的资源 ，系统的类都加载一遍吗

子类加载的时候会优先加载它父类的Class

子进程可以访问父进程的资源。zygoteinit 读取PRELOADED_CLASSES，循环遍历提前加载

```java
private static void preloadClasses() {
    final VMRuntime runtime = VMRuntime.getRuntime();

    InputStream is;
    try {
        is = new FileInputStream(PRELOADED_CLASSES);
    } catch (FileNotFoundException e) {
        Log.e(TAG, "Couldn't find " + PRELOADED_CLASSES + ".");
        return;
    }

    
    try {
        BufferedReader br =
                new BufferedReader(new InputStreamReader(is), Zygote.SOCKET_BUFFER_SIZE);

        int count = 0;
        String line;
        while ((line = br.readLine()) != null) {
            // Skip comments and blank lines.
            line = line.trim();
            if (line.startsWith("#") || line.equals("")) {
                continue;
            }

            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, line);
            try {
              	// 循环遍历提前加载
                Class.forName(line, true, null);
                count++;
            } catch (ClassNotFoundException e) {

    }

```

### 启动SystemServer

```
/* Request to fork the system server process */
pid = Zygote.forkSystemServer(
        parsedArgs.mUid, parsedArgs.mGid,
        parsedArgs.mGids,
        parsedArgs.mRuntimeFlags,
        null,
        parsedArgs.mPermittedCapabilities,
        parsedArgs.mEffectiveCapabilities);
```



### Zygote总结

1.LocalSocket 实现和别的进程通信

2.加载系统类，系统资源

3.启动SystemServer



### 6 PMS 是干什么的，你是怎么理解PMS

### 7  为什么会有AMS AMS的作用

### 8  为什么一个Activity需要声明在AndroidManifest.xml中

###  9 AMS如何管理Activity，探探AMS的执行原理

假设没有AMS、PMS要启动一个Activity要经历哪些步骤

1.遍历data/app目录

2.解压APK资源

3.dom解析Android.xml

4.定位到主Activity

5.class加载到内存

6.实例化Activity



PMS把123给做了，AMS把456给做了





