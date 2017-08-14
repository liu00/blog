*本文所引用的代码为Android 6.0版本*

[请尊重博主劳动成果，转载请标明原文链接。](http://blog.csdn.net/hwliu51/article/details/76945248)
## 简介

### 类图

先看看类图，画了几个类中主要的属性和方法。

![ClassLoader](http://img.blog.csdn.net/20170729234249278?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 大致介绍

* ClassLoader：抽象类，类加载器的顶级父类。
* BootClassLoader：ClassLoader子类，类修饰符省略，即package。该类只能被包内的程序访问。用于加载系统类，例如加载framework中的类。
* BaseDexClassLoader：ClassLoader子类。用于加载jar或apk文件。
* PathClassLoader：BaseDexClassLoader子类。用于加载安装的apk文件。
* DexClassLoader：BaseDexClassLoader子类。用于加载外部的jar或apk文件，也就是动态加载中所用的加载器。

不管是dalvik，还是art虚拟机，他们所使用的加载器基本都是这些类。

请尊重博主劳动成果，转载请标明原文链接且勿修改原文内容。

## ClassLoader

位置：android-6.0.0_r1/libcore/libart/src/main/java/java/lang/ClassLoader.java

### getSystemClassLoader()
采用静态内部来实现的单例模式。私有的静态内部类SystemClassLoader被加载进来时，会调用createSystemClassLoader()创建系统ClassLoader对象，并将其赋值给静态属性loader，以完成初始化操作。
```java
    static private class SystemClassLoader {
        public static ClassLoader loader = ClassLoader.createSystemClassLoader();
    }

    public static ClassLoader getSystemClassLoader() {
        return SystemClassLoader.loader;
    }
```
当外部使用`ClassLoader.getSystemClassLoader()`获取到的便是loader所引用的对象，即同一个进程内无论谁调用了这个方法，获取到的都是同一个对象。

在看看创建ClassLoader的createSystemClassLoader()方法。
```java
    private static ClassLoader createSystemClassLoader() {
        String classPath = System.getProperty("java.class.path", ".");
        ...

        // TODO Make this a java.net.URLClassLoader once we have those?
        return new PathClassLoader(classPath, BootClassLoader.getInstance());
    }
```
ClassLoader.getSystemClassLoader()获取到的为PathClassLoader对象。该对象的父加载器为BootClassLoader。classPath即为系统framework的类路径。

### loadClass()

ClassLoader中非常重要的一个方法，也是经常被使用到的方法。

```java
    public Class<?> loadClass(String className) throws ClassNotFoundException {
        return loadClass(className, false);
    }
```

真正执行类加载的方法
```java
    protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
        Class<?> clazz = findLoadedClass(className);

        if (clazz == null) {
            ClassNotFoundException suppressed = null;
            try {
                clazz = parent.loadClass(className, false);
            } catch (ClassNotFoundException e) {
                suppressed = e;
            }

            if (clazz == null) {
                try {
                    clazz = findClass(className);
                } catch (ClassNotFoundException e) {
                    e.addSuppressed(suppressed);
                    throw e;
                }
            }
        }

        return clazz;
    }
```
大致可以分为三步：首先从已加载的类中查找；如果没有找到，则调用父加载器查找；如果父加载器中也未找到，则调用findClass()方法获取。如果都没有找到，则抛出相应的异常。ClassLoader的findClass()方法没有做实现，需要子类去重写。

第一步，从已加载的类中查找。

```java
    protected final Class<?> findLoadedClass(String className) {
        ClassLoader loader;
        if (this == BootClassLoader.getInstance())
            loader = null;
        else
            loader = this;
        return VMClassLoader.findLoadedClass(loader, className);
    }
```
使用加载系统类的BootClassLoader作为加载器。findLoadedClass()是native方法。

第二步，父加载器查找。参数false可以忽略，这个参数在Android上无用。

第三步，findClass()方法获取，
```java
    protected Class<?> findClass(String className) throws ClassNotFoundException {
        throw new ClassNotFoundException(className);
    }
```
直接抛出ClassNotFoundException异常。这个方法需要子类去重写，也必须被重写。

## BootClassLoader

位置：在ClassLoader.java文件中，class的修饰符缺省，即package，只能包内访问。

### getInstance()

构造方法和获取实例的方法相关代码

```java
    private static BootClassLoader instance;

    @FindBugsSuppressWarnings("DP_CREATE_CLASSLOADER_INSIDE_DO_PRIVILEGED")
    public static synchronized BootClassLoader getInstance() {
        if (instance == null) {
            instance = new BootClassLoader();
        }

        return instance;
    }

    public BootClassLoader() {
        super(null, true);
    }
```

其实也是单例模式，而且父加载器为null。

### findClass

重写了该方法。
```java
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        return Class.classForName(name, false, null);
    }
```
使用Class的classForName()方法加载，改方法为native方法，且只能被包内的程序访问。

### loadClass

这个方法也被重写。
```java
@Override
    protected Class<?> loadClass(String className, boolean resolve)
           throws ClassNotFoundException {
        Class<?> clazz = findLoadedClass(className);

        if (clazz == null) {
            clazz = findClass(className);
        }

        return clazz;
    }
```
因为BootClassLoader的父加载器为null，所以没有去父加载器中查找这一步了。

## BaseDexClassLoader

位置：android-6.0.0_r1/libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java

### 构造方法
```java
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
```
参数：
* dexPath：含有class和资源的jar或apk文件的路径
* optimizedDirectory：优化后的dex文件存放的目录
* libraryPath：dex文件中引用的so文件的路径
* parent：父加载器

### findClass()

```java
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }
```
pathList为DexPathList类型对象。该类记录当前使用的加载器，已加载的dex或资源文件，以及so文件信息。从已加载的文件中查找，如果未找到，则抛出异常。

### pathList

```java
private final DexPathList pathList;
```
私有属性，记录加载的dex/resource和native库的信息。

DexPathList位置：android-6.0.0_r1/libcore/dalvik/src/main/java/dalvik/system/DexPathList.java

DexPathList部分代码：
```java
/*package*/ final class DexPathList {
    private static final String DEX_SUFFIX = ".dex";
    private static final String zipSeparator = "!/";

    /** class definition context */
    private final ClassLoader definingContext;

    /**
     * List of dex/resource (class path) elements.
     * Should be called pathElements, but the Facebook app uses reflection
     * to modify 'dexElements' (http://b/7726934).
     */
    private final Element[] dexElements;

    /** List of native library path elements. */
    private final Element[] nativeLibraryPathElements;
    ...
}
```
definingContext为加载器，dexElements记录加载过的dex/resource文件，nativeLibraryPathElements记录加载的so文件信。息。

## PathClassLoader

位置：android-6.0.0_r1/libcore/dalvik/src/main/java/dalvik/system/PathClassLoader.java

### 构造方法
```java
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }

    public PathClassLoader(String dexPath, String libraryPath,
            ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
```
两个构造方法，因为是用加载安装的apk，而apk在安装时便会被优化且优化后的dex会被存放到指定的目录，所以构造方法中的优化目录为null。其它参数与父类一致。未重写或增加其它方法。

## DexClassLoader

位置：android-6.0.0_r1/libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java

### 构造方法

```java
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
```
与父类完全一致。也未重写或增加其它方法。

## DexPathList



## 测试
测试使用的Android Studio版本为2.2，compileSdkVersion为25，buildToolsVersion为25.0.1。

测试使用的主要代码：
```java
    private void printInfo() {
        ClassLoader cl = getClassLoader();
        Log.i(tag, "getClassLoader(): " + cl);

        while(cl.getParent() != null){
            cl = cl.getParent();
            Log.i(tag, "parent classLoader: " + cl);
        }

        cl = ClassLoader.getSystemClassLoader();

        Log.i(tag, "System classLoader: " + cl);

        while(cl.getParent() != null){
            cl = cl.getParent();
            Log.i(tag, "parent classLoader: " + cl);
        }

        Log.i(tag, "Activity classLoader: " + Activity.class.getClassLoader());

        Log.i(tag, "Integer classLoader: " + Integer.class.getClassLoader());

        Log.i(tag, "CardView classLoader: " + CardView.class.getClassLoader());

        Log.i(tag, "HookUtil classLoader: " + HookUtil.class.getClassLoader());

    }
```

**debug版本打印的结果**

```
getClassLoader(): dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.cl.test-1/base.apk"],
nativeLibraryDirectories=[/vendor/lib64, /system/lib64]]]
parent classLoader: com.android.tools.fd.runtime.IncrementalClassLoader@21125647
parent classLoader: java.lang.BootClassLoader@68bf030

System classLoader: dalvik.system.PathClassLoader[DexPathList[[directory "."],nativeLibraryDirectories=[/vendor/lib64, /system/lib64]]]
parent classLoader: java.lang.BootClassLoader@68bf030

Activity classLoader: java.lang.BootClassLoader@68bf030
Integer classLoader: java.lang.BootClassLoader@68bf030


CardView classLoader: com.android.tools.fd.runtime.IncrementalClassLoader$DelegateClassLoader[
DexPathList[[
dex file "/data/data/com.cl.test/files/instant-run/dex/slice-support-annotations-23.0.1_a027f157516185b4b73d0e633f35bb54f626c969-classes.dex",
dex file "/data/data/com.cl.test/files/instant-run/dex/slice-slice_9-classes.dex",
…
dex file "/data/data/com.cl.test/files/instant-run/dex/slice-slice_0-classes.dex",
dex file "/data/data/com.cl.test/files/instant-run/dex/slice-library-2.4.0_bc70a6881aeab0fed3511105025b6d0b7eeec0ba-classes.dex",
dex file "/data/data/com.cl.test/files/instant-run/dex/slice-internal_impl-23.0.1_1852f3ccc7a05de6a49cae3dc158656c5aa77abc-classes.dex",
dex file "/data/data/com.cl.test/files/instant-run/dex/slice-com.android.support-cardview-v7-23.0.1_ced719027df2b750f55d3e237bc54549deeb6066-classes.dex",
dex file "/data/data/com.yanbober.viewdraghelper_demo/files/instant-run/dex/slice-com.android.support-appcompat-v7-23.0.1_58c0c10f3b8636e4d5d49cb29b8d62aaf58bf57e-classes.dex"],
nativeLibraryDirectories=[/vendor/lib64, /system/lib64, /vendor/lib64, /system/lib64]]]
HookUtil classLoader: ...
```
当前Activity中使用getClassLoader()获取到的加载器为PathClassLoader，DexPathList中显示的路径为apk存放的路径，即加载当前应用所使用的加载器。IncrementalClassLoader为使用Instant机制编译时动态添加的，它是一个代理ClassLoader，功能与BaseDexClassLoader相同。如果想了解可以查看[深度理解Android InstantRun原理以及源码分析](http://blog.csdn.net/nupt123456789/article/details/51828701)。PathClassLoader的父加载器为BootClassLoader。

使用`ClassLoader.getSystemClassLoader()`获取到的也是PathClassLoader，因为还未加载过apk，所以DexPathList中的文件为空。

Activity和Integer均为framework中的类，所以它们的加载器为BootClassLoader。

因为HookUtil打印的结果与CardView的一样，所以省略了。
CardView为第三方的工程，HookUtil是自定义的类，所以都是通过IncrementalClassLoader的DelegateClassLoader动态加载的。

**再看看release版本打印的结果**

```
getClassLoader(): dalvik.system.PathClassLoader[  DexPathList[[zip file "/data/app/org.plugin-1/base.apk"],  nativeLibraryDirectories=[/vendor/lib64, /system/lib64]]]
parent classLoader: java.lang.BootClassLoader@3cdf9f5b 
System classLoader: dalvik.system.PathClassLoader[  DexPathList[[directory "."],nativeLibraryDirectories=[/vendor/lib64, /system/lib64]]]
 parent classLoader: java.lang.BootClassLoader@3cdf9f5b 
Activity classLoader: java.lang.BootClassLoader@3cdf9f5b
Integer classLoader: java.lang.BootClassLoader@3cdf9f5b 
CardView classLoader: dalvik.system.PathClassLoader[  DexPathList[[zip file "/data/app/org.plugin-1/base.apk"],  nativeLibraryDirectories=[/vendor/lib64, /system/lib64]]]
 HookUtil classLoader: dalvik.system.PathClassLoader[  DexPathList[[zip file "/data/app/org.plugin-1/base.apk"],  nativeLibraryDirectories=[/vendor/lib64, /system/lib64]]]

```
release版本，Instant相关的字节码注入被移除了，所以结果信息更加直观。

framework相关的类都是使用BootClassLoader加载。apk中相关的则是PathClassLoader，而且DexPathList中文件目录都是指向apk的安装目录。可以看到不同方式获取到的BootClassLoader的哈希码都是3cdf9f5b，也就是说BootClassLoader是单例的。


参考博客：

Android ClassLoader机制
http://blog.csdn.net/mr_liabill/article/details/50497055

Android平台的ClassLoader
http://www.jianshu.com/p/a620e368389a

Android类加载器ClassLoader
http://gityuan.com/2017/03/19/android-classloader/

