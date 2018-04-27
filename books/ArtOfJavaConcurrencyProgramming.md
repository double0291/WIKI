## 第一章 并发编程的挑战
#### 1.1 上下文切换
CPU 分配给各个线程时间，通过时间片分配算法来循环执行任务，时间片一般是**几十毫秒（ms）**，**所有任务从保存到再加载的过程就是一次上下文切换**。    
实践中应避免创建不需要的线程，太多线程处于 WAITING 状态会增加上下文切换的次数和时长。

#### 1.2 死锁
避免死锁的几个常见方法：

- 避免一个线程同时获得多个锁
- 避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源
- 尝试使用定时锁，使用 lock.tryLock(time) 来替代使用内部锁机制
- 对于数据库锁，加锁和解锁必须在一个数据库连接里，否则会出现解锁失败的情况

#### 1.3 资源限制的挑战
并发编程时，程序的执行速度受限于计算机硬件资源或软件资源。

- 对于硬件资源限制，可以考虑使用集群并发执行程序
- 对于软件资源限制，可以考虑使用资源池将资源复用

#### 建议多使用JDK并发包提供的并发容器和工具类来解决并发问题，因为这些类都已经过充分的测试和优化。

## 第二章 Java 并发机制的底层实现原理
#### 2.1 volatile 的应用
volatile 是轻量级的 synchronized，在并发处理中保证**一个线程修改一个共享变量的时候，另一个线程能读到修改的值**。    
volatile 变量修饰符使用恰当的话，比 synchronized 使用和执行成本更低，因为它不会引起程序上下文的切换和调度。

###### 2.1.1 volatile 定义与原理实现
如果一个字段被声明为 volatile，那么 Java 线程内存模型确保所有的线程看到这个变量的值是一致的。    
volatile 两个实现原则：    

1. Lock前缀指令会引起处理器缓存回写到内存
2. 一个处理器缓存回写到内存会导致其他处理器的缓存失效

#### 2.2 synchronized 的实现原理与应用
一个线程试图访问同步代码块时，它首先必须得到锁；退出或抛出异常时必须释放锁。
java 中每一个对象都可以作为锁，具体有3种形式：

- 对于普通同步方法，锁是当前实例对象
- 对于静态同步方法，锁是当前类的 Class 对象
- 对于同步方法块，锁是 Synchronized 括号里配置的对象

#### 2.3 原子操作
从 Java1.5 开始，JDK的并发包提供一些类支持原子操作，如 AtomicBoolean、AtomicInteger 和 AtomicLong。这些原子包装类还提供了一些有用的工具方法，比如以原子的方式将当前值自增 1 和自减 1。

## 第三章 Java 内存模型
#### 3.1 Java 内存模型的基础
###### 3.1.1 并发编程模型的两个关键问题
线程之间的通信机制有两种：

1. 共享内存
2. 消息传递

Java 并发采用的是**共享内存模型**。

###### 3.1.2 Java 内存模型的抽象结构
java 中，所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。    
Java 线程之间的通信由 Java 内存模型（简称 JMM）控制，JMM 决定一个线程对共享变量的写入何时对另一个线程可见。    
抽象角度看，JMM 定义了线程和主内存之间的抽象关系：**线程之间的共享变量存储在主内存（ Main Memory ）中，每个线程都有一个私有的本地内存（ Local Memory ），本地内存中存储了该线程以读/写共享变量的副本。**    
    
如果线程 A 与线程 B 之间要通信的话，必须经历2个步骤。
1）线程 A 把本地内存更新过的共享变量刷新到主内存中    
2）线程 B 到主内存中去读取线程 A 之前已更新过的共享变量    
    
JMM 通过控制主内存与每个线程的本地内存之间的交互，来为 Java 程序员提供内存可见性保证。

#### 3.4 Volatile 的内存定义
###### 3.4.1 volatile 的特性
理解 volatile 特性的一个好方法是把对 volatile 变量的单个读/写，看出是使用同一个锁对这些单个读/写操作做了同步。

```java
class VolatileFeaturesExample {
	volatile long v1 = 0L;

	public void set(long l) {
		v1 = l;
	}

	public void getAndIncrement() {
		v1++;
	}

	public long get() {
		return v1;
	}
}
```
等同于下面这段代码：

```java
class VolatileFeaturesExample {
	long v1 = 0L;

	public synchronized void set(long l) {
		v1 = l;
	}

	public void getAndIncrement() {
		long temp = get();
		temp += 1L;
		set(temp);
	}

	public synchronized long get() {
		return v1;
	}
}
```


volatile 写的内存定义是：    
**当写一个 volatile 变量时，JMM 会吧该线程对应的本地内存中的共享变量值刷新到主内存中。**    
volatile 读的内存定义是：    
**当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。**

#### 3.5 锁的内存语义
线程获取锁时，JMM 会把该线程对应的本地内存置为无效，从而使被监视器保护的临界区代码必须从主内存中读取共享变量。

#### 3.8 双重检查锁定与延迟初始化
###### 3.8.1 双重检查锁定的由来
```java
public class UnsafeLazyInitialization {
	private static Instance instance;

	public static Instance getInstance() {
		if (instance == null) 			// 1：A 线程执行
			instance = new Instance();	// 2：B 线程执行

		return instance;
	}
}
```

假设 A 线程执行代码 1 的同时，B 线程执行代码 2，就可能初始化两次。    
针对这个，可以对 getInstance() 方法做同步处理来实现线程安全的延迟初始化。    

```java
public class SafeLazyInitialization{
	private static Instance instance;

	public synchronized static Instance getInstance() {
		if (instance == null)
			instance = new Instance();

		return instance;
	}
}
```

synchronized 会导致内存开销。    
如果 getInstance() 方法被多个线程频繁调用，将会导致程序执行性能的下降。反之，如果 getInstance() 方法不被多个线程频繁调用，这个方案还算令人满意。    
早期的 JVM 中，synchronized存在巨大性能消耗，所以人们想出了**双重检查锁定**，通过它来降低同步的开销：

```java
public class DoubleCheckLazyInitialization{							// 1
	private static Instance instance;								// 2

	public static Instance getInstance() {							// 3
		if (instance == null) {										// 4：第一次检查
			synchronized (DoubleCheckLazyInitialization.class) { 	// 5：加锁
				if (instance == null)								// 6：第二次检查
					instance = new Instance();						// 7：问题根源
			}														// 8
		}															// 9

		return instance;											// 10
	}																// 11
}
```

如果第一次检查 instance 非空，就不需要执行下面的加锁和初始化操作，可以大幅度降低 synchronized 带来的性能开销。    
**但这是一个错误的优化！线程执行到第 4 行，代码读取到 instance 不为 null时，instance 引用的对象可能还没有完全初始化。**    
###### 3.8.2 问题的根源
上面代码的第7行（ instance = new Instance(); ）创建了一个对象。    
这一行可以分解为3行伪代码。    

```java
memory = allocate();	// 1：分配对象的内存空间
ctorIntance(memory);	// 2：初始化对象
instance = memory;		// 3：设置 instance 指向刚分配的内存地址
```

而在一些编译器上面，上面伪代码的第 2 行和第 3 行可能会被重排，重拍后如下：

```java
memory = allocate();	// 1：分配对象的内存空间
instance = memory;		// 3：设置 instance 指向刚分配的内存地址
						// 注意，此时对象还没有被初始化！
ctorIntance(memory);	// 2：初始化对象
```

根据重排规则，重排不改变单线程执行结果，上面的重排并没有违反规则。    
这就导致线程 A 和线程 B 按次执行后，线程 B 可能看到一个没有被初始化的对象！

###### 3.8.3 基于 volatile 的解决方案
对于上面的双重检查锁定，只需要**把 instance 声明为 volatile 对象**，就可以线程安全的延迟初始化，因为声明为 volatile 后，上述的重排序会被禁止（注意，需要 JDK5 以上版本）：

```java
public class SafeDoubleCheckLazyInitialization{
	private volatile static Instance instance;

	public static Instance getInstance() {
		if (instance == null) {
			synchronized (DoubleCheckLazyInitialization.class) {
				if (instance == null)
					instance = new Instance();
			}
		}

		return instance;
	}
}
```

###### 3.8.4 基于类初始化的解决方案
JVM 会在类的初始化阶段（就是 Class 被加载后，且被多线程使用前），会执行类的初始化。    
在执行类初始化期间，JVM 会去获取一个锁，这个锁可以同步多个线程对同一类的初始化。    
基于这个特性，可以实现另一种线程安全的延迟初始化方案：

```java
public class InstanceFactory {
	private static class InstanceLoader {
		public static Instance instance = new Instance();
	}

	public static Instance getInstance() {
		return InstanceLoader.instance;	// 这里将导致 InstanceLoader 类被初始化
	}
}
```

根据 Java 语言规范，首次发生下面任意一种情况，一个类或借口类型 T 将立即初始化：

1. T 是一个类，而且一个 T 类型的实例被创建
2. T 是一个类，且 T 中声明的静态方法被调用
3. T 中声明的一个静态字段被赋值
4. T 中声明的一个静态字段被使用，而且这个字段不是一个常量字段
5. T 是一个顶级类，而且一个断言语句嵌套在 T 内部被执行

## 第四章 Java 并发编程基础
#### 4.1 线程简介
###### 4.1.1 什么是线程
操作系统运行一个程序的时候，会为其创建一个进程。    
一个进程里可以创建多个线程，这些线程拥有各自的**计数器、堆栈和局部变量**等属性，并且能够访问共享的**内存变量**。    

###### 4.1.3 线程优先级
Java 线程通过一个整形成员变量 priority 来控制优先级，优先级的范围从1 ~ 10，默认优先级是5。    
**程序正确性不能依赖优先级高低，因为有些操作系统会忽略优先级设置**。

###### 4.1.4 线程的状态
 状态命 | 说明 
 NEW | 初始状态，线程被构建，但还没有调用 start() 方法
 RUNNABLE | 运行状态，Java 线程讲操作系统中的就绪和运行两种状态笼统的称作“运行中”
 BLOCKED | 阻塞状态，表示线程阻塞于锁
 WAITING | 等待状态，进入该状态表示当前线程需要等待其他线程做出一些待定动作（通知或中断）
TIME_WAITING | 超时等待状态，不同于 WAITING，它可以在指定的时间自行返回
TERMINATED | 终止状态，表示当前线程已经执行完毕

#### 4.2 启动和终止线程
###### 4.2.1 构造线程
运行线程之前首先构造一个线程对象，构造时需要属性，包括线程所属的线程组、线程优先级、是否是Deamon线程等。

###### 4.2.2 启动线程
**启动线程前，最好给这个线程设置名称，方便排查问题**。

###### 4.2.3 中断
中断可以理解为一个程序的标志位属性。    
一个线程在运行中，其他线程调用该线程的 interrupt() 方法对其进行中断操作。    
许多声明抛出 InterruptedException 的方法（例如 Thread.sleep(long millis) 方法），这些方法在抛出 InterruptedException 之前，Java 虚拟机会先将该线程的中断标志位清除，此时调用 isInterrupted() 方法将会返回 false。

###### 4.2.4 过期的 suspend()、resume() 和 stop()
以上三个方法可以理解为 CD 机播放音乐过程中的 暂停、恢复和停止。    
suspend() 方法调用后，线程不会释放已经占有的资源（比如锁），会引发死锁问题。    
stop() 方法终结一个线程时也不保证资源正常释放。    
所以这些方法被废弃了。

###### 4.2.5 安全的终止线程
- 设置一个 volatile 的标志位来终止线程
- 通过中断操作

#### 4.3 线程间通信
###### 4.3.1 volatile 和 synchronized 关键字
Java 支持多线程同时访问一个对象或对象的成员变量。    
虽然对象以及对象的成员变量在共享内存中，但每个执行的线程还是可以拥有一份拷贝（这样做的目的是加速程序执行）。    
所以程序执行过程中，一个线程看到的变量并不一定是最新的。    
关键字 volatile 告知程序任何对该变量的访问均需要从共享内存中获取，对它的改变必须同步刷新到共享内存，保证所有线程看到的变量一致。    
过多使用 volatile 会降低程序执行效率。    

synchronized 可以修饰方法或者同步块，确保同一时刻，只有一个线程在方法或同步块中。

###### 4.3.2 等待 / 通知机制
等待 / 通知的相关方法是任意 java 对象都具备的。    
线程 A 调用了对象 O 的 wait() 方法进入等待状态，另一个线程 B 调用了对象 O 的 notify() 或 notifyAll() 方法，线程 A 收到通知后从对象 O 的 wait() 方法返回，进而执行后续操作。    
上述两个线程通过对象 O 来完成交互，而对象上的 wait() 和 notify/notifyAll() 方法就像开关信号一样，用来完成等待方和通知方的交互工作。    

- 使用 wait()、notify()、notifyAll() 时需要先对调用对象加锁
- 调用 wait() 方法后，线程状态由 RUNNING 变成 WAITING，并将当前线程放到等待队列，同时释放锁
- notify()/notifyAll() 方法调用后，等待线程依旧不会从 wait() 返回，需要等锁释放后，才有机会从 wait() 返回
- notify() 将等待队列中的一个等待线程移到同步队列中，而 notifyAll() 则将等待队列中所有线程全部移到同步队列，被移动的线程状态由 WAITING 变成 BLOCKED
- 从 wait() 方法返回的前提是获得了调用对象的锁

###### 4.3.3 等待 / 通知的经典范式
消费者：    

1. 获取对象的锁
2. 条件不满足，调用对象的 wait() 方法，被通知后仍要检查条件
3. 条件满足则执行对应的逻辑

```
synchronized(对象) {
	while(条件不满足) {
		对象.wait();
	}
	处理逻辑
}
```

生产者：    

1. 获取对象的锁
2. 改变条件
3. 通知所有等待在对象上的线程

```
synchronized(对象) {
	改变条件
	对象.notifyAll();
}
```


###### 4.3.4 管道输入 / 输出流

###### 4.3.5 Thread.join() 的使用
一个线程 A 执行了 thread.join() 语句，含义是：**当前线程 A 等待 thread 线程终止之后才从 thread.join() 返回**。

###### 4.3.5 ThreadLocal 的使用
ThreadLocal，即线程变量，是一个以 ThreadLocal 对象为键，任意对象为值的存储结构。    
这个结构被附带到线程上，也就是说一个线程可以根据一个 ThreadLocal 对象查询到绑定在这个线程上的值。

## 第五章 Java 中的锁
#### 5.1 Lock 接口
Lock 接口出现之前，Java 程序依靠 synchronized 关键字实现锁功能。    
Java SE 5 之后，并发包中新增了 Lock 接口实现锁功能。    
虽然不便捷，但其拥有多种 synchronized 关键字不具备的同步特性：

- 锁获取和释放的可操作性
- 可中断的获取锁
- 超时获取锁

Lock 的使用很简单：

```java
Lock lock = new ReentrantLock();
lock.lock();
try {

} finally {
	lock.unlock();
}
```

**释放锁的过程写在 finally 里面，而不是 try 里面。**    
Lock 接口的实现基本都是通过聚合了一个同步器的子类来完成线程访问控制的。

## 第六章 Java 并发容器和框架
#### 6.1 ConcurrentHashMap 的实现原理与使用

###### 6.1.1 为什么使用 ConcurrentHashMap
HashMap 在并发编程中可能导致死循环，而使用线程安全的 HashTable 效率又底下，所有有了 ConcurrentHashMap。    
**ConcurrentHashMap 使用锁分段技术：首先将数据分成一段一段存储，然后每一段数据配一把锁，当一个线程占用锁访问其中一段数据时，其他段数据也能被其他线程访问。**    


#### 6.2 ConcurrentLinkedQueue
ConcurrentLinkedQueue 是一个基于链接节点的无界限线程安全队列。

#### 6.3 Java 中的阻塞队列
阻塞队列（BlockingQueue）是一个支持两个附加操作的队列，这两个附加的操作支持阻塞的插入和移除方法：

1. 支持阻塞的插入方法：当队列满的时候，队列会阻塞插入元素的线程，直到队列不满。
2. 支持阻塞的移除方法：当队列为空时，获取元素的线程会等待队列变为非空。

阻塞队列经常用于生产者和消费者的场景。

#### 6.4 Fork/Join 框架
Fork 就是把一个大任务切分成若干子任务并行的执行。    
Join就是合并这些子任务的执行结果，最后得到这个大任务的结果。    


## 第七章 Java 中的13个原子操作类


## 第八章 Java 中的并发工具类
#### 8.1 等待多线程完成的 CountDownLatch
CountDownLatch 允许一个或多个线程等待其他线程完成操作。

#### 8.2 同步屏障 CyclicBarrier
CyclicBarrier 字面意思是“可循环”的“屏障”，它做的事情是让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

###### 8.2.1 CyclicBarrier 与 CountDownLatch 的区别
CountDownLatch 计数器只能使用一次，而 CyclicBarrier 的计数器可以使用 set() 方法重置。    
所以 CyclicBarrier 能处理更为复杂的业务场景，比如计算发生错误，可以重置再执行。







