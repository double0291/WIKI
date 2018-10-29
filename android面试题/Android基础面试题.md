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

# LinearLayout 和 RelativeLayout 的对比
- RelativeLayout 会让子 View 调用2次 onMeasure，LinearLayout 在有 weight 时，也会调用子 View 的2次 onMeasure。
- RelativeLayout 的子 View 如果高度和 RelativeLayout 不同，则会引发效率问题，当子 View 很复杂时，这个问题会更加严重。如果可以，尽量使用 padding 代替 margin。
- 在不影响层级深度的情况下,使用 LinearLayout 和 FrameLayout 而不是 RelativeLayout。

# Thread、AsyncTask、IntentService的使用场景与特点
- Thread线程，独立运行与于 Activity 的，当 Activity 被 finish 后，如果没有主动停止 Thread 或者 run 方法没有执行完，其会一直执行下去。
- AsyncTask 封装了两个线程池和一个 Handler，（SerialExecutor 用于排队，THREAD_POOL_EXECUTOR 为真正的执行任务，Handler 将工作线程切换到主线程），其必须在 UI 线程中创建，execute 方法必须在 UI 线程中执行，一个任务实例只允许执行一次，执行多次将抛出异常，用于网络请求或者简单数据处理。
- IntentService：处理异步请求，实现多线程，在 onHandleIntent 中处理耗时操作，多个耗时任务会依次执行，执行完毕后自动结束。

# include、merge、ViewStub 三种使用场景
- include：解决重复定义相同布局的问题。
 - xml1 中 include 了 xml2 并给其设定 id1，以 id1 为准
 - xml1 没有给 xml2 设定id，以 xml2 的根目录 id2 为准
- Merge: 帮助 include 减少视图层级，可以删除多余的层级，优化 UI。    
  include 标签的 parent ViewGroup 与包含的 layout 根容器 ViewGroup 是相同的类型，那么则可以将包含的 layout 根容器 ViewGroup 使用 merge 标签代替，从而减少一层 ViewGroup 的嵌套，从而提升 UI 性能渲染。
- ViewStub: 按需加载，减少内存使用量、加快渲染速度。**不支持 merge 标签**

# invalidate 和 requestLayout 的区别
- 调用 invalidate 只会执行 *onDraw()* 方法
- 调用 requestLayout 会执行 *onMeasure()* 方法和 *onLayout()* 方法，会根据标志位判断是否需要 *onDraw()*。

所以当我们进行 View 更新时，若仅 View 的显示内容发生改变且新显示内容不影响 View 的大小、位置，则只需调用 invalidate 方法。    
若 View 宽高、位置发生改变，需调用 requestLayout 方法。    
内容、大小、位置都变化时，就同时调用 requestLayout 和 invalidate。

# invalidate 和 postInvalidate 的区别
- invalidate 方法是 View 内部的，只能用于UI线程中。
- postInvalidate 会将更新消息发送到 ViewRootImpl 中的  ViewRootHandler 中去更新UI。在非UI线程中，使用 postInvalidate 方法可以省去使用 handler 的烦恼。

# Activity 启动模式
#### standard
新建 Activity 实例。   
 
- **Android 5.0 以前**，新建的 Activity 会放在同一个 task 下 Activity 栈的顶部，哪怕前后两个 Activity 是来自不同的应用。
- **Android 5.0 以后**，如果两个 Activity 来自不同的应用，就会新建一个 task，**这是因为 android 5.0 以后的 task 管理器有修改以更直观有效**。

#### singleTop
- 如果 Activity 不在栈顶，可以无限创建Activity。    
- 如果 Activity 实例已经在栈顶了，那么就不会创建新的 Activity 实例，而是会将 intent 传入已有的 Activity 的实例的 *onNewIntent()* 方法。

#### singeTask
**只允许一个Activity实例**。  

1. 同一个应用：  
  如果已经存在 Activity 实例，那么 task 中的这个实例会被置顶，这个过程中，Activity 栈中在其上的 Activity 都会被销毁，并且 intent 会被传给 *onNewIntent()*。
 
2. 不同的应用：
  - 系统中没有 Activity，创建新 task。
  - 系统中有一个 task 有那个 Activity，切到那个 task，同时销毁上面的 Activity，**如果用户按了返回键，就会回到唤起 Activity 的 task。**

#### singeInstance
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

# HttpURLConnection 和 HttpClient 比较
- 在 Android 2.2 版本之前，HttpClient 拥有较少的 bug；而在Android 2.3版本及以后，HttpURLConnection则是最佳的选择，它 API 简单，体积较小。
-  Apache HttpClient 早就不推荐 httpclient，5.0 之后干脆废弃，6.0 删除了 HttpClient。
-  在 Android 2.2 版本之前，HttpURLConnection 一直存在着一些令人厌烦的 bug。比如说对一个可读的 InputStream 调用 close() 方法时，就有可能会导致连接池失效了。那么我们通常的解决办法就是直接禁用掉连接池的功能。
-  Android4.4 的源码中可以看到 HttpURLConnection 已经替换成 OkHttp 实现。

# OkHttp 的优点
- Android4.4 的源码中可以看到 HttpURLConnection 已经替换成 OkHttp 实现。
- Volley停止了更新，而OkHttp得到了官方的认可，并在不断优化。
- OkHttp 是一个现代，快速，高效的 Http client，支持 HTTP/2、SPDY、连接池、GZIP 和 HTTP 缓存。
- OkHttp 实现的诸多技术如：连接池，gziping，缓存等就知道网络相关的操作是多么复杂了。
- OkHttp 扮演着传输层的角色。
- OkHttp 使用 Okio 来大大简化数据的访问与存储，Okio 是一个增强 java.io 和 java.nio 的库。
- OkHttp 处理了很多网络疑难杂症：会从很多常用的连接问题中自动恢复。如果您的服务器配置了多个 IP 地址，当第一个 IP 连接失败的时候，OkHttp 会自动尝试下一个 IP。
- OkHttp 还处理了代理服务器问题、二次连接和 SSL 握手失败问题。
- OkHttp 是一个 Java 的 HTTP+SPDY 客户端开发包，同时也支持 Android。需要 Android 2.3 以上。
- 目前支持的功能：
 * 一般的get请求
 * 一般的post请求
 * 基于Http的文件上传
 * 文件下载
 * 上传下载的进度回调
 * 加载图片
 * 支持请求回调，直接返回对象、对象集合
 * 支持session的保持
 * 支持自签名网站https的访问，提供方法设置下证书就行
 * 支持取消某个请求

# Android 中软引用与弱引用的应用场景
#### 分类
- 强引用：不回收
- 软引用（SoftReference）：内存空间不足就会去回收
- 弱引用（WeakReference）：一旦发现，不论内存足不足，都会回收
- 虚引用

#### 使用场景
**在处理一些占用内存大而且声明周期较长的对象时候，可以尽量应用软引用和弱引用技术**。    
如果只是想避免 OOM 异常的发生，则可以使用软引用。如果对于应用的性能更在意，想尽快回收一些占用内存比较大的对象，则可以使用弱引用。    
可以根据对象是否经常使用来判断选择软引用还是弱引用。如果该对象可能会经常使用的，就尽量用软引用。如果该对象不被使用的可能性更大些，就可以用弱引用。

# Service 保活
### 在 onStartCommond() 中将返回值设置为 START_STICKY
在运行 onStartCommand 后 service 进程被 kill 后，那将保留在开始状态，但是不保留那些传入的 intent。    
不久后 service 就会再次尝试重新创建，因为保留在开始状态，在创建 service 后将保证调用 onstartCommand。如果没有传递任何开始命令给 service，那将获取到 null 的 intent。    
**【亲测】**：手动返回 START_STICKY，当 service 因内存不足被 kill，当内存又有的时候，service 又被重新创建，比较不错，但是不能保证任何情况下都被重建，比如进程被干掉

### onDestroy 方法里重启 service
直接在 onDestroy（）里 startService 或 service + broadcast 方式，就是当 service 走 onDestory 的时候，发送一个自定义的广播，当收到广播的时候，重新启动 service。    
**【亲测】**：当使用类似 QQ 管家等第三方应用或是在 setting -应用-强制停止时，APP 进程可能就直接被干掉了，onDestroy 方法都进不来，所以还是无法保证

### 白色保活：启动前台 service，提升 service 优先级
调用系统 api 启动一个前台的 Service 进程，这样会在系统的通知栏生成一个 Notification，用来让用户知道有这样一个 app 在运行着，哪怕当前的 app 退到了后台，如QQ音乐。

### 灰色保活：单独起一个 Service 进程，提高该进程优先级
灰色保活，这种保活手段是应用范围最广泛。它是利用系统的漏洞来启动一个前台的Service进程，与普通的启动方式区别在于，它不会在系统通知栏处出现一个Notification，看起来就如同运行着一个后台Service进程一样。这样做带来的好处就是，用户无法察觉到你运行着一个前台进程（因为看不到Notification）,但你的进程优先级又是高于普通后台进程的。

### 黑色保活：不同 app 进程利用广播相互唤起
- 场景1：开机，网络切换、拍照、拍视频时候，利用系统产生的广播唤醒app
- 场景2：接入第三方SDK也会唤醒相应的app进程，如微信sdk会唤醒微信，支付宝sdk会唤醒支付宝。由此发散开去，就会直接触发了下面的 场景3
- 场景3：假如你手机里装了支付宝、淘宝、天猫、UC等阿里系的app，那么你打开任意一个阿里系的app后，有可能就顺便把其他阿里系的app给唤醒了。（只是拿阿里打个比方，其实BAT系都差不多）

# Android 长连接，怎么处理心跳机制
- 长连接：长连接是建立连接之后, 不主动断开. 双方互相发送数据, 发完了也不主动断开连接, 之后有需要发送的数据就继续通过这个连接发送。
- 心跳包：其实主要是为了防止 NAT 超时，客户端隔一段时间就主动发一个数据，探测连接是否断开。
- 服务器处理心跳包：假如客户端心跳间隔是固定的, 那么服务器在连接闲置超过这个时间还没收到心跳时, 可以认为对方掉线, 关闭连接. 如果客户端心跳会动态改变,  应当设置一个最大值, 超过这个最大值才认为对方掉线. 还有一种情况就是服务器通过 TCP 连接主动给客户端发消息出现写超时, 可以直接认为对方掉线。
- [Android微信智能心跳方案](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&amp;mid=207243549&amp;idx=1&amp;sn=4ebe4beb8123f1b5ab58810ac8bc5994&amp;scene=21#wechat_redirect)

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
