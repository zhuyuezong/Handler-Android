# Handler-Android
## 概述
Handler 、 Looper 、Message 这三者在Android都与异步消息处理线程相关，所以深入掌握并应用Handler异步处理机制在Android开发中显得特别重要。

## Looper
Looper的主要方法有两个，prepare()和loop()，Looper是线程用来运行消息循环(message loop)的类。默认情况下，线程并没有与之关联的Looper，可以通过在线程中调用Looper.prepare() 方法来获取，并通过Looper.loop() 无限循环地获取并分发MessageQueue中的消息，直到所有消息全部处理。<br>

首先看prepare()方法

```Java
public static final void prepare() {  
        if (sThreadLocal.get() != null) {  
            throw new RuntimeException("Only one Looper may be created per thread");  
        }  
        sThreadLocal.set(new Looper(true));  
}  
```

sThreadLocal是一个本地线程存储类，所有线程共用这个对象，但这个对象对于不同线程却又不同的值，也就是说它的值对不同的线程相对独立，每个线程对它的值修改不会影响其他线程<br>
Looper的构造方法

```Java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

看代码中源码，可以看出每个线程只能有一个Looper对象。创建一个新的消息队列，并且绑定当前线程。
再来看看Looper的loop()方法

```Java
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();//将msg回收到message回收池中
        }
    }
```
<br>

获取当前的消息队列，确认当前线程为本地线程后开启一个死循环从消息队列中获取消息并通过target对象dispatchMessage()方法发布消息。<br>

```Java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

...
private static void handleCallback(Message message) {
    message.callback.run();
}
```

这里有个优先级的问题
* Message的回调方法优先级最高，即message.callback.run()；
* Handler的回调方法优先级次之，即Handler.mCallback.handleMessage(msg)；
* Handler的默认方法优先级最低，即Handler.handleMessage(msg)。
消息方法完后对消息进行回收 msg.recycleUnchecked()

## Message
一个message对象包含一个自身的描述信息和一个可以发给handler的任意数据对象。这个对象包含了两个int 类型的extra 字段和一个object类型的extra字段。利用它们，在很多情况下我们都不需要自己做内存分配工作。 <br>

虽然Message的构造方法是public的，但实例化Message的最好方法是调用Message.obtain() 或 Handler.obtainMessage() ，因为这两个方法是从一个可回收利用的message对象回收池中获取Message实例。该回收池用于将每次交给handler处理的message对象进行回收。 <br>

Message对象包含两个额外的int类型变量和一个额外的对象，利用它们大多数情况下我们不用再做内存分配相关工作。实例化Message最好的方法是调用Message.obtain()或Handler.obtainMessage()（实际上最终调用的仍然是Message.obtain()），因为这两个方法都是从一个可回收利用的对象池中获取Messag的。<br>

## Handler
```Java
public Handler() {  
        this(null, false);  
}  
public Handler(Callback callback, boolean async) {  
        if (FIND_POTENTIAL_LEAKS) {  
            final Class<? extends Handler> klass = getClass();  
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&  
                    (klass.getModifiers() & Modifier.STATIC) == 0) {  
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +  
                    klass.getCanonicalName());  
            }  
        }  
  
        mLooper = Looper.myLooper();  
        if (mLooper == null) {  
            throw new RuntimeException(  
                "Can't create handler inside thread that has not called Looper.prepare()");  
        }  
        mQueue = mLooper.mQueue;  
        mCallback = callback;  
        mAsynchronous = async;  
    }  
```
上面源代码可以看到，通过Looper.myLooper()来获取当前线程的Looper对象，再从Looper对象获取消息队列。<br>
Handler最常用的方法是sendMessage()
```Java
public final boolean sendMessage(Message msg)  
{  
        return sendMessageDelayed(msg, 0);  
}  
...
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) 
{  
        Message msg = Message.obtain();  
        msg.what = what;  
        return sendMessageDelayed(msg, delayMillis);  
}  
...
 public final boolean sendMessageDelayed(Message msg, long delayMillis)  
 {  
        if (delayMillis < 0) {  
                delayMillis = 0;  
        }  
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);  
}  
...
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {  
       MessageQueue queue = mQueue;  
       if (queue == null) {  
           RuntimeException e = new RuntimeException(  
                   this + " sendMessageAtTime() called with no mQueue");  
           Log.w("Looper", e.getMessage(), e);  
           return false;  
       }  
       return enqueueMessage(queue, msg, uptimeMillis);  
}  
```
可以看到最终调用的是sendMessageAtTime(),最后调用enqueueMessage将消息存入消息队列，这里我们看看enqueueMessage方法

```Java
 private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
                msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
}
```
把当前的handler作为msg的target属性。最终会调用queue的enqueueMessage的方法，也就是说handler发出的消息，最终会保存到消息队列中去。<br>
最后再讲下Handler的post方法。直接上源码
```Java
public final boolean post(Runnable r)  
{  
      return  sendMessageDelayed(getPostMessage(r), 0);  
}  
```
实际上是handler发送了一条消息而已，那么getPostMessage方法作用是什么？

```Java
private static Message getPostMessage(Runnable r) {  
      Message m = Message.obtain();  
      m.callback = r;  
      return m;  
} 
```
这里看到，传入的Runnable作为Message的callback方法，如果使用这种方法发送消息，前面我们提到Looper的在loop(),无线循环调用，msg.target.dispatchMessage方法的优先级，就是先调用message.callback.run方法。也就是这里的Runnable了。
到这里就基本理解Handler的整个分发机制了。

