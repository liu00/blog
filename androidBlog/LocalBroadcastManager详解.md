*本文所引用的代码为Android 6.0版本*

[请尊重博主劳动成果，转载请标明出处。](http://blog.csdn.net/hwliu51/article/details/74752250)

###一 本地广播与全局广播区别

Android文档上LocalBroadcastManager的说明：
> Helper to register for and send broadcasts of Intents to local objects within your process.  This is has a number of advantages over sending global broadcasts with  android.content.Context#sendBroadcast:
> You know that the data you are broadcasting won't leave your app, so don't need to worry about leaking private data.
> It is not possible for other applications to send these broadcasts to your app, so you don't need to worry about having security holes they can exploit.
> It is more efficient than sending a global broadcast through the system.

本地广播与全局广播相比较。优势：不会泄漏数据，不会因广播而出现安全漏洞，发送和接收更高效。缺点：只能在应用内部使用。

###二 主要的类和代码介绍：
####1 LocalBroadcastManager
采用单例模式，一个应用进程只有实例。如果应用为多进程，在其他非主进程调用会生成新的对象。

```java
	//记录广播接收者和与之对应的广播过滤器集合的map
	//一个BroadcastReceiver对应多个IntentFilter
    private final HashMap<BroadcastReceiver, ArrayList<IntentFilter>> mReceivers
            = new HashMap<BroadcastReceiver, ArrayList<IntentFilter>>();
    //广播消息action和接收该Action的广播接收者集合的map
    //一个action对应多个ReceiverRecord
    private final HashMap<String, ArrayList<ReceiverRecord>> mActions
            = new HashMap<String, ArrayList<ReceiverRecord>>();

	//记录待处理的广播消息和对应接收者的集合
    private final ArrayList<BroadcastRecord> mPendingBroadcasts
            = new ArrayList<BroadcastRecord>();

	//消息code，用于异步处理mPendingBroadcasts的消息
    static final int MSG_EXEC_PENDING_BROADCASTS = 1;
    //用于发送和在主线程处理mPendingBroadcasts
    private final Handler mHandler;

    private static final Object mLock = new Object();
    //静态实例
    private static LocalBroadcastManager mInstance;

    public static LocalBroadcastManager getInstance(Context context) {
        synchronized (mLock) {
            if (mInstance == null) {
            	//使用的是Application的Context来创建
                mInstance = new LocalBroadcastManager(context.getApplicationContext());
            }
            return mInstance;
        }
    }
	//私有构造方法
    private LocalBroadcastManager(Context context) {
        mAppContext = context;
        mHandler = new Handler(context.getMainLooper()) {

            @Override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                	//异步处理广播
                    case MSG_EXEC_PENDING_BROADCASTS:
                        executePendingBroadcasts();
                        break;
                    default:
                        super.handleMessage(msg);
                }
            }
        };
    }
```

####2 私有的内部类ReceiverRecord
广播接收者记录类：记录广播过滤器IntentFilter和注册的BroadcastReceiver广播接收对象。

```java
   private static class ReceiverRecord {
		//广播过滤器
        final IntentFilter filter;
        //广播接收对象
        final BroadcastReceiver receiver;
        //是否被记录到处理广播的标志，以防止重复记录。
        boolean broadcasting;

        ReceiverRecord(IntentFilter _filter, BroadcastReceiver _receiver) {
            filter = _filter;
            receiver = _receiver;
        }

        @Override
        public String toString() {
            StringBuilder builder = new StringBuilder(128);
            builder.append("Receiver{");
            builder.append(receiver);
            builder.append(" filter=");
            builder.append(filter);
            builder.append("}");
            return builder.toString();
        }
    }
```

#### 3 私有的内部类ReceiverRecord
广播记录类：记录广播消息Intent和接收该广播消息的ReceiverRecord的集合。

```java
   private static class BroadcastRecord {
		//带有广播消息的Intent
        final Intent intent;
        //接收intent的消息的广播接收者的集合
        final ArrayList<ReceiverRecord> receivers;

        BroadcastRecord(Intent _intent, ArrayList<ReceiverRecord> _receivers) {
            intent = _intent;
            receivers = _receivers;
        }
    }
```

###三广播流程介绍

#### 1 注册本地广播
通过静态方法getInstance获取到LocalBroadcastManager对象，然后调用registerReceiver，传入BroadcastReceiver和IntentFilter参数注册广播。

LocalBroadcastManager#registerReceiver代码：
```java
   public void registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        synchronized (mReceivers) {
        	//创建广播接收者记录对象，记录filter和receiver
            ReceiverRecord entry = new ReceiverRecord(filter, receiver);
            //获取receiver之前注册的IntentFilter对象的集合
            ArrayList<IntentFilter> filters = mReceivers.get(receiver);
            if (filters == null) {
            	//filters为null，说明receiver没有被注册过。
            	//创建IntenFilter集合filters，将receiver和filter记录到mReceiver中
                filters = new ArrayList<IntentFilter>(1);
                mReceivers.put(receiver, filters);
            }
            //记录filter
            filters.add(filter);
            //遍历Action，将其与接收该Action的ReceiverRecord关联
            for (int i=0; i<filter.countActions(); i++) {
                String action = filter.getAction(i);
                ArrayList<ReceiverRecord> entries = mActions.get(action);
                if (entries == null) {
                    entries = new ArrayList<ReceiverRecord>(1);
                    mActions.put(action, entries);
                }
                entries.add(entry);
            }
        }
    }
```

####2 发送本地广播
发送有两种方式，调用sendBroadcast或sendBroadcastSync方法。sendBroadcast方法不会立即处理广播，而是通过mHandler发送一个MSG_EXEC_PENDING_BROADCASTS的空消，然后在主线程异步处理。而sendBroadcastSync在调用时便处理广播，即同步处理。因此sendBroadcastSync不能在子线程中调用。

先看看LocalBroadcastManager#sendBroadcast代码：
```java
   public boolean sendBroadcast(Intent intent) {
        synchronized (mReceivers) {
        	//获取广播消息的基本信息
            final String action = intent.getAction();
            final String type = intent.resolveTypeIfNeeded(
                    mAppContext.getContentResolver());
            final Uri data = intent.getData();
            final String scheme = intent.getScheme();
            final Set<String> categories = intent.getCategories();

            final boolean debug = DEBUG ||
                    ((intent.getFlags() & Intent.FLAG_DEBUG_LOG_RESOLUTION) != 0);
            if (debug) Log.v(
                    TAG, "Resolving type " + type + " scheme " + scheme
                    + " of intent " + intent);

			//从mActions中获取接收该action的广播接收者记录集合
            ArrayList<ReceiverRecord> entries = mActions.get(intent.getAction());
            //如果entries不为null，则有接收该Action的广播注册。
            if (entries != null) {
                if (debug) Log.v(TAG, "Action list: " + entries);

				//记录符合条件的广播接收者记录对象
                ArrayList<ReceiverRecord> receivers = null;
                //------ 遍历开始 ------
                for (int i=0; i<entries.size(); i++) {
                    ReceiverRecord receiver = entries.get(i);
                    if (debug) Log.v(TAG, "Matching against filter " + receiver.filter);
					//broadcasting为true，则说明该receiver已被记录到receivers
                    if (receiver.broadcasting) {
                        if (debug) {
                            Log.v(TAG, "  Filter's target already added");
                        }
                        continue;
                    }

					//匹配IntentFilter
                    int match = receiver.filter.match(action, type, scheme, data,
                            categories, "LocalBroadcastManager");
                    if (match >= 0) {//匹配成功
                        if (debug) Log.v(TAG, "  Filter matched!  match=0x" +
                                Integer.toHexString(match));
                        if (receivers == null) {//创建接收者集合
                            receivers = new ArrayList<ReceiverRecord>();
                        }
                        receivers.add(receiver);//记录receiver
                        receiver.broadcasting = true;//标记为已记录
                    } else {
                    	//debug信息
                        if (debug) {
                            String reason;
                            switch (match) {
                                case IntentFilter.NO_MATCH_ACTION: reason = "action"; break;
                                case IntentFilter.NO_MATCH_CATEGORY: reason = "category"; break;
                                case IntentFilter.NO_MATCH_DATA: reason = "data"; break;
                                case IntentFilter.NO_MATCH_TYPE: reason = "type"; break;
                                default: reason = "unknown reason"; break;
                            }
                            Log.v(TAG, "  Filter did not match: " + reason);
                        }
                    }
                }
                //------ 遍历结束 ------

				//有符合条件的广播接收者记录对象
                if (receivers != null) {
                	//重置标记
                    for (int i=0; i<receivers.size(); i++) {
                        receivers.get(i).broadcasting = false;
                    }
                    //封装到广播记录，并添加到待处理的集合
                    mPendingBroadcasts.add(new BroadcastRecord(intent, receivers));
                    //如果没有正在处理mPendingBroadcasts，则放送异步处理消息
                    if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
                        mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);
                    }
                    //发送成功
                    return true;
                }
            }
        }
        //发送失败
        return false;
    }
```

LocalBroadcastManager#sendBroadcastSync代码：
```java
   public void sendBroadcastSync(Intent intent) {
   		//通过sendBroadcast将接收广播的receiver找出并封装为BroadcastRecord记录到
   		//mPendingBroadcasts，然后调用executePendingBroadcasts立即处理
        if (sendBroadcast(intent)) {
            executePendingBroadcasts();
        }
    }
```

####3 反注册本地广播
调用unregisterReceiver注册
LocalBroadcastManager#unregisterReceiver代码：
```java
    public void unregisterReceiver(BroadcastReceiver receiver) {
        synchronized (mReceivers) {
        	//将receiver从mReceiver移除，并返回相关的IntentFilter集合
            ArrayList<IntentFilter> filters = mReceivers.remove(receiver);
            if (filters == null) {
                return;
            }
            //遍历IntentFilter集合
            for (int i=0; i<filters.size(); i++) {
                IntentFilter filter = filters.get(i);
                //遍历Action
                for (int j=0; j<filter.countActions(); j++) {
                    String action = filter.getAction(j);
                    //获取action关联的ReceiverRecord集合
                    ArrayList<ReceiverRecord> receivers = mActions.get(action);
                    if (receivers != null) {
                        for (int k=0; k<receivers.size(); k++) {
                        	//如果相同，则将其删除
                            if (receivers.get(k).receiver == receiver) {
                                receivers.remove(k);
                                k--;
                            }
                        }
                        //receivers为空，则说明与action关联的ReceiverRecord已全被移除
                        //即已无ReceiverRecord与action关联，则将action从mActions删除
                        if (receivers.size() <= 0) {
                            mActions.remove(action);
                        }
                    }
                }
            }
        }
    }
```
从mReceivers中删除receiver，到遍历filters和action来删除mActions中引用，从而彻底删除对receiver的引用。这样就可以防止应注册本地广播而导致的内存泄漏。

###四 回顾上述分析，再来比较普通广播与本地广播
从以上的代码和流程分析来看本地广播的优点：
1.**安全**：本地广播所发出的Intent只在应用内部传播，即使其他应用注册了相同的action也无法接收到广播消息。而全局广播则会传播给所有注册相同action的应用，从而导致数据泄漏。
2.**高效**：
	**a 本地广播大概流程**
	**注册**  LocalBroadcastManager#registerReceiver；
**反注册** LocalBroadcastManager#unRegisterReceiver；
**发送处理（异步）**  LocalBroadcastManager#sendBroadcast =》 mHandler#handleMessage =》BroadCastReceiver ；
	**发送处理（同步）** LocalBroadcastManager#sendBroadcastSync =》BroadCastReceiver；
		
**b 普通广播大概流程**
**注册** ContextImpl#registerReceiver =>IBinder => ActivityManagerService#registerReceiver 
**反注册** ContextImpl#unRegisterReceiver =>IBinder => ActivityManagerService#unRegisterReceiver
**发送处理（只有异步）** ContextImpl#sendBroadcast =>IBinder => ActivityManagerService#sendBroadcast=》ApplicationThread#scheduleReceiver =》H#handleMessage =》ActivityThread#handleReceiver =》BroadCastReceiver

本地广播的数据基本只在LocalBroadcastManager类和BroadcastReceiver内部流通。而普通广播则复杂的多。

本地广播的缺点也很明显，就是不能够跨进程传播。

###五 本地广播与EventBus
二者大概的设计思路比较相似。
LocalBroadcastManager采用了单例设计模式。注册时，使用集合容器来记录BroadcastReceiver和action。发送广播事消息时，从集合容器中查找BroadcastReceiver对象，然后将消息和BroadcastReceiver对象集合封装到BroadcastRecord对象，并加入到处理的集合中，再异步或同步处理给BroadcastReceiver。可以将mPendingBroadcasts看作为一个待处理的消息队列。

EventBus也使用了单例模式。注册时，通过反射获取需要调用的方法和参数消息以及其他配置信息，然后分别存入到相关的集合容器中。发送消息时，依据发送和配置的策略采取同步或异步处理。异步消息将加入到队列中，在Handler中遍历获取，根据消息的类型遍历获取被调用的方法然后传递消息。





