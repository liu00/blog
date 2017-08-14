###一 Activity中添加Fragment方式
 

####（1）Xml布局文件添加，如下

```xml
<fragment
    android:id="@+id/fragment"
    android:name="xxx.xxx.MyFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

####（2）View中动态添加
```java
Fragment fragment = getSupportFragmentManager().findFragmentByTag("tag");
if(fragment == null){
	FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();
    fragment = MyFragment.newInstance();
    transaction.add(R.id.fragment_container, fragment, "tag");
    transaction.commit();
}

```
或

```java
Fragment fragment = null;
if(savedInstanceState == null){
	fragment = MyFragment.newInstance();
    FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();
    transaction.add(R.id.fragment_container, fragment, "tag");
    transaction.commit();
}else{
    fragment = getSupportFragmentManager().findFragmentByTag("tag");
}
```

**之所以在添加Fragment前判断是否有存在相同的Fragment，是因为在内存不足或横竖屏切换（如未在android:configChanges设置）时，会保存其内的Fragment状态信息。当Activity被创建显示时，Fragment会被自动恢复（恢复Fragment和状态的代码在Fragment Activity的onCreate方法中）。如果不判断，则添加多个相同的Fragment。**
在添加Fragment时，如果需要向其传递一些初始化的参数，则可使用Bundle来传递。在Fragment内使用getArguments()获取参数。代码如下：

```java
//------ 传递参数 ------
Bundle data = new Bundle();
//除了基本数据类型的值，还可以传递序列化和实现Parcelable类型的数据
data.putString(key, value);
...
fragment.setArguments(data);

//------ 获取参数 ------
Bundle data = getArguments();
if(data != null){
	//获取参数
	String value = data.getString(key);
	...
}
```

 
####（3）使用TabLayout和ViewPager添加
在布局文件添加：

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <android.support.design.widget.TabLayout
        android:id="@+id/tab_layout"
        android:layout_width="match_parent"
        android:layout_height="40dp" />

    <android.support.v4.view.ViewPager
        android:id="@+id/view_pager"
        android:layout_below="@+id/tab_layout"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
</RelativeLayout>
```
给ViewPager设置Fragment适配器和TabLayout绑定ViewPager：

```java
TabLayout table_layout = (TabLayout) findViewById(R.id.tab_layout);
ViewPager view_pager = (ViewPager) findViewById(R.id.view_pager);
//设置自定义的Fragment适配器
view_pager.setAdapter(new MyFragAdapter(getSupportFragmentManager()));
//TabLayout绑定ViewPager
table_layout.setupWithViewPager(view_pager);
table_layout.setTabMode(TabLayout.MODE_FIXED);
```
自定义的Fragment适配器必须要继承FragmentStatePagerAdapter或FragmentPagerAdapter，而这两个类则继承了android.support.v4.view.PagerAdapter。FragmentStatePagerAdapter与FragmentPagerAdapter区别则体现在Fragment的添加，移除和状态保存这三个方面。FragmentStatePagerAdapter添加Fragment时会缓存到mFragments集合中，并恢复其状态（如果有保存状态），而当不可见时则会先使用Fragment.SavedState来保存其状态信息，将其在mFragments的缓存置为null，然后直接从FragmentManager中remove掉。而FragmentPagerAdapter直接使用FragmentManager管理Fragment缓存，当不可见时，只会调用detach将其与Activity分离。
FragmentStatePagerAdapter的instantiateItem和destroyItem方法代码（使用...省略部分代码）：

```java
@Override
public Object instantiateItem(ViewGroup container, int position) {
	...
    //本地缓存有（非FragmentManager缓存），则直接返回
	if (mFragments.size() > position) {
         Fragment f = mFragments.get(position);
         if (f != null) {
             return f;
         }
    }
	...
    //创建实例
    Fragment fragment = getItem(position);
    ...
    if (mSavedState.size() > position) {
        Fragment.SavedState fss = mSavedState.get(position);
        //恢复保存的状态信息
        if (fss != null) {
            fragment.setInitialSavedState(fss);
         }
    }
	//增加本地缓存
	while (mFragments.size() <= position) {
          mFragments.add(null);
	}
    ...
    //添加到本地缓存
    mFragments.set(position, fragment);
    mCurTransaction.add(container.getId(), fragment);
	return fragment;
}

@Override
public void destroyItem(ViewGroup container, int position, Object object) {
	...
	//增加状态缓存
	while (mSavedState.size() <= position) {
          mSavedState.add(null);
	}
	//判断是否保存Fragment状态信息
    mSavedState.set(position, fragment.isAdded()
                ? mFragmentManager.saveFragmentInstanceState(fragment) : null);
    //清除Fragment对象缓存
    mFragments.set(position, null);
    //从FragmentManager中移除
	mCurTransaction.remove(fragment);
}
...
@Override
public Parcelable saveState() {
	Bundle state = null;
	//保存状态信息
	...
	return state;
}

@Override
public void restoreState(Parcelable state, ClassLoader loader) {
	if (state != null) {
		//恢复状态信息
        ...
     }
}
```

FragmentPagerAdapter的instantiateItem和destroyItem方法代码：

```java
@Override
public Object instantiateItem(ViewGroup container, int position) {
	...
    final long itemId = getItemId(position);
    String name = makeFragmentName(container.getId(), itemId);
    //获取FragmentManager缓存
    Fragment fragment = mFragmentManager.findFragmentByTag(name);
    if (fragment != null) {
        //attach到当前Activity
        mCurTransaction.attach(fragment);
    } else {
        //创建实例
        fragment = getItem(position);
        mCurTransaction.add(container.getId(), fragment, makeFragmentName(container.getId(), itemId));
    }
    if (fragment != mCurrentPrimaryItem) {
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
    }
    return fragment;
}

@Override
public void destroyItem(ViewGroup container, int position, Object object) {
	if (mCurTransaction == null) {
    mCurTransaction = mFragmentManager.beginTransaction();
    }
    mCurTransaction.detach((Fragment)object);
}
...
@Override
public Parcelable saveState() {
	//默认未做状态信息保存
	return null;
}

@Override
public void restoreState(Parcelable state, ClassLoader loader) {
	//默认未恢复状态信息
}
```
由这几个方法对比可知：

**FragmentStatePagerAdapter适用于管理大数量Fragemnt，如新闻页面，根据不同的频道显示对应的新闻信息，占用内存较少，保存状态信息有利于再次快速回显数据。**
**FragmentPagerAdapter使用少量的固定的Tab页面，所有被显示的Fragment都会保存在内存中，这使得切换页面时显示更快，同时也会占用一定的内存。**

在使用ViewPager作为Fragment容器时，需要根据显示Fragmet的数量来使用Adapter，以免占用过多的内存或切换页面时内容显示缓慢。


###二 Fragment中使用Fragment
Fragment中添加子Fragment的方式与Activity中添加方式基本相同，只是获取FragmentManager对象的方法不同。Activity中使用getSupportFragmentManager()方法获取，Fragment中需要使用getChildFragmentManager()，而不是调用getFragmentManager()。

Fragment的getFragmentManager()代码注释：

> Return the FragmentManager for interacting with fragments associated with this fragment's activity

获取与Fragment关联的Activity交互的FragmentManager。

Fragment的getFragmentManager()代码注释：
>Return a private FragmentManager for placing and managing Fragments inside of this Fragment.

获取Fragment内一个管理子Fragment的私有的FragmentManager。

子Fragment使用startActivityForResult()方法在低版本时onActivityResult得不到执行完成的回调问题。
先看看android 4.4源码中的FragmentActivity的onActivityResult代码：
```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
	mFragments.noteStateNotSaved();
	int index = requestCode>>16;
	if (index != 0) {
	index--;
	...
	//mFragments为FragmentManagerImpl实例，通过getSupportFragmentManager()获取的即该实例。
	Fragment frag = mFragments.mActive.get(index);
	if (frag == null) {
		...
	} else {
		frag.onActivityResult(requestCode&0xffff, resultCode, data);
	}
	return;
	}
	super.onActivityResult(requestCode, resultCode, data);
}
```
通过以上注释可知：
**FragmentActivity的onActivityResult方法只查找了通过getSupportFragmentManager()关联的Fragment，而通过getFragmentManager()与Fragment关联的子Fragment则直接被忽略了，从而导致在子Fragment无法收到回调。**

这个问题在23.2.0以下的support-v4包中都存在，23.2.0版本才修复了这个Bug，改用以上版本则可避免。或着在自定义的Activity基类中重写此方法，自己实现对子Fragment的onActivityResult回调。**建议还是使用高版本的v4包，低版本的Fragment中问题实在是太多了。**

###三 Fragment内部数据保存和恢复
在onSaveInstanceState()保存Fragment数据，自定义的业务类尽量实现Parcelable接口（建议不使用Serializable)。代码如下：

```java
@Override
public void onSaveInstanceState(Bundle outState) {
	super.onSaveInstanceState(outState);
	//保存数据
    outState.putString(key, value);
}
```
**在Fragment被调用onSaveInstanceState方法后，如果有操作DialogFragment，需使用commitAllowingStateLoss()提交，使用commit()则会抛出异常。**

在onViewStateRestored()恢复数据并更新UI。

```
@Override
public void onViewStateRestored(@Nullable Bundle savedInstanceState) {
	super.onViewStateRestored(savedInstanceState);
	if(savedInstanceState != null){
		//恢复数据
		String value = savedInstanceState.getString(key);
		...
	}
}
```

###四 ViewPager中Fragment延时加载数据
自定义一个Fragment基类：

```java
public abstract class BaseFragment extends Fragment {
    /**
     * 是否已加载数据
     */
    protected boolean hasLoadData = false;

	@Override
    public void onResume() {
	    super.onResume();
        if (getUserVisibleHint() && !hasLoadData) {
	        //加载数据
            loadData(true);
            hasLoadData = true;
        }
	}
    @Override
    public void onHiddenChanged(boolean hidden) {
        super.onHiddenChanged(hidden);
        if(getUserVisibleHint() && !hasLoadData){
            //加载数据
            loadData(true);
            hasLoadData = true;
        }
    }

    /**
     * 加载数据
     * @param isRefresh true为刷新，false为加载更多
     */
    public abstract void loadData(boolean isRefresh);
}
```
子中实现数据加载，在onSaveInstanceState方法中保存已经加载的数据，onViewStateRestored方法中恢复数据并将数据更新到UI。这样既可以避免加载未显示的Fragment，又可以在Fragment被重新显示时无需加载即可快速恢复数据并显示。

###五 Fragment获取Activity
Fragment中获取FragmentActivity的getActivity方法代码：

```java
final public FragmentActivity getActivity() {
        return mHost == null ? null : (FragmentActivity) mHost.getActivity();
}
```

Fragment是通过属性mHost（FragmentHostCallbackde实例）关联FragmentActivity，该属性没有限定标识符，即给其赋值的类在同一包名中。既然管理Fragment的是FragmentManager，那么给mHost赋值的操作应该在其方法内。而FragmentManager为抽象类，子类为FragmentManagerImpl，则赋值操在其方法内。通过查看Fragment添加和显示理财，可知具体设置mHost值是在moveToState方法中：

```java
void moveToState(Fragment f, int newState, int transit, int transitionStyle,
            boolean keepActive) {
	...
	switch (f.mState) {
            case Fragment.INITIALIZING: 
				if (f.mSavedFragmentState != null) {
                    ...
					//恢复保存的状态信息
                 }
				 //设置mHost
                 f.mHost = mHost;
				 //
                 f.mParentFragment = mParent;
				 //
                 f.mFragmentManager = mParent != null ? mParent.mChildFragmentManager : mHost.getFragmentManagerImpl();
                 f.mCalled = false;
				 //执行Fragment的onAttach方法
				 f.onAttach(mHost.getContext());
				 ...
			case Fragment.CREATED:
				if (newState < Fragment.CREATED) {
					if (f.mAnimatingAway != null) {
                        ...
                    } else {
                        if (!f.mRetaining) {
			                  f.performDestroy();
                        }
                        f.mCalled = false;
						//执行Fragment的onDetach
                        f.onDetach();
                        ...
						//如果不需要保活此Fragment
                        if (!keepActive) {
                           if (!f.mRetaining) {
								makeInactive(f);
                            } else {
								//将Fragment中的mHost置为null
								f.mHost = null;
								f.mParentFragment = null;
								f.mFragmentManager = null;
								f.mChildFragmentManager = null;
                            }
                  }
         }
	}
```
在onAttach之后才可以调用getActivity获取到FragmentActivity对象，在onDetach之后一般情况下是获取不到值。

[请尊重博主劳动成果，转载请标明原文链接。](http://blog.csdn.net/hwliu51/article/details/69054521)













