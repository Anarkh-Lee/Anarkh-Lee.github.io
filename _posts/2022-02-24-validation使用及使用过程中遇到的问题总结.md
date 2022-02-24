---
layout: post
title: 'validation使用及使用过程中遇到的问题总结'
date: 2022-02-24
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: 轮子






---

> 1.SpringBoot下集成validation及使用；2.自定义校验注解；3.使用过程中遇到的问题

### 1.SpringBoot下集成validation及使用

#### maven

```xml
<dependency>
    <groupId>javax.validation</groupId>
    <artifactId>validation-api</artifactId>
    <version>2.0.1.Final</version>
</dependency>
```

#### 定义工厂类使用

```java
private static ValidatorFactory factory = Validation.buildDefaultValidatorFactory();

public static <T> List<String> validate(T t) {
    Validator validator = factory.getValidator();
    Set<ConstraintViolation<T>> constraintViolations = validator.validate(t);

    List<String> messageList = new ArrayList<>();
    for (ConstraintViolation<T> constraintViolation : constraintViolations) {
        messageList.add(constraintViolation.getMessage());
    }
    return messageList;
}
```

```java
Entity entity = new Entity();
List<String> vallist = validate(entity);
vallist.forEach(row -> {
    System.out.println(row.toString());
});
```

#### validation注解

Bean Validation 中内置的 constraint:

| 注解                       | 作用                                                     |
| -------------------------- | -------------------------------------------------------- |
| @Null                      | 被注释的元素必须为 null                                  |
| @NotNull                   | 被注释的元素必须不为 null                                |
| @AssertTrue                | 被注释的元素必须为 true                                  |
| @AssertFalse               | 被注释的元素必须为 false                                 |
| @Min(value)                | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |
| @Max(value)                | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |
| @DecimalMin(value)         | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |
| @DecimalMax(value)         | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |
| @Size(max=, min=)          | 被注释的元素的大小必须在指定的范围内                     |
| @Digits(integer, fraction) | 被注释的元素必须是一个数字，其值必须在可接受的范围内     |
| @Past                      | 被注释的元素必须是一个过去的日期                         |
| @Future                    | 被注释的元素必须是一个将来的日期                         |
| @Pattern(regex=,flag=)     | 被注释的元素必须符合指定的正则表达式                     |

Hibernate Validator 附加的 constraint:

| 注解                       | 作用                                   |
| -------------------------- | -------------------------------------- |
| @NotBlank                  | 验证字符串非null，且长度必须大于0      |
| @Email                     | 被注释的元素必须是电子邮箱地址         |
| @Length(min=,max=)         | 被注释的字符串的大小必须在指定的范围内 |
| @NotEmpty                  | 被注释的字符串的必须非空               |
| @Range(min=,max=,message=) | 被注释的元素必须在合适的范围内         |

### 2.自定义校验注解

#### Java Bean 中属性长度校验

由于使用的是Oracle数据库，使用`select userenv('language') from dual;`查询数据库字符集使用的是：`AMERICAN_AMERICA.ZHS16GBK`，中文占2个字节，英文占1个字节。而已有的@Size注解无论中英文都是按照长度算的，如果使用@Size就会出现一个问题：如果某个字段（ID）数据库中为VARCHAR2(4)，输入4个中文汉字，validation校验是通过的，但是持久化到数据库中时，4个汉字对应的就是8个字节，就会超出长度。因此使用GBK字节对长度进行校验：

```java
package com.neusoft.lpleaf6.job.app.externalInterface.validation;

import com.neusoft.lpleaf6.job.app.externalInterface.annotation.ByteSize;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;
import java.io.UnsupportedEncodingException;

/**
 * @author li.ba
 * @desc Java bean 属性长度校验实现
 * @date 2022/02/23
 */

public class ByteSizeValidator implements ConstraintValidator<ByteSize, String> {

    private int max;

    @Override
    public void initialize(ByteSize byteSize) {
        this.max = byteSize.max();
    }

    @Override
    public boolean isValid(String o, ConstraintValidatorContext constraintValidatorContext) {
        try {
            return o == null || o.getBytes("GBK").length <= this.max;
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return false;
    }
}

```

```java
package com.neusoft.lpleaf6.job.app.externalInterface.annotation;

import com.neusoft.lpleaf6.job.app.externalInterface.validation.ByteSizeValidator;

import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.*;

/**
 * @author li.ba
 * @desc Java bean 属性长度校验注解
 * @date 2022/02/23
 */

@Documented
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = {ByteSizeValidator.class})
public @interface ByteSize {
    String message() default "代码项不合法";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    int max();
}

```

#### Java Bean 中属性数据字典校验

举个例子：某个代码项字段（SEX:1代表男，0代表女），只能限制SEX是0或者1。一般来说数据字典都会存储到数据库中，如果要校验大量的这种数据字典字段，挨个校验很费劲，遂采用将数据字典缓存到redis中，然后编写validation自定义校验将缓存中的数据字典取出，对代码项字段进行校验。

```java
package com.neusoft.lpleaf6.job.app.externalInterface.validation;

import com.alibaba.fastjson.JSONArray;
import com.neusoft.lpleaf6.common.api.dto.c.CodeListDto;
import com.neusoft.lpleaf6.job.app.externalInterface.annotation.DataDictionaryValidation;
import com.neusoft.lpleaf6.job.app.externalInterface.util.BeanUtil;
import org.hibernate.validator.internal.engine.constraintvalidation.ConstraintValidatorContextImpl;
import org.springframework.data.redis.core.StringRedisTemplate;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.List;

/**
 * @author li.ba
 * @desc Java bean 属性代码项校验实现
 * @date 2022/02/23
 */

public class DataDictionaryValidator implements ConstraintValidator<DataDictionaryValidation, Object> {

    private StringRedisTemplate stringRedisTemplate;

    @Override
    public void initialize(DataDictionaryValidation dataDictionaryValidation) {
        stringRedisTemplate = BeanUtil.getBean(StringRedisTemplate.class);
    }

    @Override
    public boolean isValid(Object o, ConstraintValidatorContext constraintValidatorContext) {

        Class clazz = ConstraintValidatorContextImpl.class;
        try {
            Field field = clazz.getDeclaredField("basePath");
            field.setAccessible(true);
            String aaa100 = field.get(constraintValidatorContext).toString();
            List<CodeListDto> list = null;
            String codelist = stringRedisTemplate.opsForValue().get("codelist:nmgswyt:"+aaa100.toLowerCase());
            if(codelist != null) {
                JSONArray jsonArray = JSONArray.parseArray(codelist);
                list=jsonArray.toJavaList(CodeListDto.class);
            }

            List<String> strList = new ArrayList<>();
            if(list == null){
                return false;
            }else {
                for (CodeListDto dto:list) {
                    String str = dto.getCODEVALUE();
                    strList.add(str);
                }

                if(strList.contains(o)){
                    return true;
                }else {
                    return false;
                }
            }


        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return false;
    }
}

```

```java
package com.neusoft.lpleaf6.job.app.externalInterface.annotation;

import com.neusoft.lpleaf6.job.app.externalInterface.validation.DataDictionaryValidator;

import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.*;

/**
 * @author li.ba
 * @desc Java bean 属性代码项校验注解
 * @date 2022/02/23
 */


@Documented
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = {DataDictionaryValidator.class})
public @interface DataDictionaryValidation {
    String message() default "代码项不合法";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}

```

### 3.自定义校验注解时遇到的问题：实现ConstraintValidator接口的类中注入为null

从上文中可以看到，在校验数据字典的自定义注解中，我们在实现类中使用StringRedisTemplate用于取出redis中的缓存，可以看到StringRedisTemplate并没有使用@Autowire直接注入，是因为直接注入的时候，由于此实现类并没有受Spring管理，遂注入的时候为null，在此加了@Componet注解也不好用，遂这种想了个办法，就是写了一个bean工具类，在实现类初始化的时候将StringRedisTemplate取出来赋值，这样就可以正常的使用StringRedisTemplate了。

附上BeanUtil代码：

```java
package com.neusoft.lpleaf6.job.app.externalInterface.util;

import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Service;

@Service
public class BeanUtil implements ApplicationContextAware {

    private static ApplicationContext context;

    @Override

    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {

        context = applicationContext;

    }

    public static <T> T getBean(Class<T> beanClass) {

        return context.getBean(beanClass);

    }

}
```

### 附录：使用注解

注解bean：

```java
package com.neusoft.lpleaf6.job.app.externalInterface.domain.business;

import com.neusoft.lpleaf6.job.app.externalInterface.annotation.ByteSize;
import com.neusoft.lpleaf6.job.app.externalInterface.annotation.DataDictionaryValidation;
import io.swagger.annotations.ApiModel;
import io.swagger.annotations.ApiModelProperty;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Pattern;

@ApiModel(value = "Cb33", description = "招聘会信息表-对外接口专用")
public class Cb33ExtDTO implements java.io.Serializable {
    private static final long serialVersionUID = 1L;
    @ApiModelProperty(value = "招聘会编号")
    @NotBlank(message = "acb330不能为空")
    @ByteSize(max = 32,message = "acb330长度超出范围")
    private String acb330; // 招聘会编号
    @ApiModelProperty(value = "是否外链招聘会")
    @NotBlank(message = "acb33b不能为空")
    @ByteSize(max = 3,message = "acb33b长度超出范围")
    @DataDictionaryValidation(message = "acb33b代码项非法")
    private String acb33b; // 是否外链招聘会
    @ApiModelProperty(value = "停止预订时间")
    @NotBlank(message = "acb33e不能为空")
    @ByteSize(max = 8,message = "acb33e长度超出范围")
    //校验格式yyyyMMdd
    @Pattern(regexp = "^([\\d]{4}((((0[13578]|1[02])((0[1-9])|([12][0-9])|(3[01])))|(((0[469])|11)((0[1-9])|([12][0-9])|30))|(02((0[1-9])|(1[0-9])|(2[0-8])))))|((((([02468][048])|([13579][26]))00)|([0-9]{2}(([02468][048])|([13579][26]))))(((0[13578]|1[02])((0[1-9])|([12][0-9])|(3[01])))|(((0[469])|11)((0[1-9])|([12][0-9])|30))|(02((0[1-9])|(1[0-9])|(2[0-9]))))){4})$"
            ,message = "acb33e非法的时间格式范围")
    private String acb33e; // 停止预订时间
   
    private String plstatus;
    
//    @JSONField(name="pl_status")
    public String getPlstatus() {
        return plstatus;
    }
//    @JSONField(name="pl_status")
    public void setPlstatus(String plstatus) {
        this.plstatus = plstatus;
    }

    public String getAcb330() {
        return acb330;
    }

    public void setAcb330(String acb330) {
        this.acb330 = acb330;
    }

    public String getAcb33b() {
        return acb33b;
    }

    public void setAcb33b(String acb33b) {
        this.acb33b = acb33b;
    }

    public String getAcb33e() {
        return acb33e;
    }

    public void setAcb33e(String acb33e) {
        this.acb33e = acb33e;
    }
}

```

调用：

```java
private static ValidatorFactory factory = Validation.buildDefaultValidatorFactory();

public static <T> List<String> validate(T t) {
        Validator validator = factory.getValidator();
        Set<ConstraintViolation<T>> constraintViolations = validator.validate(t);

        List<String> messageList = new ArrayList<>();
        for (ConstraintViolation<T> constraintViolation : constraintViolations) {
            messageList.add(constraintViolation.getMessage());
        }
        return messageList;
}

//调用
public void involeVali(){
	Cb33ExtDTO entity = new Cb33ExtDTO();
	entity.setAcb330("111");
	List<String> vallist = validate(entity);
	vallist.forEach(row -> {
   	  System.out.println(row.toString());
	});
}
```







<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>