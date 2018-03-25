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

根据 Java 语言规范，首次发生下面任意一种情况，一个类或借口类型 T 将立即初始化。

1. T 是一个类，而且一个 T 类型的实例被创建
2. T 是一个类，且 T 中声明的静态方法被调用
3. T 中声明的一个静态字段被赋值
4. T 中声明的一个静态字段被使用，而且这个字段不是一个常量字段
5. T 是一个顶级类，而且一个断言语句嵌套在 T 内部被执行


