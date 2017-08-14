
*本文参考的代码为Android6.0版本*

[请尊重博主劳动成果，转载请标明出处。](http://blog.csdn.net/hwliu51/article/details/74944749)

### 一  编写测试代码，debug查看帧栈信息
测试代码：

```java
   View v = ViewUtil.findView(this, R.id.bt1);

   v.setOnTouchListener(new View.OnTouchListener() {
      @Override
      public boolean onTouch(View v, MotionEvent event) {
         return false;
      }
   });
```

将断点打在`return false`这行，点击debug，然后进入调试模式。查看该事件的帧栈信息，如下截图：
![Frames信息](http://img.blog.csdn.net/20170710232525442?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![Frames信息](http://img.blog.csdn.net/20170710232604273?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

ZygoteInit是用来初始化当前App进程，不用关心。Method是用来反射获取ActivityThread，而进程中所有对Activity的操作都是调用ActivityThread中的相关方法。Looper#loop是在ActivityThread的main方法中被调用，于是Looper便加入无限循环地获取Message。ActivityThread和Looper会一直运行到App进程退出才停止。这些都不是本文所关心的，继续往上看。

在Looper循环获取Message时，调用了MessageQueue#nativePollOnce方法。这个native方法又调用了InputEventReceiver#dispathInputEvent方法，从这里framework层便开始了touch事件的传递。

从以上的帧栈信息，可知touch事件传递的大概流程：感应器元件接收信息，然后通过底层驱动传递给系统服务。InputEventReceiver与系统服务相关联，于是系统服务将touch事件传递给ViewRootImpl。ViewRootImpl接收到事件后，又传递给PhoneWindow的内部类DecorView。DecorView又传递给Activity，Activity将事件传递给PhoneWindow。PhoneWindow将事件分发给DecorView，DecorView于是又分发给自己的子View。子View又继续分发，直到事件被处理。

### 二 主要的类和方法介绍
#### 1 ViewRootImpl相关介绍

关于nativePollOnce如何调用InputEventReceiver#dispathInputEvent方法请阅读这篇博客：[事件处理系统](http://www.cnblogs.com/lcw/p/3373214.html)。
查看InputEventReceiver的源码，这是一个抽象类。它是哪个类的实例呢？不着急，点击Frames的`dispatchInputEvent:185,InputEventReceiver(android.view)`这一行，进入查看Variables，显示信息如下：
![](http://img.blog.csdn.net/20170711001806355?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
原来它是ViewRootImpl的内部类WindowInputEventReceiver。于是这对象调用了自己的onInputEvent方法将touch事件通过调用ViewRootImpl#enqueueInputEvent方法传递给了ViewRootImpl。于是Touch事件进入了ViewRootImpl。

看看InputEventReceiver#dispathInputEvent代码：

```java
    // Map from InputEvent sequence numbers to dispatcher sequence numbers.
    private final SparseIntArray mSeqMap = new SparseIntArray();
    
    // Called from native code.
    @SuppressWarnings("unused")
    private void dispatchInputEvent(int seq, InputEvent event) {
        mSeqMap.put(event.getSequenceNumber(), seq);
        onInputEvent(event);
    }
```
WindowInputEventReceiver代码：

```java
    final class WindowInputEventReceiver extends InputEventReceiver {
        public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
            super(inputChannel, looper);
        }

        @Override
        public void onInputEvent(InputEvent event) {
        	//ViewRootImpl的方法
            enqueueInputEvent(event, this, 0, true);
        }

        @Override
        public void onBatchedInputEventPending() {
            if (mUnbufferedInputDispatch) {
                super.onBatchedInputEventPending();
            } else {
            	//ViewRootImpl的方法
                scheduleConsumeBatchedInput();
            }
        }

        @Override
        public void dispose() {
        	//ViewRootImpl的方法
            unscheduleConsumeBatchedInput();
            super.dispose();
        }
    }
```
ViewRootImpl内部对事件做了一系列的处理，最后判断该事件不是键盘输入事件传递出去了。
内部类ViewPostImeInputStage的onProcess方法来执行事件类别判断，代码如下：
```java
        @Override
        protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                return processKeyEvent(q);
            } else {
                final int source = q.mEvent.getSource();
                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                    return processPointerEvent(q);
                } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
                    return processTrackballEvent(q);
                } else {
                    return processGenericMotionEvent(q);
                }
            }
        }
        
        private int processPointerEvent(QueuedInputEvent q) {
            final MotionEvent event = (MotionEvent)q.mEvent;

            mAttachInfo.mUnbufferedDispatchRequested = false;
            final View eventTarget =
                    (event.isFromSource(InputDevice.SOURCE_MOUSE) && mCapturingView != null) ?mCapturingView : mView;
            mAttachInfo.mHandlingPointerEvent = true;
            boolean handled = eventTarget.dispatchPointerEvent(event);
            maybeUpdatePointerIcon(event);
            mAttachInfo.mHandlingPointerEvent = false;
            if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
                mUnbufferedInputDispatch = true;
                if (mConsumeBatchedInputScheduled) {
                    scheduleConsumeBatchedInputImmediately();
                }
            }
            return handled ? FINISH_HANDLED : FORWARD;
        }
```

再看看mCapturingView和mView代码上的注释。
```
View mView;

// The view which captures mouse input, or null when no one is capturing.
//获取鼠标输入相关的视图
View mCapturingView;
```
所以在processPointerEvent方法中事件被mView分发了。

而mView通过调用ViewRootImpl#setView方法设置的。代码如下：
```java
    /**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                ...//省略部分代码
            }
        }
    }
```
这个View只能被设置一次，也就是说它就是Activity的根View。这个根View传递给DecorView。这个mView是哪种类型的View呢？继续查看帧栈中的变量信息。点击`dispatchPoninterEvent:8799,View(android.view)`这一行，查看Variables中显示的信息。如下图所示：
![mView信息](http://img.blog.csdn.net/20170711111301000?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
它是DecorView的实例，也就是说DecorView的对象是当前Activity中最顶级的View，其它的View都是他的子孙View，状态栏和底部的导航栏都是在DecorView或其子孙View的内部。

##### **插个小话题，讲讲ViewRootImpl创建和setView的调用**

而ViewRootImpl是何时创建，mView（DecorView对象）又是何时被setView方法添加到ViewRootImpl？这个mView（DecorView对象）既然是根View，可以猜测管理它的对象一定是WindowManagerGlobal的。WindowManagerGlobal是管理Android中所有根View的超级大管家，而且也只管理根View。

先看看WindowManagerGlobal中相关的代码：
```java
	//记录根View
    private final ArrayList<View> mViews = new ArrayList<View>();
    //记录ViewRootImpl
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    //记录WindowManager.LayoutParams
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
            
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ...//省略部分代码
        
        //ViewRootImpl出现
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            ...//
            
            //创建ViewRootImpl对象
            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);
            
            //记录信息，方便管理，页面重新显示和移除时可以从中查找。
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }

        // do this last because it fires off messages to start doing things
        try {//将根View设置到ViewRootImpl，便能从ViewRootImpl传递事件给View
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
    }
```
WindowManagerImpl调用WindowManagerGlobal的相关代码：
```java
public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

	...//省略部分代码
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
    ...//省略部分代码
}
```

ActivityThread#handleResumeActivity方法中调用了WindowManger来添加DecorView。

```java
    final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        …//省略代码
        if (r != null) {
            final Activity a = r.activity;

            …//省略代码
            
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                //获取DecorView
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                //WindowManager为接口，实际为WindowMangerImpl对象
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                
                ...//省略部分代码
                
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    //添加到WindowMangerGoble
                    wm.addView(decor, l);
                }

            ...//省略代码
           
        } else {
            ...//
        }
    }
```

#### 2 PhoneWindow和DecorView相关介绍
继续看touch事件传递。刚才分析到ViewRootImpl的子类ViewPostImeInputStage
在processPointerEvent方法中将事件传递给了DecorView的dispatchTouchEvent。
继续看DecorView的dispatchTouchEvent方法代码：
```java
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        final Window.Callback cb = mWindow.getCallback();
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
    }
```

调用的是PhoneWindow#getCallback来获取，即Window的getCallback方法。这个mCallback其实就是当前Activity。Activity也实现了Window.Callback接口，用来接收窗口的事件。
Window的getCallback方法代码：
```java
public final Callback getCallback() {
	return mCallback;
}
```

在Activity#attach方法内，创建了PhoneWindow，并将自己设置给PhoneWindow。
Activity#attach方法代码：
```java
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
        //创建PhoneWindow
        mWindow = new PhoneWindow(this, window);
        mWindow.setWindowControllerCallback(this);
        //将Activity自己设置给PhoneWindow
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        ...//省略
        //设置WindowManagerImpl
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        ...//省略
    }
```
Window中setWindowManager代码：
```java
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName, boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated
                || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
```

#### 3 Activity相关介绍
touch事件终于传递到Activity了。咦，Activity怎么没有传递给内部的View，居然传递给PhoneWindow。
Activity#dispatchTouchEvent代码：
```java
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        //传递给PhoneWindow
        if (getWindow().superDispatchTouchEvent(ev)) {
       //如果事件被处理，则返回true
            return true;
        }
        return onTouchEvent(ev);
    }
```

PhoneWindow#superDispatchTouchEvent代码：
```
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
    	//将事件传递给DecorView
        return mDecor.superDispatchTouchEvent(event);
    }
```

DecorView的superDispatchTouchEvent代码：
```
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```
因为DecorView继承了FrameLayout，super.dispatchTouchEvent也调用了ViewGroup的dispatchTouchEvent和diaptchTransformedTouchEvent方法来继续对子View进行touch事件分发，直至事件被处理。关于ViewGroup和View如何分发和处理touch事件，请阅读博客：[Android触摸屏事件派发机制详解与源码分析一(View篇)](http://blog.csdn.net/yanbober/article/details/45887547)和[Android触摸屏事件派发机制详解与源码分析二(ViewGroup篇)](http://blog.csdn.net/yanbober/article/details/45912661) 。

于是事件传递到断点所在的代码行，整个touch事件传递个过程分析完毕。

#### 4 类图

图中画的是这篇文章中所涉及到的类，画了主要的属性和方法。

![touch事件UML](http://img.blog.csdn.net/20170716233210653?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

注：如果是Android 6.0或以上代码，DecorView已从PhoneWindow中抽出，成独立的一个类。

**补充**：如果不想用断点，也可以在onTouch方法中使用`Thread.dumpStack();`以日志的方式将帧栈中的信息打印出来，慢慢分析。

dump的stack信息：
```
java.lang.Throwable: stack dump
at java.lang.Thread.dumpStack(Thread.java:490)
at test.TestMainActivity$1.onTouch(TestMainActivity.java:32)
at android.view.View.dispatchTouchEvent(View.java:8582)
at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2519)
at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2171)
at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2519)
at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2171)
at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2519)
at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2171)
at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2519)
at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2171)
at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2519)
at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2171)
at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2519)
at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2171)
at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2519)
at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2171)
at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2519)
at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2171)
at com.android.internal.policy.impl.PhoneWindow$DecorView.superDispatchTouchEvent(PhoneWindow.java:2482)
at com.android.internal.policy.impl.PhoneWindow.superDispatchTouchEvent(PhoneWindow.java:1798)
at android.app.Activity.dispatchTouchEvent(Activity.java:2827)
at android.support.v7.view.WindowCallbackWrapper.dispatchTouchEvent(WindowCallbackWrapper.java:67)
at com.android.internal.policy.impl.PhoneWindow$DecorView.dispatchTouchEvent(PhoneWindow.java:2443)
at android.view.View.dispatchPointerEvent(View.java:8799)
at android.view.ViewRootImpl$ViewPostImeInputStage.processPointerEvent(ViewRootImpl.java:4706)
at android.view.ViewRootImpl$ViewPostImeInputStage.onProcess(ViewRootImpl.java:4537)
at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4051)
at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:4104)
at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:4070)
at android.view.ViewRootImpl$AsyncInputStage.forward(ViewRootImpl.java:4207)
at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:4078)
at android.view.ViewRootImpl$AsyncInputStage.apply(ViewRootImpl.java:4264)
at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4051)
at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:4104)
at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:4070)
at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:4078)
at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4051)
at android.view.ViewRootImpl.deliverInputEvent(ViewRootImpl.java:6507)
at android.view.ViewRootImpl.doProcessInputEvents(ViewRootImpl.java:6481)
at android.view.ViewRootImpl.enqueueInputEvent(ViewRootImpl.java:6434)
at android.view.ViewRootImpl$WindowInputEventReceiver.onInputEvent(ViewRootImpl.java:6666)
at android.view.InputEventReceiver.dispatchInputEvent(InputEventReceiver.java:185)
at android.os.MessageQueue.nativePollOnce(Native Method)
at android.os.MessageQueue.next(MessageQueue.java:148)
at android.os.Looper.loop(Looper.java:151)
at android.app.ActivityThread.main(ActivityThread.java:5759)
at java.lang.reflect.Method.invoke(Native Method)
at java.lang.reflect.Method.invoke(Method.java:372)
at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:1042)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:837)
```


本文参考的博客：
【Android】事件处理系统
http://www.cnblogs.com/lcw/p/3373214.html

Touch事件如何传递到Activity
http://www.jianshu.com/p/7d442ed0a355

