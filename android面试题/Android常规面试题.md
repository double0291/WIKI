Android 常规面试题
================
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