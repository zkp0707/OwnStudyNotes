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

### 3.1 spring IoC基础：

#### 3.1.1 IoC的纯xml模式回顾：

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

#### 3.1.2 Bean的创建方式以及Bean标签属性的回顾：

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

**Bean标签属性中的"init-method"属性，"destory-method"属性**。

```xml
<!--applicationContext.xml-->
<bean id="accountDao" class="com.lagou.own.impl.JdbcTemplateDaoImpl" scope="singleton" init-method="init" destroy-method="destory">
    <!--set注入使用property标签，如果注入的是另外一个bean那么使用ref属性，如果注入的是普通值那么使用的是value属性-->
    <property name="ConnectionUtils" ref="connectionUtils"/>
</bean>
```

```java
//在JdbcAccountDaoImpl.java中：
public void init(){
    system.out.println("初始化方法。。。");
}
public void destory(){
    system.out.println("销毁方法。。。");
}
```

这个时候我们运行textIoC方法，发现只打印了“初始化方法。。。”，因为我们的对象只有一个，初始化只有一次，方法只会被调用一次。那销毁方法呢？其实我们整个applicationContext容器并没有关闭，在下面追加applicationContext.close();即可调用destory()方法。

注意到，此时的scope="singleton"如果是"prototype"这个多例类型，多例类型的对象，是不在容器的管理范围之内的，容器给你创建对象之后就丢出去不管了，那么销毁的时候就调不到销毁方法了，因为对象已经不在管理范围之内了。

#### 3.1.3 spring DI依赖注入配置回顾：

DI，依赖注入，往一个对象传值。不考虑spring如何来做的话，就一个普通的对象，用最基本的方式往里面传值的话，也是通过①构造函数（顾名思义，就是利用带参构造函数实现对类成员变量的数据赋值）②set方法（通过类成员的set方法实现数据的注入，也是使用最多的）。其实spring在实现依赖注入的时候，也是这么去做的。

其实目前我们的代码当中依赖关系的维护，向一个对象传入另一个对象，使用的是set方法。例如Dao层的实现类，我们需要传入一个connectionUtils，声明了一个属性，然后给该属性添加了set方法，外部就可以传参进来了。

```java
public class JdbcAccountDaoImpl implements AccountDao{
    privaate ConnectionUtils connectionUtils;
  	public void setConnectionUtils(ConectionUtils connectionUtils){
        this.connectionUtils = connectionUtils;
    }... ...
```

```xml
<!--applicationContext.xml-->
<bean id="accountDao" class="com.lagou.own.impl.JdbcTemplateDaoImpl" scope="singleton" init-method="init" destroy-method="destory">
    <!--set注入使用property标签，如果注入的是另外一个bean那么使用ref属性，如果注入的是普通值那么使用的是value属性-->
    <property name="ConnectionUtils" ref="connectionUtils"/>
</bean>
```

```xml
 <!--set注入使用property标签，如果注入的是另外一个bean那么使用ref属性，如果注入的是普通值那么使用的是value属性-->
<property name="ConnectionUtils" ref="connectionUtils"/>
<property name="name" value="zhangsan"/>
<property name="sex" value="1"/>
<property name="money" value="100.3"/>


<constructor-arg index="0" ref="connectionUtils"/>
<constructor-arg index="1" value="zhangsan"/>
<constructor-arg index="2" value="1"/>
<constructor-arg index="3" value="100.5"/>

<!--name：按照参数名称注入，index按照参数索引位置注入-->
<constructor-arg name="connectionUtils" ref="connectionUtils"/>
<constructor-arg name="name" value="zhangsan"/>
<constructor-arg name="sex" value="1"/>
<constructor-arg name="money" value="100.6"/>
```

```java
public class JdbcAccountDaoImpl implements AccountDao {

private ConnectionUtils connectionUtils;
private String name;
private int sex;
private float money;

public void setConnectionUtils(ConnectionUtils connectionUtils) {
    this.connectionUtils = connectionUtils;
}
public void setName(String name) {
    this.name = name;
}

public void setSex(int sex) {
    this.sex = sex;
}

public void setMoney(float money) {
    this.money = money;
}
    
public JdbcAccountDaoImpl(ConnectionUtils connectionUtils, String name, int sex, float money) {
    this.connectionUtils = connectionUtils;
    this.name = name;
    this.sex = sex;
    this.money = money;
}
```

```xml
<!--set注入注入复杂数据类型-->
<property name="myArray">
    <array>
        <value>array1</value>
        <value>array2</value>
        <value>array3</value>
    </array>
</property>
<property name="myMap">
    <map>
        <entry key="key1" value="value1"/>
        <entry key="key2" value="value2"/>
    </map>
</property>
<property name="mySet">
    <set>
        <value>set1</value>
        <value>set2</value>
    </set>
</property>
<property name="myProperties">
    <props>
        <prop key="prop1">value1</prop>
        <prop key="prop2">value2</prop>
    </props>
</property>
//注：set标签和array数组标签可以替换，props也可以和map替换。原因：array和set都是单执行的集合，底层都一样。map和props是键值对的形式。
```

```java
public class JdbcAccountDaoImpl implements AccountDao {
    private String[] myArray;
    private Map<String,String> myMap;
    private Set<String> mySet;
    private Properties myProperties;
    ... ...
        public Account queryAccountByCardNo(String cardNo) throws Exception {
        //从连接池获取连接
        // Connection con = DruidUtils.getInstance().getConnection();
        Connection con = connectionUtils.getCurrentThreadConn();//此处打断点
        ... ...
    }... ...
}//看一下connectionUtil中是否包含内容。
```

**xml和注解相结合模式回顾**：

xml+注解结合模式，xml文件依然存在，所以，springIoC容器的启动仍然从家中xml开始。一般情况，第三方jar中的bean定义在xml，自己开发的bean定义使用注解。

### 3.2 spring IoC高级特性：

#### 3.2.1 lazy-init 延迟加载：

ApplicationContext 容器的默认⾏为是在启动服务器时将所有 singleton bean 提前进⾏实例化。提前实例化意味着作为初始化过程的⼀部分，ApplicationContext 实例会创建并配置所有的singleton bean。

```xml
<!--lazy-init：配置bean对象的延迟加载，true/false，默认为false，立即加载。-->
<bean id="lazyResult" class="com.own.pojo.Result" lazy-init="false">
```

```java
// 测试bean的lazy-init属性
@Test
public void testBeanLazy(){
//启动容器（容器初始化）
ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
//getBean获取bean对象使用
Object lazyResult = applicationContext.getBean("lazyResult");//打断点
//applicationContext->beanFactory->singletonObjects(map)->lazyResult.
//说明在容器初始化的时候，bean对象完成了创建。此时再getBean时，就从容器（缓存）中拿了出来。
System.out.println(lazyResult);
}
```

```xml
<!--lazy-init：配置bean对象的延迟加载，true/false，默认为false，立即加载。-->
<bean id="lazyResult" class="com.own.pojo.Result" lazy-init="true">
```

此时的singletonObjects就没有了lazyResult。往下走一步时，便出现了。

注解模式：在pojo对象上添加@lazy注解。默认为true。

**全局bean配置lazy-init**：在applicationContext.xml的头beans标签的头中加：default-lazy-init="true/default/false"。

**应用场景**：

①开启延迟加载⼀定程度提⾼容器启动和运转性能。
②对于不常使⽤的 Bean 设置延迟加载，这样偶尔使⽤的时候再加载，不必要从⼀开始该 Bean 就占
⽤资源。

#### 3.2.2 FactoryBean 和 BeanFactory:

BeanFactory接⼝是容器的顶级接⼝，定义了容器的⼀些基础⾏为，负责⽣产和管理Bean的⼀个⼯⼚，具体使⽤它下⾯的⼦接⼝类型，⽐如ApplicationContext；此处我们重点分析FactoryBean。

Spring中Bean有两种，⼀种是普通Bean，⼀种是⼯⼚Bean（FactoryBean），FactoryBean可以⽣成某⼀个类型的Bean实例（返回给我们），也就是说我们可以借助于它⾃定义Bean的创建过程。

Bean创建的三种⽅式中的静态⽅法和实例化⽅法和FactoryBean作⽤类似，FactoryBean使⽤较多，尤其在Spring框架⼀些组件中会使⽤，还有其他框架和Spring框架整合时使⽤。

进入FactoryBean(接口)：

```java
// 可以让我们⾃定义Bean的创建过程（完成复杂Bean的定义）
public interface FactoryBean<T> {
    @Nullable
    // 返回FactoryBean创建的Bean实例，如果isSingleton返回true，则该实例会放到Spring容器的单例对象缓存池中(Map
    T getObject() throws Exception;
    @Nullable
    // 返回FactoryBean创建的Bean类型
    Class<?> getObjectType();
    // 返回作⽤域是否单例
    default boolean isSingleton() {
    return true;
	}}
```

**案例**：使用FactoryBean创建一个Bean对象(company)，加入到springIoC容器中管理。先在pojo包中创建company实体类，然后实现FactoryBean接口。

```java
public class CompanyFactoryBean implements FactoryBean<Company> {
    private String companyInfo; // 公司名称,地址,规模
    public void setCompanyInfo(String companyInfo) {
    	this.companyInfo = companyInfo;
	}
@Override
public Company getObject() throws Exception {
    // 模拟创建复杂对象Company
    Company company = new Company();
    String[] strings = companyInfo.split(",");
    company.setName(strings[0]);
    company.setAddress(strings[1]);
    company.setScale(Integer.parseInt(strings[2]));
    return company;
}
@Override
public Class<?> getObjectType() {
	return Company.class;
}
@Override
public boolean isSingleton() {
	return true;
}}
```

怎么把它配置在springIoC中呢？在applicationContext.xml文件中：

```xml
<bean id="companyBean" class="com.own.factory.CompanyFactoryBean">
	<property name="companyInfo" value="哈哈,上海,100"/>
</bean>
```

接下来启动容器，获取companyFactoryBean。此时获取的，是FactoryBean实现类类型，还是FactoryBean帮我们产生的Company类型。

```java
Object companyBean = applicationContext.getBean("companyBean");
System.out.println("bean:" + companyBean);
//结果：bean:Company{name='哈哈', address='上海', scale=100}
```

说明FactoryBean属性是帮我们生成另外一个Bean，虽然我们在xml中指定的是FactoryBean，但是它的功能就是抛出另外一个Bean。若非要获取CompanyFactoryBean，则需在getBean(name:"&Beanid")。FactoryBean在spring一些组件，包括其他框架向spring整合时，底层用的还是很多。

#### 3.2.3 后置处理器：

在框架中，但凡说到XXX器，其实往往都是预留的扩展接口，以什么形式来提供扩展呢?往往是提供了一个接口让我们去实现。

spring提供了两种后处理bean的扩展接口：①、BeanPostProcesssor ②、BeanFactoryPostProcessor。

一个产品要生产出来要先有工厂。工厂初始化(BeanFactory)->Bean对象。

①在BeanFactory初始化之后，可以使用BeanFactoryPostProcessor进行后置处理做一些事情。

②在Bean对象实例化（并不是Bean整个生命周期完成）之后，可以使用BeanPostProcesssor 进行后置处理做一些事情。 

spring中的Bean肯定是一个对象，但一个对象不一定是springBean，因为要经过一个流水线，走完这个流水线才能称之为springBean。

springBean生命周期：

```java
Ⅰ、实例化Bean Ⅱ、设置属性值 Ⅲ、调用BeanNameAware的setBeanName方法 Ⅳ、调用BeanFactoryAware的setBeanFactory方法
Ⅴ、调用ApplicationContextAware的setApplicationContext方法Ⅵ、调用BeanPostProcessor的预初始化方法
Ⅶ、调用InitializingBean的afterProperties方法 Ⅷ、调用定制的初始化方法init-method.
Ⅸ、调用BeanPostProcessor的后初始化方法：prototype:将准备就绪的Bean交给调用者。singleton：spring缓存池中准备就绪的Bean->Bean销毁：调用DisposableBean的方法/调用destroy-method属性配置的销毁方法。

Bean生命周期的整个执行过程描述:
1）根据配置情况调用Bean构造方法或工厂方法实例化Bean。
2）利用依赖注入完成Bean中所有属性值的配置注入。
3）如果Bean实现了BeanNameAware接口，则Spring调用Bean的setBeanName()方法传入当前Bean的id值。
4）如果Bean实现了BeanFactoryAware接口，则Spring调用setBeanFactory)方法传入当前工厂实例的引用。
5）如果Bean实现了ApplicationContextAware接口，则Spring调用setApplicationContext(方法传入当前ApplicationContext 实例的引用。
6）如果BeanPostProcessor和Bean关联，则Spring 将调用该接口的预初始化方法postProcessBeforelnitialzation)对Bean进行加工操作，此处非常重要，Spring	的AOP就是利用它实现的。
7）如果Bean实现了InitializingBean接口，则Spring将调用afterPropertiesSet()方法。
8）如果在配置文件中通过init-method属性指定了初始化方法，则调用该初始化方法。
9）如果BeanPostProcessor和Bean关联，则Spring将调用该接口的初始化方法postProcessAfterlnitialization()。此时，Bean已经可以被应用系统使用了。
10)如果在<bean>中指定了该Bean的作用范围为scope="singleton"，则将该Bean放入Spring loC的缓存池中，
	将触发Spring对该Bean的生命周期管理;如果在<bean>中指定了该Bean的作用范围为scope="prototype"，则将该Bean交给调用者。
11）如果Bean实现了DisposableBean接口，则Spring 会调用destory)方法将Spring 中的Bean销毁;
	如果在配置文件中通过destory-method属性指定了Bean的销毁方法，则Spring将调用该方法对Bean进行销毁。
注意∶Spring为Bean 提供了细致全面的生命周期过程，通过实现特定的接口或<bean>的属性设置，都可以对Bean的生命周期过程产生
	虽然可以随意配置<bean>的属性，但是建议不要过多地使用Bean实现接口，因为这样会导致代码和Spring 的聚合过于紧密。
```

### 3.3 spring IoC源码深度剖析：

**Spring源码构建**：
①下载源码（github②安装gradle ③导⼊（耗费⼀定时间）④编译⼯程（顺序：core-oxm-context-beans-aspects-aop）⑤⼯程—>tasks—>compileTestJava。

我们在spring-framework里面写一个测试类。IoCTest.java

#### 3.3.1 spring IoC容器初始化主体流程：

①**springIoC的容器体系**：

```java
public void testIoC() {
    // ApplicationContext是容器的高级接口，BeanFacotry（顶级容器/根容器，规范了/定义了容器的基础行为）
    // Spring应用上下文，官方称之为 IoC容器（错误的认识：容器就是map而已；准确来说，map是ioc容器的一个成员，
    // 叫做单例池, singletonObjects,容器是一组组件和过程的集合，包括BeanFactory、单例池、BeanPostProcessor等以及之间的协作流程）
    /**
    * Ioc容器创建管理Bean对象的，Spring Bean是有生命周期的
    * 构造器执行、初始化方法执行、Bean后置处理器的before/after方法、：AbstractApplicationContext#refresh#finishBeanFactoryInitialization
    * Bean工厂后置处理器初始化、方法执行：AbstractApplicationContext#refresh#invokeBeanFactoryPostProcessors
    * Bean后置处理器初始化：AbstractApplicationContext#refresh#registerBeanPostProcessors
    */
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
    OwnBean ownBean = applicationContext.getBean(OwnBean.class);
    System.out.println(ownBean);
}
```

以 ClasspathXmlApplicationContext 为例，深⼊源码说明 IoC 容器的初始化流程。

②**Bean⽣命周期关键时机点**：

