# Android源码设计模式笔记——建造者模式

## 建造者模式

builder模式是一步一步创建复杂对象的的创造型模式，可以将一个复杂的对象的构建和它的表示分离，使得同样的构建过程可以创建不同的表示。

具体的表现形式在都是用于构造复杂的并且可定制化的对象，例如一些框架的config对象，alerdialog的构建都是运用了builder模式，通常会省略掉director对象。

优点：

1. 良好的封装性，使用buidler模式可以使客户端不必知道产品内部的组成细节。
2. 建造者独立，容易拓展

缺点：

1. 容易产生冗余代码

## AlertDialog源码分析

AlertDialog就是典型的建造者模式，由一个内部Builder类来构建，通过内部的AlertController来控制内容的构建，包括title、message、button等。

而dialog的show方法主要就是:

1. 通过dispatchOnCreate来调用dialog的onCreate方法
2. 然后调用onStart方法
3. 最后将dialog的decorView添加到windowManager（wm）中

所以，我们可以这么认为，dialog的内容视图是最终通过wm显示到手机上的。

## 深入分析WindowManager

不止dialog，activity、toast都是通过wm来操作的，因为之前的文章也分析过，setContentView方法就是通过window的setContentView方法来调用，最终还是会与wm相关联。

来分析下wm的流程：

1. 服务的初始化

   ```java
   registerService(Context.WINDOW_SERVICE, WindowManager.class,
           new CachedServiceFetcher<WindowManager>() {
       @Override
       public WindowManager createService(ContextImpl ctx) {
           return new WindowManagerImpl(ctx);
       }});
   ```

   可以看出主要是WindowManagerImpl来实现，而WindowManagerImpl也只是一个代理类，其主要是调用了WindowManagerGlobal.getInstance()这个类的方法。

2. window的创建主要是通过PolicyManager.makeNewWindow()方法来创建，然后通过window.setWindowManager方法与wm关联。

3. 分析addView方法，可以看出主要是通过ViewRootImpl来构建视图

   1. 构建ViewRootImpl
   2. 将布局参数设置给View
   3. 存储这些到ViewRootImpl、View、lp到列表中
   4. 通过ViewRootImpl的setView方法将View显示到窗口中

   其实ViewRootImpl继承自Handler，是一个作为native层的wms和Java层View系统通信的桥梁，当然这个通信机制肯定也似binder。

4. 在ViewRootImpl构造函数中，通过mWindowSession = WindowManagerGlobal.getWindowSession();这个方法以及WindowManagerGlobal的作用来看，这就是通过mWindowSession来与native层建立通信。在getWindowSession方法中，通过IWindowManager windowManager = getWindowManagerService();这一句与wms进行联系，一看就知道binder机制，然后通过这个Session来通信。

5. 需要注意的是WMS只负责管理手机屏幕View的z-order，也就是WMS管理当前状态下哪个View在屏幕最上层。当WMS建立Session后就调用ViewRootImpl的setView方法:

   1. reqeustLayout
   2. 想WMS发起显示当前window的请求

   reqeustLayout会调用 scheduleTraversals，继而调用mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();这个方法发送消息给handler，最终执行performTraversals方法。而performTraversals方法又会执行以下几步：

   	1. 获取Surface对象
   	2. 得到整个视图树的各个View的大小，调用performMeasure方法
   	3. 布局整个视图树，调用performLayout
   	4. 绘制，调用performDraw

6. 我们可以分析下performDraw这个方法：

   1. 判断使用CPU还是GPU绘制
   2. 获取绘制表面Surface对象
   3. 通过Surface对象获取并且锁住Canvas对象
   4. 从DecorView开始发起整棵树的绘制
   5. Surface对象解锁Canvas，并且通知SurfaceFlinger更新视图

   内容绘制完成后就会请求WMS显示该窗口的内容，至此，activity、dialog的内容就会显示到屏幕上。

## 小结

通过这篇对WM的分析，更加深入理解了Android的绘制原理，对视图的显示、绘制有了更清晰的认识。