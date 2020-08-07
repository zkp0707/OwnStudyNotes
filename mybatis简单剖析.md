## 1.JDBC的问题分析：

1.数据库配置信息存在硬编码问题             						     =>	配置文件

2.频繁创建释放数据库链接                          						   =>	连接池

3.sql语句、设置参数、获取结果集存在硬编码问题             =>	配置文件

4.手动封装返回结果集，较为繁琐										 =>	反射、内省

## 2.自定义框架设计：

**使用端（项目）**：引入jar包。

​	提供两部分配置信息：

​		1.**sqlMapConfig.xml**：存放数据库配置信息，存放mapper.xml的全路径

​		2.**mapper.xml**:存放sql配置信息（sql语句、参数类型、返回值类型）

**自定义持久层框架本身（工程）**：封装JDBC。

1. **加载配置文件**：根据配置文件的路径，加载配置文件成字节输入流，存储在内存中

   ​	创建Resources类		方法：InputSteam getResourceAsSteam(String path)

2. **创建两个javaBean(容器对象)**：存放的就是对配置文件解析出来的内容

   ​	Configuration（核心配置类）：存放sqlMapConfig.xml解析出来的内容

   ​	MappedStatement（映射配置类）：存放mapper.xml解析出来的内容

3. **解析配置文件**：dom4j

   ​	**创建类**：SqlSessionFactoryBuilder 方法：build( Inputsteam in )

   ​		**第一**：使用dom4j解析配置文件，将解析出来的内容封装到容器对象中

   ​		**第二**:创建SqlSessionFactory对象：生产sqlSession: 会话对象（工厂模式）

4. **创建SqlSessionFactory借口口及实现类DefaultSqlSessionFactory**

   ​	openSession(): 生产sqlSession

5. **创建SqlSession接口及实现类DefaultSession**

   ​	定义对数据库的CRUD操作：selectList()、selectOne()、update()、delete()

6. **创建Executor接口及实现类SimpleExecutor实现类**
       query(Configuration, MappedStatement,Object... params): 执行JDBC代码

## 3.自定义持久层框架问题分析：

​	1.Dao层使用自定义持久层框架，存在代码复用，整个操作的过程模版重复（加载配置文件、创建sqlSessionFactory、生产sqlSession）。

​	2.statementId存在硬编码问题。

​	**解决思路**：使用代理模式生成Dao层接口的代理实现类。 

![](https://s1.ax1x.com/2020/08/06/agN1Wd.png)

getMapper->使用JDK动态代理，为你传递过来的Dao接口生成代理对象proxyInstance->返回给userDao，所以userDao的类型就是proxy类型。->需明确：代理对象调用接口中的任意方法，都会执行invoke方法。

## 4.mybatis简单回顾：

**1.传统开发方式**：

```
@Test
public void test1() throws IOException{
    //1.Resources工具类，配置文件的加载，把配置文件加载成字节输入流
    Inputstream resourceAsstream = Resources.getResourceAsStream("sqlMapConfig.xml"
   	//2.解析了配'置文件，并创建了sqLsessionFactory工厂
    sqlsessionFactory sqlsessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);
    //3.生产sqLlsession。默认开启一个事务，但是该事务不会自动提交,在进行增删改操作时，要手动提交事务
    sqlSession sqlSession = sqlSessionFactory.openSession();
   	//4.sqlSession调用方法，查询所有selectList查询单个: selectone添加: insert修改:update删除: delete    
    List<User> users = sqlsession.selectList( s: "user.findAl1");
    for(user user : users) {
    system.out.println(user);
    sqlsession.close();
}
```

​	**注**：若在"openSession(b:ture)"自动提交，则不需要"sqlSession.commit()" 手动提交。

**2.代理开发方式（主流）**：

```
@Test
public void test5() throws IOException {
    Inputstream resourceAsstream = Resources.getResourceAsStream("sqlMapConfig.xml");
    sqlSessionFactory sqlsessionFactory = new SqlSessionFactoryBuilder().build(resourceAsstream);
    sqlsession sqlsession = sqlsessionFactory.openSession();
    
    IUserDao mapper = sqlsession.getMapper(IUserDao.class);
    List<User> all = mapper.findAll();
    for (User user : all) {
		system.out.println(user);
	}
```

**Mapper接口开发需要遵循以下规范**：

①Mapper.xml文件汇总的namespace与mapper接口的全限定名相同

②Mapper接口方法名和Mapper.xml中定义的每个statement的id相同

③Mapper接口方法的输入参数类型和mapper.xml中定义的每个sql的parameterType的类型相同

④Mapper接口方法的输出参数类型和mapper.xml中定义的每个sql的resulttype的类型相同

![](https://s1.ax1x.com/2020/08/06/agNudO.png)

## 5.mybatis映射开发：

### 1.一对一回顾：

**一对一查询需求**：查询一个订单，与此同时查询出该订单所属的用户信息。

![](https://s1.ax1x.com/2020/08/06/agRNnS.png)

```
public void test1() throws IOException {
    InputStream resourceAsStream = Resources.getResourceAsStream("sqlMapConfig.xml");
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);
    SqlSession sqlSession = sqlSessionFactory.openSession();
    IOrderMapper mapper = sqlSession.getMapper(IOrderMapper.class);
    List<Order> orderAndUser = mapper.findOrderAndUser();
    for (Order order : orderAndUser) {
    System.out.println(order);
    }
}
```

**Tips**：①我们通常引用映射配置文件是:<mappers> <mapper resource="IUerMapper.xml"></mapper></mappers>。

​			  ②直接引入某包下所有接口。需保证：映射配置文件，与该接口同包同名：（扫描接口和与其同包同名的xml）

​					<mappers><package name="com.own.mapper"></mappers>

​			  ③加载某接口，同时把接口中对应的sql注解与当前方法进行加载：（没有②好用）

​					<mapper class="com.own.mapper.IUserMapper"></mapper>

### 2.一对多回顾：			 

**一对多查询需求**：查询所有用户，与此同时查询出该用户具有的订单。

```
<resultMap id="userMagtype="com.own.pojo.User">
    <result property="id" column="uid"></result>
    <result property="username" column="username"></result>
    <collection property="orderList" ofType="com.own.pojo.0rder">
        <result property="id" column="id"></result>
        <result property="orderTime" column="orderTime"></result>
        <result property="total" column="total"></result>
    </collection>
</resultHap>
	
<select id="findAll" resultMap="userMap">
    select * from user u left join orders o on u.id = o.uid
</select>
```

### 3.多对多回顾：

**多对多查询需求**：查询用户同时查询出该用户的所有角色。

```
<resultMap id="userRoleMap" type="com.own.pojo.User">
    <result property="id" column="userid"></result>
    <result property="username" column="username"></result>
	<collection property="roleList" ofType="com.own.pojo.Role">
        <result property="id" column="roleid"></result>
        <result property="roleName" column="roleName"></result>
        <result property="roleDesc" column="roleDesc"></result>
	</collection>
</resultMap>

<select id="findAllUserAndRole" resultMap="userRoleMap">
	select * from user u left join sys_user_role ur on u.id = ur.userid
								left join sys_role r on r.id = ur.roleid
</select>
```

##  6.mybatis注解开发：

### 1.一对一回顾：

```
@Results({
    @Result(property = "id",column = "id"),
    @Result(property = "orderTime",column = "orderTime"),
    @Result(property = "total",column = "total"),
    @Result(property = "user",column = "uid",javaType = User.class,
           one=@One(select = "com.own.mapper.IUserMapper.findUserById"))
})//sql先执行，然后按照@Results配置关系封装，然后根据@One定位到第二段sql语句。
//colume="uid"是我们要传递的参数
@Select("select * from orders")
public List<Order> findOrderAndUser();	
	
//根据id查询用户，接受到的参数即是传递的uid的值。把查询结果封装成User对象。
//封装好的User对象，返回到 property="user"。最终封装完成List集合。
@Select({"select * from user where id = #{id}"})
public User findUserById(Integer id);
```

```
private IUserMapper userMapper;
private IOrderMapper orderMapper;

@Before
public void befor() throws IOException {
    InputStream resourceAsStream = Resources.getResourceAsStream("sqlMapConfig.xml");
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);
    SqlSession sqlSession = sqlSessionFactory.openSession(true);
    userMapper = sqlSession.getMapper(IUserMapper.class);
    orderMapper = sqlSession.getMapper(IOrderMapper.class);
}
    
@Test
public void oneToOne(){
    List<Order> orderAndUser = orderMapper.findOrderAndUser();
    for (Order order : orderAndUser) {
        System.out.println(order);
    }
}
```

### 2.一对多回顾：

**注意**：此时我们并不采用左外连接，而是通过写两条sql语句来完成。

```
select * from user;
select * from orders where uid=查询用户id;
```

```
//查询所有用户、同时查询每个用户关联的订单信息
@Select("select * from user")
@Results({
    @Result(property = "id",column = "id"),
    @Result(property = "username",column = "username"),
    @Result(property = "orderList",column = "id",javaType = List.class,
    	many=@Many(select = "com.lagou.mapper.IOrderMapper.findOrderByUid"))
})
public List<User> findAll();
    
@Select("select * from orders where uid = #{uid}")
public List<Order> findOrderByUid(Integer uid);
```

### 3.多对多回顾：

同样：此时我们并不采用左外连接，而是通过写两条sql语句来完成。

```
select * from user;
select * from role r,user_role ur where r.id=ur.role_id and ur.uer_id=用户的id
```

```
//查询所有用户、同时查询每个用户关联的角色信息
@Select("select * from user")
@Results({
    @Result(property = "id",column = "id"),
    @Result(property = "username",column = "username"),
    @Result(property = "roleList",column = "id",javaType = List.class,
   	 many = @Many(select = "com.own.mapper.IRoleMapper.findRoleByUid"))
})
public List<User> findAllUserAndRole();
    
@Select("select * from sys_role r,sys_user_role ur where r.id = ur.roleid and ur.userid = #{uid}")
public List<Role> findRoleByUid(Integer uid);
```

## 7.mybatis缓存：

**mybatis分为一级缓存和二级缓存**：
①一级缓存是**SqlSession级别**的缓存。在操作数据库时需要**构造sqlSession对象**，在对象中有一个数
据结构(HashMap)用于存储缓存数据。不同的sqlSession之间的缓存数据区域(HashMap)相互不影响。
②二级缓存是**mapper级别**的缓存，多个SqlSession去操作同一个Mapper的sql语句，多个SqlSession可以共用二级缓存，**二级缓存是跨SqlSession的**。

### 一级缓存原理探究与源码分析：

**sqlSession一级缓存(HashMap​)**->key:**cacheKey**(stetementid,params,boundSql,rowBounds)/value:user对象。

```
@Test
public void firstLevelCache(){
    // 第一次查询id为1的用户
    User user1 = userMapper.findUserById(1);
	//首先去一级缓存中去查询->有：直接返回。
	//没有：查询数据库，同时将查询出来的结果存到一级缓存中。
	
    //更新用户
    User user = new User();
    user.setId(1);
    user.setUsername("tom");
    userMapper.updateUser(user);
    sqlSession.commit();
    sqlSession.clearCache();
//结论：做增删改操作，并进行了事务提交，就是刷新一级缓存->clearCashe():手动刷新一级缓存。
	
    // 第二次查询id为1的用户
    User user2 = userMapper.findUserById(1);
   	//首先去一级缓存中去查询->有：直接返回。
	//没有：查询数据库，同时将查询出来的结果存到一级缓存中。 
    System.out.println(user1==user2);//true
}
```



点进sqlSession，打开structure找到clearCache()接口。

```
clearCahche-->claearCache-->clearLocalCache-->clearLocalCache-->clear||结束
SqlSession-->DefaultSqlSession-->Executor-->BaseExecutor-->PerpetualCache
```

在PerpetualCache.java中不难发现：

```
private Map<Object, Object> cache = new HashMap<Object, Object>();

@Override
public void clear() {
cache.clear();
}

```

**cache何时被创建呢**？我们回退到Executor.java——在自定义mybatis中，我们知道Executor就是一个执行器，它的作用就是来执行sql，执行JDBC——我们找到createCacheKey—>进入BaseExecutor.java。

```
@Override
//RowBounds:分页对象。BoundSql：底层要执行的sql语句
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
    	throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());//拿到namespace.id
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());//上行和此行，设置分页参数，就是RowBounds里两个成员属性。
    cacheKey.update(boundSql.getSql());//获取sql
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
   	... ...
   	//由自定义基础知，configuration对象对应的是sqlMapConfig.xml
    if (configuration.getEnvironment() != null) {//判断<environment></environment>不为空
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());//我们在这里进入update
    }
    return cacheKey;
}

this.updateList = new ArrayList<Object>();
public void update(Object object) {
	... ....
	updateList.add(object);//完成对CacheKey四个值的封装
}
```

**createCacheKey方法何时被调用的呢**？我userMapper.findUserById();第一次发起查询的时候，生成cacheKey。然后会使用cacheKey以及查询出来的结果放到一级缓存中。所以，我们需要寻找底层执行sql方法。不管usesMapper调用什么方法，底层最终执行的都是Executor执行器中的query方法。

```
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);//要执行的sql在boundSql封装着
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);//就是要存放到一级缓存中的key值
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);//接下来怎么去放的呢？进入query
}

  @Override
public <E> List<E> query(...) throws SQLException {
   ... ...
    try {
      queryStack++;
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;//根据你刚刚生成的cacheKey从一级缓存中获取
      if (list != null) {//一级缓存中有内容，一般情况为第一次执行以后的执行。
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);//返回结果
      } else {//查询数据库的方法，进去看看 
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    
private <E> List<E> queryFromDatabase(...) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);//查询数据库
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);//cacheKey和查询数据库的返回结果，一起存到了一级缓存中。
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  } 
```

### 二级缓存回顾以及整合redis：

假设有三个sqlSession，这三个sqlSession对同一个UserMapper操作。三个sqlSession共享当前mapper二级缓存区域。

①sqlSession1执行UserMapper查询，先查询二级缓存区域，为空，向数据库发起查询，并把数据库查询出来的结果存一份到二级缓存中。一存。

②sqlSession2执行UserMapper查询，查询二级缓存区域，直接读取。一取。

③sqlSession3执行事物操作(插入、更新、删除)，会清空二级缓存。

在mybatis中默认开启一级缓存，二级缓存需要配置使用。**首先**在sqlMapConfig.xml配置：

**注意**：settings需要配置在properties标签下。

```
<!--开启二级缓存  -->
<settings>
	<setting name="cacheEnabled" value="true"/>
</settings>
```

**其次**：①配置开发：在UserMapper.xml中开启缓存：<cache></cache>

​		    ②注解开发：在IUserMapper.java中开启缓存：@CacheNamespace(implementation=PerpetualCache.class)

```
@Test//sqlSession1,sqlSession2
	public void SecondLevelCache(){
        SqlSession sqlSession1 = sqlSessionFactory.openSession();
        SqlSession sqlSession2 = sqlSessionFactory.openSession();
        SqlSession sqlSession3 = sqlSessionFactory.openSession();

        IUserMapper mapper1 = sqlSession1.getMapper(IUserMapper.class);
        IUserMapper mapper2 = sqlSession2.getMapper(IUserMapper.class);
        IUserMapper mapper3 = sqlSession3.getMapper(IUserMapper.class);

        User user1 = mapper1.findUserById(1);//在此处打断点
        //Cache Hit Ratio=0.0。创建链接，向数据库查询。存入二级缓存。
        sqlSession1.close(); //清空一级缓存
        User user2 = mapper2.findUserById(1);
        //Cache Hit Ratio=0.5
        System.out.println(user1==user2);//false
        //二级缓存缓存的并不是对象，缓存的是对象中的数据。采用二级缓存的方式，在第二次查询对象时，底层重新创建了一个user对象，
        //并且把二级缓存中提前缓存好的数据重新封装成一个对象进行返回。
}
```

**注意**：开启了二级缓存后，还需要将要缓存的pojo实现Serializable接口，为了将缓存数据取出执行反序列化操作，因为二级缓存数据存储介质多种多样，不一定只存在内存中，有可能存在硬盘中，如果我们要再取这个缓存的话，就需要反序列化了。所以mybatis中的pojo都去实现Serializable接口。

```
 @Test//sqlSession3
    public void SecondLevelCache(){
		... ...
        User user1 = mapper1.findUserById(1);
        sqlSession1.close(); //清空一级缓存
        
        User user = new User();
        user.setId(1);
        user.setUsername("lisi");
        mapper3.updateUser(user);
        sqlSession3.commit();
        
        User user2 = mapper2.findUserById(1);//debug看第二次查询是否发送sql语句。
        System.out.println(user1==user2);
    }
```

**补充**：①还可以配置useCache来设置是否禁用二级缓存，默认为true。

```
xml:
<select id="selectUserByUserId" useCache="false" resultType="com.own.pojo.User" parameterType="int">
	select * from user where id=#{id}
</select>
注解：
@Options(useCache = false)
@Select({"select * from user where id = #{id}"})
public User findUserById(Integer id);
```

​		②在mapper的同一个namespace中，如果有其它insert、update、delete操作数据后需要刷新缓如果改成false则不会刷新。使用缓存时如果手动修改数据库表中的查询数据会出现脏读。设置statement配置中的flushCache="true”属性，默认情况下为true，即刷新缓存，如果不执行刷新缓存会出现脏读。

mybatis二级缓存是基于PerpetualCache(mybatis默认实现缓存功能的类)实现的，自定义缓存类必须实现Cache接口。我们进入：PerpetualCache

```
External Libraries->Maven:org.mybatis:mybatis:3.4.5->org->apache->ibatis->cache

//mybatis二级缓存底层仍然是为HashMap存储
private Map<Object, Object> cache = new HashMap<Object, Object>();
```



​	mybatis自带的二级缓存是单服务器工作，无法实现分布式缓存。什么是分布式缓存呢?假设现在有两个服务器1和2，用户访问的时候访问了1服务器，查询后的缓存就会放在1服务器上，假设现在有个用户访问的是2服务器，那么他在2服务器上就无法获取刚刚那个缓存。为了解决这个问题，就得找一个分布式的缓存，专门用来存储缓存数据的，这样不同的服务器要缓存数据都往它那里存，取缓存数据也从它那里取。


​	mybatis提供了一个cache接口，如果要实现自己的缓存逻辑，实现cache接口开发即可。mybatis本身默认实现了一个，但是这个缓存的实现无法实现分布式缓存，所以我们要自己来实现。redis分布式缓存就可以，mybatis提供了一个针对cache接口的redis实现类,该类存在mybatis-redis包。

```
在IUserMapper.java中开启缓存。
@CacheNamespace(implementation=RedisCache.class)
```

#### RedisCache源码分析：

```
//RedisCache在mybatis初始化的时候，由MyBatis的CacheBuilder创建，在创建的过程中有参构造就要执行。
public final class RedisCache implements Cache {
    private final ReadWriteLock readWriteLock = new DummyReadWriteLock();
    private String id;
    private static JedisPool pool;

    public RedisCache(String id) {
        if (id == null) {
            throw new IllegalArgumentException("Cache instances require an ID");
        } else {//执行
            this.id = id;
            //RedisConfig是redis的默认配置信息对象，因此当我们移除redis.properties也能正常执行。但如果我们编写了redis.properties则会对其覆盖。
            //如何拿到这个配置信息对象呢？进parseConfiguration看看。
            RedisConfig redisConfig = RedisConfigurationBuilder.getInstance().parseConfiguration();
            pool = new JedisPool(...);
        }
}

public RedisConfig parseConfiguration(ClassLoader classLoader) {
	Properties config = new Properties();
	//根据配置文件的路径，把这个配置文件加载成字节输入流，那这个文件路径是什么呢？进去看看。
	InputStream input = classLoader.getResourceAsStream(this.redisPropertiesFilename);
	
final class RedisConfigurationBuilder {
    private static final RedisConfigurationBuilder INSTANCE = new RedisConfigurationBuilder();
    private static final String SYSTEM_PROPERTY_REDIS_PROPERTIES_FILENAME = "redis.properties.filename";
    private static final String REDIS_RESOURCE = "redis.properties";//所以redis配置文件名称固定。
    private final String redisPropertiesFilename = System.getProperty("redis.properties.filename", "redis.properties");
    ... ...
}
```

继续RedisCache.class往下看：

```
public void putObject(final Object key, final Object value) {//向redis存值
	//模版方法	
    this.execute(new RedisCallback() {
    	//具体实现模版方法execute()中的doWithRedis方法
        public Object doWithRedis(Jedis jedis) {
        	//底层结构：hash
        	//1.hash的key值，2.项的key值，3.把要缓存的数据通过工具类实现序列化，存到缓存中
            jedis.hset(RedisCache.this.id.toString().getBytes(), key.toString().getBytes(), SerializeUtil.serialize(value));
            return null;
        }
    });
}
public Object getObject(final Object key) {//向redis取值
    return this.execute(new RedisCallback() {
        public Object doWithRedis(Jedis jedis) {
            return SerializeUtil.unserialize(jedis.hget(RedisCache.this.id.toString().getBytes(), key.toString().getBytes()));
        }
    });
}
```

## 7.mybatis插件：

**mybatis四大核心对象**：

```
						  ->ParameterHandler
Executor->StatementHandler
						  ->ResultSetHandler
mybatis允许拦截的方法：						  
Executor:执行器，主要负责增删改查的行为。(update/query/commit/rollback)
StatementHandler:sql语法构建器，完成sql预编译。(prepare/parameterize/batch/update/query)
ParameterHandler：参数处理器：设置参数。(getParameterObject/setParameters)
ResultSetHandler：结果集处理器：处理返回结果集。(handleResultSets/handleOutputParameters)
```

使用插件对这四大核心对象进行拦截，对于mybatis来说，插件就是拦截器，用来增强核心对象的功能，增强本质就是借助底层的动态代理实现。

换句话说，当前这四大组件，返回的时候，返回的并不是原生的对象，而是经过代理后的代理对象。

## **Note：**

**resultType**:声明实体类的全路径，才能通过**反射**获取实体类属性，才能完成字段值和属性值的自动映射封装。 

**StatementId**：是namespace.id。

**attributeValue**:获取对象信息	=>	**getTextTrim**：获得除去空格的文本。

