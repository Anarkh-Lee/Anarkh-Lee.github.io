---
layout: post
title: '解决 Spring Boot 多数据源环境下事务管理器冲突问题（非Neo4j请求标记了 @Transactional 尝试启动Neo4j的事务管理器）'
subtitle: "解决 Spring Boot 多数据源环境下事务管理器冲突问题（非Neo4j请求标记了 @Transactional 尝试启动Neo4j的事务管理器）"
date: 2025-02-16
author: Anarkh-Lee
cover: './assets/img//cover_img/Neo4j.png'
tags: 图数据库



---

# 0. 写在前面

到底遇到了什么问题？

**简洁版：**

在 Oracle 与 Neo4j 共存的多数据源项目中，一个仅涉及 Oracle 操作的请求，却因为 Neo4j 连接失败而报错。根本原因是 Spring 的默认事务管理器错误地指向了 Neo4j，导致不相关的请求也受到了 Neo4j 连接状态的影响。

**详细版：**

在包含 Oracle 和 Neo4j 数据库的多数据源 Spring Boot 项目中，一个业务逻辑上仅需访问 Oracle 数据库的 API 请求（标记了 @Transactional ），在执行时却意外地尝试启动 Neo4j 事务。当 Neo4j 数据库无法连接时，这个本应只与 Oracle 交互的请求，反而因为 Neo4j 的连接或事务错误而失败。

# 1. 背景

本项目是一个基于 Spring Boot 的应用，集成了多种数据源：

- 两个 Neo4j 图数据库实例（分别用于开发/生产环境，通过 spring.dev.neo4j.* 和 spring.prod.neo4j.* 配置）。
- 一个 Oracle 关系型数据库（通过 dynamic-datasource-spring-boot-starter 管理，主数据源名为 dsPrimary ）。
- 使用 Mybatis-Plus 作为 Oracle 数据库的 ORM 框架。
- 使用 Spring 的 @Transactional 注解进行事务管理。

# 2. 问题描述

在开发过程中，当两个 Neo4j 数据库实例宕机或无法连接时，调用一个 仅涉及 Oracle 数据库 的 API（例如 /xxx/xx/saveXxx ）时，应用程序抛出异常，导致该 API 不可用。

初始错误：

```plain
org.springframework.transaction.TransactionSystemException: Could not open a new Neo4j session: Unable to connect to [Neo4j IP]:7687...; nested exception is org.neo4j.driver.exceptions.ServiceUnavailableException: Unable to connect to [Neo4j IP]:7687...
    at org.springframework.data.neo4j.core.transaction.Neo4jTransactionManager.doBegin(Neo4jTransactionManager.java:313)
    ...
```

这表明即使 API 不直接操作 Neo4j，Spring 仍然尝试启动一个 Neo4j 事务。

# 3. 分析过程

1. **初步诊断 ：**错误发生在 `Neo4jTransactionManager` 的 doBegin 方法中。这通常意味着 Neo4j 的事务管理器被配置为 Spring 的 默认（Primary）事务管理器 。当 Spring 遇到 @Transactional 注解且未指定特定事务管理器时，它会尝试使用默认的事务管理器，即使该方法本身不涉及 Neo4j。检查发现， `ProdNeo4jConfig.java` 中的 prodTransactionManager Bean 可能被标记了 @Primary 。
2. **尝试移除 Neo4j 的 @Primary ：**移除了 prodTransactionManager Bean 上的 @Primary 注解。
3. **出现新错误 ：**移除后，再次调用该 API，出现 NoUniqueBeanDefinitionException 。

```plain
org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'org.springframework.transaction.TransactionManager' available: expected single matching bean but found 2: devTransactionManager,prodTransactionManager
```

   这表明：

- 该 API 确实需要事务管理（其对应 Service 方法上有 @Transactional 注解）。
- Spring 容器中存在多个 PlatformTransactionManager 类型的 Bean（至少有 devTransactionManager 和 prodTransactionManager ）。
- 由于没有 Bean 被标记为 @Primary ，Spring 无法确定默认使用哪一个。

4. **区分默认数据源与主事务管理器 ：**我在 `application-dev.yml` 中配置了 spring.datasource.dynamic.primary: dsPrimary 。需要明确，此配置仅告知 dynamic-datasource-spring-boot-starter 库哪个数据源是默认的， 并不能 指定哪个 PlatformTransactionManager Bean 是 Spring 事务管理的 @Primary Bean。
5. **尝试切换数据源配置 ：**为了简化问题，尝试将数据源配置从 dynamic-datasource 改回标准的 spring.datasource.druid.* 。
   - **问题 5.1 ：**启动报错 CannotFindDataSourceException: dynamic-datasource can not find primary datasource 。原因是 `application.yml` 中排除了 com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure ，导致 Spring Boot 无法根据 spring.datasource.druid.* 自动创建 DataSource 。
   - **解决 5.1 ：**移除对 DruidDataSourceAutoConfigure 的排除。

```yaml
spring:
  autoconfigure:
    exclude: 
      # - com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure
```

```
- **问题 5.2 ：**启动后仍然报错 NoUniqueBeanDefinitionException ，且错误信息中 只列出了 Neo4j 的事务管理器 ( devTransactionManager , prodTransactionManager )。这表明即使启用了 Druid 自动配置，Oracle 对应的 DataSourceTransactionManager 也没有被成功创建或注册为 Bean，或者 Spring 因某种原因未能找到它。
```

6. **确定最终方向 ：**无论是使用标准 Druid 配置还是 dynamic-datasource ，最可靠的方法是 显式地在 Java 配置中定义 Oracle 数据库（即 dsPrimary ）对应的事务管理器，并将其标记为 @Primary 。

# 4. 解决方案

决定继续使用 dynamic-datasource-spring-boot-starter 以保留其灵活性，并通过 Java 配置显式定义主事务管理器。

在 对应工程的对应目录下新增（或者修改对应的）一个配置类 DataSourceConfig.java ：

```java
package com.xxx.xxxx.config; // 使用项目实际的包路径

import javax.sql.DataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class DataSourceConfig {

    /**
     * 显式定义与动态数据源关联的事务管理器。
     * @param dataSource Spring 容器会自动注入由 dynamic-datasource-spring-boot-starter 创建的代理 DataSource Bean。
     *                   这个代理 DataSource 知道如何根据上下文切换到 dsPrimary 或其他数据源。
     * @return 标记为 @Primary 的事务管理器
     */
    @Bean("transactionManager") // 使用标准的 "transactionManager" 作为 Bean 名称
    @Primary // <--- 关键：标记为主要事务管理器
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        // 使用注入的动态数据源代理来创建事务管理器
        return new DataSourceTransactionManager(dataSource);
    }
}
```

**实施效果：**

添加此配置类后，Spring 容器中存在三个 PlatformTransactionManager Bean：

- devTransactionManager (Neo4j)
- prodTransactionManager (Neo4j)
- transactionManager (Oracle, 使用动态数据源代理, @Primary )

当调用仅涉及 Oracle 且标记了 @Transactional 的 API 时，Spring 会自动选用被 @Primary 标记的 transactionManager ，不再尝试使用 Neo4j 的事务管理器，也解决了 NoUniqueBeanDefinitionException 。应用程序在 Neo4j 宕机时，涉及 Oracle 的 API 可以正常工作。

# 5. 关键点总结

- 在 Spring Boot 中， @Primary 注解用于指定在存在多个同类型 Bean 时应优先注入或使用的 Bean。对于事务管理，它指定了默认的 PlatformTransactionManager 。
- dynamic-datasource-spring-boot-starter 的 spring.datasource.dynamic.primary 配置项用于指定该库内部的默认数据源，与 Spring 的 @Primary 事务管理器是两个不同的概念。
- 在包含多个事务管理器（例如，连接不同类型数据库）的环境中，必须明确指定一个事务管理器为 @Primary ，以供未显式指定事务管理器名称的 @Transactional 注解使用。
- 当使用 dynamic-datasource-spring-boot-starter 时，配置 @Primary 的 DataSourceTransactionManager 需要注入由该库提供的 代理 DataSource Bean 。
- 注意检查 spring.autoconfigure.exclude 配置，避免意外排除了必要的自动配置类。

# 6. 涉及文件

application.yml

```yaml
server:
  port: 8080
  servlet:
    context-path: /xxx

spring:
  profiles:
    active: dev
  autoconfigure:
    exclude: 
      - com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure
      - org.springframework.boot.autoconfigure.data.neo4j.Neo4jReactiveDataAutoConfiguration
  data:
    neo4j:
      database: neo4j
  prod:
    neo4j:
      uri: bolt://ip:port1
      authentication:
        username: xxx
        password: xxxx
      database: xxx
  dev:
    neo4j:
      uri: bolt://ip:port2
      authentication:
        username: xxx
        password: xxxxx
      database: xxx
  datasource: 
    dynamic: 
      strict: false
      primary: dsPrimary
      druid: 
        validation-query: SELECT 1 FROM DUAL
        initial-size: 5
        min-idle: 0
        max-active: 100
        max-wait: 10000
        # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
        time-between-eviction-runs-millis: 30000
        # 配置一个连接在池中最小生存的时间，单位是毫秒
        min-evictable-idle-time-millis: 1800000
        test-while-idle: true
        test-on-borrow: false
        test-on-return: false
        #线程溢出检测控制
        remove-abandoned: true
        #线程溢出时间控制（秒）
        remove-abandoned-timeout-millis: 120
        #线程溢出日志
        log-abandoned: false
        # 是否缓存preparedStatement，也就是PSCache
        pool-prepared-statements: false
        max-pool-prepared-statement-per-connection-size: 0
        # 通过connectProperties属性来打开mergeSql功能；慢SQL记录
        connection-properties: 
          druid: 
            stat: 
              # 合并参数化的SQL
              mergeSql: true
              slowSqlMillis: 5000
        # 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙，'log4j'是用来输出统计数据的
        filters: stat

mybatis-plus:
  configuration:
    # 驼峰命名，默认true-开启
    map-underscore-to-camel-case: false
    jdbc-type-for-null: 'null'
  global-config:
    db-config:
      # 字段验证策略，not-null默认策略，不会对null做处理
      update-strategy: ignored
      insert-strategy: not-null
  mapper-locations: classpath*:/mapper/*Mapper.xml,classpath*:/mapper/**/*Mapper.xml

```

application-dev.yml

```yaml
spring: 
  neo4j:
    uri: bolt://ip:port
  data:
    neo4j:
      database: xxx
  datasource: 
    dynamic: 
      datasource: 
        dsPrimary: 
          driver-class-name: oracle.jdbc.OracleDriver
          url: jdbc:oracle:thin:@ip:port/xxx
          username: xxx
          password: xxxxxx
neo4j:
  authentication:
    username: xxx
    password: xxxxx
```

DevNeo4jConfig.java

```java
import org.neo4j.driver.AuthToken;
import org.neo4j.driver.Config;
import org.neo4j.driver.Driver;
import org.neo4j.driver.GraphDatabase;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.neo4j.core.DatabaseSelectionProvider;
import org.springframework.data.neo4j.core.Neo4jClient;
import org.springframework.data.neo4j.core.Neo4jTemplate;
import org.springframework.data.neo4j.core.mapping.Neo4jMappingContext;
import org.springframework.transaction.PlatformTransactionManager;

import java.net.URI;

@Configuration
@ConditionalOnProperty(prefix = "spring.dev.neo4j", name = "uri")
public class DevNeo4jConfig extends AbstractMultiNeo4jConfig {

    @Bean("devCypherService")
    public CypherService devCypherService(@Qualifier("devNeo4jClient") Neo4jClient neo4jClient) {
        return new CypherServiceImpl(neo4jClient);
    }

    @Bean("devCypherQueryService")
    public CypherQueryService devCypherQueryService(@Qualifier("devNeo4jClient") Neo4jClient neo4jClient) {
        return new CypherQueryServiceImpl(neo4jClient);
    }

    @Bean("devNeo4jClient")
    public Neo4jClient neo4jClient(@Qualifier("devDriver") Driver driver,@Qualifier("devDatabaseSelectionProvider") DatabaseSelectionProvider databaseNameProvider) {
        return Neo4jClient.create(driver, databaseNameProvider);
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.dev.neo4j")
    public KbNeo4jProperties devNeo4jProperties() {
        return new KbNeo4jProperties();
    }

    /**
     * The driver to be used for interacting with Neo4j.
     *
     * @return the Neo4j Java driver instance to work with.
     */
    @Bean("devDriver")
    @Override
    public Driver driver() {
        AuthToken authToken = mapAuthToken(devNeo4jProperties().getAuthentication());
        Config config = mapDriverConfig(devNeo4jProperties());
        URI serverUri = determineServerUri(devNeo4jProperties());
        return GraphDatabase.driver(serverUri, authToken, config);
    }

    @Bean("devNeo4jTemplate")
    @Override
    public Neo4jTemplate neo4jTemplate(final @Qualifier("devNeo4jClient") Neo4jClient neo4jClient,
            final Neo4jMappingContext mappingContext,
            @Qualifier("devDatabaseSelectionProvider") DatabaseSelectionProvider databaseNameProvider) {
        return new Neo4jTemplate(neo4jClient, mappingContext, databaseNameProvider);
    }

    @Bean("devTransactionManager")
    @Override
    public PlatformTransactionManager transactionManager(@Qualifier("devDriver") Driver driver,
            @Qualifier("devDatabaseSelectionProvider") DatabaseSelectionProvider databaseNameProvider) {
        return super.transactionManager(driver, databaseNameProvider);
    }

    @Bean("devDatabaseSelectionProvider")
    @Override
    protected DatabaseSelectionProvider databaseSelectionProvider() {
        String database = devNeo4jProperties().getDatabase();
        return (database != null) ? DatabaseSelectionProvider.createStaticDatabaseSelectionProvider(database)
                : DatabaseSelectionProvider.getDefaultSelectionProvider();
    }

    @Bean("devNeo4jImportService")
    public Neo4jImportServiceImpl neo4jImportService(@Qualifier("devDriver") Driver driver) {
        return new Neo4jImportServiceImpl(driver);
    }
}
```

ProdNeo4jConfig.java

```java
import org.neo4j.driver.AuthToken;
import org.neo4j.driver.Config;
import org.neo4j.driver.Driver;
import org.neo4j.driver.GraphDatabase;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.neo4j.core.DatabaseSelectionProvider;
import org.springframework.data.neo4j.core.Neo4jClient;
import org.springframework.data.neo4j.core.Neo4jTemplate;
import org.springframework.data.neo4j.core.mapping.Neo4jMappingContext;
import org.springframework.transaction.PlatformTransactionManager;

import java.net.URI;

@Configuration
@ConditionalOnProperty(prefix = "spring.prod.neo4j", name = "uri")
public class ProdNeo4jConfig extends AbstractMultiNeo4jConfig {

    @Bean("prodCypherService")
    public CypherService prodCypherService(@Qualifier("prodNeo4jClient") Neo4jClient neo4jClient) {
        return new CypherServiceImpl(neo4jClient);
    }

    @Bean("prodCypherQueryService")
    public CypherQueryService prodCypherQueryService(@Qualifier("prodNeo4jClient") Neo4jClient neo4jClient) {
        return new CypherQueryServiceImpl(neo4jClient);
    }

    @Bean("prodNeo4jClient")
    public Neo4jClient neo4jClient(@Qualifier("prodDriver") Driver driver,@Qualifier("prodDatabaseSelectionProvider") DatabaseSelectionProvider databaseNameProvider) {
        return Neo4jClient.create(driver, databaseNameProvider);
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.prod.neo4j")
    public KbNeo4jProperties prodNeo4jProperties() {
        return new KbNeo4jProperties();
    }

    /**
     * The driver to be used for interacting with Neo4j.
     *
     * @return the Neo4j Java driver instance to work with.
     */
    @Bean("prodDriver")
    @Override
    public Driver driver() {
        AuthToken authToken = mapAuthToken(prodNeo4jProperties().getAuthentication());
        Config config = mapDriverConfig(prodNeo4jProperties());
        URI serverUri = determineServerUri(prodNeo4jProperties());
        return GraphDatabase.driver(serverUri, authToken, config);
    }

    @Bean("prodNeo4jTemplate")
    @Override
    public Neo4jTemplate neo4jTemplate(final @Qualifier("prodNeo4jClient") Neo4jClient neo4jClient,
            final Neo4jMappingContext mappingContext,
            @Qualifier("prodDatabaseSelectionProvider") DatabaseSelectionProvider databaseNameProvider) {
        return new Neo4jTemplate(neo4jClient, mappingContext, databaseNameProvider);
    }

    @Bean("prodTransactionManager")
//    @Primary
    @Override
    public PlatformTransactionManager transactionManager(@Qualifier("prodDriver") Driver driver,
            @Qualifier("prodDatabaseSelectionProvider") DatabaseSelectionProvider databaseNameProvider) {
        return super.transactionManager(driver, databaseNameProvider);
    }

    @Bean("prodDatabaseSelectionProvider")
    @Override
    protected DatabaseSelectionProvider databaseSelectionProvider() {
        String database = prodNeo4jProperties().getDatabase();
        return (database != null) ? DatabaseSelectionProvider.createStaticDatabaseSelectionProvider(database)
                : DatabaseSelectionProvider.getDefaultSelectionProvider();
    }

    @Bean("prodNeo4jImportService")
    public Neo4jImportServiceImpl neo4jImportService(@Qualifier("prodDriver") Driver driver) {
        return new Neo4jImportServiceImpl(driver);
    }
}

```

DataSourceConfig.java

```java
import javax.sql.DataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class DataSourceConfig {

    /**
     * 显式定义与动态数据源关联的事务管理器。
     * @param dataSource Spring 容器会自动注入由 dynamic-datasource-spring-boot-starter 创建的代理 DataSource Bean。
     *                   这个代理 DataSource 知道如何根据上下文切换到 dsPrimary 或其他数据源。
     * @return 标记为 @Primary 的事务管理器
     */
    @Bean("transactionManager") // 使用标准的 "transactionManager" 作为 Bean 名称
    @Primary // <--- 关键：标记为主要事务管理器
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        // 使用注入的动态数据源代理来创建事务管理器
        return new DataSourceTransactionManager(dataSource);
    }

    // 通常不需要在这里手动配置 DataSource Bean，
    // dynamic-datasource-spring-boot-starter 会根据 application-dev.yml 中的配置自动完成。
}

```









<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>