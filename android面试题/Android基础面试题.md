Android 基础面试题
================

# 广播
## 广播的两种注册方法，有什么区别？
- 动态注册：通过调用 Context.registerReceiver() 进行注册，跟随组件的生命周期，特定时刻监听广播。
- 静态注册：在 AndroidMainfest.xml 中声明，程序关闭后仍会调用，常驻广播，时刻监听广播。

**跨进程场景只能用 BroadCast**。 

## LocalBroadcast   
android 官方建议在一个进程内通信使用 localbroadcast 来代替 broadcast。    
本地广播是无法通过静态注册来实现的，因为静态注册是为了让程序未启动也能接收广播。

## EventBus 和 Broadcast 的区别？
官方解释：

>和Android的 BroadcastReceiver/Intent 不同，EventBus 用了标准java类的事件，提供了更方便的API。当你要用很多case去区分事件时，EventBus很一个很好的解决方案，你不需要大量设置intent、给intent添加extras、实现大量broadcast receivers、再获得intent、从intent里提取数据。
>而且，EventBus 的开销低得多。

**广播的优势体现在它与 android sdk 链接的更紧密，当我们需要同 android 交互的时候，广播提供的便捷性抵消掉了它过多的资源消耗。但是对于不需要同 android 交互或是只做很少的交互的时候，使用广播往往是一种浪费。**    
许多系统级的事件都是使用广播来进行通知的，像常用的电量变化、网络状态变化、短信发送接收的状态等等，这就使得与 android 系统相关的通知，广播往往成了唯一的选择。

# EventBus

EventBus 作为 Android 开发中常用的框架，拥有着许多优点：

- 调度灵活。不依赖于 Context，使用时无需像广播一样关注 Context 的注入与传递。父类对于通知的监听和处理可以继承给子类，这对于简化代码至关重要；通知的优先级，能够保证 Subscriber 关注最重要的通知；粘滞事件（sticky events）能够保证通知不会因 Subscriber 的不在场而忽略。**可继承、优先级、粘滞，是 EventBus 比之于广播、观察者等方式最大的优点，它们使得创建结构良好组织紧密的通知系统成为可能**。
- 使用简单。EventBus 的 Subscriber 注册非常简单，调用 eventBus 对象的 register 方法即可，如果不想创建 eventBus 还可以直接调用静态方法 EventBus.getDefault() 获取默认实例，Subscriber 接收到通知之后的操作放在 onEvent 方法里就行了。成为 Publisher 的过程就更简单了，只需要调用合适的 eventBus（自己创建的或是默认的）的 post 方法即可。
- 快速且轻量。作为 github 的明星项目之一， EventBus 的源代码中许多技巧来改善性能。

EventBus 的缺点是在 Subscriber 注册的时候，Subscriber 中的方法会被遍历查找以 onEvent 开头的 public 方法，一旦对代码进行混淆，就无法查找到了，所以一个程序既用到了 EventBus 又需要进行代码混淆时，就得设置混淆规则。    
**不过新版可能用注解来实现，能解决混淆问题**。

# SQlite 的 getReadableDatabase() 和 getWritableDatabase() 
创建或打开一个现有数据库，返回一个可对数据库进行读写操作的对象。    
当数据库不可写入的时候 **getReadableDatabase()** 以只读的方式返回数据库对象，**getWritableDatabase()** 返回异常。

# include、merge、stub 三种使用场景
- include：解决重复定义相同布局的问题。
 - xml1 中 include 了 xml2 并给其设定 id1，以 id1 为准
 - xml1 没有给 xml2 设定id，以 xml2 的根目录 id2 为准
- Merge: 帮助 include 减少视图层级，可以删除多余的层级，优化 UI。    
  include 标签的 parent ViewGroup 与包含的 layout 根容器 ViewGroup 是相同的类型，那么则可以将包含的 layout 根容器 ViewGroup 使用 merge 标签代替，从而减少一层 ViewGroup 的嵌套，从而提升 UI 性能渲染。
- ViewStub: 按需加载，减少内存使用量、加快渲染速度。**不支持 merge 标签**

# Activity 启动模式
## standard
新建 Activity 实例。   
 
- **Android 5.0 以前**，新建的 Activity 会放在同一个 task 下 Activity 栈的顶部，哪怕前后两个 Activity 是来自不同的应用。
- **Android 5.0 以后**，如果两个 Activity 来自不同的应用，就会新建一个 task，**这是因为 android 5.0 以后的 task 管理器有修改以更直观有效**。

## singleTop
- 如果 Activity 不在栈顶，可以无限创建Activity。    
- 如果 Activity 实例已经在栈顶了，那么就不会创建新的 Activity 实例，而是会将 intent 传入已有的 Activity 的实例的 *onNewIntent()* 方法。

## singeTask
**只允许一个Activity实例**。  

1. 同一个应用：  
  如果已经存在 Activity 实例，那么 task 中的这个实例会被置顶，这个过程中，Activity 栈中在其上的 Activity 都会被销毁，并且 intent 会被传给 *onNewIntent()*。
 
2. 不同的应用：
  - 系统中没有 Activity，创建新 task。
  - 系统中有一个 task 有那个 Activity，切到那个 task，同时销毁上面的 Activity，**如果用户按了返回键，就会回到唤起 Activity 的 task。**

## singeInstance
类似 singleTask。    
新建这个 Activity 的时候，就会创建一个新的task。
这种模式很少用。

# Bitmap 与 OOM
## OOM 一般在什么时候发生？
1. Bitmap 的使用不当
 - 单个 ImageView 加载高清大图的时候
 - ListView 或者 GridView 等批量快速加载图片的时候
  
2. 线程管理不当（利用线程池）

## 如何高效加载大图？
通过 BitmapFactory.Options 通过指定的采样率来缩小图片的分辨率。

# LRU 缓存
## Hash 原理
通过 Hash 函数将 key 转为 value。

## 碰撞
不同的关键字，通过Hash函数取到的Hash值相同。

## HashMap 怎么处理碰撞？
**拉链法**。    
![](http://double0291.github.io/Collection/collision_1.png)

## LinkedHashMap 是什么？
LinkedHashMap 在 HashMap 的基础上**维护着一个运行于所有条目的双重链接列表，此链接列表定义了迭代顺序，该迭代顺序可以是插入顺序或者是访问顺序**。    
LinkedHashMap 的构造函数有个变量：**accessOrder**，设置为true则为访问顺序，为false，则为插入顺序。

## LRU 是什么？
LRU 是“Least Recently Used”的缩写，就是最近最少使用。    
LRU 缓存把最近最少使用的数据移除，让给最新读取的数据。    
LRU 缓存用到的数据结构就是 **LinkedHashMap**。    
**LruCache中维护了一个集合LinkedHashMap，该LinkedHashMap是以访问顺序排序的。当调用put()方法时，就会在结合中添加元素，并调用trimToSize()判断缓存是否已满，如果满了就用LinkedHashMap的迭代器删除队尾元素，即近期最少访问的元素。当调用get()方法访问缓存对象时，就会调用LinkedHashMap的get()方法获得对应集合元素，同时会更新该元素到队头。**

# Android 版本新特征
- 5.0:
 1. Material Design
 2. 支持多种设备
 3. 全新通知中心
 4. 支持 64 位 ART 虚拟机
 5. 电池续航改进
 6. 全新“最近应用程序”
 7. 安全性改进
 8. 不同数据独立保存
 9. 改进搜索
 10. 支持蓝牙 4.1、USB Audio、多人分享等
- 6.0：
 1. 动态权限管理
 2. 系统层支持指纹识别
 3. APP 关联
 4. Android Pay
 5. 电源管理
 6. TF 卡默认存储
- 7.0：
 1. 分屏多任务
 2. 下拉快捷开关
 3. 新通知消息
 4. 夜间模式
 5. 流量保护模式
 6. 全新设置样式
 7. 改进 Doze 休眠机制
 8. 系统级电话黑名单
 9. 菜单键快速切换应用
- 8.0：
 1. 画中画
 2. 通知标志
 3. 自动填充框架
 4. 系统优化
 5. 后台限制
