## 第2章 创建和销毁对象
- 如果类的构造器或静态工厂中具有多个参数，设计这种类时，**Builder** 模式就是不错的选择。
- 只有静态方法和静态域的工具类，通过显式的私有构造函数来强制该类不能实例化。
- 静态工厂方法优于构造器，例如 Boolean.valueOf(String) 优于 Boolean(String)。
- Java中常见的内存泄漏：
    1. 自己管理内存的类
    2. 缓存
    3. 监听器和其他回调

## 第3章 对于所有对象都通用的方法
- 覆盖 equals 要遵守通用约定（自反、对称、传递、一致）。
- 覆盖 equals 时总要覆盖 hashCode。
- toString 方法返回对象中值得关注的对象。

## 第4章 类和接口
- 使类和成员的可访问性最小化，尽可能使每个类或成员不被外界访问。
- 组合优于继承。
- 非静态成员类的每个实例都隐含地与外围类的一个实例相关联。
  **如果声明的成员类不要求访问外围类实例，就要声明为 static 类**，否则消耗时间空间，也容易造成内存泄漏。

## 第5章 泛型
- 使用泛型时，不确定或者不关心实际的类型参数，可以使用问号（？）代替。

```java
static int f(Set<?> s1, Set<?> s2) {
    int result = 0;
    for (Object o : s1) {
        if (s2.contains(o))
            result++;
	}
	return result;
}
```

- 无法消除警告，同时**确定引起警告的代码是类型安全的，可以用 @SuppressWarinings("unchecked") 注解来禁止这条警告**，尽量在方法上加，不要在类上加（那有可能掩盖重要警告），同时添加一条注释。

## 第7章 方法
- 不要导出具有相同参数数目的重载方法。
- 返回空的数组或集合，而不是 null。

## 第8章 通用程序设计
- 使局部变量的作用域最小化，使方法小而集中。
- for-each 循环优于传统的 for 循环。
- **如果需要精确的答案，避免使用 float 和double。（记住资金的 bug 案例）。
- 基本类型优于装箱基本类型。
- 字符串连接消耗性能。
- 如果有合适的**接口类型**，那么对于参数、返回值、变量和域来说，都应该使用接口类型进行声明。
- 每次试图优化的前后，要对性能进行测量。

## 第10章 并发
- 多个线程共享可变数据时，每个读或写数据的线程都必须执行同步。

一个隐蔽的同步 bug：

```java
private static volatile long nextSerialNum = 0;

public static long generateSerialNum() {
    return nextSerialNum++;
}
```
这段代码有问题，因为**增量操作符（++）不是原子的**。    
解决的方法：

1. 给 generateSerialNum() 方法添加 synchronized 修饰符
2. 使用 AtomicLong

```
private static final AtomicLong nextSerialNum = new AtomicLong();

public static long generateSerialNum() {
    return nextSerialNum.getAndIncrement();
}
```

- 避免过度同步，应该在同步区域内做尽可能少的工作。
- 观察者模式可以用 CopyOnWriteArrayList 实现，CopyOnWriteArrayList 里处理写操作（包括add、remove、set等）是先将原始的数据通过 JDK1.6 的 Arrays.copyof() 来生成一份新的数组，然后在新的数据对象上进行写，写完后再将原来的引用指向到当前这个数据对象，所以**适合使用在读操作远远大于写操作的场景里，比如缓存**。
- executor 和 task 优于自己开启线程。
- 始终使用 wait 循环模式来调用 wait 方法，永远不要在循环之外调用 wait 方法。
- 尽量使用 notifyAll，因为它保证唤醒所需要唤醒的线程，虽然可能唤醒一些其他线程，但是那些线程醒来，会去检查所等待的条件，发现条件不满足就继续等待。
- 如果能使用同步器（例如 CountDownLatch 或 CyclicBarrier）就尽量不要用 wait 和 notify。
- 慎用懒加载模式，需要用的话：1、基于类初始化；2、双重加锁检查模式。
- 不要让程序的正确性依赖于线程调度器（yield 或 调整线程优先级）。

## 第10章 序列化
- “将一个对象编码成一个字节流”的过程叫做**序列化**，相反的过程就叫“反序列化”。
- **实现 Serializable 接口而付出的最大代价就是，一旦一个类发布，就大大降低了“改变这个类”的灵活性**。
- 如果没有显式的指定序列版本 UID，系统会自动根据类来调用一个复杂的运算过程，从而在运行时产生 UID，这个 UID 跟类名称、实现的接口名称以及所有公有的或受保护成员的名称有关，一旦改变这些，自动产生的 UID 也会变，兼容性会被破坏。
- 实现 Serializable 接口另一个代价是，随着类发行新的版本，相关的测试负担也增加了。因为要检查“新版本中序列化一个实例，能否在旧版本中反序列化”。
- 内部类不应该实现 Serializable。