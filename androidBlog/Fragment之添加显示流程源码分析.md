==*本文所引用的代码均为support-v4-23.0.1包中的源码，使用‘...’表示省略部分代码。*==

当在Activity的onCreate方法中通过一下方式添加Fragment，运行程序，便可以修饰在屏幕上。
```java
	FragmentManager fm = getSupportFragmentManager();
    FragmentTransaction transaction = fm.beginTransaction();
    transaction.add(resId, fragmentA);
    transaction.commit();
```
这几行代码让Fragment经历了怎样的一番旅途，最终被关联到宿主Activity并显示。我准备扒一扒support-v4-23.0.0的源码，探索这一段神奇的旅途。

[请尊重博主劳动成果，转载请标明原文链接。](http://blog.csdn.net/hwliu51/article/details/69841068)

##一 相关的类##

与此相关的类主要有10个，分别为：

**FragmentManger**为抽象类，定义对Fragment的操作行为，子类为**FragmentMangerImpl**。**FragmentMangerImpl**又实现了LayoutInflaterFactory接口，是真正对Fragment执行初始化，显示和移除等等操作的类。

**FragmentTransaction**为抽象类，定义对Fragment操作事务行为，子类为**BackStackRecord**。**BackStackRecord**又实现了Runnable和FragmentManger.BackStackEntry接口，是执行Fragment添加，显示，隐藏和移除等等事务的类。**Op**为**BackStackRecord**的静态内部类，当使用**FragmentTransaction**添加,替换,隐藏,移除等等操作便会在**BackStackRecord**对象中生成一个记录操作行为和Fragment的**Op**对象，该对象以链表的方式存储在**BackStackRecord**中。

**FragmentHostCallback<E>**为抽象类，又继承了**FragmentContainer**，子类为**FragmentActivity**的内部类**HostCallback**。

**FragmentController**封装对**FragmentMangerImpl**和**HostCallback**获取和操作的方法。

**FragmentActivity**用作**Fragment**的容器。

类图（Fragment的全家福）：
![这里写图片描述](http://img.blog.csdn.net/20170711215435590?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

##流程分析
###一，获取FragmentManager
在我们自定义的Activity中经常使用使用getSupportFragmentManager方法获取当前的FragmentManager（即FragmentMagagerImpl对象），那就先看看FragmentActivity类中的代码：
```java
public FragmentManager getSupportFragmentManager() {
	return mFragments.getSupportFragmentManager(); 
}
```
mFragments是什么类型，何时初始化与赋值？接着往FragmentActivity中查看：
```java
//在对象创建是便已赋值
final FragmentController mFragments = FragmentController.createController(new HostCallbacks());
```

创建HostCallbacks对象，那就看看它的构造方法代码：
```java
public HostCallbacks() {
    		//调用父类FragmentHostCallback的构造方法进行初始化
    		super(FragmentActivity.this /*fragmentActivity*/); 
}
```
接着继续看FragmentHostCallback代码：
```java
public abstract class FragmentHostCallback<E> extends FragmentContainer {
	//当前关联的FragmentActivity对象
	private final Activity mActivity;
	//当前关联的Context对象，其实就是FragmentActivity对象
	final Context mContext;
	//当前FragmentActivity的mHandler属性
	private final Handler mHandler;
	final int mWindowAnimations;
	//FragmentActivity获取的FragmentManager就是这个属性
	final FragmentManagerImpl mFragmentManager = new FragmentManagerImpl();
	//当前key为who，value为加载器LoaderManagerImpl
	private SimpleArrayMap<String, LoaderManager> mAllLoaderManagers;
	//
	private LoaderManagerImpl mLoaderManager; 	private boolean mCheckedForLoaderManager; 	private boolean mLoadersStarted;  	public FragmentHostCallback(Context context, Handler handler, int windowAnimations) {
		this(null /*activity*/, context, handler, windowAnimations);
	} 
	//FragmentActivity的内部类HostCallbacks调用赋值的构造方法
	FragmentHostCallback(FragmentActivity activity) {
		//使用activity给mContext赋值，activity.mHandler给mHandler赋值
		this(activity, activity /*context*/, activity.mHandler, 0 /*windowAnimations*/);
	} 
	//执行赋值
	FragmentHostCallback(Activity activity, Context context, Handler handler, int windowAnimations) {
		Activity = activity;
		mContext = context;
		mHandler = handler;
		mWindowAnimations = windowAnimations;
	}
	...
}
```
再往回看看FragmentActivity的onCreate方法：
```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
	//**这一步很重要**
	//将FragmentController的mHost赋值给HostCallBack的mFragmentManager，便完成了当前Activity与FramgmentManger关联。
	mFragments.attachHost(null /*parent*/);
	super.onCreate(savedInstanceState); 
	NonConfigurationInstances nc = (NonConfigurationInstances) getLastNonConfigurationInstance();
	if (nc != null) {
		mFragments.restoreLoaderNonConfig(nc.loaders);
	}
	if (savedInstanceState != null) {
		Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
		mFragments.restoreAllState(p, nc != null ? nc.fragments : null);
	}
	//通知mFragmentManager当前Activity执行了onCreate
	mFragments.dispatchCreate();
}
```

FragmentController中attachHost和dispatchCreate方法代码：
```java
public void attachHost(Fragment parent) {
	mHost.mFragmentManager.attachController(mHost, mHost /*container*/, parent);
}

public void dispatchCreate() {    
	mHost.mFragmentManager.dispatchCreate(); 
}
```

调用FragmentManagerImpl相关方法和属性的代码：
```java
public void attachController(FragmentHostCallback host, FragmentContainer container, Fragment parent) {
	if (mHost != null) throw new IllegalStateException("Already attached");
	//将FragmentController中的mHost赋值给FragmentManagerImpl的mHost，自此便完成当前Activity与FramgmentManger关联
	mHost = host;
	mContainer = container;
	mParent = parent; }

//FragmentManagerImpl当前状态值，默认为初始化
int mCurState = Fragment.INITIALIZING;

public void dispatchCreate() {
	mStateSaved = false;
	//更新当前状态mCurState为Fragment.CREATED
	moveToState(Fragment.CREATED, false);
}

void moveToState(int newState, boolean always) {
	moveToState(newState, 0, 0, always);
}

void moveToState(int newState, int transit, int transitStyle, boolean always) {
	if (mHost == null && newState != Fragment.INITIALIZING) {
		throw new IllegalStateException("No host");
	}
	if (!always && mCurState == newState) {
		return;
	}
	//更新当前状态mCurState为Fragment.CREATED
	mCurState = newState;
	//当从FragmentActivity的onCreate执行到此时，由于还未有添加Fragment，则mActive为null，便返回。
	//执行完这些代码后，FragmentManagerImpl的mCurState为Fragment.CREATED
	if (mActive != null) {
		boolean loadersRunning = false;
		for (int i=0; i<mActive.size(); i++) {
			Fragment f = mActive.get(i);
			if (f != null) {
				moveToState(f, newState, transit, transitStyle, false);
				...
			}
		}
		...
	}
}
```
当Activity的onCreate方法执行完super.onCreate()，我们便能够通过getSupportFragmentManager方法获取到一个可以正常使用的FragmentManagerImpl对象。

###二，获取FragmentTransaction
调用FragmentManager的beginTransaction()方法获取FragmentTransaction。看看代码，究竟执行了哪些操作。
```java
	public FragmentTransaction beginTransaction() {
		//创建了一个BackStackRecord对象，并把自己作为参数传递给它。
		return new BackStackRecord(this);
	}


	public BackStackRecord(FragmentManagerImpl manager){
		//对mManager进行赋值，完成BackStackRecord与FragmentManagerImpl关联
		mManager = manager;
	}
```
###三，添加Fragment
调用FragmentTransaction的add方法添加Fragment又执行了BackStackRecord哪些方法？继续看代码：
```java
	//添加Fragment并绑定tag
	public FragmentTransaction add(Fragment fragment, String tag) {
        doAddOp(0, fragment, tag, OP_ADD);
        return this;
    }

	 //向指定id的View作为容器，添加Fragment
    public FragmentTransaction add(int containerViewId, Fragment fragment) {
        doAddOp(containerViewId, fragment, null, OP_ADD);
        return this;
    }

	 //指定View容器和tag，添加Fragment
    public FragmentTransaction add(int containerViewId, Fragment fragment, String tag) {
        doAddOp(containerViewId, fragment, tag, OP_ADD);
        return this;
    }

	private void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
		//将与Activity关联的FragmentMangerImpl对象赋值给Fragment
        fragment.mFragmentManager = mManager;
        //如果Fragment的tag属性已赋值，则添加时指定的tag必须要与其相同
        if (tag != null) {
            if (fragment.mTag != null && !tag.equals(fragment.mTag)) {
                throw new IllegalStateException("Can't change tag of fragment "
                        + fragment + ": was " + fragment.mTag
                        + " now " + tag);
            }
            fragment.mTag = tag;
        }

			//如果Fagment已关联显示容器的Id（即在View中显示），则添加时不能添加到其他的View
        if (containerViewId != 0) {
            if (fragment.mFragmentId != 0 && fragment.mFragmentId != containerViewId) {
                throw new IllegalStateException("Can't change container ID of fragment "
                        + fragment + ": was " + fragment.mFragmentId
                        + " now " + containerViewId);
            }
            fragment.mContainerId = fragment.mFragmentId = containerViewId;
        }
		//创建Op，指定执行操作类型和关联Fragment
        Op op = new Op();
        op.cmd = opcmd;
        op.fragment = fragment;
        addOp(op);
    }

	//将Op对象加入到Op链表中
	void addOp(Op op) {
        if (mHead == null) {
            mHead = mTail = op;
        } else {
        	//加入到Op链表尾部
            op.prev = mTail;
            mTail.next = op;
            mTail = op;
        }
        ...
        mNumOp++;
    }
```
自此Fragment便被添加到FragmentTransaction（即BackStackRecord对象中）。

###四，提交Fragment添加事务
调用FragmentTransaction的commit方法，便提交Fragment的添加事务。我们只要运行代码，Fragment便会显示在屏幕上。commit操作执行了哪些方法，commit之后Actvity执行onStart和onResume时Fragment又经历了哪些操作。继续看代码分析。

```java
	public int commit() {
		return commitInternal(false);
	}

	public int commitAllowingStateLoss() {
		return commitInternal(true);
	}
    
    int commitInternal(boolean allowStateLoss) {
        if (mCommitted) throw new IllegalStateException("commit already called");
        if (FragmentManagerImpl.DEBUG) {
            Log.v(TAG, "Commit: " + this);
            LogWriter logw = new LogWriter(TAG);
            PrintWriter pw = new PrintWriter(logw);
            dump("  ", null, pw, null);
        }
        mCommitted = true;
        if (mAddToBackStack) {
            mIndex = mManager.allocBackStackIndex(this);
        } else {
            mIndex = -1;
        }
        //将创建的BackStackRecord对象，提交给FragmentManagerImpl
        mManager.enqueueAction(this, allowStateLoss);
        return mIndex;
    }
```
因为BackStackRecord实现了Runnable，故在FragmentManagerImpl定义一个ArrayList<Runnable>类型的集合mPendingActions来存储添加的BackStackRecord对象。在看看enqueueAction以及与其相关的方法和属性的代码：
```java
	private void checkStateLoss() {
		//如果在已执行了保存状态信息操作后执行commit，则抛出异常
        if (mStateSaved) {
            throw new IllegalStateException(
                    "Can not perform this action after onSaveInstanceState");
        }
        if (mNoTransactionsBecause != null) {
            throw new IllegalStateException(
                    "Can not perform this action inside of " + mNoTransactionsBecause);
        }
    }
	public void enqueueAction(Runnable action, boolean allowStateLoss) {
		//是否允许状态信息丢失
        if (!allowStateLoss) {
        	//检查状态
            checkStateLoss();
        }
        synchronized (this) {
            if (mDestroyed || mHost == null) {
                throw new IllegalStateException("Activity has been destroyed");
            }
            if (mPendingActions == null) {
                mPendingActions = new ArrayList<Runnable>();
            }
            //将提交的BackStackRecord添加到集合中
            mPendingActions.add(action);
            if (mPendingActions.size() == 1) {
				//移除还未执行的Runnalbe：mExecCommit
				mHost.getHandler().removeCallbacks(mExecCommit);
                //发送一个延时执行的Runnalbe：mExecCommit
                mHost.getHandler().post(mExecCommit);
            }
        }
    }
    
    //延时执行的Runnable对象
    Runnable mExecCommit = new Runnable() {
        @Override
        public void run() {
        	//运行mPendingActions中的Runnable对象
            execPendingActions();
        }
    };
```
从enqueueAction中第一个判断的checkStateLoss方法代码可知：**不能在Activity执行onSaveInstanceState操作后再调用commit。如果提交的Fragment显示重要的信息，建议也尽量不要使用commitAllowingStateLoss。因为执行onSaveInstanceState时会保存所有Fragment的状态信息 ，在其后执行commitAllowingStateLoss则会丢失提交的Fragment信息，导致状态恢复时出现一些很奇怪的现象。**

执行commit提交的BackStackRecord在enqueueAction方法里并没有被直接执行，而是存储在mPendingActions中等待执行。同时，会移除正在Message队列中等待执行的mExecCommit，并使用FragmentActivity的mHandler发送新的的延时执行信息mExecCommit。mExecCommit的run方法调用了execPendingActions，也就是说真正执行BackStackRecord的是execPendingActions。execPendingActions方法中有一个while(true)循环来遍历mPendingActions中的Runnable并执行run方法，直至mPendingActions为空集合才退出。

execPendingActions方法代码：
```java
	/**
     * Only call from main thread!
     */
    public boolean execPendingActions() {
        if (mExecutingActions) {
            throw new IllegalStateException("Recursive entry to executePendingTransactions");
        }
        //不能在非主线程中调用此方法
        if (Looper.myLooper() != mHost.getHandler().getLooper()) {
            throw new IllegalStateException("Must be called from main thread of process");
        }

        boolean didSomething = false;
		//采用无限循环可以让在执行execPendingActions时提交的BackStackRecord被及时执行run方法。
        while (true) {
            int numActions;
            
            synchronized (this) {
            	//如果mPendingActions为null（即未提交BackStackRecord），或mPendingActions为空集合（即所有的BackStackRecord都执行完），则退出循环。
                if (mPendingActions == null || mPendingActions.size() == 0) {
                    break;
                }
                
                numActions = mPendingActions.size();
                if (mTmpActions == null || mTmpActions.length < numActions) {
                    mTmpActions = new Runnable[numActions];
                }
                //将mPendingActions集合中的数据存储到mTmpActions数组
                mPendingActions.toArray(mTmpActions);
                //清空集合
                mPendingActions.clear();
                //移除延时执行消息
                mHost.getHandler().removeCallbacks(mExecCommit);
            }
            
            mExecutingActions = true;
            for (int i=0; i<numActions; i++) {
            	//**这一步执行很重要**
				//执行BackStackRecord的run方法
                mTmpActions[i].run();
                mTmpActions[i] = null;
            }
            mExecutingActions = false;
            didSomething = true;
        }
        
        ...
        return didSomething;
    }
```
关键步骤mTmpActions[i].run()执行的是BackStackRecord的run方法。其代码：
```java
	public void run() {
        ...

		//循环执行Op链表中的Op对象
        Op op = mHead;
        while (op != null) {
            int enterAnim = state != null ? 0 : op.enterAnim;
            int exitAnim = state != null ? 0 : op.exitAnim;
            switch (op.cmd) {
            	//添加Fragment
                case OP_ADD: {
                    Fragment f = op.fragment;
                    f.mNextAnim = enterAnim;
                    mManager.addFragment(f, false);
                } break;
                //替换Fragment
                case OP_REPLACE: {
                    Fragment f = op.fragment;
                    int containerId = f.mContainerId;
                    if (mManager.mAdded != null) {
                        for (int i=0; i<mManager.mAdded.size(); i++) {
                            Fragment old = mManager.mAdded.get(i);
                            if (FragmentManagerImpl.DEBUG) Log.v(TAG,
                                    "OP_REPLACE: adding=" + f + " old=" + old);
                            if (old.mContainerId == containerId) {
                                if (old == f) {
                                    op.fragment = f = null;
                                } else {
                                    if (op.removed == null) {
                                        op.removed = new ArrayList<Fragment>();
                                    }
                                    op.removed.add(old);
                                    old.mNextAnim = exitAnim;
                                    if (mAddToBackStack) {
                                        old.mBackStackNesting += 1;
                                        if (FragmentManagerImpl.DEBUG) Log.v(TAG, "Bump nesting of "
                                                + old + " to " + old.mBackStackNesting);
                                    }
                                    mManager.removeFragment(old, transition, transitionStyle);
                                }
                            }
                        }
                    }
                    if (f != null) {
                        f.mNextAnim = enterAnim;
                        mManager.addFragment(f, false);
                    }
                } break;
                //
                case OP_REMOVE: {
                    Fragment f = op.fragment;
                    f.mNextAnim = exitAnim;
                    mManager.removeFragment(f, transition, transitionStyle);
                } break;
                //隐藏Fragment
                case OP_HIDE: {
                    Fragment f = op.fragment;
                    f.mNextAnim = exitAnim;
                    mManager.hideFragment(f, transition, transitionStyle);
                } break;
                //显示Fragment
                case OP_SHOW: {
                    Fragment f = op.fragment;
                    f.mNextAnim = enterAnim;
                    mManager.showFragment(f, transition, transitionStyle);
                } break;
                //detach Fragment
                case OP_DETACH: {
                    Fragment f = op.fragment;
                    f.mNextAnim = exitAnim;
                    mManager.detachFragment(f, transition, transitionStyle);
                } break;
                //attach Fragment
                case OP_ATTACH: {
                    Fragment f = op.fragment;
                    f.mNextAnim = enterAnim;
                    mManager.attachFragment(f, transition, transitionStyle);
                } break;
                default: {
                    throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
                }
            }
			//执行下一个Op对象
            op = op.next;
        }
		//调用FragmentManagerImpl的moveToState，更新当前Fragment状态
        mManager.moveToState(mManager.mCurState, transition, transitionStyle, true);

        ...
    }
```
使用FragmentTransaction的add方法添加Fragment时，会创建Op对象，cmd为OP_ADD。执行到此run方法时，则会进入case OP_ADD，在其中调用了FragmentManagerImpl的addFragment方法添加提交的Fragment。回顾一下之前FragmentActivity的onCreate中的介绍，当我们在Activity的onCreate方法代码super.onCreate后调用getSupportFragmentManager时，与Actvity关联的**FragmentManagerImpl对象的mCurState已被更新为Fragment.CREATED**。此时被提交的**fragment的mState属性还是Fragment.INITIALIZING**默认值。

继续查看FragmentManagerImpl的addFragment方法和与其相关联的makeActive方法代码：
```java
	void makeActive(Fragment f) {
        if (f.mIndex >= 0) {
            return;
        }
        
        if (mAvailIndices == null || mAvailIndices.size() <= 0) {
            if (mActive == null) {
                mActive = new ArrayList<Fragment>();
            }
            //设置其位置和父Fragment（在Activity中添加的，mParent为null；只有在Fragment中添加的才有Parent Fragment）。
            f.setIndex(mActive.size(), mParent);
            //添加到mActive集合中
            mActive.add(f);
            
        } else {
            //更新信息
            f.setIndex(mAvailIndices.remove(mAvailIndices.size()-1), mParent);
            mActive.set(f.mIndex, f);
        }
        ...
    }

	public void addFragment(Fragment fragment, boolean moveToStateNow) {
        if (mAdded == null) {
            mAdded = new ArrayList<Fragment>();
        }
        if (DEBUG) Log.v(TAG, "add: " + fragment);
        //添加到mActive或更新信息
        makeActive(fragment);
        if (!fragment.mDetached) {
        	//如果已添加到mAdd，则抛出异常
            if (mAdded.contains(fragment)) {
                throw new IllegalStateException("Fragment already added: " + fragment);
            }
            //添加到mAdded
            mAdded.add(fragment);
			//修改标志mAdded为true（即已添加）
            fragment.mAdded = true;
            fragment.mRemoving = false;
            ...
            if (moveToStateNow) {
            	//更新fragment状态，并执行相应操作
                moveToState(fragment);
            }
        }
    }
```
因为BackStackRecord的run方法中case OP_ADD代码块中的addFragment的moveToStateNow为false，所以if (moveToStateNow)不成立，即不会进入moveToState(fragment)。待BackStackRecord的run方法中的Op链表的数据被执行完，便会执行底部代码：mManager.moveToState(mManager.mCurState, transition, transitionStyle, true)。此时**mManager.mCurState为Fragment.CREATED**，而**fragment.mState为Fragment.INITIALIZING**。

初始化Fragment：
```java
	void moveToState(int newState, int transit, int transitStyle, boolean always) {
        …
        mCurState = newState;
        if (mActive != null) {
            boolean loadersRunning = false;
            //遍历mActive集合
            for (int i=0; i<mActive.size(); i++) {
            	//获取Fragment
                Fragment f = mActive.get(i);
                if (f != null) {
                	//执行更新操作并更新f.mState
                    moveToState(f, newState, transit, transitStyle, false);
                    …
                }
            }

            …
        }
    }
```
调用mManager.moveToState传入newState值与mCurState相同，mCurState值没有被改变。执行moveToState(f, newState, transit, transitStyle, false)代码。还是继续看代码，分析。

先看看Fragment的几种状态值：
```java
	static final int INITIALIZING = 0;     // Not yet created.
    static final int CREATED = 1;          // Created.
    static final int ACTIVITY_CREATED = 2; // The activity has finished its creation.
    static final int STOPPED = 3;          // Fully created, not started.
    static final int STARTED = 4;          // Created and started, not resumed.
    static final int RESUMED = 5;          // Created started and resumed.
```
**当前FragmentManagerImpl的mCurState为Fragment.CREATED（即1），而mAtive中的Fragment的mState为Fragment.INITIALIZING（即0）。**
```java
	void moveToState(Fragment f, int newState, int transit, int transitionStyle, boolean keepActive) {
        ...
        
        if (f.mState < newState) {
            // For fragments that are created from a layout, when restoring from
            // state we don't want to allow them to be created until they are
            // being reloaded from the layout.
            //如果f是从布局文件创建的，当恢复状态时，没有执行执行从布局重新加载操作，则返回
            if (f.mFromLayout && !f.mInLayout) {
                return;
            }  
            ...
            switch (f.mState) {
            	//执行Fragment初始化
                case Fragment.INITIALIZING:
                    //如果有保存状态信息，则恢复
                    if (f.mSavedFragmentState != null) {
                        f.mSavedFragmentState.setClassLoader(mHost.getContext().getClassLoader());
                        f.mSavedViewState = f.mSavedFragmentState.getSparseParcelableArray(
                                FragmentManagerImpl.VIEW_STATE_TAG);
                        f.mTarget = getFragment(f.mSavedFragmentState,
                                FragmentManagerImpl.TARGET_STATE_TAG);
                        if (f.mTarget != null) {
                            f.mTargetRequestCode = f.mSavedFragmentState.getInt(
                                    FragmentManagerImpl.TARGET_REQUEST_CODE_STATE_TAG, 0);
                        }
                        f.mUserVisibleHint = f.mSavedFragmentState.getBoolean(
                                FragmentManagerImpl.USER_VISIBLE_HINT_TAG, true);
                        if (!f.mUserVisibleHint) {
                            f.mDeferStart = true;
                            if (newState > Fragment.STOPPED) {
                                newState = Fragment.STOPPED;
                            }
                        }
                    }
                    //mHost赋值FragemntContooler，即与Activity关联
                    f.mHost = mHost;
                    //设置父Fragemnt（默认为null，只有子Fragemnt才有值）
                    f.mParentFragment = mParent;
                    //关联FragmentManagerImpl
                    f.mFragmentManager = mParent != null
                            ? mParent.mChildFragmentManager : mHost.getFragmentManagerImpl();
                    f.mCalled = false;
                    f.onAttach(mHost.getContext());
                    if (!f.mCalled) {
                        throw new SuperNotCalledException("Fragment " + f
                                + " did not call through to super.onAttach()");
                    }
                    if (f.mParentFragment == null) {
                        mHost.onAttachFragment(f);
                    }

                    if (!f.mRetaining) {
                        f.performCreate(f.mSavedFragmentState);
                    }
                    f.mRetaining = false;
                    //从布局文件创建的Fragemnt
                    if (f.mFromLayout) {
                        // For fragments that are part of the content view
                        // layout, we need to instantiate the view immediately
                        // and the inflater will take care of adding it.
                        f.mView = f.performCreateView(f.getLayoutInflater(
                                f.mSavedFragmentState), null, f.mSavedFragmentState);
                        if (f.mView != null) {
                            f.mInnerView = f.mView;
                            if (Build.VERSION.SDK_INT >= 11) {
                                ViewCompat.setSaveFromParentEnabled(f.mView, false);
                            } else {
                                f.mView = NoSaveStateFrameLayout.wrap(f.mView);
                            }
                            if (f.mHidden) f.mView.setVisibility(View.GONE);
                            f.onViewCreated(f.mView, f.mSavedFragmentState);
                        } else {
                            f.mInnerView = null;
                        }
                    }
                //创建Fragment的View
                case Fragment.CREATED:
                    if (newState > Fragment.CREATED) {
                        if (DEBUG) Log.v(TAG, "moveto ACTIVITY_CREATED: " + f);
                        //非布局文件创建，即直接new
                        if (!f.mFromLayout) {
                            ViewGroup container = null;
                            if (f.mContainerId != 0) {
                                container = (ViewGroup)mContainer.onFindViewById(f.mContainerId);
                                if (container == null && !f.mRestored) {
                                    throwException(new IllegalArgumentException(
                                            "No view found for id 0x"
                                            + Integer.toHexString(f.mContainerId) + " ("
                                            + f.getResources().getResourceName(f.mContainerId)
                                            + ") for fragment " + f));
                                }
                            }
                            f.mContainer = container;
                            //执行创建View
                            f.mView = f.performCreateView(f.getLayoutInflater(
                                    f.mSavedFragmentState), container, f.mSavedFragmentState);
                            if (f.mView != null) {
                                f.mInnerView = f.mView;
                                if (Build.VERSION.SDK_INT >= 11) {
                                    ViewCompat.setSaveFromParentEnabled(f.mView, false);
                                } else {
                                    f.mView = NoSaveStateFrameLayout.wrap(f.mView);
                                }
                                if (container != null) {
                                    Animation anim = loadAnimation(f, transit, true,
                                            transitionStyle);
                                    if (anim != null) {
                                        setHWLayerAnimListenerIfAlpha(f.mView, anim);
                                        f.mView.startAnimation(anim);
                                    }
                                    container.addView(f.mView);
                                }
                                if (f.mHidden) f.mView.setVisibility(View.GONE);
                                f.onViewCreated(f.mView, f.mSavedFragmentState);
                            } else {
                                f.mInnerView = null;
                            }
                        }

                        f.performActivityCreated(f.mSavedFragmentState);
                        if (f.mView != null) {
                            f.restoreViewState(f.mSavedFragmentState);
                        }
                        f.mSavedFragmentState = null;
                    }
                //执行Fragment的start操作
                case Fragment.ACTIVITY_CREATED:
                case Fragment.STOPPED:
                    if (newState > Fragment.STOPPED) {
                        if (DEBUG) Log.v(TAG, "moveto STARTED: " + f);
                        f.performStart();
                    }
                //显示Fragemnt
                case Fragment.STARTED:
                    if (newState > Fragment.STARTED) {
                        if (DEBUG) Log.v(TAG, "moveto RESUMED: " + f);
                        f.mResumed = true;
                        f.performResume();
                        f.mSavedFragmentState = null;
                        f.mSavedViewState = null;
                    }
            }
        } else if (f.mState > newState) {
            ...
        }
        //更新fragment的mState值
        f.mState = newState;
    }
```
执行moveToState(f, newState, transit, transitStyle, false)代码时，传入的newState为1，而f.mState为0，则进入case Fragment.INITIALIZING代码块。待执行完Fragment.INITIALIZING代码块，由于f.mState值比Fragment.CREATED，Fragment.ACTIVITY_CREATED，Fragment.STOPPED和Fragment.STARTED都小，则无法进入这些代码块。最终执行f.mState = newState，更新f.mState为Fragment.CREATED。

到此时Activity的代码也刚执行完onCreate，接着执行onStart，onResume并显示。接着看FragemntActivity中的这些方法代码：
```java
	final Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                ...
                case MSG_RESUME_PENDING:
                    onResumeFragments();
                    //遍历并执行execPendingActions中BackStackRecord
                    mFragments.execPendingActions();
                    break;
                default:
                    super.handleMessage(msg);
            }
        }

    };
    
	protected void onStart() {
        super.onStart();
        ...
        if (!mCreated) {
            mCreated = true;
            //更新状态为ActivityCreated
            mFragments.dispatchActivityCreated();
        }

        mFragments.noteStateNotSaved();
        //遍历并执行execPendingActions中BackStackRecord
        mFragments.execPendingActions();

        mFragments.doLoaderStart();
        //更新状态为Start
        mFragments.dispatchStart();
        mFragments.reportLoaderStart();
    }
    
	protected void onResume() {
        super.onResume();
        //发送执行execPendingActions消息
        mHandler.sendEmptyMessage(MSG_RESUME_PENDING);
        mResumed = true;
        //遍历并执行execPendingActions中BackStackRecord
        mFragments.execPendingActions();
    }
```
FragmentController中execPendingActions，dispatchActivityCreated和dispatchStart代码：
```
	public void dispatchActivityCreated() {
        mHost.mFragmentManager.dispatchActivityCreated();
    }

	public void dispatchStart() {
        mHost.mFragmentManager.dispatchStart();
    }

	public void dispatchResume() {
        mHost.mFragmentManager.dispatchResume();
    }
    
	public boolean execPendingActions() {
        return mHost.mFragmentManager.execPendingActions();
    }
```
FragmentController并没有做什么事情，而是FragmentManagerImpl执行了对应的操作。再看看FragmentManagerImpl中这些方法的代码。execPendingActions方法代码在上文有展示，此处便不再列出。

```java
	public void dispatchActivityCreated() {
        mStateSaved = false;
        moveToState(Fragment.ACTIVITY_CREATED, false);
    }
    
    public void dispatchStart() {
        mStateSaved = false;
        moveToState(Fragment.STARTED, false);
    }
    
    public void dispatchResume() {
        mStateSaved = false;
        moveToState(Fragment.RESUMED, false);
    }
```
当执行moveToState(Fragment.ACTIVITY_CREATED, false)，  则FragmentManagerImpl的mCurState更新为ACTIVITY_CREATED，Fragment便会进入到case Fragment.CREATED代码块，执行create相关操作。依次类推Activity执行onResume，fragment执行performResume()即可以显示在屏幕上。

    
##Fragment添加子Fragment
使用getChildFragmentManager获取FragmentManager。

```java
final public FragmentManager getChildFragmentManager() {
        if (mChildFragmentManager == null) {
            instantiateChildFragmentManager();
            if (mState >= RESUMED) {
                mChildFragmentManager.dispatchResume();
            } else if (mState >= STARTED) {
                mChildFragmentManager.dispatchStart();
            } else if (mState >= ACTIVITY_CREATED) {
                mChildFragmentManager.dispatchActivityCreated();
            } else if (mState >= CREATED) {
                mChildFragmentManager.dispatchCreate();
            }
        }
        return mChildFragmentManager;
    }

void instantiateChildFragmentManager() {
        mChildFragmentManager = new FragmentManagerImpl();
        mChildFragmentManager.attachController(mHost, new FragmentContainer() {
            @Override
            @Nullable
            public View onFindViewById(int id) {
                if (mView == null) {
                    throw new IllegalStateException("Fragment does not have a view");
                }
                return mView.findViewById(id);
            }

            @Override
            public boolean onHasView() {
                return (mView != null);
            }
        }, this);
    }

```
管理子Fragment的mChildFragmentManager的mHost与Activity中mHost是同一个对象。调用getChildFragmentManager方法时，也会更新mChildFragmentManager的mCurState状态和子Fragment的状态。更新流程与上文所讲述的一致。
