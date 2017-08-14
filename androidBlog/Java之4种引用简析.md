*本文所引用的代码为JDK 1.8版本*

Java语言，引用分为4种类型：强引用，软引用，弱引用和虚引用。强引用为直接引用。除强引用外，其它3种引用都需要通过各自的包装类来实现，并通过get()方法获取。下文将通过类图，类的代码和测试用例这三个步骤来分析和验证这四种引用。

[请尊重博主劳动成果，转载请标明出处。](http://blog.csdn.net/hwliu51/article/details/75108818)

## 类图
![四种引用类图](http://img.blog.csdn.net/20170713150431963?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
软引用为SoftReference，弱引用为WeakReference，虚引用为PhantomReference。它们都继承了Reference。Reference是一个抽象类，负责对存入的对象的具体操，如存入，获取和回收。Reference.Lock是Reference内的一个空类，用于生产lock对象。Reference.ReferenceHandler为Reference的内部类，它继承了Thread，在Reference中非常重要。ReferenceQueue是用来存储Reference，它不是一个真正的Queue，是以链表的形式来存储。

## 源代码分析

### 强引用
因为强引用为直接引用，所以它没有包装类。
强引用使用样例：
```java
Object obj = new Object();
```
JVM执行完第一行代码，便生产了一个强引用obj，它指向创建的Object对象。Object对象被分配在Yuong Generation的Eden Space区域。如果有一个生命周期很长的对象引用到obj，则Object对象会留在内存中，但它存储在内存中位置会随着内存的变化和GC的次数而改变。如果没有被释放，当Eden Space内存满了，它会移动到Survivor Space的某个子区域。当这个子区域的内存也满了，它又会移到到另外一个子区域。当它在这些子区中来回移动到达一定次数时，或者Survivor Space的内存被占满，又会被移动到Tenured Space，即进入Old Generation。如果经历多次GC，obj仍然被引用，那它会一直存在这。（关于内存分配和管理，请阅读[Java内存管理](http://blog.csdn.net/hwliu51/article/details/74853664)这篇博客所介绍的文章。）
这就有可能造成内存泄漏。如果引用obj的对象已经不需要再使用它了，则这肯定是内存泄漏。于是，需要在不在使用的时候断开引用或者使用`obj = null;`来释放。当obj被置为null后，GC时Objcet对象会被回收，占用的内存便被释放。这就是强引用，使用不当时极易造成内存泄漏。

### Reference引用系列
由于强引用使用不当容易造成内存泄漏，于是Java语言开发的大神在JDK 1.2版本引入Reference系列的引用。Reference定义绝大部分的行为和执行了几乎全部的操作。继承者SoftReference，WeakReference和PhantomReference只是稍微有所添加或改动。所以分析软引用，弱引用和虚引用之前，先重点分析它们的父类Reference。这一部分英文注释比较多，如果看不太明白可以使用[谷歌翻译](https://translate.google.cn)。

#### **Reference**
Reference类的代码比较少，但是功能比较复杂。下面会将Reference分为两步来分析：Reference本身；Reference的内部类ReferenceHandler和静态代码块。
第一步：Reference本身
代码：

```java
public abstract class Reference<T> {

    //引用中存储的对象，在条件合适时，它会被VM回收
    private T referent;         /* Treated specially by GC */

    //queue被赋值（非null），则当创建的Reference的子类对象从active状态改变时会被加入到这个queue
    volatile ReferenceQueue<? super T> queue;

    /* When active:   NULL
     *     pending:   this
     *    Enqueued:   next reference in queue (or this if last)
     *    Inactive:   this
     */
    @SuppressWarnings("rawtypes")
    //下一个实例，如果当前对象的状态为Active，则next为null；Pending状态，则为本身；
    //Enqueued状态，则为queue.head.next(即下一个对象）；Inactive状态，为本身
    Reference next;

    /* When active:   next element in a discovered reference list maintained by GC (or this if last)
     *     pending:   next element in the pending list (or null if last)
     *   otherwise:   NULL
     */
    //如果当前对象的状态为Active，discovered链表由VM维护（无其它节点时，为本身）；
    //如果状态为Pending,则为pending（Reference的pending属性，非Pending状态）链表
    //的下一个元素；Enqueued和Inactive状态，discovered为null。
    transient private Reference<T> discovered;  /* used by VM */


    /* Object used to synchronize with the garbage collector.  The collector
     * must acquire this lock at the beginning of each collection cycle.  It is
     * therefore critical that any code holding this lock complete as quickly
     * as possible, allocate no new objects, and avoid calling user code.
     */
    static private class Lock { };
    //用在ReferenceHandler的run()方法的synchronized上，以维护Reference的pending链表
    private static Lock lock = new Lock();


    /* List of References waiting to be enqueued.  The collector adds
     * References to this list, while the Reference-handler thread removes
     * them.  This list is protected by the above lock object. The
     * list uses the discovered field to link its elements.
     */
    //链表结构，由discovered指向下一个元素。pending的referent已被回收
    private static Reference<Object> pending = null;

    //获取存入的对象
    public T get() {
        return this.referent;
    }

    //清除存入的对象
    public void clear() {
        this.referent = null;
    }


    //判断对象是否入queue。如果创建对象时未传入或传入null，则返回false
    public boolean isEnqueued() {
        //ReferenceQueue.ENQUEUED为ReferenceQueue内部类Null的对象
        return (this.queue == ReferenceQueue.ENQUEUED);
    }

    //添加到queue。（这个方法只会被代码调用。GC处理时，不会调用该方法，会直接将references对象入队）
    public boolean enqueue() {
        return this.queue.enqueue(this);
    }

    /* -- Constructors -- */
    Reference(T referent) {
        //queue传入null
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        //如果queue为null，则将其赋值为ReferenceQueue.NULL（ReferenceQueue中的内部类Null对象）
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }

}
```

源码头部的注释对四种状态规则说明：
> A Reference instance is in one of four possible internal states:

>      Active: Subject to special treatment by the garbage collector.  Some
>      time after the collector detects that the reachability of the
>      referent has changed to the appropriate state, it changes the
>      instance's state to either Pending or Inactive, depending upon
>      whether or not the instance was registered with a queue when it was
>      created.  In the former case it also adds the instance to the
>      pending-Reference list.  Newly-created instances are Active.

>      Pending: An element of the pending-Reference list, waiting to be
>      enqueued by the Reference-handler thread.  Unregistered instances
>      are never in this state.

>      Enqueued: An element of the queue with which the instance was
>      registered when it was created.  When an instance is removed from
>      its ReferenceQueue, it is made Inactive.  Unregistered instances are
>      never in this state.

>      Inactive: Nothing more to do.  Once an instance becomes Inactive its
>      state will never change again.

> The state is encoded in the queue and next fields as follows:

>      Active: queue = ReferenceQueue with which instance is registered, or
>      ReferenceQueue.NULL if it was not registered with a queue; next =
>      null.

>      Pending: queue = ReferenceQueue with which instance is registered;
>      next = this

>      Enqueued: queue = ReferenceQueue.ENQUEUED; next = Following instance
>      in queue, or this if at end of list.

>      Inactive: queue = ReferenceQueue.NULL; next = this.

> With this scheme the collector need only examine the next field in order
> to determine whether a Reference instance requires special treatment: If
> the next field is null then the instance is active; if it is non-null,
> then the collector should treat the instance normally.

> To ensure that a concurrent collector can discover active Reference
> objects without interfering with application threads that may apply
> the enqueue() method to those objects, collectors should link
> discovered objects through the discovered field. The discovered
> field is also used for linking Reference objects in the pending list.

’the next field‘是指Reference的next属性。
大概意思：Reference子类的实例有4种状态，分别为：Active（活动）,Pending（挂起）, Enqueued（入队）, Inactive（失活）。一个实例在某个时间点只能处在其中的一种状态。

新创建的Reference子类的实例ref（假设为A）的状态为Active，同时ref.next为null。当GC检测到它的referent（即存储的对象）的引用可达到性改变到合适的状态（即无外部的强引用），则会将referent置为null（即释放引用），并改变对象ref的状态。如果创建对象ref时传入ReferenceQueue，则将ref的状态改变为Pending，同时将ref加入到pending列表和设置ref.next=ref。如果没有，则改变为Inactive，同时将ref.next设置为本身。

当处于Pending状态，对象ref会等待被ReferenceHandler的run()方法处理，被处理后便进入了Enqueued状态，ref.next也会被设置为queue中的下一个元素（如果没有则为ref），同时
ref的queue被重置为ReferenceQueue.ENQUEUED。

当处于Enqueued状态，对象ref被从queue中移除，则进入了Inactive状态，ref.next又被设置为本身。

处于Inactive状态时，ref的状态便不能改变了，ref等待被VM回收。

关于如何VM如何将ref更新到pending链表，可以查看[weakreference实现原理分析](http://blog.csdn.net/hwliu51/article/details/75000787)。


第二步：ReferenceHandler和静态代码块
代码：

```java
    /* High-priority thread to enqueue pending References
     */
    private static class ReferenceHandler extends Thread {

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }

        //处理进入pending状态的references
        public void run() {
            for (;;) {
                Reference<Object> r;
                synchronized (lock) {
                    if (pending != null) {
                        r = pending;
                        pending = r.discovered;
                        r.discovered = null;
                    } else {
                        //防止因异常而退出
                        try {
                            try {
                                lock.wait();
                            } catch (OutOfMemoryError x) { }
                        } catch (InterruptedException x) { }
                        continue;
                    }
                }

                // Fast path for cleaners
                if (r instanceof Cleaner) {
                    ((Cleaner)r).clean();
                    continue;
                }

                ReferenceQueue<Object> q = r.queue;
                //如果创建references时传入非null的queue，则将references进入queue
                if (q != ReferenceQueue.NULL) q.enqueue(r);
            }
        }
    }

    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        //设置优先级为最高
        handler.setPriority(Thread.MAX_PRIORITY);
        //设置当前线程为守护线程
        handler.setDaemon(true);
        handler.start();
    }
```
ThreadGroup源码上一段说明：

>  A thread group represents a set of threads. In addition, a thread
>  group can also include other thread groups. The thread groups form
>  a tree in which every thread group except the initial thread group
>  has a parent.

对ThreadGroup不了解，所以不多说了。如果哪位知道可以补充下。关于守护线程概念，请阅读[Thread.setDaemon详解](http://blog.csdn.net/xyls12345/article/details/26256693)。

ReferenceQueue的部分代码：

```java
public class ReferenceQueue<T> {

    private static class Null<S> extends ReferenceQueue<S> {
        boolean enqueue(Reference<? extends S> r) {
            return false;
        }
    }

    //如果创建references时未传入ReferenceQueue对象或null，
    //则references.queue被设置为该值
    static ReferenceQueue<Object> NULL = new Null<>();
    //references由Pending状态转变为Enqueued状态，则被设置为该值
    static ReferenceQueue<Object> ENQUEUED = new Null<>();

    static private class Lock { };
    //锁对象
    private Lock lock = new Lock();
    //链表，通过next属性指向下一个
    private volatile Reference<? extends T> head = null;
    private long queueLength = 0;

    //入队的方法
    boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
        synchronized (lock) {
            // Check that since getting the lock this reference hasn't already been
            // enqueued (and even then removed)
            ReferenceQueue<?> queue = r.queue;
            //r未设置queue，或r为Enqueued状态，返回false
            if ((queue == NULL) || (queue == ENQUEUED)) {
                return false;
            }
            assert queue == this;
            //重置r.queue
            r.queue = ENQUEUED;
            //将r加入到queue的head链表头
            r.next = (head == null) ? r : head;
            head = r;
            queueLength++;
            if (r instanceof FinalReference) {
                sun.misc.VM.addFinalRefCount(1);
            }
            lock.notifyAll();
            return true;
        }
    }

    //执行从queue中移除references操作的方法
    @SuppressWarnings("unchecked")
    private Reference<? extends T> reallyPoll() {       /* Must hold lock */
        Reference<? extends T> r = head;
        if (r != null) {
            //head的下一个移除；如果head.next==head，则head置为null
            head = (r.next == r) ?
                null :
                r.next; // Unchecked due to the next field having a raw type in Reference
            //重置r.queue
            r.queue = NULL;
            r.next = r;
            queueLength--;
            if (r instanceof FinalReference) {
                sun.misc.VM.addFinalRefCount(-1);
            }
            return r;
        }
        return null;
    }
    ...//省略了poll(),remove(time)和remove()方法的代码，它们都是使用reallyPoll()来获取

}
```

ReferenceHandler主要处理进入Pending状态的references，将其从pending链表中取出，并加入references.queue中。经过ReferenceHandler处理，references便进入了Enqueued状态。



#### SoftReference
软引用源码：
```java
public class SoftReference<T> extends Reference<T> {

    /**
     * Timestamp clock, updated by the garbage collector
     */
    //更新timestamp的clock，这个值由GC来更新
    static private long clock;

    /**
     * Timestamp updated by each invocation of the get method.  The VM may use
     * this field when selecting soft references to be cleared, but it is not
     * required to do so.
     */
    //时间戳，在创建SoftReference对象和调用get()方法时，都会更新。
    //当选择清理软引用时，VM可能（仅仅是可能，不是一定）会用这个属性。
    private long timestamp;

    public SoftReference(T referent) {
        super(referent);
        //设置timestamp为clock
        this.timestamp = clock;
    }

    public SoftReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
        //设置timestamp为clock
        this.timestamp = clock;
    }

    public T get() {
        T o = super.get();
        //如果获取的对象不为null，且timestamp与VM更新的clock时间不相同，
        //则更新timestamp为最新的VM设置时间clock
        if (o != null && this.timestamp != clock)
            this.timestamp = clock;
        return o;
    }
}
```
GC维护着clock，获取对象时更新timestamp为最新的clock。VM在回收对象时可能会参考timestamp。在VM抛出OutOfMemeryError异常之前，VM会一次性地把所有的软引用都释放。而强引用则不会释放。更详细的说明可以查看源码文件头部的注释。

#### WeakReference
弱引用源码：
```java
public class WeakReference<T> extends Reference<T> {

    //无ReferenceQueue的构造方法
    public WeakReference(T referent) {
        super(referent);
    }

    //
    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```
与父类Reference没区别。一般情况，创建WeakReference引用后的第一次GC，如果内部的referent无外部强引用，便会被释放。弱引用适合用来实现规范化的映射，映射那些创建速度比较快的对象。

#### PhantomReference
虚引用源码：
```java
public class PhantomReference<T> extends Reference<T> {

    //直接返回null
    public T get() {
        return null;
    }

    //如果不q为null，则创建的实例无用
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```
重写了get()，直接返回null。所以以这方式：`PhantomReference<Object> pro = new PhantomReference(new Object(), null);`，pro.get()获取到的为null，并不代表创建Object被立即被回收，而是下一次GC时才被回收。Java官方建议不要直接使用PhantomReference，而是继承该类来实现业务的需求。

##测试
测试的操作系统Mac 10.12。JDK为64位，版本为1.8.0_05。
### TestCase 1
以WeakReference为例来测试。WeakReference基本没有修改Reference的行为，所以用它来测试能更好的体现Reference。
自定义的测试对象类代码：
```java
class Case{
	int id = -1;
	public Case(int i){
		id = i;
	}
	
	@Override
	public String toString() {
		return "id=" + id + "; " + super.toString();
	}
	@Override
	protected void finalize() throws Throwable {
		System.out.println("call finalize, " + "" + toString() );
		super.finalize();
	}
}
```
测试方法代码：

```java
	static void testReference() {
		//外部强引用
		ReferenceQueue<Case> queue = new ReferenceQueue<Case>();
		WeakReference<Case> wrc0 = new WeakReference<Case>(new Case(0), queue);
		WeakReference<Case> wrc1 = new WeakReference<Case>(new Case(1), queue);
		WeakReference<Case> wrc2 = new WeakReference<Case>(new Case(2), null);
		
		System.out.println("wrc0: " + wrc0 + " , " + wrc0.get());
		System.out.println("wrc1: " + wrc1 + " , " + wrc1.get());
		System.out.println("wrc2: " + wrc2 + " , " + wrc1.get());
		
 		Case c = wrc0.get();//测试存在外部强引用，所以wrc0不能被释放
		
		System.out.println("------- start GC --------");
		System.gc();
		
		try {
			Thread.sleep(100);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		
		refPrint(queue.poll());
		refPrint(queue.poll());
		refPrint(queue.poll());
	}


	static void refPrint(Reference<? extends Object> ref){
		System.out.println("ref: " + ref);
		if(ref != null){
			System.out.println("ref.get :" + ref.get());
		}
	}
```
在main方法中运行testReference()，测试结果如下图：
![testCase 00](http://img.blog.csdn.net/20170714113835943?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
GC后，弱引用wrc1和wrc2所引用的对象被释放，如红色框所示。wrc0的referent因有外表强引用c，所以不会被释放。wrc2因为无queue，wrc1没有被释放，所以queue.poll()取到的只有wrc1。结果如蓝色框内容

注释`Case c = wrc0.get();`这行代码并保存运行，测试结果：
![testCase 01](http://img.blog.csdn.net/20170714113859023?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
GC后，弱引用wrc0，wrc1和wrc2所引用的对象都被释放。wrc0和wrc1都能从queue通过poll()方法取到。

###TestCase 2

Eclipse -> Perferences -> Java -> Installed JREs -> 选择当前使用的jre，然后点击“Edit” -> 在“Defalut VM arguments”项输入：-Xms512m -Xmx1024m。限制最大VM最大分配的内存为1G，设置之后，测试出现OutOfMemoryError就比较容易。（注：Xms和Xmx值需要根据硬件实际内存来设置。测试完成后，需要改为原来的配置值。）

用来占用内存的测试对象类的代码：
```java
class BigData {
	int size = 0;
	byte[] bigData = null;

	BigData(int mb) {
		size = mb;
		//防止程序退出
		try{
			bigData = new byte[1024 * 1024 * mb];
		}catch(Error e){
			e.printStackTrace();
		}
	}

	@Override
	public String toString() {
		return "size=" + size + " MB; " + super.toString();
	}

	@Override
	protected void finalize() throws Throwable {
		bigData = null;
		System.out.println("call finalize, " + "" + toString());
		super.finalize();
	}
}
```
测试代码：
```java
	static void testSoftReference() {
		ReferenceQueue<BigData> queue = new ReferenceQueue<BigData>();
		ArrayList<SoftReference<BigData>> bigDataList = new ArrayList<SoftReference<BigData>>(10);

		//VM被设置最大内存为1G后，程序的可使用的堆内存在600MB到700MB之间。（不同的机器可能会有差异）
		//创建8个占用内存超过100MB的BigData对象
		for (int i = 0; i < 8; i++) {
			bigDataList.add(new SoftReference<BigData>(new BigData(100), queue));
			System.out.println("srBig_" + i + ": " + bigDataList.get(i) + " , " + bigDataList.get(i).get());
		}
		
		//查看软引用入队情况
		for(int i=0; i< 8; i++){
			refPrint(queue.poll());
		}
	}
```
运行结果：
![这里写图片描述](http://img.blog.csdn.net/20170714142828375?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
在抛出OutOfMemoryError之前，VM将之前分配的软引用的referent对象全部释放。这些被释放的软引用被被加入到queue。






参考博客：

java内存泄露分析，java弱引用(weakreference)
http://blog.csdn.net/liangguo03/article/details/7839111

weakreference实现原理分析
http://blog.csdn.net/hwliu51/article/details/75000787


