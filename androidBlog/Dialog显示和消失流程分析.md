*本文所引用的代码为Android 5.0（API 22）版本*

Dialog类实现了DialogInterface, Window.Callback, KeyEvent.Callback, OnCreateContextMenuListener, Window.OnWindowDismissedCallback这个五个接口。比较常用到的三个接口：Window.Callback是接收屏幕touch事件；KeyEvent.Callback为接收键盘按键事件和实体键消息；Window.OnWindowDismissedCallback为接收窗口消失回调。

本文从Dialog的创建，隐藏，显示和移除步骤来分析相关的代码流程。最后补充了几个常见的异常分析。

[请尊重博主劳动成果，转载请标明出处。](http://blog.csdn.net/hwliu51/article/details/75040297)

### 创建
看看构造方法代码：

```java
	//提供给外部调用人口
    public Dialog(Context context) {
        this(context, 0, true);
    }
    
    //提供给外部调用人口
    public Dialog(Context context, int theme) {
        this(context, theme, true);
    }
    
    //真正执行初始化的构造方法
    Dialog(Context context, int theme, boolean createContextThemeWrapper) {
        if (createContextThemeWrapper) {
            if (theme == 0) {
                TypedValue outValue = new TypedValue();
                context.getTheme().resolveAttribute(com.android.internal.R.attr.dialogTheme, outValue, true);
                theme = outValue.resourceId;
            }
            mContext = new ContextThemeWrapper(context, theme);
        } else {
            mContext = context;
        }
        
        //获取WindowManagerImpl对象，这个对象一般为Activity的windowManger
        mWindowManager = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
        //创建PhoneWindow对象
        Window w = PolicyManager.makeNewWindow(mContext);
        //赋值，mWindow即PhoneWindow对象，不是WindowManger
        mWindow = w;
        设置Window.Callback回调
        w.setCallback(this);
        //window dismiss回调
        w.setOnWindowDismissedCallback(this);
        //调用Window的setWindowManager创建PhoneWindow内部的WindowMangerImpl
        w.setWindowManager(mWindowManager, null, null);
        //默认设置为居中
        w.setGravity(Gravity.CENTER);
        //接收异步消息的Handler
        mListenersHandler = new ListenersHandler(this);
    }
```

PolicyManager#makeNewWindow相关代码：
路径：/sources/android-22/com/android/internal/policy/PolicyManager.java
```java
    public static Window makeNewWindow(Context context) {
        return sPolicy.makeNewWindow(context);
    }
```
sPolicy为Policy类的对象：
路径：/sources/android-22/com/android/internal/policy/impl/Policy.java
```java
    public Window makeNewWindow(Context context) {
        return new PhoneWindow(context);
    }
```
Window的setWindowManager方法相关代码：
```java
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName) {
        setWindowManager(wm, appToken, appName, false);
    }

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
    
### 显示
show方法代码：
```java
    public void show() {
        if (mShowing) {//dialog已调用过show方法，调用dismiss会设置为false
            if (mDecor != null) {
                if (mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
                    mWindow.invalidatePanelMenu(Window.FEATURE_ACTION_BAR);
                }
                //显示Dialog的根View，即Dialog便可见
                mDecor.setVisibility(View.VISIBLE);
            }
            return;
        }

        mCanceled = false;
        
        if (!mCreated) {
            dispatchOnCreate(null);
        }

        onStart();
        //创建DecorView
        mDecor = mWindow.getDecorView();

        if (mActionBar == null && mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
            final ApplicationInfo info = mContext.getApplicationInfo();
            mWindow.setDefaultIcon(info.icon);
            mWindow.setDefaultLogo(info.logo);
            mActionBar = new WindowDecorActionBar(this);
        }
        
        //设置布局参数
        WindowManager.LayoutParams l = mWindow.getAttributes();
        if ((l.softInputMode
                & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) == 0) {
            WindowManager.LayoutParams nl = new WindowManager.LayoutParams();
            nl.copyFrom(l);
            nl.softInputMode |=
                    WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
            l = nl;
        }

        try {
            //添加到当前显示的窗口，便可以看见dialog
            mWindowManager.addView(mDecor, l);
            mShowing = true;
            //发送dialog显示消息
            sendShowMessage();
        } finally {
        }
    }
```

mDecor为DecorView对象，DecorView继承了FrameLayout。在Android 6.0以下版本，DecorView为PhoneWindow的内部类；6.0及以上版本，则被抽出，为一个独立的类。

我们在使用Dialog调用findViewById和setContentView方法，其实都是PhoneWindow中的方法在执行相应的操作。
Dialog中相关方法的代码：
```java
    public View findViewById(int id) {
        return mWindow.findViewById(id);
    }

    public void setContentView(int layoutResID) {
        mWindow.setContentView(layoutResID);
    }

    public void setContentView(View view) {
        mWindow.setContentView(view);
    }

    public void setContentView(View view, ViewGroup.LayoutParams params) {
        mWindow.setContentView(view, params);
    }

    public void addContentView(View view, ViewGroup.LayoutParams params) {
        mWindow.addContentView(view, params);
    }
```

再看看PhoneWindow中的相关代码：

```java
    //获取DecorView
    @Override
    public final View getDecorView() {
        if (mDecor == null || mForceDecorInstall) {
	        //如果没有，则创建
            installDecor();
        }
        return mDecor;
    }
    
    //创建和初始化DecorView
    private void installDecor() {
        if (mDecor == null) {//如果为null，则创建
            mDecor = generateDecor();
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        }
        ...//省略部分代码
    }
    
    //创建DecorView对象
    protected DecorView generateDecor() {
        //创建DecorView，-1为布局资源id，即不解析xml布局文件
        return new DecorView(getContext(), -1);
    }
    
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID, getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        //在Dialog的mWindow中，cb为Dialog本身
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }

    @Override
    public void setContentView(View view) {
        setContentView(view, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    }

    @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            view.setLayoutParams(params);
            final Scene newScene = new Scene(mContentParent, view);
            transitionTo(newScene);
        } else {
            mContentParent.addView(view, params);
        }
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }

    @Override
    public void addContentView(View view, ViewGroup.LayoutParams params) {
        if (mContentParent == null) {
            installDecor();
        }
        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            // TODO Augment the scenes/transitions API to support this.
            Log.v(TAG, "addContentView does not support content transitions");
        }
        mContentParent.addView(view, params);
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```

这里就不细讲DecorView是如何被创建的，有兴趣的可以去看看源码，或看这篇博客：[Android应用setContentView与LayoutInflater加载解析机制源码分析](http://blog.csdn.net/yanbober/article/details/45970721)。

再看看mWindowManager.addView(mDecor, l)这步操作。从**创建**这一节的分析可知mWindowManager为WindowMangerImpl的对象。
先介绍下WindowMangerImpl代码：

```java
public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    //
    private final Display mDisplay;
    private final Window mParentWindow;

    private IBinder mDefaultToken;
    
    ...//省略部分代码
    
    //添加View
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mDisplay, mParentWindow);
    }

    private void applyDefaultToken(@NonNull ViewGroup.LayoutParams params) {
        // Only use the default token if we don't have a parent window.
        if (mDefaultToken != null && mParentWindow == null) {
            if (!(params instanceof WindowManager.LayoutParams)) {
                throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
            }

            // Only use the default token if we don't already have a token.
            final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
            if (wparams.token == null) {
                wparams.token = mDefaultToken;
            }
        }
    }
    
    ...//省略部分代码
    
    //移除View，后面的dimiss会调用到这个方法
    public void removeViewImmediate(View view) {
        //true为立即移除
        mGlobal.removeView(view, true);
    }
    
    ...//省略部分代码
}
```

继续看WindowManagerGobal的addView方法代码：

```java
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ...//省略

        //ViewRootImpl接收接收底层服务发送来的屏幕触摸事件，然后将这些事件传递给DecorView，
        //并返回结果（true或false）给底层服务。
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }

            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }

            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }

            //创建对象
            root = new ViewRootImpl(view.getContext(), display);
            //设置布局属性
            view.setLayoutParams(wparams);

            //添加到记录集合，方便以后查找和移除
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }

        // do this last because it fires off messages to start doing things
        try {
            //添加DecorView到ViewRootView，执行完这一步，dialog便能接收屏幕事件
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
关于ViewRootImpl如何接收和传递屏幕事件，可以阅读[Activity touch事件传递流程分析](http://blog.csdn.net/hwliu51/article/details/74944749)。
经历了setContent和show，漂亮的Dialog便显示在手机上。

### 隐藏和消失
隐藏调用的是hide方法。如果dialog在当前页面被频繁调用，则可以用这个方法。它不会将Dialog的根视图mDecor从当前移除，仅仅是将其设置为不可见。
hide方法代码
```java
    public void hide() {
        if (mDecor != null) {
            mDecor.setVisibility(View.GONE);
        }
    }
```
当我们调用代码dismiss，点击back键或点击dialog外部区域，便可以将dialog隐藏。

```java
    //提供给外部调用的方法
    public void dismiss() {
        if (Looper.myLooper() == mHandler.getLooper()) {
            //在主线程，则直接调用
            dismissDialog();
        } else {
            //非主线程，则发送消息，让mListenersHandler处理
            mHandler.post(mDismissAction);
        }
    }
    
    //真正执行dismiss的方法
    void dismissDialog() {
        //如果mDecor为null或dismiss的，则返回
        if (mDecor == null || !mShowing) {
            return;
        }

        //如果PhoneWindow被销毁，则返回
        if (mWindow.isDestroyed()) {
            Log.e(TAG, "Tried to dismissDialog() but the Dialog's window was already destroyed!");
            return;
        }

        try {
            //移除mDecor，其实是mWindowManager的内部WindowManagerGlobal对象执行移除操作
            mWindowManager.removeViewImmediate(mDecor);
        } finally {
            if (mActionMode != null) {
                mActionMode.finish();
            }
            mDecor = null;//置为null，释放
            mWindow.closeAllPanels();
            onStop();
            mShowing = false;//修改标志为false

            //发送dismiss事件
            sendDismissMessage();
        }
    }
```

这个方法里最重要的操作为mWindowManager.removeViewImmediate(mDecor)，这步操作将Dialog直接从当前视图移除，并销毁释放。所以在页面中被频繁显示的的dialog采用hide比dimiss更高效。

继续看WindowManagerGlobal代码：
```java
    public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            //查找当前View在记录集合中的位置
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            //移除ViewRootImpl
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            //如果要移除的View与记录中的不同，则抛出异常
            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }

    //查找位置
    private int findViewLocked(View view, boolean required) {
        final int index = mViews.indexOf(view);
        if (required && index < 0) {
            //如果该View已被移除，则抛出异常
            throw new IllegalArgumentException("View=" + view + " not attached to window manager");
        }
        return index;
    }

    private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();

        if (view != null) {//断开键盘与当前View的关联
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        //ViewRootImpl执行die操作
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }
```

经过以上三个步骤的分析，我们清楚知道了Dialog是如何创建，显示和消失移除。

### 补充：常见的异常分析
工作中可能会出现的一些异常，现在就分析下它们出现的原因。

#### WindowLeaked
异常信息：
```
android.view.WindowLeaked: Activity xx.xxActivity has leaked window com.android.internal.policy.impl.PhoneWindow$DecorView{3e7fbc08 V.E..... R.....I. 0,0-960,231} that was originally added here
at android.view.ViewRootImpl.<init>(ViewRootImpl.java:462)
at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:278)
at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:85)
at android.app.Dialog.show(Dialog.java:311)
```
这个是在Activity执行destroy时，dialog还没有被执行dismiss操作而抛出的。**需要在onPause()或onStop()中调用dismiss释放dialog窗口。**

如果跟着日志显示的类去查找，估计会转晕。其实，它在ActivityThread执行handleDestroyActivity操作时抛出的。如果跟着代码显示的日志，该异常出现在onDestroy的日志后面。
相关代码：
```java
    //ActivityThread的handleDestroyActivity方法代码
    private void handleDestroyActivity(IBinder token, boolean finishing,
            int configChanges, boolean getNonConfigInstance) {
        ActivityClientRecord r = performDestroyActivity(token, finishing,
                configChanges, getNonConfigInstance);
        if (r != null) {
            cleanUpPendingRemoveWindows(r);
            WindowManager wm = r.activity.getWindowManager();
            View v = r.activity.mDecor;
            if (v != null) {
                if (r.activity.mVisibleFromServer) {
                    mNumVisibleActivities--;
                }
                //当前Activity的window token，
                IBinder wtoken = v.getWindowToken();
                if (r.activity.mWindowAdded) {
                    if (r.onlyLocalRequest) {
                        // Hold off on removing this until the new activity's
                        // window is being added.
                        r.mPendingRemoveWindow = v;
                        r.mPendingRemoveWindowManager = wm;
                    } else {
                        wm.removeViewImmediate(v);
                    }
                }
                if (wtoken != null && r.mPendingRemoveWindow == null) {
                    //关闭所有与wtoken相关的所有DecorView
                    WindowManagerGlobal.getInstance().closeAll(wtoken,
                            r.activity.getClass().getName(), "Activity");
                }
                r.activity.mDecor = null;
            }
            if (r.mPendingRemoveWindow == null) {
                // If we are delaying the removal of the activity window, then
                // we can't clean up all windows here.  Note that we can't do
                // so later either, which means any windows that aren't closed
                // by the app will leak.  Well we try to warning them a lot
                // about leaking windows, because that is a bug, so if they are
                // using this recreate facility then they get to live with leaks.
                //关闭所有与token相关的所有DecorView
                WindowManagerGlobal.getInstance().closeAll(token,
                        r.activity.getClass().getName(), "Activity");
            }

            …//省略代码
        }
        …//省略代码
    }
    
    //WindowManagerGlobal的closeAll方法
    public void closeAll(IBinder token, String who, String what) {
        synchronized (mLock) {
            int count = mViews.size();
            //Log.i("foo", "Closing all windows of " + token);
            //遍历DecorView，对比token
            for (int i = 0; i < count; i++) {
                //Log.i("foo", "@ " + i + " token " + mParams[i].token
                //        + " view " + mRoots[i].getView());
                //如果当前传入的token为null，或DecorView的token与之相同
                if (token == null || mParams.get(i).token == token) {
                    ViewRootImpl root = mRoots.get(i);

                    //Log.i("foo", "Force closing " + root);
                    //当前传入的who为‘Activity’
                    if (who != null) {
                        //即这个窗口没有被移除，出现泄漏了
                        WindowLeaked leak = new WindowLeaked(
                                what + " " + who + " has leaked window "
                                + root.getView() + " that was originally added here");
                        leak.setStackTrace(root.getLocation().getStackTrace());
                        Log.e(TAG, "", leak);
                    }

                    //将其移除
                    removeViewLocked(i, false);
                }
            }
        }
    }
```
#### IllegalArgumentException
这个异常调用dismiss时出现的，常出现在WindowLeaked之后。在长时间的异步操作后，Activity可能被回收或调用finish释放，再在主线程调用dismiss时可能会出现这个异常。主要出现在网络请求结束关闭加载框时。解决方法：**在dismiss前使用isFinishing()判断当前Activity是否被finish，或者在onStop()或onPaus()方法中调用dismiss，再在onStart方法中判断是否需要显示。**
异常信息：

```
java.lang.IllegalArgumentException: View=com.android.internal.policy.impl.PhoneWindow$DecorView{3e7fbc08 V.E..... R.....I. 0,0-960,231} not attached to window manager
at android.view.WindowManagerGlobal.findViewLocked(WindowManagerGlobal.java:416)
at android.view.WindowManagerGlobal.removeView(WindowManagerGlobal.java:342)
at android.view.WindowManagerImpl.removeViewImmediate(WindowManagerImpl.java:116)
at android.app.Dialog.dismissDialog(Dialog.java:354)
at android.app.Dialog.dismiss(Dialog.java:337)
```
抛出异常的代码在**隐藏和消失**这节的WindowManagerGlobal的findViewLocked方法中。因为mDecor已经被移除了，而dialog又没有收到关闭消息，因此dialog认为mDecor还在当前视图中，所以再次remove便报错了。

#### BadTokenException
这个异常是使用new创建Dialog是没有传入Activity对象，而是其它的Context。导致token验证没有通过，而抛出异常。如果具有系统权限，则可以以其它的Context创建。
```
android.view.WindowManager$BadTokenException: Unable to add window -- token null is not valid; is your activity running?
at android.view.ViewRootImpl.setView(ViewRootImpl.java:685)
at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:289)
at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:85)
at android.app.Dialog.show(Dialog.java:311)
```

ViewRootImpl.setView的代码：

```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                …//省略
                int res; /* = WindowManagerImpl.ADD_OKAY; */

                …//省略
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
                    //验证token
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets, mInputChannel);
                } catch (RemoteException e) {
                    …//省略
                } finally {
                    …//省略
                }

                …//省略
                
                //判断token是否可用
                if (res < WindowManagerGlobal.ADD_OKAY) {
                    mAttachInfo.mRootView = null;
                    mAdded = false;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    switch (res) {
                        case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                        case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not valid; is your activity running?");
                        …//省略
                    }
                    throw new RuntimeException(
                            "Unable to add window -- unknown error code " + res);
                }

                …//省略
            }
        }
    }
```



