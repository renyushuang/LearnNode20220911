| PhoneWindow实例化流程                 |
| ------------------------------------- |
| DecorView创建过程                     |
| ViewRootImpl渲染机制                  |
| window的显示次序，尺寸计算            |
| window添加view流程分析                |
| token的 含义,自己实现子线程刷新UI框架 |

https://www.processon.com/view/link/61a7ebe80e3e746f85f57285#map

Wk3B

## 从启动到UI绘制过程

需要了解整个绘制过程中AMS、WMS、App进程的角色扮演



Activity 从启动到View加载过程

创建ActvityThread

进行ActvityThread.attach

AMS.getServices获取AMS

传个AMS一个ApplicatonThread

attachApplicatonLock

通过handler，发送消息给App

Instrumentaton构建，主要用于Applicaton&Activity的相关所有活动构建new/生命周期

makeApplicaton，通过反射创建Application

callApplicationOnCreare 



AMS下面还没有结束，继续AMS中的操作

获取ActivityRecord，拿到主Activity

realStartActivity

LauncherActivityItem.excute

最终调用handlerlaunchactivity

通过反射创建Activity

attach

callActivityOnCreate



总结：Activity是绘制数据的管理者，以及绘制状态

## setContentView做什么

getWindow().setContentView

attach时创建了PhoneWindow

WindowManagerImpl专注于与WMS进行通信，乔接模式

构建了一个专门负责与WMS交互的对象WindowManagerGloble



创建顶层View DecorView

给顶层View根据主题 做一个基础布局

解析出来Activity定义的layout

createFromTag，处理最根部的View

rInflatechiledren 将下面所有的子View全部加载，进行递归调用 

注意：这一层可能会因为XML的层级过多，导致栈溢出



截止到目前为止，数据准备完毕



## View数据准备好，给谁，给过去做什么

回到Activity的生命周期接着向下看

resumeItem handleResumeActivity

resume时，将dector添加到WindowManager中

ViewRootImpl管理所有的View进行绘制策略，怎么绘制，这个类说的算

将所有View进行存储，setView

requesetLayout

scheduleTraversals，编舞者开始进行绘制节奏的控制



## 编舞者

基础知识的补充

刷新率、帧速率

屏幕：负责展示数据



cpu/gpu：负责制作数据，制作图形数据

当制作图片大于刷新率的时候，出现跳针

当制作图片小于刷新率的时候，闪屏，屏幕白了下，或者黑了下



解决办法 projectbuffer、vAsync、triplebuffer（ 帧缓冲区）、编舞者

vAsync同步信号，每16ms进行一次同步

 帧缓冲区，是Linux为显示设备提供的一个接口，把显存抽象后的一种设备，它允许上层应用程序在图形模式下，直接对显示缓冲区进行读写操作。

当前帧、预备帧

```java
private Choreographer(Looper looper, int vsyncSource) {
  	// 上一帧时间
    mLastFrameTimeNanos = Long.MIN_VALUE;
		// 每帧时间间隔
    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
}
```

Choreographer 内部维护一个handler

​	handler上来先post一个消息

​	mFrameScheduled控制保证是由一帧数据进行绘制

```java
// 如果是主线程，直接调用，否则通过消息进行调用，保证主线程绘制
if (isRunningOnLooperThreadLocked()) {
    scheduleVsyncLocked();
} else {
    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
    msg.setAsynchronous(true);
    mHandler.sendMessageAtFrontOfQueue(msg);
}
```





