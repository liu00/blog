由[Android类加载器源码简析](http://blog.csdn.net/hwliu51/article/details/76945248)这篇博客可知：可以使用DexClassLoader动态加载含有dex的jar或apk文件，然后就可以使用loadClass()方法来加载Class；而应用（即App）是由PathClassLoader加载的。这两个类都继承了BaseDexClassLoader。它有一个私有的属性pathList，这个属性为DexPathList类型，记录加载的dex和so文件的信息。DexPathList中使用一个Element[]类型的dexElements属性记录加载的dex。如果我们先使用DexClassLoader加载jar或apk文件，然后通过反射获取到它的dexElememts。将其与PathClassLoader中dexElememts合并，然后设置给PathClassLoader的pathList的dexElements属性。这样就可以在App中通过类路径直接获取Class了。这就是动态加载dex的基本原理。

### Dex动态加载

反射获取Field的方法代码：
```java
    /**
     * 获取Class中指定名称的属性Field
     * @param clazz Class对象
     * @param fieldName 属性名称
     * @return 属性Field对象
     * @throws NoSuchFieldException 如果未查找到对应属性，则抛出
     */
    private Field getField(Class clazz, String fieldName) throws NoSuchFieldException {
        Field field = null;
        String classStr = clazz.toString();
        //从子类向父类循环查找Field
        while (clazz != null) {
            try {
                Log.i("Test", "parent class = " + clazz);
                field = clazz.getDeclaredField(fieldName);
                if (field != null) {
                    if (!field.isAccessible()) {
                        field.setAccessible(true);
                    }
                    return field;
                }
            } catch (Exception e) {
                //忽略错误信息
                //e.printStackTrace();
            }
            //获取父类字节码，从父类查找
            clazz = clazz.getSuperclass();
        }

        //如果未查找到，则抛出异常
        if (field == null) {
            throw new NoSuchFieldException("field: " + fieldName + " not in " + classStr);
        }

        return field;
    }
```
如果class中未获取到，则从其父类的class获取，直至父类为null或获取到Field对象为止。如果对反射不了解，可以查看[Java反射和代理简介](http://blog.csdn.net/hwliu51/article/details/76945255)这篇博客。

动态加载和复制合并dex的方法代码：
```java
    /**
     * 动态加载dex，并将dex合并到应用加载器中
     * @param ctx Context对象
     */
    private void copyDexElements(Context ctx){
        File file = new File(ctx.getCacheDir(), "apksDir/test.dex");
        if(!file.exists()){
            Log.i("Test", "Dex file not exists, file: " + file.getAbsolutePath());
            return;
        }
        //含有dex的jar或apk文件的路径
        String dexPath = file.getAbsolutePath();
        
        File dir = new File(ctx.getCacheDir(), "optDir");
        if(!dir.exists()){
            dir.mkdir();
        }
        //优化后的dex文件存放的目录的路径
        String optimizedDirectory = dir.getAbsolutePath();

        try {
            //应用的加载器
            ClassLoader pathClassLoader = ctx.getClassLoader();
            //动态加载器，加载的dex文件
            //因为没有从外部引入so文件，所以第3个参数为null
            DexClassLoader dexClassLoader = new DexClassLoader(dexPath, optimizedDirectory, null, pathClassLoader);

            //1, 获取DexClassLoader的pathList
            Field dexPathListField = getField(dexClassLoader.getClass(), "pathList");
            Object dexPathList = dexPathListField.get(dexClassLoader);
            if(dexPathList == null){
                Log.i("Test", "Fail to get pathList from DexClassLoader, dexClassLoader: " + dexClassLoader);
                return;
            }

            //2, 获取pathList的dexElements
            Field dexDexElementsField = getField(dexPathList.getClass(), "dexElements");
            Object[] dexDexElements = (Object[]) dexDexElementsField.get(dexPathList);
            if(dexDexElements == null){
                Log.i("Test", "Fail to get dexElements from pathList in DexClassLoader, dexClassLoader: " + dexClassLoader);
                return;
            }

            if(dexDexElements.length == 0){
                Log.i("Test", "The size of dexElements from pathList in DexClassLoader is 0, dexClassLoader: " + dexClassLoader);
                return;
            }

            //3, 获取应用加载器PathClassLoader的pathList
            Field pathPathListField = getField(pathClassLoader.getClass(), "pathList");
            Object pathPathList = pathPathListField.get(pathClassLoader);
            if(pathPathList == null){
                Log.i("Test", "Fail to get pathList from application ClassLoader, classLoader: " + pathClassLoader);
                return;
            }

            //4, 获取应用加载器的pathList的dexElements
            Field pathDexElementsField = getField(pathPathList.getClass(), "dexElements");
            Object[] pathDexElements = (Object[]) pathDexElementsField.get(pathPathList);
            if(pathDexElements == null){
                Log.i("Test", "Fail to get dexElements from pathList in application ClassLoader, classLoader: " + pathClassLoader);
                return;
            }

            //5, 创建新数组，并复制Element到新数组
            //创建长度为dexDexElements.length + pathDexElements.length的Element[]
            Object[] newDexElements = (Object[]) Array.newInstance(pathDexElements.getClass().getComponentType(), dexDexElements.length + pathDexElements.length);
            //将动态加载的dexElements复制到newDexElements，范围：[0，dexDexElements.length-1]
            System.arraycopy(dexDexElements, 0, newDexElements, 0, dexDexElements.length);
            //将应用的dexElements复制到newDexElements，范围：[dexDexElements.length, n]
            System.arraycopy(pathDexElements, 0, newDexElements, dexDexElements.length, pathDexElements.length);

            //6，将newDexElements设置给应用加载器的pathList的dexElements
            pathDexElementsField.set(pathClassLoader, newDexElements);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

**首先需要将测试的class文件打包为dex文件，然后将文件命名test.dex并复制到应用的cache目录中的apksDir目录。**关于class如何打包为dex可以阅读[手工构建Android应用](http://blog.csdn.net/hwliu51/article/details/76945265)。

动态加载dex和合并到应用的加载器中的步骤：
1，使用DexClassLoader加载jar或apk文件，获取pathList属性中的dexElements值；
2，获取应用加载器PathClassLoader的pathList的class的dexElements属性的Field对象，然后获取值；
3，合并获取到的两个dexElements值，将新的Element[]设置给获取应用加载器的pathList的dexElements属性。
执行完这3个步骤后，便可使用Class.forName("classPath")加载test.dex中的class，以反射的方式创建对象和调用方法及属性。

### Apk动态加载
Apk动态加载与dex加载的步骤基本一致，只不过还需要处理pathList中so信息，以及Android不同版本之间DexPathList兼容问题。

DexPathList类不同版本属性信息（可以从这个[网站](http://androidxref.com/)（http://androidxref.com/）查看）：
位置：android/libcore/dalvik/src/main/java/dalvik/system/DexPathList.java
这里只列了4.0到7.1版本的，4.0以下的就不考虑了，8.0的源码现在还没有。

1，Android 4.0 - 4.3
```java
    /** class definition context */
    private final ClassLoader definingContext;

    /**
     * List of dex/resource (class path) elements.
     * Should be called pathElements, but the Facebook app uses reflection
     * to modify 'dexElements' (http://b/7726934).
     */
    private final Element[] dexElements;

    /** List of native library directories. */
    private final File[] nativeLibraryDirectories;
```
只需要合并记录dex的dexElements和记录so的nativeLibraryDirectories。

2，Android 4.4 - 5.1
```java
    /** class definition context */
    private final ClassLoader definingContext;

    /**
     * List of dex/resource (class path) elements.
     * Should be called pathElements, but the Facebook app uses reflection
     * to modify 'dexElements' (http://b/7726934).
     */
    private final Element[] dexElements;

    /** List of native library directories. */
    private final File[] nativeLibraryDirectories;

    /**
     * Exceptions thrown during creation of the dexElements list.
     */
    private final IOException[] dexElementsSuppressedExceptions;
```
添加了IOException[]，需要合并dexElementsSuppressedExceptions。

3，Android 6.0 - 7.1
```java
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

    /** List of application native library directories. */
    private final List<File> nativeLibraryDirectories;

    /** List of system native library directories. */
    private final List<File> systemNativeLibraryDirectories;

    /**
     * Exceptions thrown during creation of the dexElements list.
     */
    private final IOException[] dexElementsSuppressedExceptions;
```
nativeLibraryPathElements由之前的File[]改为了Element[]，除此之外还添加了nativeLibraryDirectories和systemNativeLibraryDirectories。

目前Apk动态加载和合并需要兼容这3个版本的代码。

### Apk加壳和dex保护

从**Dex动态加载**这一节的介绍我们熟悉了如何动态加载dex。如果我们将Apk的classes.dex从中抽离出来，将其进行加密处理再放入assets目录。然后创建一个壳Apk，在壳的Application中进行解密和动态加载dex，通过反射创建应用的Application，再替换ActivityThread和应用的Application中相关属性。这样就可以在一定的程度上防止我们的Apk被反编译，逻辑代码被他人查看。市面上大部分的Apk加固软件都是采用这中逻辑来防止反编译。

还有一种简单的方式：将重要的逻辑代码单独打成一个dex文件，在使用时动态加载，用完便卸载。这样也可以在一定程度上保护重要代码不被反编译。

当然这种动态加载机制会导致App启动或运行变慢，需要综合考虑。使用模拟器或root的手机运行加壳的应用，然后使用adb命令dump dex信息，还是可以获取到dex信息。







