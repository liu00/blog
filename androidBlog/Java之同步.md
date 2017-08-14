[请尊重博主劳动成果，转载请标明出处。](http://blog.csdn.net/hwliu51/article/details/74852642)

### volatile
使用volatile对属性字段做同步时，必须保证对该属性的操作是原子性的。如果有非原子性的操作，则volatile可能会无效。

**Java并发编程：volatile关键字解析**
http://www.cnblogs.com/dolphin0520/p/3920373.html
这一篇博客对volatile讲解的很详细，值得仔细阅读。


### synchronized，wait，notify与notifyAll
synchronized可用在代码块和方法上。

```java
class T{
	public synchronized static void method(){
		....
	}
}
或
class T{
	public void method(){
		synchronized(T.class){
			...
		}
	}
}
或
class T{
	private static byte[] lock = new byte[0];
	public void method(){
		synchronized(lock){
			...
		}
	}
}
```
如果是以上三种方式，则是对T这个类加锁，T类型的所有对象竞争的是同一个锁。

```java
class T{
	public synchronized void method(){
		....
	}
}
或
class T{
	private byte[] lock = new byte[0];
	public void method(){
		synchronized(lock){
			...
		}
	}
}
```
如果是以上两种方式，则是对T类型的对象加锁，锁只对同一个对象有效。

更详细的讲解，请查看博客：[**Java中Synchronized的用法**](http://blog.csdn.net/luoweifu/article/details/46613015)这篇博客。

wait()方法只能在synchronized内使用，调用此方法会让对象释放持有的锁，并加入等待池。notify()方法会从等待池任意唤醒一个对象，待执行完synchronized代码，这个被唤醒的对象便能够获取到锁。notityAll()方法会唤醒等待池中所有的对象，这些被唤醒的对象进入锁池竞争锁。

**锁池**：线程A获取到锁，紧接着来了B，C，D三个线程。因为A持有锁，所以B，C，D就进入这个锁的锁池，等待A释放锁。
**等待池**：进入synchronized代码中的线程A调用了wait()方法，则A释放锁进入该对象等待池，等待被唤醒。

更详细的内容请阅读博客
[**java中的锁池和等待池**](http://blog.csdn.net/emailed/article/details/4689220)和[**线程间协作：wait、notify、notifyAll**](http://wiki.jikexueyuan.com/project/java-concurrency/collaboration-between-threads.html)

### 单例模式
双重检测：

```java
public class T{
	private volatile static T instance;
	private T(){}

	public static T getInstance(){
		if(instance == null){
			synchronized(T.class){
				if(instance == null){
					instance = new T();
				}
			}
		}
		return instance;
	}
}
```
另一种更简洁的方式：

```java
public class T{
	private static class Holder{
		static T instance = new T();
	}

	public static void getInstance(){
		return Holder.instance;
	}
}
```
参考了[**Java 中的双重检查（Double-Check）**](http://blog.csdn.net/dl88250/article/details/5439024)和[**Initialization-on-demand holder idiom**](https://en.wikipedia.org/wiki/Initialization-on-demand_holder_idiom)这两篇博客。

### Lock

**Java 并发开发：Lock 框架详解**
http://www.cnblogs.com/aishangJava/p/6555291.html
这篇博客对Lock的知识讲解的非常详细，值得仔细阅读。

### 指令重排序

Java内存访问重排序的研究
https://tech.meituan.com/java-memory-reordering.html
如果有时间，可以细细阅读。