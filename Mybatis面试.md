> mybatis最近还在看源码，还在整理。。。

Mybatis面试
===

说一下mybatis
---

`mybatis`是一个半自动ORM（对象关系映射）框架，内部封装了JDBC。使用`mybatis`时，我们只需要关注**sql语句**，而无需关注其他操作（如创建销毁Connection、Statement、sql的执行）。



什么是半自动ORM？
---

全自动就是连sql语句都被封装好了，我们只需要用，但是我们就不能对sql语句进行优化。

半自动就是自己写sql语句，其他连接数据库等操作交给框架。



#{}和${}的区别
---

`${}`的作用跟`#{}`一样，都是可以获取参数。但两者实现的原理不同。

在第三节知道了`#{}`的底层是使用`Preparestatement`来进行操作的，在执行sql语句时会把它**替换成`?`**。

而`${}` 的底层是使用`Statement`来进行操作的，在执行sql语句时会获取参数，然后在将参数使用 **字符串拼接的方式** 拼接起来。

比如：`"select * from student where name=${name}"`，mybatis执行时会先把它变成`"select * from student where name="+"'name'"` ，然后再执行。

学过数据库的人都知道这种方式会发生**sql注入**，在实际开发中是很危险的。

那`${}`这么危险，为什么还会存在。它之所以存在是因为它还留有一手`#{}`不能实现的功能，不然早给淘汰了。

`${}`的特殊功能：**可以替换表名或列名**。

因为在执行sql语句时，表名跟列名不能使用占位符。

你想想要是表名跟列名用占位符替换就成这样子了`select * from ? where ?=1;`数据库看到这都不开心：我\*\*你\*\*个\*\*，这不是为难我数据库吗。

综上所述：

- \#使用？在sql语句中做展位符，使用的是Preparestatement对象，它可以防止sql注入，更安全更高效
- $使用字符串拼接的方式，使用的是Statement对象，有sql注入风险，不安全。一般用它来替换表名或列名



会话 跟 执行器
---

在mybatis中，使用执行器来操作数据库，执行真正的sql语句的。所以执行器需要做的有： **获取连接**、**获取事务类型**以及**获取配置信息**，根据这些信息创建执行器，接着执行器执行sql。

简单版的执行器执行过程如下：

```java
// transaction封装了连接信息、隔离级别、是否自动提交
// configuration为配置类
SimpleExecutor executor = SimpleExecutor(configuration, transaction);
// MappedStatement封装了sql信息
MappedStatement ms = configuration.getMappedStatement("com.test.UserMapper.selectById");
// rowBounds封装分页信息
// resultHandler结果处理器
// boundSql真正执行的sql
executor.doQuery(ms, parameter, rowBounds, resultHandler, boundSql)
```

为了高效，mybatis还采用了缓存的技术，将结果缓存在执行器中。而上面代码加点判断缓存的逻辑就更接近mybatis的实现了。

又因为需要使用多个执行器，每个执行器都需要获取连接、存放缓存数据等操作。所以可以加一层抽象类，将相同操作提取出来，因而形成了**BaseExecutor**。

> 执行器包括三个：SimpleExecutor、ReuseExecutor、BatchExecutor。**BaseExecutor**是他们的父类，这种设计模式为**模板方法**。

会话跟执行器是一对一的关系，执行器内部会经过复杂的操作后才执行sql，而会话是对执行器内部的封装，会话只需要调用方法执行即可，无需关心执行器内部的具体逻辑，这属于**外观模式。**



mybatis有哪些执行器，他们之间的区别
---

执行器有三种：simple（默认）、reuse、batch

- simpleExecutor：每执行一次sql开启一个statement对象，用完立刻关闭。
- reuseExecutor：执行器内会缓存**预编译**通过的 sql，相同的sql使用同一个statement对象，即不会重复编译。对象用完后不关闭，而是放入map中.
- batchExecutor：将所有的sql添加到批处理中，统一执行。更新大量数据时有明显的速度提升，如果是查询则跟simpleExecutor没什么区别。
  - **批处理操作必须手动处理/提交事物** 

如何指定执行器？settings标签中指定ExecutorType的方式或者openSession(ExecutorType et) 的方式。



能否简单说下Mybatis加载的流程？
---

1. **加载配置文件和映射文件**：其中全局配置文件（mybatis-config.xml）配置了Mybatis 的运行环境信息(数据源、事务等)，SQL映射文件（Mapper.xml）中配置了与 SQL 执行相关的信息。
2. **创建会话工厂**：MyBatis通过读取配置文件的信息来构造出会话工厂(SqlSessionFactory)，即通过SqlSessionFactoryBuilder 构建 SqlSessionFactory。
3. **创建会话**：拥有了会话工厂，MyBatis就可以通过它来创建会话对象(SqlSession)。会话对象是一个接口，该接口中包含了对数据库操作的增删改查方法。
4. **创建执行器**：因为会话对象本身不能直接操作数据库，所以它使用了一个叫做数据库执行器(Executor)的接口来帮它执行操作。
5. **封装SQL对象**：执行器(Executor)将待处理的SQL信息封装到一个对象中(MappedStatement)，该对象包括SQL语句、输入参数映射信息和输出结果映射信息。
6. **操作数据库**：拥有了执行器和SQL信息封装对象就使用它们访问数据库了，最后再返回操作结果，结束流程。



指定mapper映射文件方式
---

指定mapper映射文件方式有四种：resource、url、class、package。

```xml
<mappers>
    <!-- 除了package其他都是mapper标签 -->
    <package name="dao"/>
    <mapper resource="com/dao/EmployeeMapper.xml"/>
    <mapper url="file:E:/Study/Test/src/com/EmployeeMapper.xml"/>
    <mapper class="dao.adminDao.EmployeeMapper"/>
</mappers>
```



mapper接口跟xml绑定的过程
---

1. 首先会将所有接口的动态代理工厂放到MapperRegistry中。在调用方法前会使用MapperRegistry.getMapper(接口的Class类型)的方式获取动态代理工厂

2. 动态代理工厂利用jdk动态代理的方式创建mapper的代理对象；

   ```java
   protected T newInstance(MapperProxy<T> mapperProxy) {
       return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
   }
   ```

3. 代理对象执行接口方法时会判断该方法是否为Object的方法或者接口的默认方法，是的话直接执行。否则创建mapper方法的MapperMethod对象；

   - MapperMethod中封装了**方法名、参数和返回类型**、以及**sql类型** 
   - 这里方法名为`接口全限定名.方法名`，所以要求xml中指定`namespace为接口全限定名，id为方法名` 
   - mapper接口跟xml就是通过方法名来连接的。

4. MapperMethod对象根据sql类型（commonType）调用sqlsession对应的方法（CRUD），将方法名和方法参数传递过去；

5. sqlsession根据方法名找到对应的MapperStatement，并交给执行器执行。



mybatis是否支持延迟加载？如何进行的？
---

MyBatis中的延迟加载，也称为懒加载，是指在进行表的关联查询时，按照设置 延迟规则 推迟对关联对象的select查询。

例如在进行一对多查询的时候，只查询出一方，当程序中需要多方的数据时，mybatis再发出sql语句进行查询，这样子延迟加载就可以的减少数据库压力。

延迟加载的应用**要求**：关联对象的查询与主加载对象的查询必须是**分别**进行的select语句，**不能是使用多表连接**所进行的select查询。因为，多表连接查询，实质是对一张表的查询，对由多个表连接后形成的一张表的查询。会一次性将多张表的所有信息查询出来。

相关配置：

```xml
<settings>
	<!-- 开启延迟加载-->
    <setting name="lazyLoadingEnabled" value="true"/>
    <!-- 配置侵入式延迟加载, 默认为false（深度加载）
          侵入式：默认只会执行主加载SQL，那么当访问主加载对象的详细信息时才会执行关联对象的SQL查询
          深度延迟：默认只执行主加载SQL，那么当调用到主加载对象中关联对象的信息时才会执行关联对象的SQL查询
        -->
    <setting name="aggressiveLazyLoading" value="false"/>
</settings>
```

***

Mybatis仅支持association关联对象和collection关联集合对象的延迟加载，association指的就是一对一，collection指的就是一对多查询（查询结果是list）

例：

```xml
<mapper namespace="cn.xh.dao.UserDao">
    <select id="findCategory" resultMap="categoryMap">
        select * from category
    </select>
    <resultMap id="categoryMap" type="cn.xh.pojo.Category">
        <id column="cid" property="cid"></id>
        <result column="cname" property="cname"></result>
        <!--  
		 select属性表示要执行的sql语句
		 column表示执行sql语句要传的参数
		 property表示查询出来的结果给哪个属性
		-->
        <collection property="books" column="cid" select="findBook"></collection>
    </resultMap>

    <select id="findBook" parameterType="int" resultType="cn.xh.pojo.Book">
        select * from book where cid = #{cid}
    </select>
</mapper>
```



延迟加载的原理
---

使用CGLIB创建目标对象的代理对象，当调用目标方法时，进入拦截器方法，比如调用a.getB().getName()，拦截器invoke()方法发现a.getB()是null值，那么就会**单独执行事先保存好的查询关联B对象的sql**，把B查询上来，然后调用a.setB(b)，于是a的对象b属性就有值了，接着完成a.getB().getName() 方法的调用。

即每次先查询主表的数据，当需要获取其他表数据为null时会去数据库中查询该表的数据。



分页插件原理
---

> mybatis插件的原理都是通过**动态代理**跟**拦截器**实现的。每个插件都有一个对应的拦截器，mybatis会给每个插件创建代理对象，代理对象会获取插件指定的方法，然后交给插件对应的拦截器执行。

Executor会为**所有的插件创建代理对象**（JDK方式），代理类为Plugin类，Plugin的invoke方法里面会**获取query方法**然后交给 对应拦截器实现的**intercept方法** 去执行。这里的intercept方法的实现就是PageHelper拦截器的intercept方法。

所有的查询在mybatis底层都是调用query方法，所以执行查询方法时都会被代理类获取然后调用PageHelper拦截器实现的intercept方法。

在PageHelper拦截器实现的intercept方法中，首先拿到所有的参数（根据MappedStatement获取），然后获取到用户设置的分页参数（**被PageHelper封装成Page对象放到ThreadLocal中**，保证线程安全）。

拿到这些参数之后会**根据不同的数据库进行sql语句的拼接**，最后执行拼接后的sql语句。

比如Mysql，会在原sql上使用`new StringBuilder.append("limit ? ?")`的方式，拼接sql语句。







