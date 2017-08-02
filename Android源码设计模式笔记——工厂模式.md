# Android源码设计模式笔记——工厂模式

## 定义

工厂模式定义了一个用于创建对象的接口，让子类决定实例化哪个类。

## 场景

在任何需要生成复杂对象的地方，都可以使用工厂模式。复杂对象适合使用工厂模式，用new就可以完成创建的对象无需使用工厂模式。

## 方式

抽象工厂模式

```java
public interface Shape {
   void draw();
}
public class Rectangle implements Shape {

   @Override
   public void draw() {
      System.out.println("Inside Rectangle::draw() method.");
   }
}
public class Square implements Shape {

   @Override
   public void draw() {
      System.out.println("Inside Square::draw() method.");
   }
}
public class ShapeFactory {
	
   //使用 getShape 方法获取形状类型的对象
   public Shape getShape(String shapeType){
      if(shapeType == null){
         return null;
      }		
      if(shapeType.equalsIgnoreCase("CIRCLE")){
         return new Circle();
      } else if(shapeType.equalsIgnoreCase("RECTANGLE")){
         return new Rectangle();
      } else if(shapeType.equalsIgnoreCase("SQUARE")){
         return new Square();
      }
      return null;
   }
}

public class FactoryPatternDemo {

   public static void main(String[] args) {
      ShapeFactory shapeFactory = new ShapeFactory();

      //获取 Circle 的对象，并调用它的 draw 方法
      Shape shape1 = shapeFactory.getShape("CIRCLE");

      //调用 Circle 的 draw 方法
      shape1.draw();

      //获取 Rectangle 的对象，并调用它的 draw 方法
      Shape shape2 = shapeFactory.getShape("RECTANGLE");

      //调用 Rectangle 的 draw 方法
      shape2.draw();

      //获取 Square 的对象，并调用它的 draw 方法
      Shape shape3 = shapeFactory.getShape("SQUARE");

      //调用 Square 的 draw 方法
      shape3.draw();
   }
}
```

也可以使用一个反射的方式来使用。

## Android源码中的工厂模式

### Collection接口

List和Set都继承了Collection接口，而Collection接口继承于Iterable接口，Iterable需要实现iterator方法，意味着List和Set接口也会实现该方法，ArrayList和HashSet中的iterator方法的时下就是返回一个迭代器对象。

### onCreate方法

不同的Activity都需要实现onCreate方法，并且通过setContentView方法来设置布局返回给framework处理，则就相当于工厂模式。

探究onCreate方法的是实现：
1. 关于onCreate有两个方法，只有一个参数的方法是在activity重新构建的时候，一般情况是当屏幕进行旋转的时候，会重新调用protected void onSaveInstanceState(Bundle outState)这个方法，进行数据的存储，做到不影响用户界面。onCreate方法以及足够强大,但是他能否更加强大？有没有这样一种情况，手机由于过热，没电或者第三方定制Rom由于卡顿而异常关机的情况？当用户在操作前台数据的时候手机突然关机了，这个时候就会调用public void onCreate(@Nullable Bundle savedInstanceState, @Nullable PersistableBundle persistentState)这方方法。
  首先，我们需要在Android 的清单文件的Activity中指定如下属性：

```java
android:persistableMode="persistAcrossReboots"
```
接着重载onSaveInstanceState或者onRestoreInstance：

```java
@Override
public void onSaveInstanceState(Bundle outState, PersistableBundle outPersistentState) {
        super.onSaveInstanceState(outState, outPersistentState);
}

@Override
public void onRestoreInstanceState(Bundle savedInstanceState, PersistableBundle persistentState) {
    super.onRestoreInstanceState(savedInstanceState, persistentState);
}
 
```
他们对应着一个PersistableBundle类型的persistentState。对齐进行操作就OK了。

补充：上面说到重载onSaveInstanceState或者onRestoreInstance。这里解释一下这两个方法onSaveInstanceState调用时机是当前Activity即将被销毁而还未被销毁的时候。而当系统调用了onRestoreInstance就表示这个Activity已经被销毁了。

2. 关于onCreate的调用流程

  1. ActivityThread–>main 
    ActivityThread类中main方法就是一个应用程序的主入口，当启动一个应用，会先由Zygote进程孵化出新的进程后，会执行AcivityThread的main方法。在main方法中的关键是ActivityThread的attach方法将其绑定到ActivityManagerService中。

  2. ActivityThread–>attach 
    在attach方法中只关注非system的部分，获取ActivityManagerService调用其attachApplication方法将ActivityThread添加到AMS中。

  3. ActivityManagerService–>attachApplication 
    在attachApplication方法中主要调用AMS的attachApplicationLocked方法。

  4. ActivityManagerService–>attachApplicationLocked 
    在attachApplicationLocked方法中我们只关心其中两个重要的方法bindApplication方法和attachApplicationLocked(ProcessRecord)。其中bindApplication方法参数非常多，主要作用就是讲ApplicationThread绑定到AMS，我们不做关注。主要看mStackSupervisor.attachApplicationLocked(ProcessRecord)。

  5. ActivityStackSupervisor–>attachApplicationLocked 
    ActivityStackSupervisor类中的attachApplicationLocked方法中真正关心的是其realStartActivityLocked方法，进入启动Activity的逻辑中。

  6. ActivityStackSupervisor–>realStartActivityLocked 
    realStartActivityLocked方法很长，主要都是做启动Activity的准备，当准备完成后就会调用ApplicationThread的scheduleLaunchActivity方法准备去启动Activity。

  7. ApplicaionThread–>schedulLaunchActivity 
    schedulLaunchActivity方法很简单对启动信息进行准备然后构造ActivityClientRecord对象，最后通过sendMessage方法发送一个LAUNCH_ACTIVITY的消息。在ActivityThread的内部有一个继承Handler的内部类H，消息将由内部类H处理。

  8. H–>LAUNCH_ACTIVITY 
    内部类H处理消息调用handleLaunchActivity方法

  9. ActivityThread–>handleLaunchActivity 
    handleLaunchActivity方法的逻辑比较简单，重点是调用了performLaunchActivity方法。

  10. ActivityThread–>performLaunchActivity 
    performLaunchActivity方法开始了真正处理具体的Activity启动。其中Instrumentationcal的callActivityOnCreate方法就会执行对Activity onCreate的调用。

  11. Instrumentation–>callActivityOnCreate 
     在callActivityOnCreate方法中调用Activity的performCreate方法

  12. Activity–>performCreate

     ```java
     final void performCreate(Bundle icicle){
         onCreate();
         mActivityTransitionState.readState(icicle);
         performCreateCommon();
     }
     ```

     至此所有的调用流程都已经完成
     ![activity启动流程](https://github.com/nickming/AndroidNoteArticle/blob/master/pic/acitivity.jpg)
     
  ## 小结
  总体来说，onCreate就是activity的启动过程，了解activity的启动过程，更有利于了解Android系统的原理，写出更好的程序。





















