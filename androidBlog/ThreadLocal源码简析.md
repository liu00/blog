*本文所引用的代码为JDK 1.8版本*

ThreadLocal是用来为当前线程提供存储和获取变量的操作，被操作的变量存储在当前线程的threadLocals中。这些变量不能被其它线程所使用，只能被当前线程所独享，所以**ThreadLocal不是用来提供多线程共享操作的类**。下文将通过类和源码来分析ThreadLocal如何存储和获取线程独享的变量，以及ThreadLocal内存泄漏的原因（其实不是ThreadLocal导致内存泄漏）。

[请尊重博主劳动成果，转载请标明出处。](http://blog.csdn.net/hwliu51/article/details/75137103)

###类图
![ThreadLocal相关的类图](http://img.blog.csdn.net/20170713150746717?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这张图中画了主要的类，以及相关的属性和方法。

重点关注的类：ThreadLocal，ThreadLocalMap和Entry。

ThreadLocalMap为ThreadLocal的内部类，它使用内部类Entry作为存储数据的基本单元。Entry继承了WeakReference，key为ThreadLocal对象，并将key使用WeakReference来引用。而Entry的value，则是ThreadLocal的初始化的值或set的变量。Entry对key的引用为弱引用（即对ThreadLocal弱引用），而对value的引用是强引用。如果对引用类型不是很了解可以去阅读J[ava之4种引用简析](http://blog.csdn.net/hwliu51/article/details/75108818)。Thread的内部属性threadLocals为ThreadLocalMap类型对象。而对threadLocals的创建和存取变量的操作都是由ThreadLocal来代为执行。


### ThreadLocal方法源码介绍

ThreadLocal的源码：

```java
public class ThreadLocal<T> {

    //确定在ThreadLocalMap的table数组中位置的因子
    private final int threadLocalHashCode = nextHashCode();
    
    ...//省略代码

    //初始化方法，
    //调用get时，如果当前线程threadLocals为null，或者threadLocals中无对应的entry，
    //这个方法都会被调用
    protected T initialValue() {
        return null;
    }

    ...//省略代码

    //获取当前线程threadLocals中ThreadLocal对应的value
    public T get() {
        Thread t = Thread.currentThread();
        //获取当前线程threadLocals
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            //获取ThreadLocal对应的entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        //如果当前线程threadLocals为null，或者threadLocals无对应的Entry，
        //则设置并返回初始化值
        return setInitialValue();
    }

    //设置初始化值
    private T setInitialValue() {
        //获取初始化值
        T value = initialValue();
        Thread t = Thread.currentThread();
        //获取当前线程的threadLocals
        ThreadLocalMap map = getMap(t);
        if (map != null)//存在，则将初始化值传入
            map.set(this, value);
        else//不存在，则创建ThreadLocalMap对象并存入，再将对象赋值给ç
            createMap(t, value);
        //返回初始化值
        return value;
    }

    //存储变量value到当前线程的threadLocals中（如果不存在，则创建ThreadLocalMap）
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)//map存在，则存入
            map.set(this, value);
        else//不存在，就创建并存入
            createMap(t, value);
    }

     //将存有自己的Entry从当前线程的threadLocals中移除（如果threadLocals不为null）
     //这个方法在JDK 1.5开始引入
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }

    //获取线程t的threadLocals
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    //创建ThreadLocalMap，存入自己和变量value，同时将创建的对象赋值给线程t的threadLocals变量
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

    ...//省略代码

}
```

ThreadLocal的内部类ThreadLocalMap的源码：
```java
static class ThreadLocalMap {
        //Entry类，存储ThreadLocal对象和被设置的变量
        static class Entry extends WeakReference<ThreadLocal<?>> {
            //value值强引用，即被设置的变量被强引用
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                //ThreadLocal对象传入到父类WeakReference，即key为弱引用
                super(k);
                value = v;
            }
        }

        ...//省略部分代码
        
        //i的下一个位置索引（循环获取）
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

        
        //i的前一个位置索引（循环获取）
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }
        
        ...//省略部分代码
        
        //根据ThreadLocal对象获取存储变量的entry
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }

        //获取丢失的entry
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {//遍历table数组
                ThreadLocal<?> k = e.get();
                if (k == key)//key相同，则返回该entry
                    return e;
                if (k == null)//即ThreadLocal对象被释放，则移除该entry
                    expungeStaleEntry(i);
                else//下一个索引
                    i = nextIndex(i, len);
                e = tab[i];//下一个entry
            }
            //以上的遍历方式，可能会将table中ThreadLocal被释放的Entry全部移除。
            //如果不存在持有key的entry或者持有key的entry恰好在遍历的最后一个，
            //则tableThreadLocal被释放的Entry会全部被移除。其它情况，则看运气。
            
            //找不到，返回null
            return null;
        }

        //存入ThreadLocal设置的变量
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            //从i往后遍历（可能为循环遍历）
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {//已存在，则只需要更新value
                    e.value = value;
                    return;
                }

                if (k == null) {
                    //ThreadLocal对象被释放，则查找并替换更新table
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            //存入
            tab[i] = new Entry(key, value);
            int sz = ++size;
            //如果清理完陈旧的entry后，sz >= table.length*2/3，则重构table
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

        //移除ThreadLocal对象对应的Entry
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();//断开ThreadLocal对象引用
                    //释放第i个entry，并重新调整table中entry的位置
                    expungeStaleEntry(i);
                    return;
                }
            }
        }

        //替换entry，并更新table
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // Back up to check for prior stale entry in current run.
            // We clean out whole runs at a time to avoid continual
            // incremental rehashing due to garbage collector freeing
            // up refs in bunches (i.e., whenever the collector runs).
            int slotToExpunge = staleSlot;
            //查找staleSlot位置前面为null或者ThreadLocal对象被释放的entry的位置，
            //并该索引赋值给slotToExpunge
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            // Find either the key or trailing null slot of run, whichever
            // occurs first
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                // If we find key, then we need to swap it
                // with the stale entry to maintain hash table order.
                // The newly stale slot, or any other stale slot
                // encountered above it, can then be sent to expungeStaleEntry
                // to remove or rehash all of the other entries in run.
                if (k == key) {//找到要更新的entry
                    //更新value
                    e.value = value;

                    //交换i与staleSlot位置的entry
                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    //只有i位置的数据需要被移除
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    
                    //替换完成，返回
                    return;
                }

                // If we didn't find stale entry on backward scan, the
                // first stale entry seen while scanning for key is the
                // first still present in the run.
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // If key not found, put new entry in stale slot
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // If there are any other stale entries in run, expunge them
            //有需要更新的entry
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }

        //移除索引为staleSlot的entry并释放引用。返回更新table结束时的索引
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            //释放entry
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            //从staleSlot位置的下一个开始循环删除ThreadLcoal对象被释放的entry，
            //如果下一个entry为null，则结束
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {//ThreadLcoal对象被释放,释放entry和存储的变量
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {//不相等，则说明entry的数据发生变化，需要更新其所在的位置
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            //返回循环结束时的索引
            return i;
        }

        //从i的下一个开始更新table，清理table陈旧的entry
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                //i的下一个索引
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {//即e的ThreadLocal对象被释放
                    n = len;//更新n，增加循环次数
                    removed = true;
                    //删除i位置entry，获取重新更新的索引
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);//n = n/2,如果n为0，退出
            return removed;
        }

        private void rehash() {
            //先删除无用数据
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            //如果size>=table.length/2,则重新构建table
            if (size >= threshold - threshold / 4)
                resize();
        }

        //重新构建table
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            //创建长度为用来2倍的新数组
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            //遍历，重新存放entry，并删除陈旧数据
            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {//清理旧数据
                        e.value = null; // Help the GC
                    } else {//计算新的存放位置，并更新
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }

        //删除table中无用的entry，并更新table
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.get() == null)
                    expungeStaleEntry(j);
            }
        }
    }

```
因为ThreadLocalMap的操作方法都是私有的，所以只有ThreadLocal能调用。

当ThreadLocal对象在存取和删除时，当前线程的threadLocals都会对table中的entry进行检查，发现陈旧的entry会删除并更新table。但是并不一定会清理完table中所有陈旧的entry。对于不再使用或者要隔很长时间再用的ThreadLocal，最好调用其remove()方法将存储在threadLocals中的entry删除。这是最为保险的方法。

### ThreadLocal内存泄漏

通过以上的源码分析，可知ThreadLocal并不直接或间接持有对value的引用。它只是方便和规范了对线程独享的变量的存取和删除等操作。所以value的泄漏与ThreadLocal无关系。真正造成泄漏的原因：**Thread对threadLocals（ThreadLocalMap对象）持有强引用，而threadLocals通过table数组对entry持有强引用，entry对value持有强引用。当线程无限或长时间的运行，它通过引用链间接对value持有强引用。于是value无法被GC释放，便出现了内存泄漏。**

对长时间运行或停不下来的线程，不需要使用value的时候，就调用ThreadLocal#remove()方法将其从threadLocals移除。需要使用的时候在去使用ThreadLocal#get()获取。

### 测试
测试所用的类的代码

```java
class Value{
	@Override
	protected void finalize() throws Throwable {
		System.out.println("call finalize, " + "" + hashCode());
		super.finalize();
	}
}

class LocalValue extends ThreadLocal<Value> {
	
	@Override
	protected Value initialValue() {
		System.out.println(Thread.currentThread().getName() + " call initialValue");
		return new Value();
	}
	
}
```

测试方法代码：

```java
	private static final LocalValue mLocalValue = new LocalValue();
	
	static void testThreadLocal(){
		//测试initialValue(),get()和remove()方法
		Thread t0 = new Thread(new Runnable() {
			@Override
			public void run() {
				Value value = mLocalValue.get();
				System.out.println(Thread.currentThread().getName() + " get value=" + value.hashCode());
				//移除
				mLocalValue.remove();
				//假设进行耗时操作
				try {
					Thread.sleep(200);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				
				System.out.println( Thread.currentThread().getName() + " get other value=" + mLocalValue.get().hashCode());
			}
		}, "t0");
		
		//测试不同线程调用ThreadLocal的get()时，initialValue()调用情况。（与t0对比）
		Thread t1 = new Thread(new Runnable(){
			@Override
			public void run() {
				System.out.println( Thread.currentThread().getName() + " get value=" + mLocalValue.get().hashCode());
			}
		}, "t1");
		
		//测试set()对initialValue()的影响
		Thread t2 = new Thread(new Runnable(){
			@Override
			public void run() {
				Value value = new Value();
				System.out.println( Thread.currentThread().getName() + " create value=" + value);
				mLocalValue.set(value);
				
				System.out.println( Thread.currentThread().getName() + " get value=" + mLocalValue.get().hashCode());
			}
		}, "t2");
		
		t0.start();
		t1.start();
		t2.start();
		
		//测试与t0,t1和t2对比，entry的value释放情况
		System.out.println(Thread.currentThread().getName() + " get value=" + mLocalValue.get().hashCode());
		
		//等待t0,t1和t2执行完毕
		try {
			Thread.sleep(500);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		
		//强制进行GC，用于查看t0,t1和t2中创建的Value对象释放情况
		System.gc();
		
		//等待显示Value的GC
		try {
			Thread.sleep(10);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		
	}
```

在main方法中运行testThreadLocal()，代码执行结果：
![ThreadLocal测试结果](http://img.blog.csdn.net/20170714213128612?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

分析结果：

main，t0和t1线程在调用mLocalValue的get()方法时，都会执行initialValue()，即为每一个线程创建一个Value对象。t2线程自己创建了Value对象，并设置到threadLocals，使用get()获取到的是自己设置的。这四个线程（main,t0,t1和t2）获取到的Value对象的hashcode分别为1735600054,454934315,912486366和563339240。这个结果显示每个线程通过initialValue()创建或set()设置的变量都是该线程独有，其它线程是无法访问到的。

t2线程使用mLocalValue的set()方法设置自己创建的Value对象，并通过mLocalValue的get()方法获取到Value对象。在这个过程中，mLocalValue都没有没有调用initialValue()。当线程的threadLocals中存在值时，使用get()不会调用initialValue()。

t0线程中mLocalValue调用了2次initialValue()方法。使用remove()方法之后再使用get()，ThreadLocal会再次设置初始化值。

GC显示的信息，通过Value对象的hashcode对比，可知t0,t1和t2线程结束之后便释放了mLocalValue设置到threadLocals中的变量引用，只有还在运行中的main线程没有释放引用。可以证明：线程没有结束，threadLocals中存储的value值是不会被释放。如果线程长时间运行，且不再使用到threadLocals中的变量，就会造成内存泄漏。
