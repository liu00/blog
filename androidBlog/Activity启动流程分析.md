
*本文引用的代码为Android 6.0版本*

本文将从Activity，Instrumentation，ActivityManagerService，ActivityStackSupervisor，ActivityStack和ActivityThread这几个主要的类的交互流程来分析点击桌面图标启动Activity的过程。

[请尊重博主劳动成果，转载请标明原文链接。](http://blog.csdn.net/hwliu51/article/details/69361691)

## Activity相关步骤

### Launcher

位置：/android-6.0.0_r1/packages/apps/Launcher2/src/com/android/launcher2/Launcher.java

启动android-6.0.0_r1版本源码编译的模拟器，点击桌面上的应用图标。代码执行的流程：启动Launcher这个Activity，点击图标时会调用到onClick()。

onClick()方法中被调用的代码：
```java
    public void onClick(View v) {
        ...
        Object tag = v.getTag();
        if (tag instanceof ShortcutInfo) {
            // Open shortcut
            final Intent intent = ((ShortcutInfo) tag).intent;
            int[] pos = new int[2];
            v.getLocationOnScreen(pos);
            intent.setSourceBounds(new Rect(pos[0], pos[1],
                    pos[0] + v.getWidth(), pos[1] + v.getHeight()));

            boolean success = startActivitySafely(v, intent, tag);

            ...
        } else if (tag instanceof FolderInfo) {
            ...
        } else if (v == mAllAppsButton) {
            ...
        }
    }
```

然后会执行startActivitySafely()方法。
```java
    boolean startActivitySafely(View v, Intent intent, Object tag) {
        boolean success = false;
        try {
            success = startActivity(v, intent, tag);
        } catch (ActivityNotFoundException e) {
            ...
        }
        return success;
    }
```

方法内又调用的Launcher重写的startActivity()方法
```java
    boolean startActivity(View v, Intent intent, Object tag) {
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);

        try {
            ...
            if (useLaunchAnimation) {
                ...
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    // Could be launching some bookkeeping activity
                    startActivity(intent, opts.toBundle());
                } else {
                    ...
                }
            } else {
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    startActivity(intent);
                } else {
                    ...
                }
            }
            return true;
        } catch (SecurityException e) {
            ...
        }
        return false;
    }
```
终于调用了Activity的startActivity()方法。

### Activity

位置：/android-6.0.0_r1/frameworks/base/core/java/android/app/Activity.java

Activity的startActivity()方法最终会调用到内部的startActivityFoResult()方法。

```java
    public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            ...
        } else {
            ...
        }
    }
```

mInstrumentation为Instrumentation类型，方法内调用了其execStartActivity()方法。

## Instrumentation步骤

### Instrumentation

位置：/android-6.0.0_r1/frameworks/base/core/java/android/app/Instrumentation.java

```java
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ...
        try {
            ...
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            //检查执行结果，如果失败，则抛出对应的异常
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```
ActivityManagerNative.getDefault()获取的事ActivityManagerProxy对象，即调用了其startActivity()方法。checkStartActivityResult()比较简单，具体处理流程可以去查看源码。

## ActivityManagerService相关步骤

### ActivityManagerNative

位置：/android-6.0.0_r1/frameworks/base/core/java/android/app/ActivityManagerNative.java
ActivityManagerProxy也在这个java文件内。

ActivityManagerNative.getDefault()相关代码：

```java
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        return new ActivityManagerProxy(obj);
    }

    static public IActivityManager getDefault() {
        return gDefault.get();
    }

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            ...
            IActivityManager am = asInterface(b);
            ...
            return am;
        }
    };
```

由以上代码可知getDefault()获取的事ActivityManagerProxy对象。startActivity()是ActivityManagerProxy中的方法。

### ActivityManagerProxy
```java
    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        //写数据用的buffer
        Parcel data = Parcel.obtain();
        //读数据用的buffer
        Parcel reply = Parcel.obtain();
        ...//写入数据
        
        //将数据发送给ActivityManagerService
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        ...
        return result;
    }
```
mRemote为IBinder对象，通过它ActivityManagerProxy可以与服务进程的ActivityManagerService进行相互传递消息。transact()方法会将数据传递给ActivityManagerService的onTransact()方法。

### ActivityManagerService
位置：/android-6.0.0_r1/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

onTransact()方法

```java    
	public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        if (code == SYSPROPS_TRANSACTION) {
            ...
        }
        try {
            return super.onTransact(code, data, reply, flags);
        } catch (RuntimeException e) {
            // The activity manager only throws security exceptions, so let's
            // log all others.
            ...
        }
    }
```
return时调用了父类ActivityManagerNative的onTransact()方法来处理。

ActivityManagerNative的onTransact()方法中处理的相应代码片段：

```java
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case START_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            ...//解析数据
            int result = startActivity(app, callingPackage, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
            ...
            return true;
        }
    }
```
onTransact()方法内根据code又调用了ActivityManagerService的startActivity()方法，并传入了获取到的数据。

ActivityManagerService的startActivity()方法

```java
        public int startActivity(IBinder whoThread, String callingPackage,
                Intent intent, String resolvedType, Bundle options) {
            checkCaller();

            int callingUser = UserHandle.getCallingUserId();
            TaskRecord tr;
            IApplicationThread appThread;//启动Activity的ApplicationThread
            synchronized (ActivityManagerService.this) {
                tr = mStackSupervisor.anyTaskForIdLocked(mTaskId);
                ...
                appThread = ApplicationThreadNative.asInterface(whoThread);
                ...
            }
            return mStackSupervisor.startActivityMayWait(appThread, -1, callingPackage, intent,
                    resolvedType, null, null, null, null, 0, 0, null, null,
                    null, options, false, callingUser, null, tr);
        }
```

## ActivityStackSupervisor和ActivityStack相关步骤

### ActivityStackSupervisor
位置：/android-6.0.0_r1/frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java

ActivityStackSupervisor的startActivityMayWait()方法

```java
    final int startActivityMayWait(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
            ...
            
            //获取Activity的相关信息
            // Collect information about the target of the Intent.
            ActivityInfo aInfo = resolveActivity(intent, resolvedType, startFlags, profilerInfo, userId);

            //HeavyWeight检查（非HeavyWeight Process，不用关心）
            ...

            //执行启动操作
            int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);

            //outResult设置。（此次outResult为null）
            ...
}
```

方法内根据Intent获取相应的信息并封装到ActivityInfo对象，然后调用startActivityLocked()方法。

startActivityLocked()方法

```java
    final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
            //默认为调用成功
            int err = ActivityManager.START_SUCCESS;
            
            //获取调用者进程信息
            ...
            
            //检查是否合法，是否具有权限等等
            ...
            
            ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                requestCode, componentSpecified, voiceSession != null, this, container, options);
            
            ...
            
            err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);
            
            //调用失败处理
            ...
            
            return err;
   }
```

方法内做了一大堆判断，之后将信息封装到ActivityRecord对象，最后调用了startActivityUncheckedLocked()方法。

startActivityUncheckedLocked()方法
```java
    final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
            boolean doResume, Bundle options, TaskRecord inTask) {
        //判断r.launchMode，即当前需要启动的Activity的launchMode类型
        ...
        
        ActivityStack targetStack;
        
        //根据launchMode以及其它条件判断，是否从已有的activity栈中唤起
        ...
        
        // Should this be considered a new task?
        if (r.resultTo == null && inTask == null && !addingToTask
                && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
            newTask = true;
            targetStack = computeStackFocus(r, newTask);
            targetStack.moveToFront("startingNewTask");

            if (reuseTask == null) {
                r.setTask(targetStack.createTaskRecord(getNextTaskId(),
                        newTaskInfo != null ? newTaskInfo : r.info,
                        newTaskIntent != null ? newTaskIntent : intent,
                        voiceSession, voiceInteractor, !launchTaskBehind /* toTop */),
                        taskToAffiliate);
                if (DEBUG_TASKS) Slog.v(TAG_TASKS,
                        "Starting new activity " + r + " in new task " + r.task);
            } else {
                r.setTask(reuseTask, taskToAffiliate);
            }
            
            ...
        }else if (sourceRecord != null) {
          ...
        }else if (inTask != null) {
          ...
        }else{
          ...
        }
        
        targetStack.mLastPausedActivity = null;
        targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
        
        	   
        	return ActivityManager.START_SUCCESS;   
   }
```

Launcher的startActivity方法内设置了Intent.FLAG_ACTIVITY_NEW_TASK，而且有时新进程的Activitvy，所以会新建一个task来存放启动的Activity。
`startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true, options, inTask);`调用时doResume为true。

### ActivityStack

位置：/android-6.0.0_r1/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java

往下继续查看ActivityStack的startActivityLocked()方法

```java
    final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
        //将r.task添加到历史栈mTaskHistory，并移动到顶部
        TaskRecord rTask = r.task;
        final int taskId = rTask.taskId;
        // mLaunchTaskBehind tasks get placed at the back of the task stack.
        if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
            // Last activity in task had been removed or ActivityManagerService is reusing task.
            // Insert or replace.
            // Might not even be in.
            insertTaskAtTop(rTask, r);
            mWindowManager.moveTaskToTop(taskId);
        } 
         
        TaskRecord task = null;
        ...
         
        task = r.task;
        ...
        task.addActivityToTop(r);
        task.setFrontOfTask();
         
        r.putInHistory();
        if (!isHomeStack() || numActivities() > 0) {
            // We want to show the starting preview window if we are
            // switching to a new task, or the next activity's process is
            // not currently running.
            boolean showStartingIcon = newTask;
            ProcessRecord proc = r.app;
            if (proc == null) {
                //获取被启动的Activity的进程信息。（activity进程还未被启动，所以为null）
                proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);
            }
            if (proc == null || proc.thread == null) {
                showStartingIcon = true;
            }
            ...
            
        } else {
            // If this is the first activity, don't do any fancy animations,
            // because there is nothing for it to animate on top of.
            ...
        }
        
        if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }   
```
上个调用者传入的doResume为true，所以会执行ActivityStackSupervisor的resumeTopActivitiesLocked()方法。

回到ActivityStackSupervisor，进行查看resumeTopActivitiesLocked方法

```java
    boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
        //targetStack是否在栈顶
        if (isFrontStack(targetStack)) {
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }

        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (stack == targetStack) {
                    // Already started above.
                    continue;
                }
                //stack是否在栈顶
                if (isFrontStack(stack)) {
                    stack.resumeTopActivityLocked(null);
                }
            }
        }
        return result;
    }
```
这里会执行两次resumeTopActivityLocked()方法。第一次调用`targetStack.resumeTopActivityLocked(target, targetOptions);`，会给ActivityRecord赋值给ActivityStack的属性mResumedActivity。第二次，会执行`pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);`。

又回到ActivityStack，继续查看resumeTopActivityLocked()相关的方法
```java
    final boolean resumeTopActivityLocked(ActivityRecord prev) {
        return resumeTopActivityLocked(prev, null);
    }

    final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        ...
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            ...
            //参数：prev=null, options=null
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
    
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
        ...
        
        //获取正在显示的Activity
        // Find the first activity that is not finishing.         final ActivityRecord next = topRunningActivityLocked(null);
        
        ...
        
        //prev=null，所以prevTask为null
        final TaskRecord prevTask = prev != null ? prev.task : null;
        if (next == null) {
           ...
        }
        
        next.delayedResume = false;
        
        ...
        
        // We need to start pausing the current activity so the top one
        // can be resumed...
        boolean dontWaitForPause = (next.info.flags&ActivityInfo.FLAG_RESUME_WHILE_PAUSING) != 0;
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, true, dontWaitForPause);
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }
        
        ...
        
        if (next.app != null && next.app.thread != null) {
            ...
            
            next.state = ActivityState.RESUMED;
            mResumedActivity = next;
            next.task.touchActiveTime();
            ...
        }
             
    }
```

接着看ActivityStack的startPausingLocked()方法

```java
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming, boolean dontWait) {
        ...
       
        ActivityRecord prev = mResumedActivity;
        
        ...
        
        mResumedActivity = null;
        mPausingActivity = prev;
        mLastPausedActivity = prev;
        
        ...
        
                if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                ...
                //调用prev所在进程（ProcessRecord）的ApplicationThread的方法通知
                //prev所在进程的ActivityThread对其进行pause操作。
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing, serLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                …
            }
        } else {
            ...
        }
    
    }
```
因为startPausingLocked()是在服务进程中调用的，所以thread为IBinder对象。因而`thread.schedulePauseActivity()`调用的是ApplicationThreadProxy的schedulePauseActivity()方法。使用ApplicationThreadProxy的mRemote向Launcher进程的ActivityThread的mAppThread（ApplicationThread对象）传递pause Activity操作相关的数据。

## ActivityThread相关步骤

### ApplicationThreadProxy
位置：/android-6.0.0_r1/frameworks/base/core/java/android/app/ApplicationThreadNative.java
ApplicationThreadProxy在这个java文件内。

schedulePauseActivity()方法
```java
    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) throws RemoteException {
        Parcel data = Parcel.obtain();
        ...//写入数据
        mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
```

Launcher进程的ApplicationThread对象的onTransact()方法会接收到mRemote.transact()方法提交的信息。

### ApplicationThread

位置：/android-6.0.0_r1/frameworks/base/core/java/android/app/ActivityThread.java
ApplicationThread为ActivityThread的内部类。

ApplicationThread没有重写onTransact()方法，所以处理mRemote.transact()的是父类ApplicationThreadNative的方法。

ApplicationThreadNative的onTransact()方法

```java
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case SCHEDULE_PAUSE_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            IBinder b = data.readStrongBinder();
            boolean finished = data.readInt() != 0;
            boolean userLeaving = data.readInt() != 0;
            int configChanges = data.readInt();
            boolean dontReport = data.readInt() != 0;
            schedulePauseActivity(b, finished, userLeaving, configChanges, dontReport);
            return true;
        }
        ...
        }
     }
```

在父类的onTransact()方法中根据code又调用了ApplicationThread的schedulePauseActivity()方法，并传递了接收的信息。


ApplicationThread的schedulePauseActivity()方法

```java
        public final void schedulePauseActivity(IBinder token, boolean finished,
                boolean userLeaving, int configChanges, boolean dontReport) {
            sendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
                    configChanges);
        }
        
        ...
        //执行发送消息的方法
        private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
           ...
           mH.sendMessage(msg);
        }

```
mH为H类型，H继承了Handler。mH为ActivityThread对象的属性，H类为ActivityThread内部类。

mH处理消息的方法：handleMessage()方法部分代码
```java
        public static final int RESUME_ACTIVITY         = 107;
        ...

        public void handleMessage(Message msg) {
           switch (msg.what) {
                ...
                case PAUSE_ACTIVITY:
                    //调用ActivityThread的方法对Activity进行pause操作
                    handlePauseActivity((IBinder)msg.obj, false, (msg.arg1&1) != 0, msg.arg2, (msg.arg1&2) != 0);
                    ...
                    break;
                ...
           }
       }
```
mH的handleMessage()方法根据msg.what调用了ActivityThread的handlePauseActivity(),并传入了msg中带有的数据。

### ActivityThread

位置：/android-6.0.0_r1/frameworks/base/core/java/android/app/ActivityThread.java

ActivityThread类的部分代码，ApplicationThread，H 和handlePauseActivity()方法：

```java
public final class ActivityThread {

    ...
    final ApplicationThread mAppThread = new ApplicationThread();
    final H mH = new H();
    ...
    
    private void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
            ...
            if (userLeaving) {
                performUserLeavingActivity(r);
            }

            r.activity.mConfigChangeFlags |= configChanges;
            performPauseActivity(token, finished, r.isPreHoneycomb());

            ...

            // Tell the activity manager we have paused.
            if (!dontReport) {
                try {
                    ActivityManagerNative.getDefault().activityPaused(token);
                } catch (RemoteException ex) {
                }
            }
            mSomeActivitiesChanged = true;
        }
    }
    
    ...
    
    private class ApplicationThread extends ApplicationThreadNative {
        ...
    }

    private class H extends Handler {
        ...
    }
}
```
performUserLeavingActivity()方法会调用Activity的performUserLeaving()方法，通知应用用户已离开这个页面。performPauseActivity()中先会判断是否有状态消息需要保存，如果有则会通过Instrumentation调用onSaveInstanceState()对Activity的状态进行保存。然后调用Instrumentation的callActivityOnPause()来调用Activity的performPause()进行pause操作，最后将Activity的paused属性设置为true。执行完performPauseActivity()操作，Activity便进入了pause状态。继续调用ActivityManagerProxy的activityPaused()方法向ActivityManagerService发送Activity的pause完成消息。**ActivityManagerService相关步骤**这一节已介绍过ActivityManagerProxy和ActivityManagerService调用的详细过程，此处便不再介绍。

## ActivityManagerService与ActivityThead相关步骤
### ActivityManagerService

又回到了ActivityManagerService，onTransact()接收到数据后会调用activityPaused()。

```java
    @Override
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityPausedLocked(token, false);
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
```
这个token对象是ActivityStack的startPausingLocked()方法中mResumedActivity的appToken。appToken是ActivityManagerService持有的与ApplicationThread交互的IBinder对象。根据appToken来查找被锁定的ActivityStack，然后调用其的activityPausedLocked()方法。

### ActivityStack

转回到ActivityStack，继续看看activityPausedLocked()

```java
    final void activityPausedLocked(IBinder token, boolean timeout) {
        …

        final ActivityRecord r = isInStackLocked(token);
        if (r != null) {
            mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
            if (mPausingActivity == r) {
                ...
                completePauseLocked(true);
            } else {
                ...
            }
        }
    }
```
接着会继续执行完成pause锁定，于是进入了completePauseLocked()


```java
    private void completePauseLocked(boolean resumeNext) {
        ActivityRecord prev = mPausingActivity;
        ...
        
        if (resumeNext) {
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            if (!mService.isSleepingOrShuttingDown()) {
                mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
            } else {
                ...
            }
        }
        
        ...
        
    }
```
resumeNext为true，接着会执行ActivityStackSupervisor的resumeTopActivitiesLocked()。该方法又会调用ActivityStack的resumeTopActivityLocked()方法，而resumeTopActivityLocked()会执行到resumeTopActivityInnerLocked()方法。

看看resumeTopActivityInnerLocked()方法

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
        //
        ...
        
        //获取正在显示的Activity
        // Find the first activity that is not finishing.
        final ActivityRecord next = topRunningActivityLocked(null);
        
        ...
        
        if (next.app != null && next.app.thread != null) {
            ...
        }else{
            // Whoops, need to restart this activity!
            ...
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }
        ...    
}
```

由于Launcher进入了pause状态，所以栈顶正在运行的是待启动的Activity。又因为该Activity的应用还没有被启动，所以next.app为null。接着会执行ActivityStackSupervisor的startSpecificActivityLocked()方法。

### ActivityStackSupervisor

转回到ActivityStackSupervisor，继续看startSpecificActivityLocked()方法

```java
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        //根据包名和uid获取进程
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        r.task.stack.setLaunchTime(r);

        if (app != null && app.thread != null) {
            try {
                …
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                ...
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        //创建进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0, "activity", r.intent.getComponent(), false, false, true);
    }
```
首先会根据r的进程名称和uid从进程记录者集合中查找。因为r的进程还未创建，所以返回为null。接着会调用ActivityManagerSevice的startProcessLocked()方法来准备创建进程。

### ActivityManagerService

又回到ActivityManagerService，脑袋快要转晕了。继续看startProcessLocked()

```java
    final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
            boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
            String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
        long startTime = SystemClock.elapsedRealtime();
        ProcessRecord app;
        …

        String hostingNameStr = hostingName != null
                ? hostingName.flattenToShortString() : null;

        if (app == null) {
            ...
            //创建进程记录者ProcessRecord
            app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
            if (app == null) {
                Slog.w(TAG, "Failed making new process record for "
                        + processName + "/" + info.uid + " isolated=" + isolated);
                return null;
            }
            app.crashHandler = crashHandler;
            ...
        } else {
            ...
        }
        
        ...
        //从ProcessRecord开启进程
        startProcessLocked( app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
        ...
        return (app.pid != 0) ? app : null;
    }
```
先调用newProcessRecordLocked()创建进程记录者，然后再调用startProcessLocked()执行创建进程准备工作。

newProcessRecordLocked()方法

```java
    final ProcessRecord newProcessRecordLocked(ApplicationInfo info, String customProcess,
            boolean isolated, int isolatedUid) {
        ...
        final ProcessRecord r = new ProcessRecord(stats, info, proc, uid);
        ...
        //添加到ActivityManagerService的集合中
        addProcessNameLocked(r);
        return r;
    }
```
创建ProcessRecord，并记录到集合中。

startProcessLocked()方法

```java
    private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
        ...
        try {
            ...

            boolean isActivityProcess = (entryPoint == null);
            //ActivityThread的类路径，使用反射来启用ActivityThread
            //fork的进程在完成初始化之后会执行ActivityThread的main方法
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
            ...
            //Process.start发送进程相关的配置信息和进程创建之后执行的类entryPoint
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
            ...
        } catch (RuntimeException e) {
            ...
        }
        ...
    }
```

Process创建进程的具体流程可以阅读[Activity之应用进程创建流程简析](http://blog.csdn.net/hwliu51/article/details/75579130)。从这篇博客可以知道ActivityManagerService接收到ApplicationThread的IBinder后会调用attachApplicationLocked()方法来创建Application对象。在创建完Application之后，还会调用ActivityStackSupervisor的attachApplicationLocked()方法来启动Activity。

继续看attachApplicationLocked()方法代码：

```java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

        ...

        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
                //判断是否有需要启动的Activity，并执行相应操作
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                ...
            }
        }

        ...
    }
```
在将待启动的Activity的进程的ApplicationThread的IBinder传递给ActivityManagerService之后，会去判断是否有待启动Activity。如果有，则会执行启动和现实Activity操作。

### ActivityStackSupervisor

回到ActivityStackSupervisor，继续看attachApplicationLocked()方法

```java
    boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (!isFrontStack(stack)) {
                    continue;
                }
                ActivityRecord hr = stack.topRunningActivityLocked(null);
                if (hr != null) {
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {
                            // 是否需要启动Activity，并执行相应操作
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            ...
                        }
                    }
                }
            }
        }
        ...
        return didSomething;
    }
```

遍历查找所有的ActivityStack，获取栈顶的ActivityRecord，并判断该Activity是否需要启动现实。

realStartActivityLocked()方法

```java
    final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {

        ...

        final ActivityStack stack = task.stack;
        try {
            ...
            
            //发送启动和现实默认Activity的信息
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);

            …

        } catch (RemoteException e) {
            …
        }

        …

        return true;
    }
```
在**ActivityThread相关步骤**这一节中已详细介绍了thread与ActivityThread中的mAppThread是如何传递信息，所以此不在介绍。直接进入ApplicationThread的scheduleLaunchActivity()方法进行分析。

### ApplicationThread

ApplicationThread的scheduleLaunchActivity()方法

```java
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            ...

            ActivityClientRecord r = new ActivityClientRecord();

            ...//获取信息

            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
```

sendMessage()方法内调用了mH发送LAUNCH_ACTIVITY类型的消息和数据，在H的handleMessage()方法中会调用ActivityThread的handleLaunchActivity()方法来启动默认的Activity。

### ActivityThread

终于会到启动和显示Activity的ActivityThread，查看启动的方法handleLaunchActivity()
```java
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...

        //创建Activity，执行onCreate，onStart操作
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            //添加Activity的View和执行onResume()方法
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);

            ...
        } else {
            // If there was an error, for any reason, tell the activity
            // manager to stop us.
            ...
        }
    }
```

这里的执行过程可以分为两个步骤。

第一步：执行了`performLaunchActivity(r, customIntent)`，该方法会创建Activity，并执行显示前的准备工作。

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            //通过反射创建Activity
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            ...
        } catch (Exception e) {
            ...
        }

        try {
            ...

            if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                ...
                //执行Activity的attach()方法
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);

                ...
                //设置主题
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                //执行Activity的onCreate()方法
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ...
                //执行Activity的start()方法
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) {
                    //执行Activity的onRestoreInstanceState()方法
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                ...
            }
            r.paused = true;
            //添加到Activity集合
            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            ...
        }

        return activity;
    }
```

第二步：执行`handleResumeActivity(r.token, false, r.isForward, !r.activity.mFinished && !r.startsNotResumed);`，该方法先执行Activity的onPause()方法，然后获取Activity的根视图DecorView，并添加到窗口和显示。

1，执行onPause()的performResumeActivity()方法

```java
    public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide) {
        ...
        if (r != null && !r.activity.mFinished) {
            ...
            try {
                ...
                //将会执行Activity的onPause()
                r.activity.performResume();
                ...
            } catch (Exception e) {
                ...
            }
        }
        return r;
    }
```

2，获取和添加根视图，然后显示Activity的handleResumeActivity()
```java
    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {
        ...
        // TODO Push resumeArgs into the activity for consideration
        ActivityClientRecord r = performResumeActivity(token, clearHide);

        if (r != null) {
            final Activity a = r.activity;
            ...
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                //Activity的根View
                View decor = r.window.getDecorView();
                //设置状态为NVISIBLE（不可见）
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    //添加decor到窗口
                    wm.addView(decor, l);
                }
                ...
            } else if (!willBeVisible) {
                ...
            }

            ...

            // The window is now visible if it has been added, we are not
            // simply finishing, and we are not starting another activity.
            if (!r.activity.mFinished && willBeVisible
                    && r.activity.mDecor != null && !r.hideForNow) {
                ...
                r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;
                if (r.activity.mVisibleFromClient) {
                    //显示Activity
                    r.activity.makeVisible();
                }
            }

            ...

            // Tell the activity manager we have resumed.
            if (reallyResume) {
                try {
                    //通知ActivityManagerService，Activity已显示。
                    ActivityManagerNative.getDefault().activityResumed(token);
                } catch (RemoteException ex) {
                }
            }

        } else {
            // If an exception was thrown when trying to resume, then
            // just end this activity.
            ...
        }
    }
```

当进程的ActivityThread对象执行完performLaunchActivity()方法，被启动Activity便显示在屏幕。

## 总结

### **点击桌面图标启动Activity的大致流程**：

1，Launcher接收点击事件，调用startActivity发出启动请求，最终会调用到父类Activity的startActivityForResult()方法。

2，在startActivityForResult()方法请求传递到Instrumentation的execStartActivity()。

3，execStartActivity()方法调用ActivityManagerNative.getDefault()获取ActivityManagerProxy对象，使用该对象的startActivity()方法向服务进程ActivityManagerService发送启动Activity的信息。ActivityManagerService从父类ActivityManagerNative继承的onTransact()方法接收到数据，会根据code调用startActivity()方法。

4，startActivity()方法又调用ActivityStackSupervisor的startActivityMayWait()方法。ActivityStackSupervisor继续调用ActivityStack的方法，二者相互调用。查找当前栈顶显示的Activity的ActivityRecord，在ActivityStack的startPausingLocked()方法中调用ActivityRecord的app（Launcher进程记录者ProcessRecord）的thread（与Launcher进程的ApplicationThread对象交互的IBinder对象）向ApplicationThread对象发送pause Activity的信息。

5，ApplicationThread的onTransact()方法接收到thread发送的数据，然后调用schedulePauseActivity()方法。该方法会调用mH发送PAUSE_ACTIVITY类型的消息，在mH的handleMessage()方法又会调用Launcher进程的ActivityThread对象对Launcher这个Activity执行pause操作。执行完pause操作，继续调用ActivityManagerProxy的activityPaused()方法向ActivityManagerService发出Launcher已经进入pause状态消息。

6，ActivityManagerService的activityPaused()方法接收到回调，便会查找被pause的Activity的ActivityStack，调用它的activityPausedLocked()方法最终会调用到ActivityStackSupervisor的startSpecificActivityLocked()方法。如果被启动的Activity进程存在，则调用realStartActivityLocked()，该方法会调用该进程的ApplicationThread的IBinder对象去启动Activity。否则，会调用ActivityManagerService的startProcessLocked()去创建进程记录者和创建Socket通知Zygote进程fork出新进程。

7，创建进程成功后，ActivityManagerService的attachApplicationLocked()方法会被调用，以接收到该进程的ApplicationThread的IBinder。改方法内会先创建应用的Application对象，然后调用ActivityStackSupervisor的attachApplicationLocked()方法检查是否有待启动的Activity并调用IBinder去通知ApplicationThread发送启动消息。

8，H收到消息，然后调用ActivityThread的handleLaunchActivity()启动Activity。到此，Activity便能显示在模拟器上。


### **精简流程**：

1，startActivity会通过ActivityManagerProxy将启动Activity消息传递给ActivityManagerService。

2，ActivityManagerService收到消息，会调用ActivityStackSupervisor和ActivityStack通过ActvityThread将当前显示的Activity进入pause状态。

3，Activity进入pause状态后，调用ActivityManagerProxy通知ActivityManagerService。

4，ActivityManagerService收到pause通知，调用ActivityStackSupervisor去启动Activity。

5，ActivityStackSupervisor发现Activity的进程为null，于是让ActivityManagerService通知Zygote需要fork一个进程。

5，Zygote收到消息。然后fork一个进程。进程进行初始化，然后创建ActivityThread。

6，ActivityThread将ApplicationThread的IBinder通过ActivityManagerProxy传递给ActivityManagerService。

7，ActivityManagerService收到IBinder，然后创建应用的Aplication，再通知ActivityStackSupervisor去启动Activity

8，ActivityStackSupervisor通过IBinder通知到ActivityThread去启动Activity。


----------



参考博客：

Android应用程序的Activity启动过程简要介绍和学习计划： 
 http://blog.csdn.net/luoshengyang/article/details/6685853

android应用程序启动过程的源代码分析：
 http://blog.csdn.net/luoshengyang/article/details/6689748

Android应用程序内部启动Activity过程（startActivity）的源代码分析：  
http://blog.csdn.net/Luoshengyang/article/details/6703247





