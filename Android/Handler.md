将子线程的内容传递给主线程，用于多个线程之间或者组件之间的通信。
[[Binder]]/Socket 用于进程间通信，$Handler$ 用于同进程线程间通信。
为什么需要？
因为子线程不允许访问$UI$ ，假若子线程允许访问 UI，则在多线程并发访问情况下，会使得 UI 控件处于不可预期的状态。而如果使用传统方法，加锁，但会使得UI访问逻辑变的复杂，其次降低 UI 访问的效率。
# 基本使用
```Java
public class MainActivity extends AppCompatActivity {
    @SuppressLint({"MissingInflatedId", "ResourceType"})
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_my_view_pager);
        Handler handler = new Handler(new Handler.Callback() {
            @Override
            public boolean handleMessage(@NonNull Message msg) {
                //接收并处理数据
                if (msg.what == 0) {
                    Log.e("child thread ", "receive message from main thread");
                }
                return true;
            }
        });
        //处理耗时工作
        new Thread(new Runnable() {
            @Override
            public void run() {
              //创建Message对象
                Message msg = Message.obtain();
                //设置what标志位
                msg.what = 0;
                //在子线程中发送数据
                handler.sendMessage(msg);
            }
        }).start();
    }
}
```
# 消息机制的流程
## 相关类
### Message
消息，理解为线程间通讯的数据单元。例如后台线程在处理数据完毕后需要更新UI，则可发送一条包含更新信息的`Message`给UI线程。
包含了描述信息和数据对象，其中的数据对象主要包括两个整型域和一个对象域，通过这些域可以传递信息。整型的代价是最小的，所以尽量使用整型域传递信息。
#### 创建方式
`Message`有公共构造函数`Message msg = new Message()`，但是推荐使用其静态构造方法`obtain()`：
```java
public static Message obtain() {
        synchronized (sPoolSync) {
          //缓存池存在直接获取Message对象
            if (sPool != null) {
              //获取对象
                Message m = sPool;
              //指针后移
                sPool = m.next;
                m.next = null;
              //清除标记位
                m.flags = 0; // clear in-use flag
              //更新缓存池大小
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```
这种方式会重复利用缓存池中的对象而不是重新创建新的对象，避免内存中出现太多对象，避免可能的性能问题。
`Message`不再需要的时候，会在`MessageQuene`中取出分发给`Handler`时被添加到缓存，具体使用`recycleUnchecked`函数：
```java
void recycleUnchecked() {
        // Mark the message as in use while it remains 
  			//in the recycled object pool.
        // Clear out all other details.
  //设置标志位为正在使用中，从缓存中取出时会清除标志位
        flags = FLAG_IN_USE;
  //Message对象放入缓存池信息清空
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
          //缓存池默认大小为50
            if (sPoolSize < MAX_POOL_SIZE) {
              //把对象放在缓存池中的链表首部
                next = sPool;
                sPool = this;
              //更新缓存池大小
                sPoolSize++;
            }
        }
    }
```
### Looper
循环器，扮演`MessageQueue`和`Handler`之间桥梁的角色，循环取出`MessageQueue`里面的`Message`，并交付给相应的`Handler`进行处理。
注意是运行在**创建Handler**的线程中的，所以作用域是线程。
#### 创建
线程在默认的情况下是没有消息队列`MessageQueue`的，也无法在其内部进行消息循环，它决定了消息的存储方式。如果想要开启消息循环则需要使用`prepare()`方法创建消息队列；
`Looper.prepare()`主要功能为：  
初始化`Looper`对象并绑定到当前线程中，并且维护一个`MessageQuene`；
```java
public static void prepare() {
  //创建可退出的消息队列
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
  // 如果当前线程已经有了 Looper 对象就直接抛出异常，
  // 因为一个线程只能有一个消息队列。
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
  //创建 Looper 对象并和线程关联。
  //通过ThreadLocal获取Looper对象
  //由此得知Looper的作用域是线程
    sThreadLocal.set(new Looper(quitAllowed));
  //需要注意的是线程是默认没有Looper的，必须使用Handler为线程创建Looper
}
//私有构造函数
private Looper(boolean quitAllowed) {
   mQueue = new MessageQueue(quitAllowed);
  //和当前线程关联起来
   mThread = Thread.currentThread();
}
```
>线程对应的`Looper`是在`ThreadLocal`里面存储，它是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储以后，只有在指定线程中可以获取到存储的数据，对于其它线程来说无法获取到数据。`ThreadLocal`它的作用是可以在不同的线程之中互不干扰地存储并提供数据（就相当于一个Map集合,键位当前的Thead线程，值为Looper对象）

#### 退出
$$
\begin{cases}
quit()\quad直接退出Looper\\
quitSafely()\quad设置一个退出标记，所有消息处理完之后退出
\end{cases}
$$
退出后`Handler`消息会发送失败，返回`false`。如果手动创建`Looper`就应该使用`quit()`终止消息队列。
### MessageQueue
持有消息列表的类，消息`Message`由与`Looper`相联的`Handler`进行发送，并最终由`Looper`分发。
本质上是单项链表。
### Handler
`Handler`是`Message`的主要处理者，负责将`Message`添加到消息队列以及对消息队列中的`Message`进行处理。
```java
public Handler() {
    this(null, false);
}

public Handler(Looper looper, Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    mLooper = looper;
    mQueue = looper.getQueue();
    mCallback = callback;
    mAsynchronous = async;
}
```
### ThreadLocal
#### 基本介绍
1. 线程内部的数据存储类，通过它可以在指定线程存储和访问数据；
2. 复杂逻辑下的对象传递。

有时候一个线程的任务过于复杂，我们需要监听器贯穿整个线程的执行过程，就可以使用`ThreadLocal`使监听器作为全局对象存在。

如果不使用`ThreadLocal`：
- 使监听器作为参数调用；  
    函数调用栈很深的时候不适合。
- 作为静态变量；  
    不具有可扩张性。
#### 内部实现
`set()`
```java
public void set(T value) {
        Thread t = Thread.currentThread();
//存储数据
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
            //如果为空则初始化
        else
            createMap(t, value);
    }
```
`get()`
```java
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);

    if (map != null) {
        // 在ThreadLocalMap中查找ThreadLocal实例对应的值
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }

    // 如果没有找到值，则调用initialValue方法创建一个值并存储
    return setInitialValue();
}
```
值得注意的是，如果你使用 `ThreadLocal` 存储数据，务必小心处理资源泄漏的问题，因为如果不在恰当的时机清理 `ThreadLocal` 变量，可能会导致内存泄漏。一般来说，应该在不再需要这些线程本地变量时，手动调用 `ThreadLocal.remove()` 方法来清理。
#### ThreadLocalMap
用于存储每个线程的 `ThreadLocal` 实例和相应的值。
每个线程都拥有自己的 `ThreadLocalMap` 实例，用于存储与该线程相关的 `ThreadLocal` 变量。
是一个哈希表，通过 `ThreadLocal` 实例作为键，相应的值是该线程的局部变量。
注：
一个线程可以有多个`ThreadLocal` 实例，每个实例代表一个特定的本地变量。这些都存储在`ThreadLocalMap` 中。
## 消息机制
1. 消息发送：通过`Handler`向关联的`MessageQuene`发消息；
2. 消息存储：把发送的消息以`Message`的形式存储在`MessageQuene`中；
3. 消息循环：通过`Looper`不停地从`MessageQuene`获取消息，队列没有消息就阻塞等待新消息；
4. 消息分发和处理：`Looper`获取消息后发送给`Handler`处理；
![[Handler消息机制.png]]
### 创建消息队列
`Android`默认没有消息队列，也无法开启内部循环，需要我们设置。
`Looper.prepare()`
但是如果我们在主线程创建`handler`对象时，发送消息运行成功，这是因为主线程`ActivityThread`开启时自动设置了消息队列和开启消息循环。
```java
/**
 * This manages the execution of the main thread in an
 * application process, scheduling and executing activities,
 * broadcasts, and other operations on it as the activity
 * manager requests.
 *
 * {@hide}
 */
public final class ActivityThread extends ClientTransactionHandler {
    public static void main(String[] args) {
        // 记录开始，用于后续通过 systrace 检查和调试性能问题。
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
      
        // 省略无关代码...

        // 为主线程创建消息队列
        Looper.prepareMainLooper();

        // 省略无关代码...

		//创建实例
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // 记录结束，后续可以通过 systrace 观察这段代码的执行情况。
        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        
        // 开启消息循环
        Looper.loop();

        // 主线程消息循环不会退出，如果走到这意味着发生意外，抛出异常。
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
}
```
`Looper.prepareMainLooper()`
```java
public static void prepareMainLooper() {
    // 启动一个无法退出的消息循环
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException(
              "The main Looper has already been prepared.");
        }
        // 返回主线程 looper 对象
        sMainLooper = myLooper();
    }
}
```
为主线程创建的消息队列是无法退出的，因为主线程的消息队列需要处理很多的事务，如`Activity`的生命周期回调等。
### 开启消息循环
`Looper.loop()`
```java
public static void loop() {
  //获取当前的Looper对象，如果失败则抛出异常
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() 
                                       wasn't called on this thread.");
        }
        if (me.mInLoop) {
            Slog.w(TAG, "Loop again would have the queued messages be executed"
                    + " before this one completed.");
        }

        me.mInLoop = true;

        //确保为当前进程
  //Handler是在Looper关联的线程中处理分发的
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        me.mSlowDeliveryDetected = false;

  //使用无限循环获取消息，只有MessageQuene.next()返回null时退出
        for (;;) {
            if (!loopOnce(me, ident, thresholdOverride)) {
                return;
            }
        }
    }
```
`loop()`中处理消息是一个死循环，拿不到`Message`就会一直阻塞，是否会导致`ANR`？

>在应用程序的入口`ActivityThread`里面的`main` 方法中会创建一个主线程的`looper`对象和一个大`Handler`。
>
>`Android`是基于事件驱动的，通过`looper.looper()`不断接收事件，处理事件，每一个触摸事件或者是`Activity`的生命周期都是运行在`Looper.looper()`的控制之下，当收到不同`Message` 时则采用相应措施：在`H.handleMessage(msg)`方法中，根据接收到不同的msg，执行相应的生命周期。如果它停止了，应用也就停止了。也就是说我们的代码其实就是运行在这个循环里面去执行的，当然就不会阻塞。
>
>而所谓`ANR`便是`Looper.loop`没有得到及时处理，一旦没有消息，Linux的`epoll`机制则会通过管道写文件描述符的方式来对主线程进行唤醒与睡眠，`Android`里调用了`Linux`层的代码，实现在适当时会睡眠主线程。

### 发送和存储消息
#### 发送
发送消息使用`Handler`类，具体可以使用`send()`和`post()`系列方法；
```java
public final boolean post(@NonNull Runnable r) {
                                //2->
    return  sendMessageDelayed(getPostMessage(r), 0);//1->
}
```
```java
//2
private static Message getPostMessage(Runnable r) {
    // 把 Runnable 对象封装成 Message 并设置 callback，
    // 这个 callback 会在后面消息的分发处理中起到作用。
    Message m = Message.obtain();
	m.callback = r;
	return m;
}
```
```java
//1
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
	    delayMillis = 0;
	}
//将延迟时间转化为绝对时间，方便后续进行
//3->
	return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```
```java
//3
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    //消息队列
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
//将消息添加到消息队列
//4->
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```
但它们最后都是调用`Message.enqueueMessage()`将消息存储到`MessageQuene`中；
#### 存储
`Message.target`为该`Handler`对象，即`Message`封装了任务携带的信息和处理信息的`Handler`，这样确保`Looper`获取到`Message`时可以准确获取`Handler`对象，然后执行消息分发方法。
插入的消息有三种可能作为`Head`:
1. 消息队列为空；
2. `when`为0；
3. `Message`执行时间在`when`之后；
`enqueueMessage`作用是向消息队列中插入一条信息：
```java
boolean enqueueMessage(Message msg, long when) {
    //标志为null发出异常
    //意味着后续无法进行消息的分发处理
    //因为每条消息需要指定一个Handler对象
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }

        synchronized (this) {
	//如果消息正在使用就出现异常
    //消息不能并发处理
            if (msg.isInUse()) {
                throw new IllegalStateException(msg + 
                                            " This message is already in use.");
            }
	//如果消息正在退出
    //回收消息对象
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
	                msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }
			//将消息标记为正在使用中
            msg.markInUse();
            //设置消息时间
            msg.when = when;
            //获取消息队列中的第一条消息
            Message p = mMessages;
            //标记是否需要唤醒消息处理线程
            boolean needWake;
		    //将消息添加到合适的位置
            // 消息队列为空或者当前消息对象的时间最近，直接放在链表首部。
            if (p == null || when == 0 || when < p.when) {
                ///更新指针
                msg.next = p;
                mMessages = msg;
                //如果原来消息阻塞就重新唤醒
                needWake = mBlocked;
            } else {
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                //根据消息的信息寻找合适的插入位置
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                //找到合适的位置后插入链表
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }
        	//唤醒等待，继续循环消息队列
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
`next` 的作用是从消息队列中取出一条消息并从消息队列中移除：
```java
Message next() {  
    //检查Looper的底层消息循环是否已经退出。如果 `ptr` 为0，表示消息循环已被释放，就不再处理消息，直接返回null。
    if (ptr == 0) {  
        return null;  
    }  
  
    int pendingIdleHandlerCount = -1; // -1 only during first iteration  
    int nextPollTimeoutMillis = 0;  
    for (;;) {  
        if (nextPollTimeoutMillis != 0) {  
        //刷新Binder的挂起命令
            Binder.flushPendingCommands();  
        }  
		//等待消息的到来
        nativePollOnce(ptr, nextPollTimeoutMillis);  
  
        synchronized (this) {  
            // Try to retrieve the next message.  Return if found.  
            final long now = SystemClock.uptimeMillis();  
            Message prevMsg = null;  
            Message msg = mMessages;  
            if (msg != null && msg.target == null) {  
                // Stalled by a barrier.  Find the next asynchronous message in the queue.  
                do {  
                    prevMsg = msg;  
                    msg = msg.next;  
                } while (msg != null && !msg.isAsynchronous());  
            }  
            if (msg != null) {  
                if (now < msg.when) {  
                //表示消息还未准备好处理，需要设置一个超时时间，以便稍后再次尝试获取消息。
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);  
                } else {  
                    // Got a message.  
                    mBlocked = false;  
                    if (prevMsg != null) {  
                        prevMsg.next = msg.next;  
                    } else {  
                        mMessages = msg.next;  
                    }  
                    msg.next = null;  
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);  
                    msg.markInUse();  
                    return msg;  
                }  
            } else {  
                // 没有更多的消息处理
                nextPollTimeoutMillis = -1;  
            }  
  
            //处理退出消息
            if (mQuitting) {  
                dispose();  
                return null;            
            }  
  
            //处理空闲消息
            //为负表示第一次迭代
            if (pendingIdleHandlerCount < 0  
                    && (mMessages == null || now < mMessages.when)) {  
                pendingIdleHandlerCount = mIdleHandlers.size();  
            }  
            //没有空闲消息处理
            if (pendingIdleHandlerCount <= 0) {  
                //继续等待  
                mBlocked = true;  
                continue;            
            }  
  
            if (mPendingIdleHandlers == null) {  
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];  
            }  
            //将空闲消息存储在数组中
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);  
        }  

		//循环执行空闲消息处理器
        for (int i = 0; i < pendingIdleHandlerCount; i++) {  
            final IdleHandler idler = mPendingIdleHandlers[i];  
            mPendingIdleHandlers[i] = null; // release the reference to the handler  
  
            boolean keep = false;  
            try {  
                keep = idler.queueIdle();  //返回一个布尔值表示是否保留处理器
            } catch (Throwable t) {  
                Log.wtf(TAG, "IdleHandler threw exception", t);  
            }  
  
            if (!keep) {  
                synchronized (this) {  
                    mIdleHandlers.remove(idler);  
                }  
            }  
        }  
  
        // Reset the idle handler count to 0 so we do not run them again.  
        pendingIdleHandlerCount = 0;  
  
        // While calling an idle handler, a new message could have been delivered  
        // so go back and look again for a pending message without waiting.        nextPollTimeoutMillis = 0;  
    }  
}
```
**空闲消息**：用于在应用程序或系统空闲时执行任务或处理后台工作。这些消息通常被用于执行一些不紧急但需要在系统或应用程序空闲时进行的工作，以充分利用资源和提高系统性能。比如后台任务，资源管理器或者定时器等。
### 消息分发处理
当消息队列中有新的消息并且处于唤醒状态是，`Looper`会取出消息分发给合适的处理者。
消息的分发是由优先顺序的：
1. 首先考虑`Message.Callback()`处理，如果通过`post`发送的消息直接进行这一步处理，`send`发送的消息没有这个回调函数。
2. 如果不存在就调用`Handler.Callback()`处理，可以拦截消息。
3. 如果还是不存在就调用`Handler.handleMessage`，这个接口需要子类实现。
```java
public void dispatchMessage(@NonNull Message msg) {
  //消息加入消息队列时就已经设置好Callback
  //Callback的意义：可以不派生子类来创建一个对象
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
          //调用Handler的Callback处理消息
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                  // 可以拦截消息，之后 Handler.handleMessage 将无法继续处理这个消息。
                    return;
                }
            }
          //处理消息
            handleMessage(msg);
        }
    }
```
`dispatchMessage` 方法首先检查消息中是否包含 `Callback`，如果包含，则使用 `Callback` 处理消息；如果没有 `Callback`，则检查 `Handler` 自身是否设置了一个 `Callback`，如果设置了并且 `Callback` 拦截了消息，那么消息处理在这里终止；最后，如果前两者都不满足，就会调用 `handleMessage` 方法来处理消息。
#### 好处
将消息处理逻辑封装在 `Callback` 对象中，可以提高代码的模块化和可重用性。这意味着你可以创建通用的 `Callback` 对象，并在多个地方重用它们，而不必在每个地方都创建自定义的 `Handler` 子类。
