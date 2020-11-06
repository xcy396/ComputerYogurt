### Handler机制--用于线程间通讯
背景：线程B持有线程A中创建的Handler对象

1、在线程B中，Handler调用sendMessage、post方法发送Message，并插入到MessageQueue中。

2、Handler有一个Looper成员变量，创建Handler时会调用Looper.myLooper方法获取当前线程的Looper并设置给自己。Looper有一个MessageQueue成员变量，Handler持有Looper主要是为了拿到Looper对应的MessageQueue，并往其中插入消息。

3、Looper.loop方法开启消息循环，Looper会循环从其MessageQueue中提取消息，并调用消息的target成员变量（也就是Handler）进行分发处理。

5、Handler拿到Message后，先判断Message的Callback是否为空，不为空直接执行，消息处理结束；为空则判断Handler的Callback是否为空，不为空则执行，并决定是否进行拦截，拦截则消息处理结束；不拦截则执行Handler的handleMessage方法。

#### Handler--发布新消息
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

#### MessageQueue--消息队列
MessageQueue采用单链表结构，表头的Message最先被取出执行。
![image](https://user-images.githubusercontent.com/18191520/97127861-c0e86400-1775-11eb-81a9-378e563a7d71.png)

#### Looper--提取消息
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

#### Handler--处理消息
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