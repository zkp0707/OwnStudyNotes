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