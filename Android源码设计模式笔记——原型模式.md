# Android源码设计模式笔记——原型模式

## 原型模式

原型模式其实很简单，就是用原型实例指定创建对象的种类，并通过拷贝这些原型对象创建新的对象

## 理解

在以编程中，在Java中就是实现cloneable接口，在C中就是拷贝函数。但是要注意的就是拷贝也分为深拷贝和浅拷贝。

原型模式是一个非常简单的设计模式，它的核心是对原型对象的进行拷贝，这个模式最重要的就是深浅拷贝。

深拷贝：在拷贝对象的时候，对于引用的原型的字段也需要采用拷贝的形式，而不是单纯的引用，避免操作副本时影响原始对象的问题。

浅拷贝：称为影子拷贝，并不是对原始对象的所有字段重新构造一份，而是副本文档的字段引用原始文档的字段。



## Android中的原型模式

Intent就实现了cloneable接口，可以看到其实现方法只是是下了一个拷贝构造函数，与C++类似，所以我们使用clone或者new需要根据构造对象的成本来决定，对象的构造成本比较大的时候使用clone，否则使用new。

## Intent的查找与匹配

Intent是一个重要的类，它是各个组件，进程之间的通信纽带。其主要是通过PacketManagerService（简称PMS）进行管理。

当PMS启动后，会扫描系统中已安装的apk目录，系统APP目录为/system/app，第三方为/data/app，PMS会解析apk包下的AndroidManifest.xml文件得到App的相关信息，而每个AndroidManifest.xml又包含了activity、service等组件信息当PMS扫描并解析完成这些信息就构建好了这个那个apk信息树。

1. PMS构造函数会加载已安装的apk，同时加载各种framework资源和核心库，并且对指定目录下的apk文件进行解析，其函数为scanDirLI.
2. scanDirLI会调用sacnPackageLI函数机型解析apk文件，通过创建PackagePareser进行apk解析，其方法为paresePackage。
3. paresePackage会根据packageFile文件类型来选择不同的解析方式，并且会在后续解析AndroidManifest.xml文件，解析这个文件就是构建与Intent有关的信息表。
4. 继而会调用pareseApplication，解析Application下的activity、service等标签，最后回到scanPacketLI最后一步，调用scanPackageLI函数，这函数会调用sacnPackageDirtyLI，这个函数会将解析到的activity、service等添加到mActivities、mServices等列表中。
5. 最后，真个apk信息树已经建立好了，当用户使用Intent进行各种跳转会在这个信息表中查询。

### 精确匹配

1. startActivity会调用startActivityForResult，继而调用Instrumentation中的execStartActivity函数，然后会调用AMS中的startActivity方法，实际上调用了ActivityStackSupervisor中的startActivityMayWait,最后代用PMS中的resolveIntent方法。
2. resolveIntent会调用queryIntentActivities，继而返回一个ActivityInfo列表，符合Intent的ActivityInfo的列表。
3. 如果目录存在包名，则会通过包名获取对应的ActivityInfo,否则需要通过ActivityIntentResolver进行模糊查找，例如Action、Category。

## 小结

了解了intent的匹配机制以及PMS原理与作用。