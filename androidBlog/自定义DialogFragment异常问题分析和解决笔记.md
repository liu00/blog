项目里用DialogFragment替代了AlertDialog作为加载提示窗口，本地测试显示关闭均正常。上线后错误日志一直有关于加载窗口异常上报，是窗口重复添加导致的抛出异常。扒了扒相关的代码，发现是show方法之前判断窗口是否添加的代码有逻辑问题，修复测试，一切OK，貌似是解决了Bug。修复版本上线，加载窗口的异常少了逐渐减少，新版没有上报这个异常。但最近一段时间里偶尔又有“IllegalStateException: Fragment already added”异常信息，看了看抛出异常的代码，好像时多次添加DialogFragment导致（可能是手机响应慢，导致多次点击时，第一点击后没有立即显示加载弹窗，导致又响应了之后的点击）。于是准备深扒一下Fragment相关的源码，主要是FragmentManagerImpl和BackStackRecord这两个类。

创建自定义的DialogFragment测试Demo。自定义一个LoadingDailog,并继承DialogFragment。写一个测试TestDFActivity,Bug代码如下：

```java
	public class TestDFActivity extends FragmentActivity {
		//加载弹窗
    	private LoadingDialog loadingDialog;

    	@Override
    	protected void onCreate(Bundle savedInstanceState) {
        	super.onCreate(savedInstanceState);
        	setContentView(R.layout.activity_frag_test);
        	loadingDialog = new LoadingDialog();
    	}

    	@Override
    	protected void onResume() {
        	super.onResume();
        	loadingDialog.show(getSupportFragmentManager(), getClass().getName());
    	}
	}
```
先看看DialogFragment.show()方法显示，方法内添加显示DialogFragment的代码，与普通的Fragment提交方式一样。

```java
	public void show(FragmentManager manager, String tag) {
        mDismissed = false;
        mShownByMe = true;
        FragmentTransaction ft = manager.beginTransaction();
        ft.add(this, tag);
        ft.commit();
    }
```

[请尊重博主劳动成果，转载请标明原文链接。](http://blog.csdn.net/hwliu51/article/details/69663937)

run一下代码，测试app运行，loading加载窗口弹出正常显示。
###一，模拟Activity被回收后恢复，窗口重复添加###
从“设置”进入“开发者选项”，将后“台进程限制”项设置为“不允许后台进程”，模拟当内存不足时Activity被回收重新显示。点击测试app显示TestDFActivity，按home键退出，然后点击其他app，再点击测试app。页面显示正常，点击back，消失了一个弹出，但页面还有一个弹窗。

看看FragmentActivity的onCreate()方法代码：

```java
	protected void onCreate(@Nullable Bundle savedInstanceState) {
        mFragments.attachHost(null /*parent*/);

        super.onCreate(savedInstanceState);

        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            mFragments.restoreLoaderNonConfig(nc.loaders);
        }
        if (savedInstanceState != null) {
            Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
			//恢复已保存的Fragment状态
            mFragments.restoreAllState(p, nc != null ? nc.fragments : null);
        }
        mFragments.dispatchCreate();
    }
```

当Activity执行到onCreate时，如果之前有保存过Fragment相关的状态信息，则会恢复Fragemt和其相关的状态信息。这就是[Activity中使用Fragment笔记](http://blog.csdn.net/hwliu51/article/details/69054521)View中动态添加小节强调在添加Fragment之前先判断是否存在Fragment的原因。

修改下loadingDialog创建方式，将直接new一个改为以下方式：

```java
	if (savedInstanceState == null) {
        loadingDialog = new LoadingDialog();
    } else {
        loadingDialog = (LoadingDialog) getSupportFragmentManager().findFragmentByTag(getClass().getName());
    }
```

运行，测试，加载弹窗重复添加问题解决。

###二，模拟多次点击，DialogFragment重复添加异常现象###

####TestCase：两次添加同一个Fragment，Tag不相同####
修改onResume()方法：

```java
	@Override
    protected void onResume() {
        super.onResume();
        loadingDialog.show(getSupportFragmentManager(), getClass().getName());
        loadingDialog.show(getSupportFragmentManager(), getClass().getSimpleName());
    }
```

运行，Activity闪退。查看log日志，异常关键部分信息：

    Caused by: java.lang.IllegalStateException: Can't change tag of fragment LoadingTip{1db4844a tag}: was tag now xx.xxActivity
	at android.support.v4.app.BackStackRecord.doAddOp(BackStackRecord.java:418)
	at android.support.v4.app.BackStackRecord.add(BackStackRecord.java:399)
	at android.support.v4.app.DialogFragment.show(DialogFragment.java:138)


抛出异常BackStackRecord.doAddOp()的代码：

```java
	private void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
        fragment.mFragmentManager = mManager;

        if (tag != null) {
            if (fragment.mTag != null && !tag.equals(fragment.mTag)) {
                throw new IllegalStateException("Can't change tag of fragment "
                        + fragment + ": was " + fragment.mTag
                        + " now " + tag);
            }
            fragment.mTag = tag;
        }

        if (containerViewId != 0) {
            if (fragment.mFragmentId != 0 && fragment.mFragmentId != containerViewId) {
                throw new IllegalStateException("Can't change container ID of fragment "
                        + fragment + ": was " + fragment.mFragmentId
                        + " now " + containerViewId);
            }
            fragment.mContainerId = fragment.mFragmentId = containerViewId;
        }

        Op op = new Op();
        op.cmd = opcmd;
        op.fragment = fragment;
        addOp(op);
    }
```
  
结论：**如果Fragment已添加，再次显示时则不能修改tag标志。**

####TestCase：两次添加同一个Fragment，Tag相同####
改用相同的tag，修改代码：

```java
	@Override
    protected void onResume() {
        super.onResume();
        loadingDialog.show(getSupportFragmentManager(), getClass().getName());
        loadingDialog.show(getSupportFragmentManager(), getClass().getName());
    }
```

运行，Activity闪退。查看log日志，异常关键部分信息：

    Caused by: java.lang.IllegalStateException: Fragment already added: LoadingTip{36e05fa1 #0 tag}
	at android.support.v4.app.FragmentManagerImpl.addFragment(FragmentManager.java:1278)
	at android.support.v4.app.BackStackRecord.run(BackStackRecord.java:671)
	at android.support.v4.app.FragmentManagerImpl.execPendingActions(FragmentManager.java:1572)
	at android.support.v4.app.FragmentController.execPendingActions(FragmentController.java:330)


抛出异常FragmentManagerImpl.addFragment()方法的代码：

```java
    public void addFragment(Fragment fragment, boolean moveToStateNow) {
        if (mAdded == null) {
            mAdded = new ArrayList<Fragment>();
        }
        if (DEBUG) Log.v(TAG, "add: " + fragment);
        makeActive(fragment);
        if (!fragment.mDetached) {
            if (mAdded.contains(fragment)) {
                throw new IllegalStateException("Fragment already added: " + fragment);
            }
            mAdded.add(fragment);
            fragment.mAdded = true;
            fragment.mRemoving = false;
            if (fragment.mHasMenu && fragment.mMenuVisible) {
                mNeedMenuInvalidate = true;
            }
            if (moveToStateNow) {
                moveToState(fragment);
            }
        }
    }
```

**使用commit提交后，Fragment便会被添加到FragmentManagerImpl的mAdded集合（为ArrayList<Fragment>类型)中。所以，第二次提交则会抛出异常。**

既然Fragment已被添加，那么为防止重复添加则需要在第二次显示之前判断是否已被添加。Fragment中isAdded()方法是常被用来判断是否被添加，先看看该方法代码：
```java
	final public boolean isAdded() {
        return mHost != null && mAdded;
    }
```
修改代码测试，在第二次添加之前使用isAdd判断：如果为false，则继续添加；为true，则返回。Run一把代码，依旧报错：Fragment already added。

isAdd方法也不靠谱？郁闷。

回头看看isAdd和addFragment方法源码，先捋一捋思路。
第一次调用show，最终调用addFragment添加到FragmentManagerImpl的mAdded列表中，然后对fragment.mAdded也设置为true。而之后的isAdded()返回false，那么应该是mHost为null。而mHost为null，根据[Activity中使用Fragment笔记](http://blog.csdn.net/hwliu51/article/details/69054521)第五节：Fragment获取Activity，可知在调用isAdd方法时Fragment没有进行初始化操作，对mHost进行赋值。再回头看看，IllegalStateException: Fragment already added。这个异常提供了另一种思路：既然Fragment被添加到FragmentManagerImpl的mAdded中，则可以通过tag或id来查找已提交的Fragment。

再次修改代码：
```java
	@Override
    protected void onResume() {
        super.onResume();
		String tag = getClass().getName();
        loadingDialog.show(getSupportFragmentManager(), tag);
        //让FragmentManagerImpl立即执行mPendingActions中的任务
        getSupportFragmentManager().beginTransaction();
        if (!loadingDialog.isAdded()) {
            if (getSupportFragmentManager().findFragmentByTag(tag) == null) {
                loadingDialog.show(getSupportFragmentManager(), tag);
            }
        }
    }
```
至于Fragment如何被添加到FragmentManager，可以阅读[Fragment之添加显示流程源码分析](http://blog.csdn.net/hwliu51/article/details/69841068)这篇博客。
测试，没有异常，显示加载弹窗，点击“back”，弹窗消失。Bug好像搞定了。修改项目加载提示弹窗类的显示方法中判断是否已添加逻辑代码，再测试，模拟连续两次点击（时间间隔极短）请求。显示一次加载提示弹窗，程序正常运行，再看看日志，第二次点击的请求在findFragmentByTag判断返回。多次测试，程序都正常运行。Bug终于解决！！！

### dismiss异常问题
这个异常偶尔会出现，主要出现在耗时的异步操作执行完毕后通知主线程执行dismiss时。异常主要的信息：
```java
java.lang.IllegalArgumentException: View=com.android.internal.policy.impl.PhoneWindow$DecorView{3e7fbc08 V.E..... R.....I. 0,0-960,231} not attached to window manager
at android.view.WindowManagerGlobal.findViewLocked(WindowManagerGlobal.java:416)
at android.view.WindowManagerGlobal.removeView(WindowManagerGlobal.java:342)
at android.view.WindowManagerImpl.removeViewImmediate(WindowManagerImpl.java:116)
at android.app.Dialog.dismissDialog(Dialog.java:354)
at android.app.Dialog.dismiss(Dialog.java:337)
```
其实这个异常不是DialogFragment本身的，而是内部dialog在调用dismiss时，WindowManagerGlobal的findViewLocked方法抛出的。

先讲解代码调用的流程。
DialogFragment#dismiss代码：

```java
    public void dismiss() {
        dismissInternal(false);
    }

    void dismissInternal(boolean allowStateLoss) {
        ...//
        if (mDialog != null) {
	        //执行Dialog的dismiss
            mDialog.dismiss();
            mDialog = null;
        }
        ...//省略
     }
```
因为dialog的mDecor被移除了，所以dismiss时在此移除便抛出了异常。解决方法：**在dismiss前使用isFinishing()判断当前Activity是否被finish，或者在onStop()或onPaus()方法中调用dismiss，再在onStart方法中判断是否需要显示。**具体分析可以阅读[Dialog显示和消失流程分析](http://blog.csdn.net/hwliu51/article/details/75040297)这篇博客。