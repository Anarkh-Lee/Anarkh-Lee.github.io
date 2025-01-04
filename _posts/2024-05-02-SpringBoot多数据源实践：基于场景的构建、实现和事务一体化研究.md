---
layout: post
title: 'SpringBoot多数据源实践：基于场景的构建、实现和事务一体化研究'
date: 2024-05-02
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Java



---

> 文章主要探讨了 SpringBoot 多数据源的实践，包括使用场景（如业务复杂、读写分离等）、实现方式（用 MyBatis Plus 的@DS 注解结合 aop 事务拦截）、事务问题（@Transational 和@DS 冲突）及解决方案（如分布式事务、更改传播机制、用@DSTransational 注解）和事务回滚机制，最后总结了多数据源事务控制的适用情况。

# 一、多数据源使用场景

**从业务角度来看：**

1. 业务复杂（数据量大）-分库分表

数据分布在不同的数据库中，数据库拆了，应用没拆。一个公司多个子项目，各用各的数据库，涉及数据共享（这种情况也可以使用OpenFeign进行服务间调用，但是存在http调用网络损耗）

分库分表，根据业务来划分不同的库，比如与用户相关的表在db_user库，与订单相关的表在db_order库。

2. 读写分离

master和slave模式，master库只用来写入数据，slave库只用来读取数据。

为了解决数据库的读性能瓶颈（读比写性能更高，写锁会影响读阻塞，从而影响读的性能）。很多数据库拥有主从架构。也就是，一台主数据库服务器，是对外提供增删改业务的生产服务器；另一（多）台从数据库服务器，主要进行读的操作。

可以通过中间件（ShardingSphere、mycat、mysql-proxy、TDDL......），但是有一些规模较小的公司，没有专门的中间件团队搭建读写分离基础设施，因此需要业务开发人员自行实现读写分离。

**从技术角度来看：**

1. 数据库高性能场景：主从，包括一主一从、一主多从等。在主库进行增删改操作，在从库进行读操作。
2. 数据库高可用场景：主备，包括一主一备、多主多备等。在数据库无法访问时可以切换。
3. 同构或异构数据的业务处理：需要处理的数据存储在不同的数据库中，包括同构（如都是MySQL）、异构（如一个MySQL，另外是PG或者Oracle）。

**实际项目开发中遇到的情况：**

1. basic服务，既要连接菜单库、又要连接用户库
2. 风控系统、连接不同的业务库对业务进行事前、事中、事后校验（业务前校验、保存校验、提交校验、发钱之后重新校验）
3. 在征得客户同意、并且是总集或者也是自己的服务数据库的情况下（比如就业调用社保）。为什么不用接口进行数据传输？提供的接口一般都是查单个人或者单个公司的，就业业务动不动就要批量查，这种情况下循环调用的话，网络传输的损耗比较大，直查数据库速度快很多。但是这种一般不直接连对方的生产库，一般都是连对方的备份库，避免查询量较大把对方生产库直接宕机。
4. 新老数据库同步。（经办系统上新，但是友商老系统数据库仍然实时为大数据中心提供服务，需要实时写入。如果不用实时写入的话，完全可以使用数据库定时同步）

# 二、实现方式

一般来说，使用aop进行事务拦截，开启事务时，对数据源key进行判断选择用哪个数据源。

这里直接使用MyBatis Plus提供的@DS注解进行实现。

注：一般来说，都是用ThreadLocal来存储当前线程的数据库key变量的。

# 三、多数据源使用过程中的事务问题

在动态切换数据库当中，遇到了@Transational和@DS冲突的问题。

## 1.场景模拟

1.1 先导入pom.xml依赖

```xml
<dependencies>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>

  <!-- 数据源切换依赖 -->
  <dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>3.5.2</version>
  </dependency>

  <!-- MySQL依赖 -->
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.29</version>
  </dependency>

  <!-- Mybatis依赖 -->
  <dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.2</version>
  </dependency>

</dependencies>
```

1.2 <font style="color:rgb(77, 77, 77);">yml文件配置了3个数据源，主数据源是master，从数据源是slave，后续临时加了个数据源temp，为了用于事务的测试，数据库均为MySQL。</font>

```yaml
server:
  port: 8080

spring:
  datasource:
    dynamic:
      primary: master
      datasource:
        master:
          username: anarkh
          password: 123456
          url: jdbc:mysql://localhost:3306/anarkh_master?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=UTC
          driver-class-name: com.mysql.cj.jdbc.Driver

        slave:
          username: anarkh
          password: 123456
          url: jdbc:mysql://localhost:3306/anarkh_slave?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=UTC
          driver-class-name: com.mysql.cj.jdbc.Driver
        temp:
          username: anarkh
          password: 123456
          url: jdbc:mysql://localhost:3306/temp?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=UTC
          driver-class-name: com.mysql.cj.jdbc.Driver

mybatis:
  mapper-locations: classpath:mapping/*.xml
```

1.3 分别编写主数据源和从数据源的Mapper层接口

```java
@Mapper
@DS("master")
public interface MasterMapper {
    int insertUser(User user);
}
```

```java
@Mapper
@DS("slave")
public interface SlaveMapper {
    int insertRole(Role role);
}
```

1.4 分别编写对应的XML文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.demo.mapper.MasterMapper">
    <insert id="insertUser" parameterType="com.example.demo.bean.User">
        INSERT INTO user (username, password)
        VALUES(#{username}, #{password})
    </insert>
</mapper>
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.demo.mapper.SlaveMapper">
    <insert id="insertRole" parameterType="com.example.demo.bean.Role">
        INSERT INTO role (role)
        VALUES(#{role})
    </insert>
</mapper>
```

1.5 编写Service方法

```java
@Service
public class UserService {

    @Resource
    private MasterMapper masterMapper;

    @Resource
    private SlaveMapper slaveMapper;

    @Transactional
    public void Add(){
        User user = new User();
        user.setUsername("anarkh");
        user.setPassword("666666");
        masterMapper.insertUser(user);
        Role role = new Role();
        role.setRole("管理员");
        slaveMapper.insertRole(role);
    }
}
```

1.6 测试运行，报错如下

```plain
### Error updating database.  Cause: java.sql.SQLSyntaxErrorException: Table 'master.role' doesn't exist
### The error may exist in file [D:\JavaProjects\Java\demo\target\classes\mapping\SlaveMapper.xml]
### The error may involve com.example.demo.mapper.SlaveMapper.insertRole-Inline
### The error occurred while setting parameters
### SQL: INSERT INTO role (role)         VALUES(?)
### Cause: java.sql.SQLSyntaxErrorException: Table 'master.role' doesn't exist
; bad SQL grammar []; nested exception is java.sql.SQLSyntaxErrorException: Table 'master.role' doesn't exist
```

<font style="color:rgb(77, 77, 77);">从报错的表面上来看，是因为在主数据库当中不存在role这个表，可是我们已经切换了数据源呀，可为什么还是报错呢，具体的详细原因，我们往下细说。</font>

## 2.原因分析

<font style="color:rgb(77, 77, 77);">@Transactional开启事务的时候，会先从数据库连接池获取是数据库的连接（基于Spring的AOP切面），我们UserService方法上面没有打上@DS注解，所以Spring默认采用的是主数据源，而且在这之后，这个事务会通过ThreadLocal跟当前线程绑定并也报错了connection连接，通俗的来讲，在进入UserService方法的时候，当前事务已经绑定了数据源Master，在运行到SlaveMapper接口时，因为当前事务的connection连接已经存在，所以拿到的数据源还是默认的Master，于是想找到Slave当中的role表，当然是不可能的，所以只能报错了。</font>

## <font style="color:rgb(77, 77, 77);">3.解决方案</font>

目前，一般有三种解决方案。

### 3.1 采用分布式事务

### 3.2 更改事务的传播机制（有问题）

<font style="color:rgb(77, 77, 77);">其实我们只要更改一下事务的传播机制，将它设置为：Propagation.REQUIRES_NEW即可，意思就是将原有的Spring事务挂起，并创建一个新的事务并分配的一个新的connection，两者不影响，具体操作如下：</font>

1. <font style="color:rgb(77, 77, 77);">修改原有的UserService代码</font>

<font style="color:rgb(77, 77, 77);">不用通过@DS指定数据源，因为默认是Master；将slave业务操作分离出来，封装到一个Service服务类当中，再通过@Resource注解注入进来，最后还是指定一下回滚策略，遇到异常就回滚。</font>

```java
@Service
public class UserService {

    @Resource
    private MasterMapper masterMapper;

    @Resource
    private SlaveService slaveService;

    @Transactional(rollbackFor = Exception.class)
    public void Add(){
        User user = new User();
        user.setUsername("anarkh");
        user.setPassword("666666");
        masterMapper.insertUser(user);
        slaveService.slave();
    }

}
```

2. <font style="color:rgb(77, 77, 77);"> 编写SlaveService服务代码</font>

<font style="color:rgb(77, 77, 77);">必须通过@DS指定一下数据源为slave，在slave方法上面重新修改一下事务的传播机制即可</font>

```java
@Service
@DS("slave")
public class SlaveService {
    @Resource
    private SlaveMapper slaveMapper;

    @Transactional(propagation = Propagation.REQUIRES_NEW,rollbackFor = Exception.class)
    public void slave(){
        Role role = new Role();
        role.setRole("管理员");
        slaveMapper.insertRole(role);
    }
}
```

3. <font style="color:rgb(77, 77, 77);">其他的保持不变，最后我们再测试一下，看一下输出结果，成功了！</font>

![]({{ '/assets/img/多数据源/mysql1.png' | prepend: '' }})

![](.\img\多数据源\mysql1.png)

![]({{ '/assets/img/多数据源/mysql2.png' | prepend: '' }})

![](.\img\多数据源\mysql2.png)

**<font style="color:rgb(254, 44, 36);">注意：</font>**<font style="color:rgb(77, 77, 77);"> 因为Propagation.REQUIRES_NEW是开启一个新的事务并重新分配一个新的数据库连接，在同一个方法中，有A方法和一个开启新的传播事务的B方法，如果B方法中出现了异常发生了回滚，那么A方法也会随之回滚，</font><font style="color:rgb(254, 44, 36);">但是，但是，但是！</font><font style="color:rgb(77, 77, 77);">如果B方法后面有一个新方法C，当C方法中出现了异常，C方法回滚了，但是B方法根据它事务机制并且已经提交了事务，那么就会出现A事务回滚了，B事务提交了，C事务回滚了，这样ABC三个方法出出现了事务不一致的问题，在下面的</font><font style="color:rgb(254, 44, 36);">事务回滚机制的第五条有演示。</font>

### <font style="color:rgb(77, 77, 77);">3.3 @DSTransational注解代替@Transactional</font>

<font style="color:rgb(77, 77, 77);">我们可以使用@DSTransactional注解代替@Transactional即可，其他什么都不用动，也是最简单的方法。</font>

1. <font style="color:rgb(77, 77, 77);">导入pom.xml依赖</font>

```xml
    <dependency>
      <groupId>com.baomidou</groupId>
      <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
      <version>3.5.0</version>
    </dependency>
```

2. 修改UserService代码

```java
@Service
public class UserService {

    @Resource
    private MasterMapper masterMapper;

    @Resource
    private SlaveMapper slaveMapper;

    @DSTransactional
    public void Add(){
        User user = new User();
        user.setUsername("anarkh");
        user.setPassword("666666");
        masterMapper.insertUser(user);
        Role role = new Role();
        role.setRole("管理员");
        slaveMapper.insertRole(role);
    }
}
```

3. <font style="color:rgb(77, 77, 77);">运行测试，查看输出结果，成功！</font>

![]({{ '/assets/img/多数据源/mysql3.png' | prepend: '' }})
![](.\img\多数据源\mysql3.png)
![]({{ '/assets/img/多数据源/mysql4.png' | prepend: '' }})
![](.\img\多数据源\mysql4.png)

## 4.事务回滚机制

### 4.1 在Master和Slave事务执行前抛出异常

<font style="color:rgb(77, 77, 77);">UserService类：</font>

```java
@Transactional(rollbackFor = Exception.class)
public void Add(){
    User user = new User();
    user.setUsername("anarkh");
    user.setPassword("666666");
    int a = 1/0;
    masterMapper.insertUser(user);
    slaveService.slave();
}
```

<font style="color:rgb(77, 77, 77);">结果：数据保持一致</font>

### <font style="color:rgb(79, 79, 79);">4.2、当master事务和slave事务中间抛出异常</font>

<font style="color:rgb(77, 77, 77);">UserService类：</font>

```java
@Transactional(rollbackFor = Exception.class)
public void Add(){
    User user = new User();
    user.setUsername("anarkh");
    user.setPassword("666666");
    masterMapper.insertUser(user);
    int a = 1/0;
    slaveService.slave();
}
```

<font style="color:rgb(77, 77, 77);">结果：回滚master事务，slave事务无影响</font>

### <font style="color:rgb(79, 79, 79);">4.3、在slave方法中抛出异常</font>

<font style="color:rgb(77, 77, 77);">SlaveService类：</font>

```java
@Transactional(propagation = Propagation.REQUIRES_NEW,rollbackFor = Exception.class)
public void slave(){
    Role role = new Role();
    role.setRole("管理员");
    slaveMapper.insertRole(role);
    int a = 1/0;
}
```

<font style="color:rgb(77, 77, 77);">结果：master和slave事务都会进行回滚</font>

### <font style="color:rgb(79, 79, 79);">4.4、在master和slave事务之后</font>

<font style="color:rgb(77, 77, 77);">UserService类：</font>

```java
@Transactional(rollbackFor = Exception.class)
public void Add(){
    User user = new User();
    user.setUsername("anarkh");
    user.setPassword("666666");
    masterMapper.insertUser(user);
    slaveService.slave();
    int a = 1/0;
}
```

<font style="color:rgb(77, 77, 77);">结果：master事务回滚，slave已经提交事务，入库</font>

### <font style="color:rgb(79, 79, 79);">4.5、临时添加一个temp数据库，进行插入操作，并抛出异常</font>

<font style="color:rgb(77, 77, 77);">UserService类：</font>

```java
@Transactional(rollbackFor = Exception.class)
public void Add(){
    User user = new User();
    user.setUsername("anarkh");
    user.setPassword("666666");
    masterMapper.insertUser(user);
    slaveService.slave();
    tempService.temp();
}
```

<font style="color:rgb(77, 77, 77);">TempService类：</font>

```java
@Transactional(propagation = Propagation.REQUIRES_NEW,rollbackFor = Exception.class)
public void temp(){
    Car car = new Car();
    car.setCar("Benz");
    tempMapper.insertCar(car);
    int a = 1/0;
}
```

<font style="color:rgb(77, 77, 77);">结果：master回滚，slave事务提交，temp回滚</font>

### <font style="color:rgb(79, 79, 79);">4.6、嵌套</font>

<font style="color:rgb(77, 77, 77);">UserService类：</font>

```java
@Transactional(rollbackFor = Exception.class)
public void Add(){
    User user = new User();
    user.setUsername("anarkh");
    user.setPassword("666666");
    masterMapper.insertUser(user);
    slaveService.slave();
}
```

<font style="color:rgb(77, 77, 77);">SlaveService类： </font>

```java
@Transactional(propagation = Propagation.REQUIRES_NEW,rollbackFor = Exception.class)
public void slave(){
    Role role = new Role();
    role.setRole("管理员");
    slaveMapper.insertRole(role);
    tempService.temp();
}
```

<font style="color:rgb(77, 77, 77);">TempService类： </font>

```java
@Transactional(propagation = Propagation.REQUIRES_NEW,rollbackFor = Exception.class)
public void temp(){
    Car car = new Car();
    car.setCar("Benz");
    tempMapper.insertCar(car);
    int a = 1/0;
}
```

<font style="color:rgb(77, 77, 77);">结果：master回滚，slave回滚，temp回滚</font>

### <font style="color:rgb(79, 79, 79);">4.7、使用@DSTransactional注解，在slave和temp之间抛出异常</font>

<font style="color:rgb(77, 77, 77);">UserService类</font>

```java
@DSTransactional
public void Add(){
    User user = new User();
    user.setUsername("anarkh");
    user.setPassword("666666");
    masterMapper.insertUser(user);
    Role role = new Role();
    role.setRole("管理员");
    slaveMapper.insertRole(role);
    int a=1/0;
    Car car = new Car();
    car.setCar("Benz");
    tempMapper.insertCar(car);
}
```

<font style="color:rgb(77, 77, 77);">结果：master回滚，slave回滚、temp回滚</font>

### <font style="color:rgb(79, 79, 79);">4.8、使用@DSTransactional注解，在最后抛出异常</font>

```java
@DSTransactional
public void Add(){
    User user = new User();
    user.setUsername("anarkh");
    user.setPassword("666666");
    masterMapper.insertUser(user);
    Role role = new Role();
    role.setRole("管理员");
    slaveMapper.insertRole(role);
    Car car = new Car();
    car.setCar("Benz");
    tempMapper.insertCar(car);
    int a=1/0;
}
```

<font style="color:rgb(77, 77, 77);">结果：master回滚，slave回滚、temp回滚</font>

### <font style="color:rgb(79, 79, 79);">5.gitee源码地址</font>

[<font style="color:rgb(77, 77, 77);">https://gitee.com/anarkhao/transactional</font>](https://gitee.com/anarkhao/transactional)

### <font style="color:rgb(77, 77, 77);">6.总结</font>

只有一个服务，切用到多个数据源时，用@DSTransational注解比较方便，可以控制多数据源进行回滚。

为什么说只有一个服务采用@DSTransactional注解，多服务不行吗？不行。如果系统是微服务架构，db1、db2、db3都源于不同的服务，那么db3报错时，前面两个并不会回滚，因为他们都不在一个服务内，@DSTransactional注解此时派不上用场。此时只能采用分布式事务控制了（seata、Atomikos）。







<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>