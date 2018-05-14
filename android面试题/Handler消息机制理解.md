Handler 消息机制理解
==================

# 基础使用
核心是**在当前线程创建一个跟 Looper 绑定的 Handler，然后把这个 Handler 暴露给别的线程**。    
简单来说，**要给谁发消息，就要用谁的Handler。**    
使用 *mHandler = new Handler(Looper.myLooper()); * 可生成来处理当前线程的Handler对象。    
使用 *new Handler(Looper.getMainLooper());* 可生成来处理main线程的Handler对象。    
可以实现主线程向子线程发消息，子线程向主线程发消息，子线程跟子线程互发消息。
    
Handler 创建会采用 Looper 来构建内部的 MessageQueue，如果当前线程没有 Looper 就会报错。    

```java
if (mLooper == null) {
    throw new RuntimeException(    
      "Can't create handler inside thread that has not called Looper.prepare()");    
} 
```

- 在主UI线程中，系统已经初始化好一个 Looper 对象，可以直接创建 Handler 并使用即可
- 子线程中，需要 
    1. Looper.prepare()
    2. 暴露 Handler
    3. Looper.loop()

# 为什么不能在子线程中访问 UI ？
UI 控件是线程不安全的。    
对 UI 控件访问加锁会降低 UI 访问效率。

# ThreadLocal 原理
一般来说，当数据是以线程为作用域并且不同线程有不同数据副本的时候，可以考虑使用 ThreadLocal。    
比如 Looper 的作用域是线程并且不同线程具有不同 Looper，就可以通过 ThreadLocal 去实现。    
**ThreadLocal 内部会从各自的线程中取出一个数组，然后再从数组中根据当前 ThreadLocal 的索引去查找对应的 value 值。**

# MessageQueue 原理
尽管 MessageQueue 叫消息队列，其内部实际是通过一个**单链表**的数据结构来维护的。    
MessageQueue 的 next() 方法是一个无限循环，如果消息队列中没有消息，那么 next 方法就一直阻塞。    
MessageQueue是消息机制的Java层和C++层的连接纽带，大部分核心方法都交给native层来处理。   
阻塞在 next() 方法的 nativePollOnce() 方法中，这时候主线程会释放 CPU 资源进入休眠状态，直到下一个消息到达或者事务发送，才通过 pipe 管道写端写入数据来唤醒主线程。    
这里涉及到 Linux 的 pipe/epoll 机制，epoll 机制是一种 IO 多路复用机制，可以同时监控多个描述符，当某个描述符就绪（读或写就绪），就立刻通知相应程序进行读或写操作，本质同步 IO，即读写是阻塞的。    
**所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。**

# Looper 原理
通过 *Looper.prepare()* 来为当前线程创建一个 Looper，再通过 *Looper.loop()* 来开启消息循环。    
Looper 处理消息：*msg.target.dispatchMessage(msg)*，这里的 msg.target 就是发消息的 Handler，所以 Handler 发送的消息最终又交给它的 dispatchMessage() 方法来处理。    
**但 dispatchMessage() 方法是在 Looper 中，Looper 又是创建 Handler 时那个线程中的，这样就将 Message 的处理切换到指定线程中了**。

```java
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

    // 死循环 
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

        // msg.target 就是 handler
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

        msg.recycleUnchecked(); // 将msg回收到message回收池中
    }
}
```

# Handler 原理
Handler 发消息是想 MessageQueue 中插入一条消息。    
MessageQueue 的 next() 方法会将这条消息传给 Looper。    
Looper 又将消息交给 Handler 处理，即 Handler 的 dispatchMessage()。    
所以使用的核心是**在当前线程创建一个跟 Looper 绑定的 Handler，然后把这个 Handler 暴露给别的线程**。    
简单来说，**要给谁发消息，就要用谁的Handler。**

# Android中为什么主线程不会因为 Looper.loop() 里的死循环卡死？
主线程的 MessageQueue 没有消息时，便阻塞在 loop 的 queue.next() 中的 nativePollOnce() 方法里。

简单一句话是：**Android应用程序的主线程在进入消息循环过程前，会在内部创建一个Linux管道（Pipe），这个管道的作用是使得Android应用程序主线程在消息队列为空时可以进入空闲等待状态，并且使得当应用程序的消息队列有消息需要处理时唤醒应用程序的主线程**。

这一题是需要从消息循环、消息发送和消息处理三个部分理解Android应用程序的消息处理机制了，这里我对一些要点作一个总结： 
   
1.  Android应用程序的消息处理机制由消息循环、消息发送和消息处理三个部分组成的。
2.  Android应用程序的主线程在进入消息循环过程前，会在内部创建一个Linux管道（Pipe），这个管道的作用是使得Android应用程序主线程在消息队列为空时可以进入空闲等待状态，并且使得当应用程序的消息队列有消息需要处理时唤醒应用程序的主线程。
3.  Android应用程序的主线程进入空闲等待状态的方式实际上就是在管道的读端等待管道中有新的内容可读，具体来说就是是通过Linux系统的Epoll机制中的epoll\_wait函数进行的。
4.  当往Android应用程序的消息队列中加入新的消息时，会同时往管道中的写端写入内容，通过这种方式就可以唤醒正在等待消息到来的应用程序主线程。        
5.  当应用程序主线程在进入空闲等待前，会认为当前线程处理空闲状态，于是就会调用那些已经注册了的IdleHandler接口，使得应用程序有机会在空闲的时候处理一些事情。

# 注意点
1. **在子线程使用Handler前一定要先为子线程创建Looper，创建的方式是直接调用**
2. **在同一个线程里，Looper.prepare()方法不能被调用两次，不要在主线程调用Looper.prepare()方法**
3. **当我们在子线程使用Handler时，如果Handler不再需要发送和处理消息，那么一定要退出子线程的消息轮询。**    
    因为Looper.loop()方法里是一个死循环，如果我们不主动结束它，那么它就会一直运行，子线程也会一直运行而不会结束。    
    退出消息轮询的方法是：
    - *Looper.myLooper().quit()*
    - *Looper.myLooper().quitSafely()*

# Handler 内存泄漏
```java
public MyActivity extends Activity {
    ...
    Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            ...
        }
    }
    ...
}
```

**在java中非静态内部类和匿名内部类都会隐式持有当前类的外部引用**。    
由于 Handler 是非静态内部类,所以其持有当前 Activity 的隐式引用，如果 Handler 没有被释放，其所持有的外部引用也就是 Activity 也不可能被释放。    
如果 *mHandler.postDelayed()*，Activity 就可能不被释放。    

解决方案：

1. Handler 设为静态内部类 + Handler 里 Activity 引用设置为弱引用
2. 在 *onDestroy()* 中调用一下 *handler.removeCallbacksAndMessages(null);* —— 如果消息接受遗漏，最佳方案
