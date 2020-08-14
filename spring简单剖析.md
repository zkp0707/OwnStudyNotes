## 1 spring的IoC 和 AOP：

**spring架构图**：

```
Web：Ⅰ、WebⅡ、ServletⅢ、WebSocketⅣ、Portlet
Data Access:Ⅰ、JDBC Ⅱ、JMS Ⅲ、ORM Ⅳ、OXM
	|→Ⅰ、AOPⅡ、AspectsⅢ、MessagingⅣ、Instrumentation
		|Core Container→Ⅰ、CoreⅡ、ContextⅢ、BeansⅣ、SpEL
```

**IoC**：Inversion of Control（控制反转）。**控制**：对象创建（实例化、管理）的权利。**反转**：控制权交给了外部环境（spring框架、IoC容器）。

**传统开发方式**：若类A依赖于类B，往往会在类A中new一个类B对象。

**IoC思想下的开发方式**：不用去自己new对象，而是由IOC容器去自动实例化对象并且管理它，需要什么对象，去问IoC容器要即可。**我们丧失了创建、管理对象的权利，得到了不用考虑对象创建、管理等一系列事情的福利**。

**IoC和DI(Dependancy Injection)的区别**：IoC和DI描述的是同一件事情，只不过是角度不同。

​	**IoC**是站在对象的角度，对象实例化及其管理权利交给了（反转）容器。

​	**DI**是站在容器的角度，容器会把对象依赖的其他对象注入。

**AOP**: Aspect Oriented Programming（面向切面编程）。AOP是OOP的延续。

```java
public class Animal ｛
	public void eat(){
	//性能监控代码
	long start = system.currentTimeMillis();
	//业务逻辑代码
	system.out.println("I can eat..." );
	//性能监控代码
	long end = system.currentTimeMillis( );
	system.out.println("执行时长∶" +(end-start)/1000f + "s");
	}
	public void run(){
	//性能监控代码
	long start = System.currentTimeMillis( );
	//业务逻辑代码
	system.out.println("I can run. .."" );
	//性能监控代码
	long end = System.currentTimeMillis();
	System.out.println("执行时长:" + (end-start)/1000f + "s");
	｝
}

```

OOP只能解决对业务逻辑代码的抽取，AOP是对性能监控代码的抽取，即在多个纵向（顺序）流程汇总出现的相同子流程代码，为我们称之为横切逻辑代码。横切逻辑代码的使用场景很有限，一般是：事物控制、权限校验、日志。AOP在不改变原有业务逻辑情况下，增强横切逻辑代码，根本上姐耦合，避免横切代码重复。

## 2 手写实现IoC和AOP：

### 2.1 简单实现一个银行转账案例（略，A->B），及问题分析和改造。

**调用关系**(数据库：姓名、卡号、存款)：

```java
转账页面->TransferServlet->TransferService->AccouuntDao->JdbcAccountDaoImpl/MybatisAccountDaoImpl
Ⅰ、点击“转出”，页面发起ajax请求到TransferServlet。
Ⅱ、TransferServlet中实例化service层对象并调用service层对象的方法：TransferService transferService = new TransferServiceImol();
Ⅲ、TransferService对象中实例化dao层对象并调用dao层对象方法：AccountDao accountDao = new JdbcAccountDaoImpl();
Ⅳ、Dao为原生JDBC，或mybatis均可。
```

**问题分析**：

**问题一**：new关键字将service层的实现类TransferServicelmpl和Dao层的具体实现类JdbcAccountDaolmpl耦合在了一起，当需要切换Dao层实现类的时候必须得				修改service代码，不符合面向接口开发的最优原则。

​	**思考**：①：使用**反射**的方式代替**new**来实例化对象。Class.forName("全限定类名")，可以把全限定类名配置在xml中。

​			　②：使用工厂模式来通过反射技术生产对象：①读取解析xml，然后通过反射技术实例化对象。②给外部提供获取对象的接口方法。

**问题二**: service层没有添加事务控制，出现异常可能导致数据错乱，问题很严重。

### 2.2 手写IoC和AOP之new关键字耦合问题代码改造。

```xml
<!--beans.xml-->
<!--跟标签beans，里面配置一个又一个的bean子标签，每一个bean子标签都代表一个类的配置-->
<beans>
    <!--id标识对象，class是类的全限定类名-->
    <bean id="accountDao" class="com.own.dao.impl.JdbcTemplateDaoImpl">
        <property name="ConnectionUtils" ref="connectionUtils"/>
    </bean>
    <bean id="transferService" class="com.own.service.impl.TransferServiceImpl">
    	<!--set+ name 之后锁定到传值的set方法了，通过反射技术可以调用该方法传入对应的值-->
        <property name="AccountDao" ref="accountDao"></property>
    </bean>... ...
```

```java
//BeanFactory.java
public class BeanFactory {
     // 任务一：读取解析xml，通过反射技术实例化对象并且存储待用（map集合）
     // 任务二：对外提供获取实例对象的接口（根据id获取）
    private static Map<String,Object> map = new HashMap<>();  // 存储对象
    static {
        // 任务一：读取解析xml，通过反射技术实例化对象并且存储待用（map集合）
        // 加载xml
        InputStream resourceAsStream = BeanFactory.class.getClassLoader().getResourceAsStream("beans.xml");
        // 解析xml
        SAXReader saxReader = new SAXReader();
        try {
            Document document = saxReader.read(resourceAsStream);
            Element rootElement = document.getRootElement();
            List<Element> beanList = rootElement.selectNodes(xpathExpression:"//bean");
            for (int i = 0; i < beanList.size(); i++) {
                Element element =  beanList.get(i);
                // 处理每个bean元素，获取到该元素的id 和 class 属性
                String id = element.attributeValue("id");        // accountDao
                String clazz = element.attributeValue("class");  // com.own.dao.impl.JdbcAccountDaoImpl
                // 通过反射技术实例化对象
                Class<?> aClass = Class.forName(clazz);
                Object o = aClass.newInstance();  // 实例化之后的对象
                // 存储到map中待用
                map.put(id,o);
            }
            // 实例化完成之后维护对象的依赖关系，检查哪些对象需要传值进入，根据它的配置，我们传入相应的值
            // 有property子元素的bean就有传值需求
            List<Element> propertyList = rootElement.selectNodes("//property");
            // 解析property，获取父元素
            for (int i = 0; i < propertyList.size(); i++) {
                Element element =  propertyList.get(i);   //<property name="AccountDao" ref="accountDao"></property>
                String name = element.attributeValue("name");
                String ref = element.attributeValue("ref");

                // 找到当前需要被处理依赖关系的bean
                Element parent = element.getParent();

                // 调用父元素对象的反射功能
                String parentId = parent.attributeValue("id");
                Object parentObject = map.get(parentId);
                // 遍历父对象中的所有方法，找到"set" + name
                Method[] methods = parentObject.getClass().getMethods();
                for (int j = 0; j < methods.length; j++) {
                    Method method = methods[j];
                    if(method.getName().equalsIgnoreCase("set" + name)) {  // 该方法就是 setAccountDao(AccountDao accountDao)
                        method.invoke(parentObject,map.get(ref));
                    }
                }
                // 把处理之后的parentObject重新放到map中
                map.put(parentId,parentObject);
            }
        }... ...    
    }
    // 任务二：对外提供获取实例对象的接口（根据id获取）
    public static  Object getBean(String id) {
        return map.get(id);
    }
}
```

```java
//TransferServiceImpl.java
public class TransferServiceImpl implements TransferService {
	//private AccountDao accountDao = new JdbcAccountDaoImpl();
    //private AccountDao accountDao = (AccountDao) BeanFactory.getBean(id:"accountDao");
    // 最佳状态
    private AccountDao accountDao;//只声明，需赋值。
    // 构造函数传值/set方法传值
    public void setAccountDao(AccountDao accountDao) {
        this.accountDao = accountDao;
    }
    
    @Override
    public void transfer(String fromCardNo, String toCardNo, int money) throws Exception {
        Account from = accountDao.queryAccountByCardNo(fromCardNo);
        Account to = accountDao.queryAccountByCardNo(toCardNo);
        from.setMoney(from.getMoney()-money);
        to.setMoney(to.getMoney()+money);		
        accountDao.updateAccountByCardNo(from);
        accountDao.updateAccountByCardNo(to);   
    }
}
```

```java
//TransferServlet.java
@WebServlet(name="transferServlet",urlPatterns = "/transferServlet")
public class TransferServlet extends HttpServlet {
    // 1. 实例化service层对象
    //private TransferService transferService = new TransferServiceImpl();
    private TransferService transferService = (TransferService) BeanFactory.getBean("transferService");
    
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doPost(req,resp);
    }
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // 设置请求体的字符编码
        req.setCharacterEncoding("UTF-8");
        String fromCardNo = req.getParameter("fromCardNo");
        String toCardNo = req.getParameter("toCardNo");
        String moneyStr = req.getParameter("money");
        int money = Integer.parseInt(moneyStr);
        Result result = new Result();
        try {
            // 2. 调用service层方法
            transferService.transfer(fromCardNo,toCardNo,money);
            result.setStatus("200");
        } ... ...
        // 响应
        resp.setContentType("application/json;charset=utf-8");
        resp.getWriter().print(JsonUtils.object2Json(result));
    }
}
```

## 3 spingIoC高级应用和源码解析：

**beans.xml**:定义需要实例化对象的类的全限定类名以及类之间依赖关系描述。

**BeanFactory**(IoC容器)：通过反射来实例化对象并维护对象之间的依赖关系。

**spring框架的IoC实现**：①纯注解、②xml+注解、③纯注解

```java
ⅠⅡ：JavaSE应用：ApplicationContext applicationContext = new classPathXmlApplicationContext("beans.xml");
			或者：new FileSystemXmlApplicationContext("c:/beans.xml");
     JavaWeb应用：ContextLoaderListener（监听器去加载xml）
                
Ⅲ：JavaSE应用：ApplicationContext applicationContext = new AnnotationConfigApplicationContext(SpringConfig.calss);
   JavaWeb应用：ContextLoaderListener(监听器去加载注解配置类)
```

**学习技巧：找xml中标签（属性）和注解的一一对应关系即可**。

```java
(I)BeanFactory - org.springframework.beans.factory → spring容器的顶层接口。
    (I)ApplicationContext - org.springframework.context → 常用的子接口。
    	(C)ClassPathXmlApplicationContext：实现类1，从classpath下加载xml文件。
    	(C)FileSystemXmlApplicationContext：实现类2：从文件系统加载xml文件。
    	(C)AnnotationConfigApplicationContext：实现类3：纯注解模式下从java配置类加载配置信息。
    
```

**ApplicationContext为什么直接使用BeanFactory呢**？

**答**：这是spring框架设计的优雅之处，BeanFactory是一个顶层的接口，它里面定义了一些作为容器必须要具备的一些基础的功能。ApplicationContext作为它的子接口，功能更加丰富，比如说：资源加载（xml，java配置类都可以叫资源）。

### 3.1 IoC的纯xml模式回顾：

```java
@Test
public void testIoC() {
    // 通过读取classpath下的xml文件来启动容器（xml模式SE应用下推荐）
    ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
    // 不推荐使用
    //ApplicationContext applicationContext1 = new FileSystemXmlApplicationContext("文件系统的绝对路径");
    AccountDao accountDao = (AccountDao) applicationContext.getBean("accountDao");
    System.out.println(accountDao);
```

```xml
<!--applicationContext.xml-->
<!--根标签beans，里面配置一个又一个的bean子标签，每一个bean子标签都代表一个类的配置-->
<!--id标识对象，class是类的全限定类名-->
<bean id="accountDao" class="com.own.dao.impl. JdbcAccountDaoImpl">
	<property name="Connectionutils" ref="connectionUtils"/>
</bean>
<bean id="transferService" class="com.own.service.impl.TransferServiceImpl">
	<!--set+ name之后锁定到传值的set方法了，通过反射技术可以调用该方法传入对应的值-->
	<property name="AccountDao" ref="accountDao"></property>
</bean>
```

```xml
<web-app><!--web.xml-->
  <display-name>Archetype Created Web Application</display-name>
  <!--配置Spring ioc容器的配置文件-->
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext.xml</param-value>
  </context-param>
  <!--使用监听器启动Spring的IOC容器-->
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
</web-app>
```

已知在TransferServlet.java中，proxyFactory它的获取还是通过自定义的手写的BeanFactory中去拿的，包括service层对象的获取。

```java
// 首先从BeanFactory获取到proxyFactory代理工厂的实例化对象
private ProxyFactory proxyFactory = (ProxyFactory) BeanFactory.getBean("proxyFactory");
private TransferService transferService = (TransferService) proxyFactory.getJdkProxy(BeanFactory.getBean("transferService"));

```

要去掉BeanFactory。报错。那怎么在TransferServlet从容器当中拿到对象呢？

其实，web应用一启动，spring容器一初始化，这个容器已经被放到servletContext上下文当中了，此时，spring提供了一个工具类，可以方便的获取到IoC容器。容器拿到之后自然可以拿到里面的对象。

```java
@Override
private TransferService transferService = null ;

public void init() throws ServletException {
	WebApplicationContext webApplicationContext = WebApplicationContextUtils.getWebApplicationContext(this.getServletContext());
    ProxyFactory proxyFactory = (ProxyFactory)webApplicationContext.getBean("proxyFactory");
    transferService = (TransferService) proxyFactory.getJdkProxy(webApplicationContext.getBean("transferService"))
}
```

### 3.2 Bean的创建方式以及Bean标签属性的回顾：

```xml
<!--Spring ioc 实例化Bean的三种方式-->
<!--方式一：使用无参构造器（推荐）-->
<!--指定了class的全啊限定类名，底层是通过反射的技术，调用了这个class的无参构造器进行实例化-->
<bean id="connectionUtils" class="com.own.utils.ConnectionUtils"></bean>
<!--另外两种方式是为了让自己new的对象加入到SpringIOC容器管理，如果不通过它提供的接口加入进来，那么通过容器.getBean是获取不到自己new的对象的-->
<!--方式二：静态方法-->
<!--<bean id="connectionUtils" class="com.own.factory.CreateBeanFactory" factory-method="getInstanceStatic"/>-->
<!--方式三：实例化方法-->
<!--<bean id="createBeanFactory" class="com.own.factory.CreateBeanFactory"></bean>
<bean id="connectionUtils" factory-bean="createBeanFactory" factory-method="getInstance"/>-->
```

假如在程序中，自己new一个对象，如何加入呢？在factory包下创建一个CreateBeanFactory.java

```java
public class CreateBeanFactory {
    public static ConnectionUtils getInstanceStatic() {        return new ConnectionUtils();    }
    public ConnectionUtils getInstance() {	return new ConnectionUtils();	}
}
```

在test方法中写到：

```java
Object connectionUtils = applicationContext.getBean("connectionUtils");
System.out.println(connectionUtils);
```

```
<!--scope:定义bean的作用范围
singleton:单例，工OC容器中只有一个该类对象。Bean对象生命周期与容器相同。
prototype:原型(多例)，每次使用该类的对象(getBean)，都返回给你一个新的对象。Bean对象spring只负责创建，不负责销毁。
-->
```

