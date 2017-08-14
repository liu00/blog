*本文分析的代码为Android 7.0版本，这些类的设计与JDK的基本一致，某些细节地方可能有区别*

[请尊重博主劳动成果，转载请标明出处。](http://blog.csdn.net/hwliu51/article/details/74911797)

## Map与Collection的实现类的简要类图

![这里写图片描述](http://img.blog.csdn.net/20170715132153896?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

##Map实现类

### HashMap
它继承了AbstractMap，实现Map，Cloneable和Serializable接口，使用数组+链表的方式来存储数据，HashEntry数组和单向链表。
table数组，类型为HashEntry[]，resize时，以table.length*2的长度增长。
基本字段：
```java
	/**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 4;

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * An empty table instance to share when the table is not inflated.
     */
    static final HashMapEntry<?,?>[] EMPTY_TABLE = {};

    /**
     * The table, resized as necessary. Length MUST Always be a power of two.
     */
    transient HashMapEntry<K,V>[] table = (HashMapEntry<K,V>[]) EMPTY_TABLE;

    /**
     * The number of key-value mappings contained in this map.
     */
    transient int size;

    /**
     * The next size value at which to resize (capacity * load factor).
     * @serial
     */
    // If table == EMPTY_TABLE then this is the initial capacity at which the
    // table will be created when inflated.
    int threshold;

    /**
     * The load factor for the hash table.
     *
     * @serial
     */
    // Android-Note: We always use a load factor of 0.75 and ignore any explicitly
    // selected values.
    final float loadFactor = DEFAULT_LOAD_FACTOR;
```
如果不设置参数使用new HashMap()方式创建，则默认的数组table长度为4，加载因子为0.75V。HashMap的最大长度不超过int的最大值。

HashEntry：

```java
static class HashMapEntry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        HashMapEntry<K,V> next;//下一个节点
        int hash;
        ...//省略了其它代码
}
```
通过HashEntry的next属性，就可以形成一条单向链表。数组+链表的数据结构，既利用了数组查询速度快的优势，又得到了链表结构不需要连续的内存来存储数据的优点。

偷个懒，没有画图，使用文字描述一个长度为8的table所存储的数据结构。
（-->代表下一个，即next。使用...省略了部分HashEntry对象）
[0] = hashEntry0 --> hashEntry1-->hasEntry2..
[1] = hashEntry5 --> hashEntry6
[2] = hashEntry8 --> hashEntry9-->hasEntry11
[3] = hashEntry7
[4] = hashEntry13 --> hashEntry10-->hasEntry21...
[5] = hashEntry14 --> hashEntry17...
[6] = hashEntry20...
[7] = hashEntry27...

从这个文字图的描述，**可以将HashMap内部的数据看成2个维度：纵向维度为数组，即table；横向维度为单向链表，即HashEntry对象链**。

在看看resize相关的代码：

```java
	//创建新的table
   void resize(int newCapacity) {
        HashMapEntry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        HashMapEntry[] newTable = new HashMapEntry[newCapacity];
        //对新的table赋值和重新链接HashEntry对象
        transfer(newTable);
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }

	//执行table赋值和HashEntry重链接
    void transfer(HashMapEntry[] newTable) {
        int newCapacity = newTable.length;
        for (HashMapEntry<K,V> e : table) {
            while(null != e) {
                HashMapEntry<K,V> next = e.next;
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }

   //添加
   void addEntry(int hash, K key, V value, int bucketIndex) {
   		//size是否大于size*loadFactor 且table[bucketIndex]有数据
        if ((size >= threshold) && (null != table[bucketIndex])) {
        	//2倍增长
            resize(2 * table.length);
            hash = (null != key) ? sun.misc.Hashing.singleWordWangJenkinsHash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }
    
    //添加
    void createEntry(int hash, K key, V value, int bucketIndex) {
        HashMapEntry<K,V> e = table[bucketIndex];
        //新加入的数据在HashEntry链表的前面
        table[bucketIndex] = new HashMapEntry<>(hash, key, value, e);
        size++;
    }
```
resize时重新创建数组和建立对象的链接需要占用内存和CPU计算时间。使用默认构造方法创建，table数组的长度递增为4，8，16， 32，64，即2*table.length。所以使用HashMap时，最好先预估数据量，设置一个合理的初始化容量，尽量减少resize。

### Hashtable
Hashtable继承Dictionary，实现Map，Cloneable和Serializable接口，也使用数组+链表的方式来存储数据，HashtableEntry数组和单向链表。
Hashtable与HashMap最大的区别为：Hashtable为线程安全。
其它小的区别：
Hashtable没有对key和value做null判断，所以传入null时会抛出空指针异常。而HashMap有判断，且可以存入。
Hashtable默认table的长度为11，增长方式为2*table.length + 1。

两者hash：
HashMap取hash和获取index

```java
   //使用sun的类的静态方法获取
   int hash = sun.misc.Hashing.singleWordWangJenkinsHash(key);
   
   static int indexFor(int h, int length) {
        // 与长度-1做与运算
        return h & (length-1);
    }
```

Hashtable取hash和获取index
```java
	//直接从hashCode方法获取
   private static int hash(Object k) {
        return k.hashCode();
   }
   //先与 ，再取余
   int index = (hash & 0x7FFFFFFF) % tab.length;
```
***使用Hashtable存储自定义的类对象，如果有重写类的hashCode()方法，一定要小心谨慎。***


### LinkedHashMap:
LinkedHashMap是HashMap的子类，内部有一个header指向的双向循环链表，能记录数据存入的顺序和数据被访问的顺序。

看代码：
```java
	//相当于一个指针
    private transient LinkedHashMapEntry<K,V> header;
    //true为记录访问顺序，false为只记录存入顺序
    private final boolean accessOrder;

	//初始化header，指向自己
    void init() {
        header = new LinkedHashMapEntry<>(-1, null, null, null);
        header.before = header.after = header;
    }

private static class LinkedHashMapEntry<K,V> extends HashMapEntry<K,V> {
        // before指向前一个LinkedHashMapEntry，after指向后面的
        LinkedHashMapEntry<K,V> before, after;

        LinkedHashMapEntry(int hash, K key, V value, HashMapEntry<K,V> next) {
            super(hash, key, value, next);
        }

        //断开前面的LinkedHashMapEntry对象和后面的对自己的指向
        private void remove() {
            before.after = after;
            after.before = before;
        }

        //将自己添加到上一个添加或访问的元素后面。将上一个元素的after指向自己，
        //将header的before指向自己。同时，将自己的before指向上一个元素，
        //after指向header
        private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }

        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            //如果记录访问顺序，则自己移出，放入到header之前，即最近访问的元素位置
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
                addBefore(lm.header);
            }
        }

        void recordRemoval(HashMap<K,V> m) {
            remove();
        }
    }
    
    //添加
    void createEntry(int hash, K key, V value, int bucketIndex) {
        HashMapEntry<K,V> old = table[bucketIndex];
        LinkedHashMapEntry<K,V> e = new LinkedHashMapEntry<>(hash, key, value, old);
        table[bucketIndex] = e;
        //更新链表，将自己添加到最前的位置
        e.addBefore(header);
        size++;
    }
    
    //获取添加时间最久或最长时间未被访问的元素
    public Map.Entry<K, V> eldest() {
        Entry<K, V> eldest = header.after;
        return eldest != header ? eldest : null;
    }
```
当创建LinkedHashMap传入accessOrder为true时，则会记录访问顺序。
参考了该博客：http://blog.csdn.net/ns_code/article/details/37867985

测试一把，代码如下：

```java
	public static void main(String[] args) {
		LinkedHashMap<String, String> map = new LinkedHashMap<>();
		map.put("0", "a");
		map.put("1", "b");
		map.put("2", "c");
		map.put("3", "d");
		map.get("1");
		print(map.keySet());
		
		LinkedHashMap<String, String> accessOrderMap = new LinkedHashMap<String, String>(4, 0.75f, true);
		accessOrderMap.put("0", "a");
		accessOrderMap.put("1", "b");
		accessOrderMap.put("2", "c");
		accessOrderMap.put("3", "d");
		accessOrderMap.get("2");
		accessOrderMap.get("2");
		print(accessOrderMap.keySet());
		
	}
	
	static void print(Set<String> set){
		Iterator<String> iter = set.iterator();
		while(iter.hasNext()){
			System.out.print(iter.next() + ", ");
		}
		System.out.println();
	}
```
运行结果：

```
0, 1, 2, 3, 
0, 1, 3, 2, 
```

Android的LruCache中有使用到这个类。不过5.0及以下版本有个Bug，详情请查看博客：https://my.oschina.net/weichou/blog/735370

### TreeMap
下面的博客比我讲的清楚详细，就不写了。请仔细阅读以下介绍的博客吧。

阅读这篇博客可以先了解下AVL和红黑树的基本概念，以及如何旋转失衡的树。
标题：重温数据结构：平衡二叉树（AVL树）、红黑树、TreeSet与TreeMap
网站：http://blog.csdn.net/nupt123456789/article/details/23280793

这篇博客详细的介绍了如何判断旋转类型和旋转步骤。
标题：AVL树的旋转操作详解
网站：http://www.cnblogs.com/cherryljr/p/6669489.html

这篇博客对TreeMap讲解的挺详细的。
标题：Java提高篇（二七）-----TreeMap
网站：http://www.cnblogs.com/chenssy/p/3746600.html


## Android中类似Map的集合类
### ArrayMap
这个是Android中的类，android.util包下，使用Object[]来存储key和value。索引为0和奇数存储的为key，偶数存储的为value。

ArrayMap内部缓存相关代码。
```java
	//mBaseCache缓存的mArray长度
    private static final int BASE_SIZE = 4;
    //mBaseCache与mTwiceBaseCache缓存的最大次数
    private static final int CACHE_SIZE = 10;
    //缓存的长度为BASE_SIZE的mArray
    static Object[] mBaseCache;
    //mBaseCache缓存次数
    static int mBaseCacheSize;
    //缓存的长度为BASE_SIZE*2的mArray
    static Object[] mTwiceBaseCache;
    //mTwiceBaseCache缓存次数
    static int mTwiceBaseCacheSize;
    


    private void allocArrays(final int size) {
        if (mHashes == EMPTY_IMMUTABLE_INTS) {
            throw new UnsupportedOperationException("ArrayMap is immutable");
        }
        //符合缓存长度，则从缓存中查找，有效利用内存，防止过多的gc
        //增长的长度为8，先从缓存获取
        if (size == (BASE_SIZE*2)) {
            synchronized (ArrayMap.class) {
                if (mTwiceBaseCache != null) {
                    final Object[] array = mTwiceBaseCache;
                    mArray = array;
                    mTwiceBaseCache = (Object[])array[0];
                    mHashes = (int[])array[1];
                    array[0] = array[1] = null;
                    mTwiceBaseCacheSize--;
                    if (DEBUG) Log.d(TAG, "Retrieving 2x cache " + mHashes
                            + " now have " + mTwiceBaseCacheSize + " entries");
                    return;
                }
            }
        } else if (size == BASE_SIZE) {//增长的长度为4，先从缓存获取
            synchronized (ArrayMap.class) {
                if (mBaseCache != null) {
                    final Object[] array = mBaseCache;
                    mArray = array;
                    mBaseCache = (Object[])array[0];
                    mHashes = (int[])array[1];
                    array[0] = array[1] = null;
                    mBaseCacheSize--;
                    if (DEBUG) Log.d(TAG, "Retrieving 1x cache " + mHashes
                            + " now have " + mBaseCacheSize + " entries");
                    return;
                }
            }
        }
		//创建数组
        mHashes = new int[size];
        mArray = new Object[size<<1];
    }

    private static void freeArrays(final int[] hashes, final Object[] array, final int size) {
		//如果释放的mArray长度符合缓存，则进行缓存
        if (hashes.length == (BASE_SIZE*2)) {//hashes长度为8,array为16
            synchronized (ArrayMap.class) {
            	//如果未超过缓次数的存总数
                if (mTwiceBaseCacheSize < CACHE_SIZE) {
                    array[0] = mTwiceBaseCache;
                    array[1] = hashes;
                    for (int i=(size<<1)-1; i>=2; i--) {
                        array[i] = null;//释放数据
                    }
                    //缓存
                    mTwiceBaseCache = array;
                    mTwiceBaseCacheSize++;
                    if (DEBUG) Log.d(TAG, "Storing 2x cache " + array
                            + " now have " + mTwiceBaseCacheSize + " entries");
                }
            }
        } else if (hashes.length == BASE_SIZE) {//hashes长度为4，array为8
            synchronized (ArrayMap.class) {
            //如果未超过缓次数的存总数
                if (mBaseCacheSize < CACHE_SIZE) {
                    array[0] = mBaseCache;
                    array[1] = hashes;
                    for (int i=(size<<1)-1; i>=2; i--) {
                        array[i] = null;//释放数据
                    }
                    //缓存
                    mBaseCache = array;
                    mBaseCacheSize++;
                    if (DEBUG) Log.d(TAG, "Storing 1x cache " + array
                            + " now have " + mBaseCacheSize + " entries");
                }
            }
        }
    }
```
这种缓存数据设计，在储存少量数据上有非常大的优势。当如果数据很多，则就无法利用到缓存。

### SparseArray
SparseArray类的基本属性：
```java
    //标记mValues中被移除的元素
    private static final Object DELETED = new Object();
    //是否整理mKeys和mValues
    private boolean mGarbage = false;

    //key数组。与HashMap和ArrayMap相比，省去了将int转换为Integer的装箱步骤
    private int[] mKeys;
    //value数组
    private Object[] mValues;
    //长度
    private int mSize;

```

delete和remove方法代码：

```java
    public void delete(int key) {
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
            if (mValues[i] != DELETED) {
                mValues[i] = DELETED;
                mGarbage = true;
            }
        }
    }

    public void removeAt(int index) {
        if (mValues[index] != DELETED) {
            mValues[index] = DELETED;
            mGarbage = true;
        }
    }
```
从SparseArray中删除元素并不会立即更新mKeys和mValues数组，只是使用DELETED标记和更改mGarbage。在调用szie(), keyAt(),valueAt(),setValueAt(),indexOfKey(),indexOfValue()，以及put()和append()方法时才调用到gc()方法来整理mKeys和mValues数组和更新mSize。
gc()方法代码：

```java
    private void gc() {
        // Log.e("SparseArray", "gc start with " + mSize);

        int n = mSize;
        int o = 0;
        int[] keys = mKeys;
        Object[] values = mValues;

        for (int i = 0; i < n; i++) {
            Object val = values[i];

            if (val != DELETED) {
                if (i != o) {
                    keys[o] = keys[i];
                    values[o] = val;
                    values[i] = null;
                }

                o++;
            }
        }

        mGarbage = false;
        mSize = o;

        // Log.e("SparseArray", "gc end with " + mSize);
    }
```
如果有元素被删除，在for循环内，会将mKeys和mValues内的往前移动到被删除的位置，原位置被置为null。经过gc()方法后，mKeys和mValues内的数据会跟紧凑。

在看看put()方法代码：

```java
    public void put(int key, E value) {
        //二分查找获取被存入的位置
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {//找到则更新
            mValues[i] = value;
        } else {
            i = ~i;//两次取反运算，即原来值。（binarySearch返回值，和此处取反）

            if (i < mSize && mValues[i] == DELETED) {/找到则更新
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }

            if (mGarbage && mSize >= mKeys.length) {
                gc();//整理mKeys和mValues

                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }

            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            mSize++;
        }
    }
```
看看GrowingArrayUtils.insert()代码：

```java
    public static <T> T[] insert(T[] array, int currentSize, int index, T element) {
        assert currentSize <= array.length;

        if (currentSize + 1 <= array.length) {
            System.arraycopy(array, index, array, index + 1, currentSize - index);
            array[index] = element;
            return array;
        }

        @SuppressWarnings("unchecked")
        T[] newArray = ArrayUtils.newUnpaddedArray((Class<T>)array.getClass().getComponentType(),
                growSize(currentSize));
        System.arraycopy(array, 0, newArray, 0, index);
        newArray[index] = element;
        System.arraycopy(array, index, newArray, index + 1, array.length - index);
        return newArray;
    }
```
其实这个存入数据的过程与ArrayMap的put过程一样。数组长度足够，则将index及后面的数据后移一个位置，然后将新数据插入。数组容量不足，则创建新的，将原数组中0至index复制到新数组，再存入，然后将原数组剩余的再复制。ArrayUtils.newUnpaddedArray()方法中直接调用了VMRuntime.getRuntime().newUnpaddedArray()创建对象数组，效率上会更高些。


## Collection实现类
### Set
#### HashSet
继承AbstractSet，实现Set，Cloneable和Serializable，内存使用HashMap来存储数据
部分代码：
```java
    private transient HashMap<E,Object> map;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();

    /**
     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
     * default initial capacity (16) and load factor (0.75).
     */
    public HashSet() {
        map = new HashMap<>();
    }
```
#### TreeSet
继承AbstractSet，实现Set，Cloneable和Serializable，内存使用TreeMap来存储数据
部分代码：
```java
    private transient NavigableMap<E,Object> m;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();

    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }

    public TreeSet() {
        this(new TreeMap<E,Object>());
    }

    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }

    public TreeSet(Collection<? extends E> c) {
        this();
        addAll(c);
    }

    public TreeSet(SortedSet<E> s) {
        this(s.comparator());
        addAll(s);
    }
```
### List
#### ArrayList
继承AbstractList，实现List，RandomAccess，Cloneable和Serializable，采用Object[]存储数据。它需要使用连续内存空间来存储数据，数组结构让查询非常方便，而删除和添加操作则比较麻烦。

#### LinkedList
继承AbstractSequentialList，实现List, Deque, Cloneable, Serializable，采用双向链表来存储数据。
代码：

```
	//数据节点
    private static class Node<E> {
        E item;//存储的数据
        Node<E> next;//上一个节点
        Node<E> prev;//下一个节点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
    //指向链表的头
    transient Node<E> first;
    //指向链表的尾部
    transient Node<E> last;
```
这种数据结构使的LinkedList添加和删除数据非常方便。

