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



```java
private static native void nativeScheduleVsync(long receiverPtr);
```

```java
65status_t DisplayEventDispatcher::scheduleVsync() {
66    if (!mWaitingForVsync) {
67        ALOGV("dispatcher %p ~ Scheduling vsync.", this);
68
69        // Drain all pending events.
70        nsecs_t vsyncTimestamp;
71        int32_t vsyncDisplayId;
72        uint32_t vsyncCount;
73        if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
74            ALOGE("dispatcher %p ~ last event processed while scheduling was for %" PRId64 "",
75                    this, ns2ms(static_cast<nsecs_t>(vsyncTimestamp)));
76        }
77				// 请求垂直同步信号，最终请求到哪里
78        status_t status = mReceiver.requestNextVsync();
79        if (status) {
80            ALOGW("Failed to request next vsync, status=%d", status);
81            return status;
82        }
83
84        mWaitingForVsync = true;
85    }
86    return OK;
87}
```

Surfaceflinger

帧缓冲区

三个核心线程，UI线程、Binder线程、事件处理线程



发送完同步信号之后，将会分发事件

```java
private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
    onVsync(timestampNanos, builtInDisplayId, frame);
}
```

```java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {
    @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
  
				//通过handler进行绘制
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }
}
```

```java
// 所谓的垂直同步信号，就是去底层驱动，你绘制上一帧的时间
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized (mLock) {
        if (!mFrameScheduled) {
            return; // no work to do
        }

        if (DEBUG_JANK && mDebugPrintNextFrameTimeDelta) {
            mDebugPrintNextFrameTimeDelta = false;
            Log.d(TAG, "Frame time delta: "
                    + ((frameTimeNanos - mLastFrameTimeNanos) * 0.000001f) + " ms");
        }

        long intendedFrameTimeNanos = frameTimeNanos;
        startNanos = System.nanoTime();// 包括了下层的通信和回调
      	// 当前时间减去vsync来的时间，相当于主线程的耗时
        final long jitterNanos = startNanos - frameTimeNanos;
        if (jitterNanos >= mFrameIntervalNanos) {//这里整体时间出现大于一针
          // 下面的罗就就是计算到底要跳多少帧
          // 1帧是16.66毫秒，计算当前跳多少帧，比如超时166.66，那么就是跳过10帧
          final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
          	// 距离上一帧有多久，一帧是16毫秒，多出来的这一帧是多少秒
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
            if (DEBUG_JANK) {
                Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                        + "which is more than the frame interval of "
                        + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                        + "Skipping " + skippedFrames + " frames and setting frame "
                        + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
            }
          
          	// 修正时间，将时间固定在16毫秒
            frameTimeNanos = startNanos - lastFrameOffset;
        }
			
      	// 避免下一帧提前渲染，如果本次vsync执行doframe比上一帧计划的提前时间早
      	// 则将本帧放到下一个vsync上进行渲染
      	// 提前渲染就会出现，画面重叠现象16.6
        if (frameTimeNanos < mLastFrameTimeNanos) {
          	// 卡顿优化 -->> 30帧
            if (DEBUG_JANK) {
                Log.d(TAG, "Frame time appears to be going backwards.  May be due to a "
                        + "previously skipped frame.  Waiting for next vsync.");
            }
          	// 
            scheduleVsyncLocked();
            return;
        }

        if (mFPSDivisor > 1) {
            long timeSinceVsync = frameTimeNanos - mLastFrameTimeNanos;
            if (timeSinceVsync < (mFrameIntervalNanos * mFPSDivisor) && timeSinceVsync > 0) {
                scheduleVsyncLocked();
                return;
            }
        }
				//记录当前帧的原始vsync时间-修正后的时间
        mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
      	// 重置标识位，可再次进入scheduleFramelock
        mFrameScheduled = false;
      	// 记录上一次的vsync的时间
        mLastFrameTimeNanos = frameTimeNanos;
    }

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

        mFrameInfo.markInputHandlingStart();
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
				
        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
				// 一大堆的callback，回到了最初设置的地方
        mFrameInfo.markPerformTraversalsStart();
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }

    if (DEBUG_FRAMES) {
        final long endNanos = System.nanoTime();
        Log.d(TAG, "Frame " + frame + ": Finished, took "
                + (endNanos - startNanos) * 0.000001f + " ms, latency "
                + (startNanos - frameTimeNanos) * 0.000001f + " ms.");
    }
}
```

编舞者监控帧率

如果要出高刷手机 90Hz，framework一定会改过，主要是编舞者的时间，这个时间通常是在配置文件中，System.properties

```java
mChoreographer.postCallback(
        Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
```

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

```java
private void performTraversals() { //真正的开始进行绘制
    // cache mView since it is used so much below...
    final View host = mView;
		relayoutWindow
    performMeasure
    performLayout
    performDraw  
```

Sureface可以理解为实体，装一组数据，装xy，宽、高，各种描述这个板子的数据。

```java
private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
        boolean insetsPending) throws RemoteException {
		
  	// mWindowSession 作为和WMS进行通信的桥接模式 
    int relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
            (int) (mView.getMeasuredWidth() * appScale + 0.5f),
            (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
            insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
            mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
            mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingDisplayCutout,
            mPendingMergedConfiguration, mSurface);

    mPendingAlwaysConsumeNavBar =
            (relayoutResult & WindowManagerGlobal.RELAYOUT_RES_CONSUME_ALWAYS_NAV_BAR) != 0;

    if (restore) {
        params.restore();
    }

    if (mTranslator != null) {
        mTranslator.translateRectInScreenToAppWinFrame(mWinFrame);
        mTranslator.translateRectInScreenToAppWindow(mPendingOverscanInsets);
        mTranslator.translateRectInScreenToAppWindow(mPendingContentInsets);
        mTranslator.translateRectInScreenToAppWindow(mPendingVisibleInsets);
        mTranslator.translateRectInScreenToAppWindow(mPendingStableInsets);
    }
    return relayoutResult;
}
```



mWindowSession的实例对象

```java
package com.android.server.wm;
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
```

调用了WMS的relayoutWindow

```
@Override
public int relayout(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int requestedWidth, int requestedHeight, int viewFlags, int flags, long frameNumber,
        Rect outFrame, Rect outOverscanInsets, Rect outContentInsets, Rect outVisibleInsets,
        Rect outStableInsets, Rect outsets, Rect outBackdropFrame,
        DisplayCutout.ParcelableWrapper cutout, MergedConfiguration mergedConfiguration,
        Surface outSurface) {
    if (false) Slog.d(TAG_WM, ">>>>>> ENTERED relayout from "
            + Binder.getCallingPid());
    Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, mRelayoutTag);
    int res = mService.relayoutWindow(this, window, seq, attrs,
            requestedWidth, requestedHeight, viewFlags, flags, frameNumber,
            outFrame, outOverscanInsets, outContentInsets, outVisibleInsets,
            outStableInsets, outsets, outBackdropFrame, cutout,
            mergedConfiguration, outSurface);
    Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    if (false) Slog.d(TAG_WM, "<<<<<< EXITING relayout to "
            + Binder.getCallingPid());
    return res;
}
```

```
@Override
public int relayout(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int requestedWidth, int requestedHeight, int viewFlags, int flags, long frameNumber,
        Rect outFrame, Rect outOverscanInsets, Rect outContentInsets, Rect outVisibleInsets,
        Rect outStableInsets, Rect outsets, Rect outBackdropFrame,
        DisplayCutout.ParcelableWrapper cutout, MergedConfiguration mergedConfiguration,
        Surface outSurface) {
    if (false) Slog.d(TAG_WM, ">>>>>> ENTERED relayout from "
            + Binder.getCallingPid());
    Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, mRelayoutTag);
    int res = mService.relayoutWindow(this, window, seq, attrs,
            requestedWidth, requestedHeight, viewFlags, flags, frameNumber,
            outFrame, outOverscanInsets, outContentInsets, outVisibleInsets,
            outStableInsets, outsets, outBackdropFrame, cutout,
            mergedConfiguration, outSurface);
    Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    if (false) Slog.d(TAG_WM, "<<<<<< EXITING relayout to "
            + Binder.getCallingPid());
    return res;
}
```

```java
// 对于这个窗体的可见性，有变更，会进行重新计算、
// WMS记录窗体的信息
// 建立c层的surface
public int relayoutWindow(Session session, IWindow client, int seq, LayoutParams attrs,
        int requestedWidth, int requestedHeight, int viewVisibility, int flags,
        long frameNumber, Rect outFrame, Rect outOverscanInsets, Rect outContentInsets,
        Rect outVisibleInsets, Rect outStableInsets, Rect outOutsets, Rect outBackdropFrame,
        DisplayCutout.ParcelableWrapper outCutout, MergedConfiguration mergedConfiguration,
        Surface outSurface) {
		
		// mWindowMa pclass WindowHashMap extends HashMap<IBinder, WindowState>
  	// 是所有chuang IBinder是IWindow即WindowToken，WindowState保存了窗体的数据
    synchronized(mWindowMap) {
    
      //如果窗体发生变化，是否发生旋转，可见性是否变化等，则进行重新计算
      
      // 最终返回一个c层的SUrface地址 2027行
			result = createSurfaceControl(outSurface, result, win, winAnimator);
      // surfacecontrol.cpp gerSurface  
```

java层面的surface代表的是用户数据提供

c++层面的surface根据用户数据提供以及surfaceFlinger的相关关注(系统层面东西要附加上去，类似导航条，电池之类)



relayoutWindow

1. 保存信息到WMS

	2. 判定是否有需要重新计算相关坐标等数据，可见行
	2. 去顶层申请一个surface ，返回地址回来

 

```java
private boolean draw(boolean fullRedrawNeeded) {
  
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
   					// 硬件加速的硬件绘制
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback);
        } else {
          	// 常规是软件绘制
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                    scalingRequired, dirty, surfaceInsets)) {
                return false;
            }
        }
    }
    return useAsyncReport;
}
```

 



```java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
  
  
  canvas = mSurface.lockCanvas(dirty);
  
  // 主要核心目的是进行绘制请求的发送，最底层是64长度的队列，从队列中提取一个空间，一个空间是一个Surface
  // 告诉SurfaceFlinger可以消费队列中的消息，推给FrameBuffer
  surface.unlockCanvasAndPost(canvas);
  
```



Surface lock后，就持有了canvas对象，canvas实际上是bitmap

```c++
302static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
303        jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
304    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));

  
322    ANativeWindow_Buffer outBuffer;
323    status_t err = surface->lock(&outBuffer, dirtyRectPtr);
324    if (err < 0) {
325        const char* const exception = (err == NO_MEMORY) ?
326                OutOfResourcesException :
327                "java/lang/IllegalArgumentException";
328        jniThrowException(env, exception, NULL);
329        return 0;
330    }
331		 // 图像生成 ， 图像引擎生成图像数据-->显示
332    SkImageInfo info = SkImageInfo::Make(outBuffer.width, outBuffer.height,
333                                         convertPixelFormat(outBuffer.format),
334                                         outBuffer.format == PIXEL_FORMAT_RGBX_8888
335                                                 ? kOpaque_SkAlphaType : kPremul_SkAlphaType,
336                                         GraphicsJNI::defaultColorSpace());
337
338    SkBitmap bitmap;
339    ssize_t bpr = outBuffer.stride * bytesPerPixel(outBuffer.format);
340    bitmap.setInfo(info, bpr);
341    if (outBuffer.width > 0 && outBuffer.height > 0) {
342        bitmap.setPixels(outBuffer.bits);
343    } else {
344        // be safe with an empty bitmap.
345        bitmap.setPixels(NULL);
346    }
347
348    Canvas* nativeCanvas = GraphicsJNI::getNativeCanvas(env, canvasObj);
349    nativeCanvas->setBitmap(bitmap);
350
  
366    sp<Surface> lockedSurface(surface);
367    lockedSurface->incStrong(&sRefBaseOwner);
368    return (jlong) lockedSurface.get();
369}
```



总结：

1.绘制流程

数据加载——》有一个实例（Activity，View）--->

绘制--》套路（编舞者）——〉具体绘制--》自己准备一个Surface绘制数据——〉

下层准备一个Surface记录数据》canves记录数据——》生成bitmap图像数据——〉

surface再底层存储一个队列，采取的是生产者消费者模式

生产者——》上层（WMS）

消费者——〉surfaceFlinger——》拿到不同的Surface，还有系统的——〉framebuffer进行展示



2.WMS扮演的角色是数据管理者，与Surface，与surfaceflinger的通信协调



3.AMS

AMS XML数据管理者
