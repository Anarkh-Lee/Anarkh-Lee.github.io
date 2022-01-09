---
layout: post
title: 'new、@Autowired、@Resource的区别'
date: 2022-01-08
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Problem
















---

> new、@Autowired、@Resource的区别

### 1. new和@Autowired的区别

**@Autowired注入的对象在注入之前就已经实例化，是从IOC容器中获取已经初始化的对象。**

**new实例化一个对象，new对象不能注入其他对象，因为new出来的对象声明周期不受IOC容器管控，自然无法完成属性的注入。**

**换句话说，@Autowired方式是由Spring创建的对象，是单例对象，作用域是整个项目，项目一启动就创建了；而new出来的对象作用域只在此对应的类中，每个调用的时候都是会创建一个新的对象，是多例。**

使用@Autowired：

```java
import com.example.SpringBootStudy.service.TestService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping(value = "/test")
public class TestController {

    @Autowired
    private TestService testService;

    @RequestMapping(value = "/print",method = RequestMethod.GET)
    public void test() {
        testService.test();
    }
}
```

```java
import com.example.SpringBootStudy.dao.TestDao;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class TestService {

    @Autowired
    private TestDao testDao;

    public void test() {
        testDao.test();
    }
}
```

```java
import org.springframework.stereotype.Repository;

@Repository
public class TestDao {

    public void test() {
        System.out.println("调用成功！");
    }
}
```

调用输出结果：

```console
调用成功！
```

使用new：

```java
import com.example.SpringBootStudy.service.TestService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping(value = "/test")
public class TestController {

    //@Autowired
    //private TestService testService;
    private TestService testService = new TestService();

    @RequestMapping(value = "/print", method = RequestMethod.GET)
    public void test() {
        testService.test();
    }
}
```

输出结果：

![](.\img\Problem\new注入报错.png)

报出空指针异常，跟进发现是service中未获取到testDao的值：

![](.\img\Problem\new之后dao为空.png)

**总结：**

@Autowired是从IOC容器中获取已经初始化的对象，此对象中@Autowired的属性也已经通过容器完成了注入，整个生命周期都交由容器管控。然而通过new出来的对象，生命周期不受容器管控，自然无法完成属性的自动注入。

### 2. @Autowired和@Resource的区别

@Autowired和@Resource都是用来注入对象的。

#### 2.1 @Autowired默认按byType自动装配，而@Resource默认byName自动装配

**2.1.1 @Autowired的装配顺序：**

![](.\img\Problem\@Autowired装配顺序.png)

**2.1.2 @Resource的装配顺序：**

* 1.如果同时制定了name和type：

  ![](.\img\Problem\Resource装配顺序1.png)

* 2.如果指定了name：

  ![](.\img\Problem\Resource装配顺序2.png)

* 3.如果指定了type：

  ![](.\img\Problem\Resource装配顺序3.png)

* 4.如果既没有指定name，也没有指定type：

  ![](.\img\Problem\Resource装配顺序4.png)

#### 2.2 @Autowired只包含一个参数：required，表示是否开启自动准入，默认是true；而@Resource包含七个参数，其中最重要的两个参数是：name和type

@Autowired要求依赖对象必须存在，如果不存在会报错。
如果允许null值，可以设置它的required属性为false。

```java
//如果UserDao这个Bean不存在，那么会报错。 
@Autowired
private UserDao userDao; 

//设置required=false之后，为空也不报错。 
@Autowired(required=false)
private UserDao userDao; 
```



#### 2.3 @Autowired如果要使用byName，需要使用@Qualifier一起配合；而@Resource如果指定了name，则用byName自动装配，如果指定了type，则用byType自动装配

多个实现类可以通过一下两种方式来指定具体使用哪一种实现：

例子代码：

```java
//接口
public interface TestService {
    public String test();
}

//实现类1
@Servicepublic class TestServiceImpl implements TestService{

    @Override
    public String test() {
        return "TestServiceImpl";
    }
}

//实现类2
@Service
public class TestServiceImpl2 implements TestService{

    @Override
    public String test() {
        return "TestServiceImpl2";
    }
}
```

**1、 通过指定bean的名字来明确到底要实例哪一个类**

@Autowired 需要结合@Qualifier来使用，如下：

```java
@Autowired
@Qualifier("testServiceImpl")
private TestService testService;
```

@Resource可直接通过指定name属性的值即可，不过也可以使用@Qualifier(有点多此一举了...)

```java
@Resource(name = "testServiceImpl")
private TestService testService;    
```

@Resource如果不显示的指定name值，就会自动把实例变量的名称作为name的值的，所以也可以直接这样写：

```java
@Resource
private TestService testServiceImpl;
```

**2、 通过在实现类上添加@Primary注解来指定默认加载类**

```java
@Service
@Primary
public class TestServiceImpl2 implements TestService{

    @Override
    public String test() {
        return "TestServiceImpl2";
    }
}
```

这样如果在使用@Autowired/@Resource获取实例时如果不指定bean的名字，就会默认获取TestServiceImpl2的bean，如果指定了bean的名字则以指定的为准。

**注：**

当使用@Resource时，这种写法会报错：

```java
private ProductDao productDao;
private ProductMxDao productZcDao ; // 这2个名称不一样
```

会提示：productZcDao 不能作为 ProductMxDao 类型注入。

以为@Resource默认按照byName注入，Srping默认将类首字母小写作为bean名字，ProductMxDao的bean名字应该为productMxDao，如果写作productZcDao就找不到了。

如果是@Autowired，则不会报错。

#### 2.4 @Autowired能够用在：构造器、方法、参数、成员变量和注解上；而@Resource能用在：类、成员变量和方法上



#### 2.5 @Autowired是Spring定义的注解，而@Resource是JSR-250定义的注解





注：

byName 通过参数名 自动装配，如果一个bean的name 和另外一个bean的 property 相同，就自动装配。

byType 通过参数的数据类型自动自动装配，如果一个bean的数据类型和另外一个bean的property属性的数据类型兼容，就自动装配

效率上来说@Autowired/@Resource差不多，不过推荐使用@Resource一点，因为当接口有多个实现时@Resource直接就能通过name属性来指定实现类，而@Autowired还要结合@Qualifier注解来使用，且@Resource是jdk的注释，可与Spring解耦。













<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>