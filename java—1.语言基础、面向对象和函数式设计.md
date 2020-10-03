## 1 Java语言基础

**Java面向过程编程**：

```java
核心要素：
①数据结构：原生类型、对象类型、数组类型、集合类型。
②方法调用：访问性、返回类型、方法参数、异常等。
③赋值、逻辑、迭代（循环）、递归等。
```

访问性（引用举例）：Reference是java的引用，java有四种引用：强引用，弱引用，软引用，幻象引用。其实还有第五种：FinalReference（对象被回收）， 和Object对象有关:

```java
@Deprecated(since="9")//从java9开始就不应该调了
protected void finalize() throws Throwable{}
```

**面向对象设计模式**：

```java
GoF：构建、结构、行为
方法设计：名称、放问性、参数、返回类型、异常
泛型设计：类级别、方法级别
异常设计：层次性、传播性
```

java泛型属于编译时处理，运行时擦写。

### 2 Java 函数式基础

**基本特性**：

①所有的函数式接口都引用一段执行代码

②函数式接口没有固定类型，固定模式（SCFP）+Action

③固定模式来通过引用方法进行实现



**面向函数编程**：①Lambda表达式、②默认方法、③方法引用



#### **2.1 匿名内置类**

Java作为一门门面向对象的静态语言，其封装性能够屏蔽数据结构的细节，从而更加关注模块的功能性。其静态性也确保了Java 强类型的特性。随着模块功能的提升，伴随而来的是复杂度的增加，代码的语义清晰依赖于开发人员抽象和命名类或方法的能力。尽管编程思想和设计模式能够促使编程风格趋于统一，然而大多数业务系统属于面向过程的方式，这与面向对象编程在一定程度上存在一些冲突。Java 编程语言为了解决这个问题，引入了匿名内置类的方案。

Java并非完全面向对象，因为有原生类型。

**典型场景**：

①Java Event/Listener②Java Concurrent③Spring Template

**基本特性**：

​	①无名称类

​	②声明位置（执行模块）：static block/实例block/方法/构造器

​	③并非特殊的类的结构：类全名称：$(package).$(declared_class).${num}

```java
public class InnerClassDemo{
  static {//static块
    new Runnable(){
      @Override
			public woid run(){}
    }
  };
  
  {//实例块
   	new callable(){
      @Override
      public Object call() throws Exception {
        return null;
      }
    };
  }
  
  public InnerClassDemo(){//构造器
    new Comparable(){
      @Override
      public int compareTo(Object o){
        return 0;
      }
    };
  }
  
  public static void main(String[] args){
    PropertyChangeListener listener = new PropertyChangeListener() {
			@Override
      public void propertyChange(PropertyChangeEvent evt){}
  };
}
  static calss PropertyChangeListenerImpl impliments PropertyChangeListerner{
    @Override
    public void propertyChanger(PropertyChangeEvent evt){}
  }
}
```

lamba语法是根据很多java特性总结出来的，为什么只有一个或两个参数，并不是说一定实现不了，只是没必要（成本较大）。

**基本特点**：①基于多态（多数基于接口编程）②实现类无需名称③允许多个抽象方法

**编程局限**：①代码臃肿②强类型约束③接口方法升级

#### 2.2 Lambda表达式：

**基本特点**：①流程编排清晰、②函数类型编程、③改善代码臃肿、④兼容接口升级。

**实现手段**：①FunctionalInterface接口、②Lambda语法、③方法引用、④接口default方法实现。

**编程局限**：①单一抽象方法、②Lambda调试困难、③Stream API操作能力有限。

defult方法和Lambda有什么必然联系：特意去改造一些方法的时候，就需要把以前的方法defult化。

通常很少关注哪个方法是函数方法，指明接口是函数接口就行了。是函数接口必然有一个未实现抽象类。

方法引用：是为了解决通用的问题，为什么通过这个手段呢，如果不用，用匿名内置类的方式，比较臃肿。

```java
public class LambdaDemo{
  public static void main(String[] args){
    //匿名类 传统写法
    PropertyChangeListener listener = new PropertyChangeListener(){
      @Override
      public void propertyChange(PropertyChangeEvent evt){
        printLn(evt);
      }
    };

    //Lambda基本写法
    PropertyChangeListener listener2 = evt ->{
      println(evt);
    };

    //Lambda简略写法
    //PropertyChangeListener#propertyChange(PropertyChangeEvent)
    //属于有入参，没有返回，语printlin(Object)一样
    PropertychangeListener listener3 = LambdaDemo::Println;
  }
}  
//SCFP = Supplier + Consumer + Function + Predicate
//四种模式（缺少Action模式）
```

```java
//Supplier 模式 输出
private static void showSupplier(){
  String string = "Hello,World";
  
  Supplier<String> string2 = () -> "Hello,World";
  
  Supplier<String> string3 = () -> new Integer().toString();
}//所有的函数式接口都是一段执行代码

//上面三种代码写法，是Consumer模式 输入

//Function模式 输入输出
private static void showFunction(){
  //强类型约束，泛型的时候不能把包装类型和原生类型混搭
  Function<Object,Integer> f = LambdaDemo::compareTo;
}

private static Integer compareTo(String value){
  return value.compareTo("Hello,World");
}

//Action模式（自己总结的不一定对）
private static void showAction(){
  Runnable runnable = new Runnable(){
    @Override
    public void run(){}
  };
  
  Runnable runnable2 = () ->{};
  
  Runnable runnable3 = LambdaDemo::showConsumer;//方法引用
}

```

**默认方法**（使用场景）：当接口升级时，添加了新的抽象方法，此时基于老接口的实现类必然会遇到编译问题。默认方法的出现能够解决以上问题，同时也能为实现类提供默认或样板实现，减少实现类的负担，如无需再使用Adapter实现。**提示**：默认方法不列入@FunctionInterface方法计算。

### 2  Java模块化基础（java9新特性）

## 3 Java面向对象设计（上）

### 3.1 Java接口设计

**具体类设计**（常见场景）：①功能组件、②接口/抽象类实现、③数据对象、④工具辅助。

抽象类程度介于类和接口之间（Java8+通常可完全由接口替代）

String类虽然被final修饰也是可以改变的

```java
public class StringDemo(){
  public static void main(String[] args){
    String value = "Hello";//常量（语法特性）=对象类型常量化
    String value2 = new String(original;"Hello");
    
    System.out.println(value2);
    
    //从Java1.5开始对象属性可以通过反射修改
    char[] chars = "World".toCharArray();
    Field valueFiled = String.class.getDeclaredField(name:"value");//获取String类中的value字段
    valueFiled.setAccessible(true);//设置private字段可以被修改
    valueFiled.set(value2,chars);
    System.out.println(value2);
  }  
}
```



