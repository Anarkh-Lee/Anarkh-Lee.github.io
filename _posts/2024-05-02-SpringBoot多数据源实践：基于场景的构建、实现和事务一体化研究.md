---
layout: post
title: 'SpringBoot多数据源实践：基于场景的构建、实现和事务一体化研究'
date: 2024-05-02
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Java



---

> 文章主要探讨了 SpringBoot 多数据源的实践，包括使用场景（如业务复杂、读写分离等）、实现方式（用 MyBatis Plus 的@DS 注解结合 aop 事务拦截）、事务问题（@Transational 和@DS 冲突）及解决方案（如分布式事务、更改传播机制、用@DSTransational 注解）和事务回滚机制，最后总结了多数据源事务控制的适用情况。

# 1. 多数据源应用场景剖析

## 1.1 业务驱动的多数据源需求

- **数据量与业务复杂度引发的分库分表**：在现代企业级应用中，随着业务的不断拓展和用户量的持续增长，数据量呈爆炸式增长。例如，在大型电商平台中，用户数据、订单数据、商品数据等各类数据海量积累。若将所有数据存储于单一数据库，查询和管理效率将急剧下降。以用户相关数据为例，用户的基本信息、购物偏好、历史订单等数据繁多，若与商品库存、评论等数据混合存储，每次查询用户信息时都需遍历大量无关数据，严重影响性能。因此，将与用户相关的数据（如用户信息、购物车、订单历史等）存储在专门的用户库中，而与商品相关的数据（如商品信息、库存、评论等）存放在商品库，这种按业务领域划分数据库的方式，能极大提高数据管理的效率和系统的可扩展性。
  - 数据分布在不同的数据库中，数据库拆了，应用没拆。一个公司多个子项目，各用各的数据库，涉及数据共享（这种情况也可以使用OpenFeign进行服务间调用，但是存在http调用网络损耗）
  - 分库分表，根据业务来划分不同的库，比如与用户相关的表在db_user库，与订单相关的表在db_order库。
- **读写分离优化数据库性能**：数据库的读写操作在性能和资源占用上存在显著差异。写操作通常需要对数据进行加锁，以确保数据的一致性和完整性，这在高并发场景下可能导致读操作被阻塞，严重影响系统的整体读性能。例如，在电商促销活动期间，大量用户同时下单（写操作），若读写不分离，查询商品信息（读操作）的用户可能会遇到长时间的延迟。为解决此问题，许多数据库采用主从架构。主库负责处理所有的写操作（如用户下单、更新库存等），从库则专门用于处理读操作（如查询商品信息、用户订单列表等）。通过这种方式，能有效提高系统的整体性能，满足高并发场景下的读写需求，提升用户体验。
  - master和slave模式，master库只用来写入数据，slave库只用来读取数据。
  - 为了解决数据库的读性能瓶颈（读比写性能更高，写锁会影响读阻塞，从而影响读的性能）。很多数据库拥有主从架构。也就是，一台主数据库服务器，是对外提供增删改业务的生产服务器；另一（多）台从数据库服务器，主要进行读的操作。
  - 可以通过中间件（ShardingSphere、mycat、mysql-proxy、TDDL......），但是有一些规模较小的公司，没有专门的中间件团队搭建读写分离基础设施，因此需要业务开发人员自行实现读写分离。

## 1.2 技术视角下的多数据源场景

- **高性能主从架构**：在追求高性能的场景中，主从架构是一种经典且有效的解决方案。主库作为数据的主要写入点，承担所有的写操作，确保数据的一致性和完整性。从库通过复制主库的数据，实现数据的冗余备份，同时分担读操作的负载。一主一从或一主多从的配置可根据实际业务需求灵活调整。例如，在一个内容管理系统中，主库负责更新文章内容、用户评论等写操作，多个从库可用于处理用户浏览文章、搜索等读操作，根据读操作的流量大小合理分配从库数量，以平衡系统的性能和成本。
- **高可用主备模式**：对于对数据库高可用性要求极高的场景，主备模式提供了可靠的保障。主数据库正常情况下对外提供服务，而备用数据库时刻处于待命状态。当主数据库出现故障（如硬件故障、网络问题等）时，备用数据库能迅速接管服务，确保业务的连续性。多主多备的配置进一步提高了系统的容错能力和可靠性。例如，在金融交易系统中，一旦主数据库发生故障，备用数据库能立即切换，避免交易中断，保障金融业务的稳定运行。
- **同构与异构数据处理**：在实际应用中，企业可能因历史遗留问题、业务并购或技术选型等原因，需要处理存储在不同类型数据库中的数据。同构数据处理涉及到多个相同类型数据库（如多个 MySQL 数据库）之间的数据交互，这种情况相对较为简单，主要挑战在于数据的同步和一致性维护。而异构数据处理则需要在不同数据库系统（如 MySQL 与 Oracle 或 PostgreSQL）之间进行数据整合和操作，这要求系统具备强大的跨数据库兼容性和数据转换能力。例如，企业在升级数据库系统时，可能需要将旧数据库中的数据迁移到新系统中，同时确保新老系统在一段时间内的数据同步。

## 1.3 实际项目中的多数据源案例

- **多库连接的服务需求**：在企业级应用开发中，一个服务往往需要与多个数据库进行交互以满足复杂的业务需求。以一个企业内部的综合管理系统为例，其中的 basic 服务既要连接菜单库获取用户操作菜单信息，又要连接用户库进行用户权限验证和用户个性化信息获取。若菜单库和用户库未分离，随着菜单数量和用户数据的增加，查询和管理的复杂性将呈指数级增长，影响系统整体性能和可维护性。
- **风控系统的多库校验**：风控系统在保障企业业务安全方面起着至关重要的作用，它需要对业务进行全方位的校验，这通常涉及到多个业务库。在金融领域的信贷业务中，风控系统在业务流程的不同阶段（如事前、事中、事后）从相应的业务库中获取数据，进行风险评估和控制。例如，事前校验可能需要从用户信用库中获取用户信用评分、信用记录等信息，以评估用户的信用风险；事中校验可能需要查询交易流水库，实时监控交易行为是否异常；事后校验可能涉及账户状态库，确保交易完成后账户状态的正确性。通过对多个业务库的综合校验，有效降低风险，保障金融业务的稳健运行。
- **批量数据查询与直连数据库**：在某些特定业务场景下，虽然接口提供了数据查询功能，但对于批量查询操作，直接连接数据库可能更为高效。例如，在就业业务中，社保数据查询可能涉及大量人员信息。若通过接口循环调用，由于网络传输的延迟和开销，性能将受到严重影响。在获得客户同意的情况下，直接连接对方的备份库进行查询可以大大提高效率，但需要谨慎操作，避免对生产库造成过大的负载。如在批量查询企业员工社保缴纳记录时，直连备份库能快速获取数据，减少用户等待时间，提升业务处理效率。
- **新老系统数据库同步**：在系统升级或更替过程中，新老数据库的同步是一个关键问题。以企业的人力资源管理系统为例，当经办系统进行更新换代时，友商的老系统数据库可能仍在为大数据中心提供实时服务。为了确保数据的一致性和业务的连续性，需要实时将新系统中的数据写入老系统数据库，或者反之。在一些情况下，如果实时同步不可行，也可采用数据库定时同步策略，但这可能会导致一定的数据延迟。例如，新系统中员工的最新人事变动信息需要及时同步到老系统，以保证整个企业人力资源数据的完整性和准确性。

# 2. 多数据源的实现策略

## 2.1 传统 AOP 事务拦截与数据源选择

在多数据源管理中，传统的实现方式通常采用 AOP（面向切面编程）进行事务拦截。当开启事务时，通过判断数据源的键（key）来决定使用哪个数据源。这种方式需要手动管理数据源的切换逻辑，增加了代码的复杂性和维护成本。

例如，开发人员需要编写切面类，在方法执行前根据业务规则选择合适的数据源，并将其绑定到当前线程的上下文中。以下是一个简单的 AOP 事务拦截实现示例：

```java
@Aspect
@Component
public class DataSourceAspect {

    @Before("@annotation(yourTransactionAnnotation)")
    public void switchDataSource(JoinPoint joinPoint) {
        // 根据业务规则获取数据源键
        String dataSourceKey = determineDataSourceKey(joinPoint);
        // 从数据源池中获取相应数据源并绑定到当前线程
        DataSource dataSource = DataSourceManager.getDataSource(dataSourceKey);
        DataSourceContextHolder.setDataSource(dataSource);
    }

    private String determineDataSourceKey(JoinPoint joinPoint) {
        // 这里可以根据方法参数、类名等信息确定数据源键
        // 例如，如果方法参数中有特定标识，则选择对应的数据源
        Object[] args = joinPoint.getArgs();
        for (Object arg : args) {
            if (arg instanceof YourDataSourceIndicator) {
                return ((YourDataSourceIndicator) arg).getDataSourceKey();
            }
        }
        // 如果无法确定，则返回默认数据源键
        return "defaultDataSource";
    }
}
```

## 2.2 MyBatis Plus的@DS注解的优势

<font style="color:rgba(0, 0, 0, 0.85);">MyBatis Plus 提供的 @DS 注解简化了多数据源的管理过程。该注解可以直接标记在 Mapper 接口或 Service 方法上，指定要使用的数据源。其原理是基于动态数据源切换机制，在运行时根据注解的配置信息，自动从多个数据源中选择合适的数据源进行数据库操作。</font>

<font style="color:rgba(0, 0, 0, 0.85);">例如，在一个包含用户库和订单库的系统中，通过在用户相关的 Mapper 接口上标注 @DS ("user")，在订单相关的 Mapper 接口上标注 @DS ("order")，可以轻松实现数据源的切换，而无需编写复杂的 AOP 拦截逻辑。</font>

## 2.3 ThreadLocal在数据源管理中的角色

<font style="color:rgba(0, 0, 0, 0.85);">在多数据源切换过程中，ThreadLocal 起着关键作用。它用于存储当前线程的数据库键（key）变量，确保在同一线程内的数据库操作都使用相同的数据源。当一个线程进入带有 @DS 注解的方法时，@DS 注解会将对应的数据源键存入 ThreadLocal 中。在后续的数据库操作中，通过获取 ThreadLocal 中的数据源键，从数据源池中获取相应的数据库连接。这种方式保证了数据源的线程安全性，避免了不同数据源之间的干扰。</font>

<font style="color:rgba(0, 0, 0, 0.85);">以下是一个简单的 ThreadLocal 使用示例：</font>

```java
public class DataSourceContextHolder {

    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();

    public static void setDataSource(String dataSourceKey) {
        contextHolder.set(dataSourceKey);
    }

    public static String getDataSource() {
        return contextHolder.get();
    }

    public static void clearDataSource() {
        contextHolder.remove();
    }
}
```

# 3. 多数据源下的事务问题

## <font style="color:rgb(51, 51, 51);">3.1 @Transational和@DS注解冲突问题</font>

### 3.1.1 问题模拟

1. <font style="color:rgb(51, 51, 51);">先导入pom.xml依赖</font>

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

2. <font style="color:rgb(37, 41, 51);">yml文件配置了3个数据源，主数据源是master，从数据源是slave，后续临时加了个数据源temp，为了用于事务的测试，数据库均为MySQL。</font>

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

3. <font style="color:rgb(51, 51, 51);">分别编写主数据源和从数据源的Mapper层接口</font>

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

4. <font style="color:rgb(51, 51, 51);">分别编写对应的XML文件</font>

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

5. <font style="color:rgb(51, 51, 51);">编写Service方法</font>

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

6. <font style="color:rgb(51, 51, 51);">测试运行，报错如下</font>

```plain
### Error updating database.  Cause: java.sql.SQLSyntaxErrorException: Table 'master.role' doesn't exist
### The error may exist in file [D:\JavaProjects\Java\demo\target\classes\mapping\SlaveMapper.xml]
### The error may involve com.example.demo.mapper.SlaveMapper.insertRole-Inline
### The error occurred while setting parameters
### SQL: INSERT INTO role (role)         VALUES(?)
### Cause: java.sql.SQLSyntaxErrorException: Table 'master.role' doesn't exist
; bad SQL grammar []; nested exception is java.sql.SQLSyntaxErrorException: Table 'master.role' doesn't exist

```

<font style="color:rgb(77, 77, 77);">从报错的表面上来看，是因为在主数据库当中不存在role这个表，可是我们已经切换了数据源呀，可为什么还是报错呢？</font>

### <font style="color:rgb(77, 77, 77);">3.1.2 原因分析</font>

<font style="color:rgba(0, 0, 0, 0.85);">在 Spring 框架中，@Transactional 注解用于开启事务管理。当一个方法被标注为 @Transactional 时，Spring 会在方法执行前从数据库连接池获取数据库连接，并将该事务与当前线程绑定。在事务执行过程中，数据库连接通过 ThreadLocal 与当前线程绑定。一旦事务开始，所有的数据库操作都将使用该绑定的连接。当 @DS 注解尝试切换数据源时，如果事务已经开始并且连接已经绑定，新的数据源切换请求可能会被忽略，导致实际执行数据库操作时仍然使用了错误的数据源。例如，在一个同时涉及主库写操作和从库读操作的事务方法中，如果主库事务已经开始并绑定了连接，后续的从库操作可能因为无法切换数据源而失败。</font>

说回到上面的例子：@Transactional开启事务的时候，会先从数据库连接池获取是数据库的连接（基于Spring的AOP切面），我们UserService方法上面没有打上@DS注解，所以Spring默认采用的是主数据源，而且在这之后，这个事务会通过ThreadLocal跟当前线程绑定并也报错了connection连接，通俗的来讲，在进入UserService方法的时候，当前事务已经绑定了数据源Master，在运行到SlaveMapper接口时，因为当前事务的connection连接已经存在，所以拿到的数据源还是默认的Master，于是想找到Slave当中的role表，当然是不可能的，所以只能报错了。

### 3.1.3 解决方案

#### 3.1.3.1 采用分布式事务

<font style="color:rgba(0, 0, 0, 0.85);">分布式事务是解决多数据源事务一致性问题的一种方法。在分布式系统中，多个数据源可能分布在不同的节点或服务中，传统的本地事务无法满足跨数据源事务的需求。分布式事务管理框架（如 Seata、Atomikos 等）提供了强大的事务协调能力，确保在多个数据源之间的操作要么全部成功提交，要么全部回滚。例如，在一个微服务架构中，订单服务可能涉及用户库、商品库和库存库等多个数据源的操作。通过分布式事务框架，可以协调这些数据源的事务，保证数据的一致性。然而，分布式事务的实现相对复杂，需要额外的配置和基础设施支持，并且可能会对系统性能产生一定的影响。以下是一个使用 Seata 实现分布式事务的简单示例：</font>

1. 引入 Seata 相关依赖：

```xml
<dependency>
  <groupId>io.seata</groupId>
  <artifactId>seata-spring-boot-starter</artifactId>
  <version>1.4.2</version>
</dependency>

```

2. 配置 Seata 相关参数（如注册中心地址、事务分组等）：

```properties
seata.registry.type=eureka
seata.registry.eureka.application = seata-server
seata.registry.eureka.service-url = http://localhost:8761/eureka
seata.tx-service-group=my_tx_group

```

3. 在服务中使用 @GlobalTransactional 注解开启分布式事务：

```java
@Service
public class UnitServiceImpl implements UnitService {

    @Autowired
    private UserService userService;
    @Autowired
    private RoleService roleService;

    @GlobalTransactional
    @Override
    public void Add() {
        User user = new User();
        user.setUsername("anarkh");
        user.setPassword("666666");
        userService.insertUser(user);
        Role role = new Role();
        role.setRole("管理员");
        roleService.insertRole(role);
    }
}

```

#### <font style="color:rgb(37, 41, 51);">3.1.3.2 更改事务的传播机制（有问题）</font>

**Propagation.REQUIRES_NEW 机制原理：**

更改事务的传播机制为 Propagation.REQUIRES_NEW 是解决 @Transactional 与 @DS 冲突的一种方法。这种机制会挂起当前事务，并创建一个新的事务，为新事务分配新的数据库连接。新事务与原事务相互独立，各自的提交和回滚操作不会相互影响。在多数据源操作中，这可以确保每个数据源操作都在正确的事务和连接下执行。例如，在主从数据源操作中，将从库操作的事务传播机制设置为 Propagation.REQUIRES_NEW，可以使从库操作在独立的事务中执行，不受主库事务的影响。

**事务传播机制的注意事项与潜在问题：**

虽然 Propagation.REQUIRES_NEW 可以解决数据源切换问题，但也带来了一些潜在问题。在同一个方法中，如果存在多个具有不同事务传播机制的子方法，事务的一致性管理会变得复杂。例如，当一个方法 A 中调用了一个具有 Propagation.REQUIRES_NEW 的方法 B，然后方法 B 中又调用了一个方法 C。如果方法 C 出现异常并回滚，方法 B 根据其事务机制可能已经提交，而方法 A 可能会因为异常而回滚，这就导致了事务的不一致性。因此，在使用这种机制时，需要仔细考虑事务的边界和异常处理逻辑，确保整个业务流程的事务一致性。

改进：

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

2. <font style="color:rgb(77, 77, 77);">编写SlaveService服务代码</font>

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

#### <font style="color:rgb(77, 77, 77);">3.1.3.3 @DSTransational注解代替@Transactional</font>

- @DSTransactional 注解功能概述
  - <font style="color:rgba(0, 0, 0, 0.85);">@DSTransactional 注解是专门为解决多数据源事务问题而设计的。它可以直接替代 @Transactional 注解，在不改变原有代码结构的基础上，实现多数据源事务的正确管理。该注解内部实现了对数据源切换和事务管理的优化，确保在多数据源操作中事务的一致性和正确性。</font>

<font style="color:rgba(0, 0, 0, 0.85);">使用 @DSTransactional 注解非常简单。只需在原来使用 @Transactional 的地方替换为 @DSTransactional，并确保项目中引入了相应的依赖。例如，在一个包含主从数据源操作的 Service 方法中，标注 @DSTransactional 后，该方法中的所有数据库操作（涉及不同数据源）将在一个事务中正确执行，并且在出现异常时能够自动回滚所有相关数据源的操作。</font>

<font style="color:rgb(37, 41, 51);">我们可以使用@DSTransactional注解代替@Transactional即可，其他什么都不用动，也是最简单的方法。</font>

1. <font style="color:rgb(77, 77, 77);">导入pom.xml依赖</font>

```xml
    <dependency>
      <groupId>com.baomidou</groupId>
      <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
      <version>3.5.0</version>
    </dependency>

```

2. <font style="color:rgb(51, 51, 51);">修改UserService代码</font>

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

# 4. 事务回滚机制的详细解析

<font style="color:rgba(0, 0, 0, 0.85);">在多数据源事务中，事务回滚机制的行为取决于异常抛出的位置和事务传播机制的设置。以下是几种常见异常场景下的事务回滚行为分析：</font>

### <font style="color:rgb(51, 51, 51);">4.1 在Master和Slave事务执行前抛出异常</font>

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

<font style="color:rgb(77, 77, 77);">原因：</font><font style="color:rgba(0, 0, 0, 0.85);">事务还未开始执行数据库操作，异常直接导致方法终止，事务不会被提交。</font>

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

<font style="color:rgb(77, 77, 77);">原因：</font><font style="color:rgba(0, 0, 0, 0.85);">在事务执行过程中，当出现异常时，事务会根据异常情况决定是否回滚已经执行的操作。在这种情况下，master 事务已经执行了部分操作，由于异常发生在中间，根据事务的原子性，已执行的 master 事务操作将被回滚。</font>

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

<font style="color:rgb(77, 77, 77);">原因：</font><font style="color:rgba(0, 0, 0, 0.85);">slave 事务设置了 Propagation.REQUIRES_NEW，它会在独立的事务中执行，当 slave 事务中出现异常时，它会自行回滚，同时由于外层的 UserService 方法也在一个事务中（由 @Transactional 注解管理），并且 slave 事务的异常会传播到外层事务，导致外层事务也回滚，从而实现了 master 和 slave 事务的一致性回滚。</font>

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

<font style="color:rgb(77, 77, 77);">原因：</font><font style="color:rgba(0, 0, 0, 0.85);">在事务执行过程中，master 事务和 slave 事务在异常抛出前已经完成了提交操作，而事务的回滚通常是基于异常的捕获和处理机制。在这种情况下，异常发生在事务提交之后，master 事务由于受到 @Transactional 注解的管理，会根据异常情况进行回滚，但 slave 事务已经完成了提交，无法再进行回滚操作。</font>

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

<font style="color:rgb(77, 77, 77);">原因：</font><font style="color:rgba(0, 0, 0, 0.85);"> temp 事务设置了 Propagation.REQUIRES_NEW，它在独立的事务中执行，当 temp 事务中出现异常时，它会自行回滚，而 master 事务由于受到 @Transactional 注解的管理，会根据异常情况进行回滚，slave 事务由于已经提交，不受影响。</font>

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

<font style="color:rgb(77, 77, 77);">SlaveService类：</font><font style="color:rgb(77, 77, 77);"> </font>

```java
@Transactional(propagation = Propagation.REQUIRES_NEW,rollbackFor = Exception.class)
public void slave(){
    Role role = new Role();
    role.setRole("管理员");
    slaveMapper.insertRole(role);
    tempService.temp();
}

```

<font style="color:rgb(77, 77, 77);">TempService类：</font><font style="color:rgb(77, 77, 77);"> </font>

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

<font style="color:rgb(77, 77, 77);">原因：</font><font style="color:rgba(0, 0, 0, 0.85);">在嵌套事务中，内层事务（TempService 中的事务）的异常会传播到外层事务（SlaveService 中的事务），导致 SlaveService 中的事务回滚，而 SlaveService 中的事务回滚又会传播到最外层的 UserService 中的事务，导致整个事务链中的所有事务都回滚，从而保证了数据的一致性。</font>

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

# <font style="color:rgb(77, 77, 77);">5. 适用场景总结与技术选型建议</font>

## 5.1 不同解决方案的适用场景总结

- **@DSTransactional 注解**：适用于单个服务内的多数据源事务管理，简单易用，能够有效地保证事务的一致性和数据源的正确切换。在开发过程中，如果系统架构相对简单，且多数据源操作主要集中在一个服务内，@DSTransactional 是一个不错的选择。例如，在一个小型电商系统中，订单服务可能需要同时操作订单库和用户库，使用 @DSTransactional 注解可以方便地管理这两个数据源的事务，确保数据的完整性。
- **分布式事务框架（如 Seata、Atomikos）**：当系统采用微服务架构，涉及多个服务之间的多数据源事务协作时，分布式事务框架是必要的选择。它能够提供强大的事务协调能力，确保跨服务、跨数据源的事务一致性，但需要投入更多的资源进行配置和维护。例如，在一个大型电商平台中，订单服务、库存服务、支付服务等多个服务可能涉及不同的数据源，此时使用分布式事务框架可以保证整个业务流程的事务正确性，避免数据不一致的情况发生。
- **事务传播机制调整（Propagation.REQUIRES_NEW）**：在一些特定场景下，如需要对部分数据源操作进行独立事务管理时，可以使用事务传播机制调整。但需要注意其可能带来的事务不一致性问题，谨慎使用并确保对事务边界和异常处理有清晰的理解。比如，在一个复杂的业务逻辑中，某个子操作对数据源的写操作独立性要求较高，且不希望受外层事务影响时，可以考虑使用 Propagation.REQUIRES_NEW 来单独管理该子操作的事务，但同时要处理好与外层事务的交互和异常情况。

注：

只有一个服务，且用到多个数据源时，用@DSTransational注解比较方便，可以控制多数据源进行回滚。

为什么说只有一个服务采用@DSTransactional注解，多服务不行吗？不行。如果系统是微服务架构，db1、db2、db3都源于不同的服务，那么db3报错时，前面两个并不会回滚，因为他们都不在一个服务内，@DSTransactional注解此时派不上用场。此时只能采用分布式事务控制了（seata、Atomikos）。

## 5.2 技术选型决策因素与最佳实践

<font style="color:rgba(0, 0, 0, 0.85);">在选择多数据源事务管理解决方案时，需要综合考虑多个因素。首先是系统架构，微服务架构通常需要分布式事务框架来处理跨服务事务，而单体应用或简单的多模块应用可能更适合使用 @DSTransactional 注解。其次是性能要求，分布式事务框架可能会带来一定的性能开销，需要根据系统的实际性能需求进行评估。例如，如果系统对性能要求极高，且多数据源操作相对简单，可能需要谨慎考虑分布式事务框架的使用，避免性能下降。此外，开发团队的技术能力和维护成本也是重要的考虑因素。如果团队对分布式事务框架不熟悉，可能会在配置和维护过程中遇到困难，增加项目风险。最佳实践是在项目初期进行充分的技术调研和架构设计，根据系统的特点和需求选择最合适的事务管理方案，并在开发过程中遵循相关的设计原则和规范，确保事务的正确性和系统的稳定性。同时，建立完善的监控和测试机制，及时发现和解决事务管理中可能出现的问题。

# 6. gitee源码地址

https://gitee.com/anarkhao/transactional







<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>