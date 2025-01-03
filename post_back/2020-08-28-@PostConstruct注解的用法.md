---
layout: post
title: '@PostConstruct注解的用法'
date: 2020-08-28
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Spring



---

> @PostConstruct注解的用法总结

# 1.定义

  从JavaEE5规范开始，Servlet增加了两个影响Servlet生命周期的注解（Annotation）：@PostConstruct和@PreConstruct。这两个注解被用来修饰一个非静态的void()方法.而且这个方法不能有抛出异常声明。

  @PostContruct是spring框架的注解，在方法上加该注解会在项目启动的时候执行该方法，也可以理解为在spring容器初始化的时候执行该方法。

**@PostConstruct注解的方法将会在依赖注入完成后被自动调用。**

# 2.用法

```java
@PostConstruct
 
public void someMethod(){}
```

或者

```java
public @PostConstruct void someMethod(){}
```

# 3.执行顺序

  被@PostConstruct修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器执行一次。PostConstruct在构造函数之后执行，init（）方法之前执行。PreDestroy（）方法在destroy（）方法知性之后执行。

![](.\img\PostConstruct执行顺序.png)

另外，spring中Constructor、@Autowired、@PostConstruct的顺序

其实从依赖注入的字面意思就可以知道，要将对象p注入到对象a，那么首先就必须得生成对象a和对象p，才能执行注入。所以，如果一个类A中有个成员变量p被@Autowried注解，那么@Autowired注入是发生在A的构造方法执行完之后的。



**Constructor >> @Autowired >> @PostConstruct**



例如：

```java
public Class AAA {

    @Autowired
    private BBB b;

    public AAA() {
        System.out.println("此时b还未被注入: b = " + b);
    }

    @PostConstruct
    private void init() {
        System.out.println("@PostConstruct将在依赖注入完成后被自动调用: b = " + b);
    }
}

```

# 4.应用场景及作用

**作用：**

  如果想在生成对象时完成某些初始化操作，而偏偏这些初始化操作又依赖于依赖注入，那么就无法在构造函数中实现。为此，可以使用@PostConstruct注解一个方法来完成初始化，@PostConstruct注解的方法将会在依赖注入完成后被自动调用。

**应用场景：**

* 1.spring项目加载数据字典：
  @PostConstruct注解的方法在项目启动的时候执行这个方法，也可以理解为在spring容器启动的时候执行，可作为一些数据的常规化加载，比如数据字典之类的。

  ```java
  @Service
  public class SelectCodeServiceImpl implements SelectCodeService {
  
      /** 日志 */
      private static Logger LOGGER = LoggerFactory.getLogger(SelectCodeServiceImpl.class);
  
      // 缓存中相对应的key
      private static final String SELECT_CODE_VERSION_KEY = "select_code_key";
  
      // 系统启动默认版本号
      private static final int INIT_DEFAULT_VERSION = 0;
  
      // 缓存默认版本号
      private static int version = -100;
  
      // 二级代码本地缓存
      private static Map<String, Map<String, SelectCode>> selectCodeMap;
  
      @Resource
      private SelectCodeRepository selectCodeRepository;
  
      @Autowired
      private CacheManager cacheManager;
  
      /**
       * 获取代码表
       */
      @Override
      public Map<String, Map<String, SelectCode>> getSelectCodeMap() {
          // 比对版本号
          if (isVersionUpdate()) {
              // 版本号更新，同步当前全局版本号
              LOGGER.debug("发现代码版本更新..");
              refreshLocalSelectCodeMap();
          }
          return selectCodeMap;
      }
  
      /**
       * 启动时检查并设置版本号
       */
      @PostConstruct
      public void init() {
          boolean hasCodeVersion = getCachedVersion() == null ? false : true;
          if (!hasCodeVersion) {
              // 设置全局版本号为INIT_DEFAULT_VERSION
              LOGGER.debug("系统初始化设置全局二级代码版本号");
              setCachedVersion(INIT_DEFAULT_VERSION);
          }
          // 启动时加载一遍二级代码，同步当前全局版本号
          refreshLocalSelectCodeMap();
      }
  
      /**
       * 刷新缓存
       */
      @Override
  //    @Transactional(readOnly = false, propagation = Propagation.REQUIRED)
      public synchronized void notifyCodeRefresh() {
          Integer remoteVersion = getCachedVersion();
          // 增加代码版本号
          setCachedVersion(++remoteVersion);
          // 更新本地代码缓存
          refreshLocalSelectCodeMap();
      }
  
      /**
       * 判断全局版本是否更新
       * 
       * @return
       */
      private boolean isVersionUpdate() {
          return getCachedVersion() > SelectCodeServiceImpl.version;
      }
  
      /**
       * 加载数据
       */
      private synchronized void refreshLocalSelectCodeMap() {
          // 更新本地版本号
          Integer oldVersion = version;
          version = getCachedVersion();
          LOGGER.debug("刷新当前二级代码数据从版本{}到版本{}", oldVersion, version);
          // 从数据库加载数据
  //        List<SelectCode> list = selectCodeRepository.findAll();
  
          List<SelectCode> list = selectCodeRepository.findAll(new Sort("id"));
          // 包装成前台所需结构
          selectCodeMap = SelectCodeMapFactory.crtSelectCodeMap(list);
      }
  
      /**
       * 获取缓存版本号
       * 
       * @return
       */
      private Integer getCachedVersion() {
          ValueWrapper value = getSelectCodeCache().get(SELECT_CODE_VERSION_KEY);
          if (null == value) {
              return null;
          } else {
              return (Integer) value.get();
          }
      }
  
      /**
       * 设置缓存版本号
       * 
       * @return
       */
      private void setCachedVersion(Integer version) {
          LOGGER.debug("设置二级代码版本号version={}", version);
          getSelectCodeCache().put(SELECT_CODE_VERSION_KEY, version);
      }
  
      /**
       * 获取二级代码缓存
       * 
       * @return
       */
      private Cache getSelectCodeCache() {
          return cacheManager.getCache(ApplyCoreConstant.CACHE_SELECT_CODE);
      }
  
      /**
       * 获得二级代码名称
       */
      @Override
      public String getCodeName(String codeType, String value) {
          // 获取代码类
          Map<String, SelectCode> codeMap = selectCodeMap.get(codeType);
          if (null == codeMap) {
              return null;
          }
          // 获取代码对象
          SelectCode code = codeMap.get(value);
          if (null == code) {
              return null;
          }
          return code.getName();
      }
  
  }
  ```

  

* 2.spring项目的定时任务：
  spring自带的@schedule，没有开关，项目启动总会启动一个线程；
  做项目的时候就使用Java的timer，这个设置开关即可自由的控制，关闭的时候，不会启动线程；
  Java的timer也需要找到一个启动类，可以放到main函数里面启动，这样的话，代码的耦合性太高了，而使用PostConstruct是很干净的。



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>