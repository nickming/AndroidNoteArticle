# Android源码设计模式笔记——单例模式

今天开始正式阅读这本书，之前一直马马虎虎囫囵吞枣地读，总觉得不好，所以打算开始认真读，并且写下每天的阅读感想与笔记，并且做成一个系列。

## 单例模式

单例模式用得比较多，记录一下优缺点。

优点：

1. 只存在一个实例，减少内存开支，避免频繁创造一个对象。
2. 创建一个复杂对象，例如application这种巨型对象，需要很多资源。
3. 避免对资源的多重占用
4. 设置全局访问，优化和共享资源

缺点：

1. 没有接口，拓展困难
2. 如果持有context，很容易内存泄漏，大部分内存泄漏都是由于单例持有并且没有及时释放

## 创建方式

### 懒汉模式

```java
public class SingletonTest {

    // 定义私有构造方法（防止通过 new SingletonTest()去实例化）
    private SingletonTest() {   
    }   

    // 定义一个SingletonTest类型的变量（不初始化，注意这里没有使用final关键字）
    private static SingletonTest instance;   

    // 定义一个静态的方法（调用时再初始化SingletonTest，使用synchronized 避免多线程访问时，可能造成重的复初始化问题）
    public static synchronized  SingletonTest getInstance() {   
        if (instance == null)   
            instance = new SingletonTest();   
        return instance;   
    }   
} 
```

这个方法synchronized可能造成每次获取都需要进行同步，造成不必要的开销。

### DCL模式

```java
public class SingletonTest { 

    // 定义一个私有构造方法
    private SingletonTest() { 
     
    }   
    //定义一个静态私有变量(不初始化，不使用final关键字，使用volatile保证了多线程访问时instance变量的可见性，避免了instance初始化时其他变量属性还没赋值完时，被另外线程调用)
    private static volatile SingletonTest instance;  

    //定义一个共有的静态方法，返回该类型实例
    public static SingletonTest getIstance() { 
        // 对象实例化时与否判断（不使用同步代码块，instance不等于null时，直接返回对象，提高运行效率）
        if (instance == null) {
            //同步代码块（对象未初始化时，使用同步代码块，保证多线程访问时对象在第一次创建后，不再重复被创建）
            synchronized (SingletonTest.class) {
                //未初始化，则初始instance变量
                if (instance == null) {
                    instance = new SingletonTest();   
                }   
            }   
        }   
        return instance;   
    }   
}
```

如果不加volatile字段，也是线程不安全的，由于instance = new SingletonTest()不是原子操作，但是new SingletonTest()也分为三步：

1. 给SingletonTest分配内存
2. 调用SingletonTest（）的构造函数，初始化成员字段
3. 让instance指向分配的内存空间，此时instance不是null

正常书序123，但是jvm无法保证顺序，则可能是132，如果是132，则切换线程之后有可能在判断if (instance == null)的时候读取，并且报错。

### 静态内部类

```java
public class Instance {

    /**
     * 构造方法私有化
     */
    private Instance(){
    }

    private static class SingleHolder{
        private static final Instance ins = new Instance();
    }

    /**
     * 内部类方式获取单例
     * @return
     */
    public static Instance getInstance(){
        return SingleHolder.ins;
    }

}
```

推荐使用，getInstance只有在第一次调用的时候才会导致虚拟机去加载SingleHolder类，不仅保证线程安全，能够保证单例对象的唯一性。

## 源码单例模式

### LayoutInflater

1. LayoutInflater就是通过系统的context.getSystemService()，明显这个getSystemService应该是一个容器单例。

2. 总的context数量为activity+service+application

3. context创建，源码可参考ActivityThread类，

   1. 在ActivityThread的main中创建ActivityThread的实例，并且attach
   2. 在attach中会调用ActivityManagerNative，再调用ActivityManagerNative的gDefault通过binder与AMS通信，并且最终调用handleLaunchActivity。
   3. 在handleLaunchActivity中会调用performLaunchActivity，接下来就会创建application和baseContext
   4. context的创建会在createBaseContextForActivity中创建，依赖于ContextImpl.createActivityContext来创建，而ContextImpl则是Context的实现类，相关系统资源，例如sp、db、file等初始资源都是在这里初始化并且提供获取接口，然后各种服务类,在其中SystemServiceRegistry就是存储service的管理注册类。

   ```java
   private static final HashMap<Class<?>, String> SYSTEM_SERVICE_NAMES =
           new HashMap<Class<?>, String>();
   private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
           new HashMap<String, ServiceFetcher<?>>();
           
   registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                   new CachedServiceFetcher<LayoutInflater>() {
               @Override
               public LayoutInflater createService(ContextImpl ctx) {
                   return new PhoneLayoutInflater(ctx.getOuterContext());
               }});
   ```

   接着就会创建各个service并且缓存到map当中，下次直接从缓存中获取，形成容器单例。


### 深入理解LayoutInflater的原理

1. 旧版的代码是通过PolicyManager来创建的，新版代码是通过PhoneLayoutInflater来创建，不过其原理差不多。只不过新版代码进行了解耦，不再利用PolicyManager.makeNewLayoutInflater创建，直接new一个PhoneLayoutInflater对象。

2. PhoneLayoutInflater则是继承LayoutInflater，会根据控件开头前缀来创建新的view并返回。

   1. ```java
      public class PhoneLayoutInflater extends LayoutInflater {
          private static final String[] sClassPrefixList = {
              "android.widget.",
              "android.webkit.",
              "android.app."
          };

          /**
           * Instead of instantiating directly, you should retrieve an instance
           * through {@link Context#getSystemService}
           *
           * @param context The Context in which in which to find resources and other
           *                application-specific things.
           *
           * @see Context#getSystemService
           */
          public PhoneLayoutInflater(Context context) {
              super(context);
          }

          protected PhoneLayoutInflater(LayoutInflater original, Context newContext) {
              super(original, newContext);
          }

          /** Override onCreateView to instantiate names that correspond to the
              widgets known to the Widget factory. If we don't find a match,
              call through to our super class.
          */
          @Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
              for (String prefix : sClassPrefixList) {
                  try {
                      View view = createView(name, prefix, attrs);
                      if (view != null) {
                          return view;
                      }
                  } catch (ClassNotFoundException e) {
                      // In this case we want to let the base class take a crack
                      // at it.
                  }
              }

              return super.onCreateView(name, attrs);
          }

          public LayoutInflater cloneInContext(Context newContext) {
              return new PhoneLayoutInflater(this, newContext);
          }
      }
      ```

3. 具体流程如下，以setContentView为例，view层级是activity—>PhoneWindow—>DecorView—>DefaultLayout—>mContentParent—>xml，实际上setContentView调用的是window的setContentView方法，而window的实现类则是PhoneWindow

   1. ```java
      public void setContentView(int layoutResID) {
          // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
          // decor, when theme attributes and the like are crystalized. Do not check the feature
          // before this happens.
          if (mContentParent == null) {
              installDecor();
          } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
              mContentParent.removeAllViews();
          }

          if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
              final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                      getContext());
              transitionTo(newScene);
          } else {
              mLayoutInflater.inflate(layoutResID, mContentParent);
          }
          mContentParent.requestApplyInsets();
          final Callback cb = getCallback();
          if (cb != null && !isDestroyed()) {
              cb.onContentChanged();
          }
          mContentParentExplicitlySet = true;
      }
      ```

      可以看到window会先初始化一个decor，再将decor中会加载一个系统定义的布局，再将这个布局包裹mContentParent，mContentParent就是我们xml引用的布局，decor就是一个系统内置的一个framelayout布局。

   2. 接着就会调用inflate方法来引入内容，首先就用利用XmlResourceParser资源解析器来解析xml布局，然后根据相应的规则，进行解析：

      1. 解析xml根标签
      2. 如果是merge，则会将merge下的所有子view添加到根目录
      3. 如果是普通元素，则会调用createViewFromTag进行解析
      4. 调用rInflate方法遍历调用viewgroup下的子view，并且将这些子view添加到viewgroup下
      5. 返回解析的根视图

      注意rInflate方法是通过深度优先遍历来构造视图树，每次解析到一个view就会递归调用rInflate，知道构造完成。并且createViewFromTag中会调用createView方法来根据完整的路径的类名通过反射机制构造view对象。

      最后，当activity的onResume方法之后，我们的setContentView设置的内容就会展示出来。

   ## 小结

   巩固了单例模式的记忆，并且通过阅读android源码，了解了activity的启动、context的构建、window的作用、视图的绘制等原理知识，还是收获颇丰的。不过要注意的是，Android源码在不同版本是有优化改动的，所以书本上的源码还是与Android studio所看到的有所出入，可以对照着studio的源码来进行阅读。
