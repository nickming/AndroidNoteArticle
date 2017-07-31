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

