---
layout: post
title: '从SpringBoot的yml/properties配置文件中取值'
date: 2021-12-30
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: SpringBoot














---

> 从SpringBoot的yml/properties配置文件中取值

开发过程中，一些不容易变动的常量最好统一存放在同一位置，改动时只改动一个文件就可以。一般采用两种方式：

**1.定义常量类：**

```java
public class TestConstant {
	public static final String TEST = "THIS IS A TEST";
}
//使用：直接 类名.常量名 即可
TestConstant.TEST
```

**2.定义在SpringBoot的配置文件中**

SpringBoot的配置文件：application.yml/application-dev.properties，yml的优先级高于properties。

比如在application.yml中定义变量：

```properties
# 注意：paramId:前面必须空两个格，paramId:与123456之间必须空一个格，否则格式错误
test:
  paramId: 123456
```

**取值：**

```java
@Value("${test.paramId}")
```

**两种方式：**

* 1.直接以注入的形式使用：

* 2.封装成一个配置类，然后注入，使用get方法获取

  ```java
  @RestController
  @RequestMapping("/test")
  public class TestController {
  
      //方法2
      @Autowired
      private TestConfig testConfig;
  
      //方法1
      @Value("${test.paramId}")
      private String paramId;
  
      @ApiOperation(value = "测试yml")
      @PostMapping(value = "/testYml",produces = MediaType.APPLICATION_JSON_VALUE)
      public String destroyGroup(){
          String id_from_config = testConfig.getParamId();
          String id_from_this = paramId;
          System.out.println("id_from_config:"+id_from_config);
          System.out.println("id_from_this:"+id_from_this);
          return "sucess";
      }
  }
  ```

  ```java
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.stereotype.Component;
  
  @Component
  public class TestConfig {
      @Value("${test.paramId}")
      private String paramId;
  
      public String getParamId() {
          return paramId;
      }
  
      public void setParamId(String paramId) {
          this.paramId = paramId;
      }
  }
  
  ```

  输出：

  ```console
  id_from_config:123456
  id_from_this:123456
  ```

  



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>