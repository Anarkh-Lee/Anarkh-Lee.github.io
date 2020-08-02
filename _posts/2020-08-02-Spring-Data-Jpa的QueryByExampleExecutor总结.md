---
layout: post
title: 'Spring Data Jpa的QueryByExampleExecutor总结'
date: 2020-08-02
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Jpa


---

> 创建动态查询，包括Example.of()用法

### 1.JpaRepository介绍

从 JpaRepository 开始的子类，都是 Spring Data 项目对 JPA 实现的封装与扩展。JpaRepository 本身继承 PagingAndSortingRepository 接口，是针对 JPA 技术的接口，提供 flush()、saveAndFlush()、deleteInBatch()、deleteAllInBatch() 等方法。

![](.\img\jpaUML.png)

从图中其实可以发现，JPA 的实现类最关键是：SimpleJpaRepository，我们多次提到，还有一个最关键的实现类是 QuerydslJpaRepository。
从图中还可以看出来，最关键的几个接口 QueryByExampleExecutor、JpaSpecificationExecutor。
从图中还可以好好体会一些接口的用意（暴露那些该暴露的操作方法，而不是一股脑的把所有的方法都暴露给使用的人，因为不是每个场景下面都会用到所有方法。）

### 2.QueryByExampleExecutor的使用

按示例查询（QBE）是一种友好用户的查询技术，具有简单的接口，它允许动态查询创建，并且不需要编写包含字段名称的查询。

从UML图中，可以看出继承JpaRepository接口后，自动拥有了按“实例”进行查询的诸多方法，可见Spring Data团队已经认为QBE是Spring Jpa的基本功能了，继承QueryByExampleExecutor和继承JpaRepository都会有这些基本方法，所以QueryByExampleExecutor位于Spring Data Common中，而JpaRepository位于Spring Data JPA中。

#### 2.1 QueryByExampleExecutor的详细配置

QueryByExampleExecutor 的源码：

```java
public interface QueryByExampleExecutor<T> {

	/**
	 * 根据样例查找一个符合条件的对象，如果没找到将返回null;如果返回多个对象时将抛出
	 *org.springframework.dao.IncorrectResultSizeDataAccessException异常
	 */
	<S extends T> Optional<S> findOne(Example<S> example);

	/**
	 *根据样例查找符合条件的所有对象集合
	 */
	<S extends T> Iterable<S> findAll(Example<S> example);

	/**
	 *根据样例查找符合条件的所有对象集合，并根据排序条件排好序 
	 */
	<S extends T> Iterable<S> findAll(Example<S> example, Sort sort);

	/**
	 *根据样例查找符合条件的所有对象集合，并根据分页条件分页 
	 */
	<S extends T> Page<S> findAll(Example<S> example, Pageable pageable);

	/**
	 *查询符合样例条件的记录数样
	 */
	<S extends T> long count(Example<S> example);

	/**
	 *检查数据库表中是否包含符合样例条件的记录，存在返回true,否则返回false
	 */
	<S extends T> boolean exists(Example<S> example);
}
```

从源码可以看出，只要了解了Example基本上就可以掌握QueryByExampleExecutor的用法和API了。

Example的源码：

注意：Example接口在`org.springframework.data.domain`包下

```java
public interface Example<T> {

	/**
	 *创建一个泛型对象的样例，泛型对象必须是与数据库表中一条记录对应的实体类
	 */
	static <T> Example<T> of(T probe) {
		return new TypedExample<>(probe, ExampleMatcher.matching());
	}

	/**
	 * 根据实体类和匹配规则创建一个样例
	 * @return
	 */
	static <T> Example<T> of(T probe, ExampleMatcher matcher) {
		return new TypedExample<>(probe, matcher);
	}

	/**
	 *获取样例中的实体类对象
	 */
	T getProbe();

	/**
	 *获取样例中的匹配器
	 */
	ExampleMatcher getMatcher();

	/**
	 *获取样例中的实体类类型
	 */
	@SuppressWarnings("unchecked")
	default Class<T> getProbeType() {
		return (Class<T>) ProxyUtils.getUserClass(getProbe().getClass());
	}
}
```

从源码中可以看出Example主要包含以下三部分内容：

* 1.Probe：这是具有填充字段的域对象的实际实体类，即查询条件的封装类（又可以理解为：查询条件参数），必填。
* 2.ExampleMatcher：ExampleMatcher有关于如何匹配特定字段的匹配规则，它可以重复使用在多个示例，必填；如果不填，用默认的（又可以理解为参数的匹配规则）。
* 3.Example：Example有Probe探针和ExampleMatcher组成，它用户创建查询，即组合查询参数和参数的匹配规则。

#### 2.2 QueryByExampleExecutor 的使用案例

```java
//创建查询条件数据对象
Customer customer = new Customer();
customer.setName("李");
customer.setAddress("大连");
//创建匹配器，即如何使用查询条件
ExampleMatcher matcher = ExampleMatcher.matching() //构建对象
        .withMatcher("name", GenericPropertyMatchers.startsWith()) //姓名采用“开始匹配”的方式查询
        .withIgnorePaths("focus");  //忽略属性：是否关注。因为是基本类型，需要忽略掉
//创建实例
Example<Customer> ex = Example.of(customer, matcher); 
//查询
List<Customer> ls = dao.findAll(ex);
//输出结果
for (Customer bo:ls)
{
    System.out.println(bo.getName());
}
```

上述例子中，使用`Example<Customer> ex = Example.of(customer, matcher);`创建“实例”。可以看到，Example对象有customer何matcher共同创建，结合上述例子明确一下定义：

* 1.Probe：实体对象，在持久化框架中与 Table 对应的域对象，一个对象代表数据库表中的一条记录，如上例中 Customer 对象。在构建查询条件时，一个实体对象代表的是查询条件中的“数值”部分，如要查询姓“李”的客户，实体对象只能存储条件值“李*”。

* 2.ExampleMatcher：匹配器，它是匹配“实体对象”的，表示了如何使用“实体对象”中的“值”进行查询，它代表的是“查询方式”，解释了如何去查的问题。例如，要查询姓“李”的客户，即姓名以“李”开头的客户，该对象就表示了“以某某开头的”这个查询方式，如上例中 withMatcher(“name”,GenericPropertyMatchers.startsWith())。

* 3.Example：实例对象，代表的是完整的查询条件。由实体对象（查询条件值）和匹配器（查询方式）共同创建。

QueryByExample通常翻译为“实例查询”，顾名思义，就是通过一个例子来查询，要查询到是Customer对象，查询条件也是一个Customer对象，通过一个现有的客户对象作为例子，查询和这个例子相匹配的对象。

#### 2.3 QueryByExampleExecutor 的特点及约束

* 1.支持动态查询：即支持查询条件个数不固定的情况，如客户列表中有多个过滤条件，用户使用时在“地址”查询框中输入了值，就需要按地址进行过滤，如果没有输入值，就忽略这个过滤条件。对应的实现是，在构建查询条件 Customer 对象时，将 address 属性值置具体的条件值或置为 null。
* 2.不支持过滤条件分组：即不支持过滤条件用 or（或）来连接，所有的过滤查件，都是简单一层的用 and（并且）连接，如 firstname = ?0 or (firstname = ?1 and lastname = ?2)。
* 3.仅支持字符串的开始/包含/结束/正则表达式匹配和其他属性类型的精确匹配。查询时，对一个要进行匹配的属性（如：姓名 name），只能传入一个过滤条件值，如以 Customer 为例，要查询姓“李”的客户，“李”这个条件值就存储在表示条件对象的 Customer 对象的 name 属性中，针对于“姓名”的过滤也只有这么一个存储过滤值的位置，没办法同时传入两个过滤值。正是由于这个限制，有些查询是没办法支持的，例如要查询某个时间段内添加的客户，对应的属性是 addTime，需要传入“开始时间”和“结束时间”两个条件值，而这种查询方式没有存两个值的位置，所以就没办法完成这样的查询。

#### 2.4 重点理解ExampleMatcher

**2.4.1需要考虑的因素**

查询条件的表示，有两部分，一是条件值，二是查询方式。条件值用实体对象（如Customer对象）来存储，相对简单，当页面传入过滤条件值时，存入相对应的属性中，没入传入时，属性保持默认值。查询方式是用匹配器ExampleMatcher来表示，情况相对复杂些，需要考虑的因素有：

（1）Null值的处理。当某个条件值为Null,是应当忽略这个过滤条件呢，还是应当去匹配数据库表中该字段值是Null的记录？

（2）基本类型的处理。如客户Customer对象中的年龄age是int型的，当页面不传入条件值时，它默认是0，是有值的，那是否参与查询呢？

（3）忽略某些属性值。一个实体对象，有许多个属性，是否每个属性都参与过滤？是否可以忽略某些属性？

（4）不同的过滤方式。同样是作为String值，可能“姓名”希望精确匹配，“地址”希望模糊匹配，如何做到？

（5）大小写匹配。字符串匹配时，有时可能希望忽略大小写，有时则不忽略，如何做到？

**2.4.2五个配置项**

围绕上面一系列情况，ExampleMatcher中定义了5项配置来解决这些问题。

```java
public class ExampleMatcher {
	NullHandler nullHandler; //Null值处理方式
	StringMatcher defaultStringMatcher; //默认字符串匹配方式
	boolean defaultIgnoreCase; //默认大小写忽略方式
	PropertySpecifiers propertySpecifiers; //各属性特定查询方式
	Set<String> ignoredPaths; //忽略属性列表
	......
}
```

（1）`nullHandler`：Null值处理方式，枚举类型，有2个可选值，`INCLUDE`（包括），`IGNORE`（忽略）。表示作为条件的实体对象中，一个属性值（条件值）为Null时，是否参与过滤。当该选项值是INCLUDE时，表示仍参与过滤，会匹配数据库表中该字段值为Null的记录；若为IGNORE时，表示不参与过滤。

（2）`defaultStringMatcher`：默认字符串匹配方式，枚举类型，有6个可选值，`DEFAULT`（默认，效果同EXACT），`EXACT`（相等），`STARTING`（开始匹配），`ENDING`（结束匹配），`CONTAINING`（包含，模糊匹配），`REGEX`（正则表达式）。该配置对所有字符串属性过滤有效，除非该属性在`propertySpecifiers`中单独定义自己的匹配方式。

（3）`defaultIgnoreCase`：默认大小写忽略方式，布尔型，当值为false时，即不忽略，大小写不相等。该配置对所有字符串属性过滤有效，除非该属性在`propertySpecifiers`中单独定义自己的匹配方式。

（4）`propertySpecifiers`：各属性特定查询方式，描述了各个属性单独定义的查询方式，每个查询方式中包含4个元素：属性名、字符串匹配方式、大小写忽略方式、属性转换器。如果属性未单独定义查询方式，或单独查询方式中，某个元素未定义（如：字符串匹配方式），则采用ExampleMatcher中定义的默认值，即上面介绍的`defaultStringMatcher`和`defaultIgnoreCase`的值。

（5）`ignoredPaths`：忽略属性列表，忽略的属性不参与查询过滤。

| 字符串匹配方式             | 对应逻辑                                    |
| -------------------------- | ------------------------------------------- |
| DEFAULT（不忽略大小写）    | firstname = ?0                              |
| DEFAULT（忽略大小写）      | LOWER(firstname) = LOWER(?0)                |
| EXACT（不忽略大小写）      | firstname = ?0                              |
| EXACT（忽略大小写）        | LOWER(firstname) = LOWER(?0)                |
| STARTING（不忽略大小写）   | fristname like ?0 + '%'                     |
| STARTING（忽略大小写）     | LOWER(fristname) like LOWER(?0) + '%'       |
| ENDING（不忽略大小写）     | firstname like '%' + ?0                     |
| ENDING（忽略大小写）       | LOWER(firstname) like '%' + LOWER(?0)       |
| CONTAINING（不忽略大小写） | firstname like '%' + ?0 + '%'               |
| CONTAINING（忽略大小写）   | LOWER(firstname) like '%' + LOWER(?0) + '%' |

**2.4.3操作方法**

在ExampleMatcher中定义了一系列方式，用于设置这5项设置值，所有的设置方法均返回ExampleMatcher对象，所以支持链式编程配置。

（1）创建一个默认的ExampleMatcher对象。

定义：

```java
public static ExampleMatcher matching()
```

默认配置如下：

A、nullHandler：IGNORE。Null值默认处理方式：忽略

B、defaultStringMatcher：DEFAULT。默认字符串匹配方式：默认（相等）

C、defaultIgnoreCase：false。默认大小写忽略方式：不忽略

D、propertySpecifuers：空。各属性特定查询方式，空

E、ignoredPaths：空列表。忽略属性列表，空列表

（2）改变Null值处理方式。

定义：

```java
public ExampleMatcher withNullHandler(NullHandler nullHandler)
public ExampleMatcher withIncludeNullValues()
public ExampleMatcher withIgnoreNullValues()
```

产生效果：

改变配置项nullHandler，分别设为：指定值、INCLUDE（包括）、IGNORE（忽略）。

（3）改变默认字符串匹配方式。

定义：

```java
public ExampleMatcher withStringMatcher(StringMatcher defaultStringMatcher)
```

产生效果：

改变配置项defaultStringMatcher，设为指定值。

（4）改变默认大小写忽略方式。

定义：

```java
public ExampleMatcher withIgnoreCase()
public ExampleMatcher withIgnoreCase(boolean defaultIgnoreCase)
```

产生效果：

改变配置项defaultIgnoreCase，分别设为：true，指定值。

（5）向“忽略属性列表”中添加属性。

定义：

```java
public ExampleMatcher withIgnorePaths(String... ignoredPaths)
```

产生效果：

改变配置项ignoredPaths，向列表中添加一个或多个属性。

（6）配置属性特定查询方式

一个属性的特定查询方式，包含了3个信息：字符串匹配方式、大小写忽略方式、属性转换器，存储在`propertySpecifiers`中，操作时用`GenericPropertyMatcher`类来传递配置信息。有4个方法来改变配置，这4个方法操作时，内部均采用增量改变的方式，即如果没有为属性定义“特定查询方式”，则会定义一个，并根据传进来的“非空信息”进行配置，如果已经定义有，则会根据传进来的“非空信息”进行更新。如果一个“特定查询方式”中的“字符串匹配方式、大小写忽略方式”没有设置值，查询时则采用ExampleMatcher中的默认配置。

A、自定义类的方式。

定义：

```java
public ExampleMatcher withMatcher(String propertyPath, MatcherConfigurer<GenericPropertyMatcher> matcherConfigurer)
```

产生效果：

向`propertySpecifiers`中增加或更新属性“特定查询方式”的配置。

参数说明：
`propertyPath`：要配置特定查询的属性名。
`matcherConfigurer`：自定义类对象。自定义类需要实现MatcherConfigurer接口，在接口的configureMatcher() 实现方法中指定相关配置。

B、直接传入通用属性查询对象方式。

定义：
public ExampleMatcher withMatcher(String propertyPath, GenericPropertyMatcher genericPropertyMatcher)

产生效果：

向 propertySpecifiers 中增加或更新属性“特定查询方式”的配置。

参数说明：
propertyPath：要配置特定查询的属性名。
genericPropertyMatcher：直接传入一个通用查询对象。 ExampleMatcher.GenericPropertyMatchers工具类中提供了常用对象创建的静态方法，所有方法均返回 GenericPropertyMatcher 对象，所以支持链式编程配置。

GenericPropertyMatchers工具类方法

| 方法名\|效果           | 字符串匹配方式 | 大小写忽略方式 | 属性转换器 |
| ---------------------- | -------------- | -------------- | ---------- |
| storeDefaultMatching() | DEFAULT        |                |            |
| exact()                | EXACT          |                |            |
| startsWith()           | STARTING       |                |            |
| endsWith()             | ENDING         |                |            |
| contains()             | CONTAINING     |                |            |
| regex()                | REGEX          |                |            |
| ignoreCase()           |                | true           |            |
| caseSensitive()        |                | false          |            |

另外：GenericPropertyMatcher 类本身也提供了诸多方法，用于改变相关配置项。

C、改变的大小写忽略方式

定义：

```java
public ExampleMatcher withIgnoreCase(String... propertyPaths)
```

产生效果：
向 propertySpecifiers 中增加或更新属性“特定查询方式”中的“大小写忽略方式”配置。

D、设置属性转换器

定义：

```java
public ExampleMatcher withTransformer(String propertyPath, PropertyValueTransformer propertyValueTransformer)
```

产生效果：
向 propertySpecifiers 中增加或更新属性“特定查询方式”中的“属性转换器”配置。

#### 2.5 QueryByExampleExecutor使用场景和考虑因素

* 1.使用场景

  * 使用一组静态或动态约束来查询数据存储、频繁重构域对象，而不用担心破坏现有查询、简单的查询的使用场景，有时候还是挺方便的。

* 2.实际使用中我们需要考虑的因素

  查询条件的表示，有两部分，一是条件值，二是查询方式。条件值用实体对象（如Customer对象）来存储，相对简单，当页面传入过滤条件值时，存入相对应的属性中，没有传入时，属性保持默认值。查询方式是用匹配器ExampleMatcher来表示，情况相对复杂些，需要考虑的因素有以下几个：

  * （1）Null值的处理

    当某个条件值为Null时，是应当忽略这个过滤条件呢，还是应当去匹配数据库中该字段值是Null的记录？

    Null值处理方式：默认值是IGNORE（忽略），即当条件值为Null时，则忽略此过滤条件，一般业务也是采用这种方式就可满足。当需要查询数据库表中属性为Null的记录时，可将值设为INCLUDE，这时，对于不需要参与查询的属性，都必须添加到忽略列表（ignoredPaths）中，否则会出现查不到数据的情况。

  * （2）基本类型的处理

    如客户Customer对象中的年龄age是int型的，当页面不传入条件值时，它默认是0，那是否参与查询呢？

    关于基本数据类型处理方式：实体对象中，避免使用基本数据类型，采用包装器类型。如果已经采用了基本数据类型，而这个属性查询时不需要进行过滤，则把它添加到忽略列表（ignoredPaths）中。

  * （3）忽略某些属性值

    一个实体对象，有许多个属性，是否每个属性都参与过滤？是否可以忽略某些属性？

    ignoredPaths：虽然某些字段里面有值或者设置了其他匹配规则，只要放在ignoredPaths中，就会忽略此字段的，不作为过滤条件。

  * （4）不同的过滤方式

    同样是作为String值，可能“姓名”希望精准匹配，“地址”希望模糊匹配，如何做到？

    默认配置和特殊配置混合使用：默认创建匹配器时，字符串采用的是精准匹配、不忽略大小写，可以通过操作方法改变这种默认匹配，以满足大多数查询条件的需要，如将“字符串匹配方式”改为CONTAINING（包含，模糊匹配），这是比较常用的情况。对于个别属性需要特定的查询方式，可以通过配置“属性特定查询方式”来满足要求，设置propertySpecifiers的值即可。

  * （5）大小写匹配

    字符串匹配时，有时可能希望忽略大小写，有时则不忽略，如何做到？

    defaultIgnoreCase：忽略大小写的生效与否，是依赖于数据库的。例如MySQL数据库中，默认创建爱你表结构时，字段是已经忽略大小写的，所以这个配置与否，都是忽略的。如果业务要求严格区分大小写，可以改变数据库表结构属性来实现。

#### 2.6 常用查询实例

首先准备两个实体类和一些模拟数据：

有 客户信息、客户类型 两个实体类定义如下：

```java
/**
* 客户
*/
@Entity
@Table(name = "demo_lx_Customer")
public class Customer extends BaseBo
{
　　private String name; //姓名
　　private String sex; //性别
　　private int age; //年龄
　　private String address; //地址
　　private boolean focus ; //是否重点关注
　　private Date addTime; //创建时间
　　private String remark; //备注
　　@ManyToOne
　　private CustomerType customerType; //客户类型
　　......
}
```

```java
/**
* 客户类型
*/
@Entity
@Table(name = "demo_lx_CustomerType")
public class CustomerType extends BaseBo
{ 
　　private String code; //编号
　　private String name; //名称
　　private String remark; //备注
　　......
}
```

模拟数据如下：

![](.\img\jpa模拟数据.png)

**1.无匹配器的情况**

要求：查询地址是“河南省郑州市”，且重点关注的客户。
说明：对于默认匹配器满足条件时，则不需要创建匹配器。

```java
//创建查询条件数据对象
Customer customer = new Customer();
customer.setAddress("河南省郑州市");
customer.setFocus(true);
        
//创建实例
Example<Customer> ex = Example.of(customer); 
        
//查询
List<Customer> ls = dao.findAll(ex);
        
//输出结果
System.out.println("数量："+ls.size());
for (Customer bo:ls){
	System.out.println(bo.getName());
}
```

输出结果：

```
数量：4
李明
刘芳
zhang ming
ZHANG SAN
```

**2.通用情况**

要求：根据姓名、地址、备注进行模糊查询，忽略大小写，地址要求开始匹配。
说明：这是通用情况，主要演示改变默认字符串匹配方式、改变默认大小写忽略方式、属性特定查询方式配置、忽略属性列表配置。

```java
//创建查询条件数据对象
Customer customer = new Customer();
customer.setName("zhang");
customer.setAddress("河南省");
customer.setRemark("BB");

//创建匹配器，即如何使用查询条件
ExampleMatcher matcher = ExampleMatcher.matching() //构建对象
	.withStringMatcher(StringMatcher.CONTAINING) //改变默认字符串匹配方式：模糊查询
	.withIgnoreCase(true) //改变默认大小写忽略方式：忽略大小写
	.withMatcher("address", GenericPropertyMatchers.startsWith()) //地址采用“开始匹配”的方式查询
	.withIgnorePaths("focus");  //忽略属性：是否关注。因为是基本类型，需要忽略掉
        
//创建实例
Example<Customer> ex = Example.of(customer, matcher); 
        
//查询
List<Customer> ls = dao.findAll(ex);
        
//输出结果
System.out.println("数量："+ls.size());
for (Customer bo:ls){
	System.out.println(bo.getName());
}
```

输出结果：

```
数量：2
zhang ming
ZHANG SAN
```

**3.多级查询**
要求：查询所有潜在客户
说明：主要演示多层级属性查询

```java
//创建查询条件数据对象
CustomerType type = new CustomerType();
type.setCode("01"); //编号01代表潜在客户
Customer customer = new Customer();
customer.setCustomerType(type);        

//创建匹配器，即如何使用查询条件
ExampleMatcher matcher = ExampleMatcher.matching() //构建对象
	.withIgnorePaths("focus");  //忽略属性：是否关注。因为是基本类型，需要忽略掉                
        
//创建实例
Example<Customer> ex = Example.of(customer, matcher); 
        
//查询
List<Customer> ls = dao.findAll(ex);
        
//输出结果
System.out.println("数量："+ls.size());
for (Customer bo:ls){
	System.out.println(bo.getName());
}
```

输出结果：

```
数量：4
李明
李莉
张强
ZHANG SAN
```

**4.查询Null值**
要求：地址是null的客户
说明：主要演示改变“Null值处理方式”

```java
//创建查询条件数据对象
Customer customer = new Customer();

//创建匹配器，即如何使用查询条件
ExampleMatcher matcher = ExampleMatcher.matching() //构建对象
	.withIncludeNullValues() //改变“Null值处理方式”：包括
	.withIgnorePaths("id","name","sex","age","focus","addTime","remark","customerType");  //忽略其他属性
        
//创建实例
Example<Customer> ex = Example.of(customer, matcher); 
        
//查询
List<Customer> ls = dao.findAll(ex);
        
//输出结果
System.out.println("数量："+ls.size());
for (Customer bo:ls){
	System.out.println(bo.getName());
}
```

输出结果：

```
数量：2
张强
刘明
```

#### 2.7 使用Lambda简化ExampleMatcher例子

原写法：

```java
User user = new User();
user.setUsername("y");
user.setAddress("sh");
user.setPassword("admin");
ExampleMatcher matcher = ExampleMatcher.matching()
	.withMatcher("username", ExampleMatcher.GenericPropertyMatchers.startsWith())//模糊查询匹配开头，即{username}%
	.withMatcher("address" ,ExampleMatcher.GenericPropertyMatchers.contains())//全部模糊查询，即%{address}%
	.withIgnorePaths("password");//忽略字段，即不管password是什么值都不加入查询条件
Example<User> example = Example.of(user ,matcher);
List<User> list = userRepository.findAll(example);
System.out.println(list);
```

可使用Lambda简化为：

```java
ExampleMatcher matcher = ExampleMatcher.matching()
	.withMatcher("username", match -> match.startsWith())//模糊查询匹配开头，即{username}%
	.withMatcher("address" ,match -> match.contains())//全部模糊查询，即%{address}%
	.withIgnorePaths("password");//忽略字段，即不管password是什么值都不加入查询条件
```



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>