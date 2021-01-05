# 分析
## 总览：Handler机制--用于线程间通讯
背景：线程B持有线程A中创建的Handler对象

1、在线程B中，Handler调用sendMessage、post方法发送Message，并插入到MessageQueue中。

2、Handler有一个Looper成员变量，创建Handler时会调用Looper.myLooper方法获取当前线程的Looper并设置给自己。Looper有一个MessageQueue成员变量，Handler持有Looper主要是为了拿到Looper对应的MessageQueue，并往其中插入消息。

3、Looper.loop方法开启消息循环，Looper会循环从其MessageQueue中提取消息，并调用消息的target成员变量（也就是Handler）进行分发处理。

5、Handler拿到Message后，先判断Message的Callback是否为空，不为空直接执行，消息处理结束；为空则判断Handler的Callback是否为空，不为空则执行，并决定是否进行拦截，拦截则消息处理结束；不拦截则执行Handler的handleMessage方法。

## 具体分析
### Handler--发布新消息
1、发送消息
```java
// Handler.java
public final boolean sendMessage(Message msg){
    // 默认延迟时间为0ms
    return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis){
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    // 设置Message的target为Handler自己，target是Message类的一个成员变量，为Handler类型的
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

2、插入消息到MessageQueue中
```java
// MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
    // 检查Message的target不能为空，因为后面Looper取出消息后，交由这个target（Handler）处理的
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    synchronized (this) {
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        
        if (p == null || when == 0 || when < p.when) {
            // 将Message置于消息链表的表头位置
            msg.next = p;
            mMessages = msg;
        } else {
            Message prev;
            // 死循环遍历Message链表
            for (;;) {
                prev = p;
                p = p.next;
                // 如果当前Message的延迟执行时间比所对比的Message执行时间小，就插到它前面
                // 不然继续循环，和多对比Message的next所指向的Message继续对比
                if (p == null || when < p.when) {
                    break;
                }
            }
            // 将当前Message插入消息链表
            msg.next = p;
            prev.next = msg;
        }
    }
    return true;
}
```

### MessageQueue--消息队列
MessageQueue采用单链表结构，表头的Message最先被取出执行。
![image](https://user-images.githubusercontent.com/18191520/97127861-c0e86400-1775-11eb-81a9-378e563a7d71.png)

### Looper--提取消息
```java
// Looper.java
public static void loop() {
	// 从ThreadLocal中取出Looper
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    for (;;) {
        Message msg = queue.next();
        if (msg == null) {
            // 没有消息表明消息队列正在退出
            return;
        }
        //...
        try {
            msg.target.dispatchMessage(msg);
        }
        //...
    }
}
```

### Handler--处理消息
先判断Message的Callback是否为空，不为空直接执行，消息处理结束；为空则判断Handler的Callback是否为空，不为空则执行，并决定是否进行拦截，拦截则消息处理结束；不拦截则执行Handler的handleMessage方法。

```java
// Handler.java
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
```

# 问题
## Looper.loop()中MessageQueue.next()取延时消息时，都在主线程中使用了死循环为什么不会卡死？

MessageQueue在取消息时，如果是延时消息会计算得到延时时长，再次循环时会执行nativePollOnce阻塞线程相应的时长。阻塞之后被唤醒的时机是阻塞时间到；或有新的消息添加进队列后，在enqueueMessage方法中调用nativeWake来唤醒阻塞线程。

## MessageQueue是什么数据结构
MessageQueue并不是队列，而是[链表结构](https://github.com/xcy396/ComputerYogurt/issues/5#issuecomment-716344691)

## Handler的postDelay,时间准吗?为什么?它用的是system.currentTime吗？
1、不准
2、因为Looper里从MessageQueue里取出来的消息执行也是串行的。如果前一个消息是比较耗时的，那么等到执行之前延时的消息时，时间难免可能会超过延时的时间。
3、postDelay时用的是System.uptimeMillis，也就是开机时间。因为系统时间用户可以手动更改，故使用开机时间。

## 子线程run方法中使用Handler
1、首先在子线程中调用Looper.prepare()方法初始化子线程的Looper（会同时初始化MessageQueue）
2、在子线程中调用Looper.loop()方法启动消息循环
3、创建Handler将子线程的Looper对象和Handler绑定，就可以使用子线程的Handler了

UI线程中系统已经自动调用Looper.prepareMainLooper()方法初始化了UI线程的Looper

## 假设先postDelay 10ms一条消息A，再postDelay 1ms一条消息B，消息的处理流程是怎样？
如果当前MessageQueue中没有消息，会将消息直接插入队列中。由于Looper的loop方法中是一直在循环取消息的，当在MessageQueue的next方法中取出延时消息A时，会阻塞线程10ms。发送消息B时，在MessageQueue的enqueueMessage方法中会调用nativeWake方法唤醒线程。将消息B插入队列，发现又是延时1ms，再次阻塞线程。当1ms过后，线程唤醒执行消息B，后续执行消息A。

## 主线程的Looper, 第一次被调用loop方法, 在什么时候吗? 哪一个类
1、在应用被打开的时候执行
2、在ActivityThread类中依次调用`Looper.prepareMainLooper()`和`Looper.loop()`方法

## IdleHandler（闲时机制）理解
* IdleHandler是一个回调接口，可以通过MessageQueue的addIdleHandler添加实现类。
* 当MessageQueue中的任务暂时处理完了（没有新任务或者下一个任务延时在之后），这个时候会回调这个接口。返回false，执行完后就会移除它，返回true会在下次message处理完空闲的时候继续回调。

```java
Looper.myQueue().addIdleHandler(new IdleHandler() {  
    @Override  
    public boolean queueIdle() {  
        //你要处理的事情
        return false;    
    }  
});
```

## HandlerThread理解
* HandlerThread是Thread的一个子类，在其内部开启了消息循环机制
* 使用HandlerThread可以很方便的开启一个包含Looper的线程，开启线程后，可以使用和该线程绑定的Looper去构建相应的Handler

使用示例：
```java
HandlerThread handlerThread = new HandlerThread("test");
handlerThread.start();
Handler handler = new Handler(handlerThread.getLooper()) {
	@Override
	public void handleMessage(Message msg) {
	super.handleMessage(msg);
		//...
	}
};
handlerThread.quit();
handlerThread.quitSafely();
```

## Message.obtain()的理解，消息池是如何维护的呢
* Message.obtain()方法从缓冲的消息池中取出第一个消息来使用，避免消息对象的频繁创建和销毁
* 消息池使用链表形式实现
* Message在分发使用后，调用其recycleUnchecked()方法执行回收操作，将该消息内部数据清空并添加到消息链表最前边

从消息池中取出第一个消息：
```java
// Message.java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

回收消息：
```java
// Message.java
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

