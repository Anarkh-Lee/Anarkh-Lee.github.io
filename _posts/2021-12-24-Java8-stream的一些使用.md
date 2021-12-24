---
layout: post
title: 'Java8 stream的一些使用'
date: 2021-12-24
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: 轮子













---

> 实战过程中Java8 stream的一些使用

### 1.根据对象（比如list）中某个属性去重（java stream distinct() 按指定对象属性进行去重）

#### 场景：

list中有5条数据如下：

```json
[{
	name = '沈阳市',
	value = '1001',
	parentValue = '1000',
	num = '1'
}, {
	name = '大连市',
	value = '1002',
	parentValue = '1000',
	num = '2'
}, {
	name = '鞍山市',
	value = '1003',
	parentValue = '1000',
	num = '3'
}, {
	name = '长春市',
	value = '2001',
	parentValue = '2000',
	num = '4'
}, {
	name = '吉林市',
	value = '2002',
	parentValue = '2000',
	num = '5'
}]
```

假如，parentValue='1000'和‘2000’分别代表辽宁和吉林，现在想要晒出这两个省的第一个城市。

列出例子Area.class

```java
class Area{
    private String name;
    private String value;
    private String parentValue;
    private String num;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }

    public String getParentValue() {
        return parentValue;
    }

    public void setParentValue(String parentValue) {
        this.parentValue = parentValue;
    }

    public String getNum() {
        return num;
    }

    public void setNum(String num) {
        this.num = num;
    }

    @Override
    public String toString() {
        return "Area{" +
                "name='" + name + '\'' +
                ", value='" + value + '\'' +
                ", parentValue='" + parentValue + '\'' +
                ", num='" + num + '\'' +
                '}';
    }
}
```

#### 方法1：

distinct（）不提供按照属性对对象列表进行去重的直接实现。它是基于hashCode（）和equals（）工作的。如果我们想要按照对象的属性，对对象列表进行去重，我们可以通过其它方法来实现。

```java
private static <T> Predicate<T> distinctByKey(Function<? super T, ?> keyExtractor) {
	Map<Object, Boolean> seen = new ConcurrentHashMap<>();
	return t -> seen.putIfAbsent(keyExtractor.apply(t), Boolean.TRUE) == null;
}
//调用
List<Area> parentList = list.stream().filter(distinctByKey(Area::getParentValue)).collect(Collectors.toList());

```

例子代码：

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Function;
import java.util.function.Predicate;
import java.util.stream.Collectors;

public class TestMain {
    public static void main(String[] args) {
        Area area1 = new Area();
        area1.setName("沈阳市");
        area1.setValue("1001");
        area1.setParentValue("1000");
        area1.setNum("1");

        Area area2 = new Area();
        area2.setName("大连市");
        area2.setValue("1002");
        area2.setParentValue("1000");
        area2.setNum("2");

        Area area3 = new Area();
        area3.setName("鞍山市");
        area3.setValue("1003");
        area3.setParentValue("1000");
        area3.setNum("3");

        Area area4 = new Area();
        area4.setName("长春市");
        area4.setValue("2001");
        area4.setParentValue("2000");
        area4.setNum("4");

        Area area5 = new Area();
        area5.setName("吉林市");
        area5.setValue("2002");
        area5.setParentValue("2000");
        area5.setNum("5");

        List<Area> list = new ArrayList<>();
        list.add(area1);
        list.add(area2);
        list.add(area3);
        list.add(area4);
        list.add(area5);
        System.out.println("筛选前");
        System.out.println(list);

        List<Area> parentList = list.stream().filter(distinctByKey(Area::getParentValue)).collect(Collectors.toList());
        System.out.println("筛选后");
        System.out.println(parentList);


    }
    private static <T> Predicate<T> distinctByKey(Function<? super T, ?> keyExtractor) {
        Map<Object, Boolean> seen = new ConcurrentHashMap<>();
        return t -> seen.putIfAbsent(keyExtractor.apply(t), Boolean.TRUE) == null;
    }
}
```

输出：

```
筛选前
[Area{name='沈阳市', value='1001', parentValue='1000', num='1'}, Area{name='大连市', value='1002', parentValue='1000', num='2'}, Area{name='鞍山市', value='1003', parentValue='1000', num='3'}, Area{name='长春市', value='2001', parentValue='2000', num='4'}, Area{name='吉林市', value='2002', parentValue='2000', num='5'}]
筛选后
[Area{name='沈阳市', value='1001', parentValue='1000', num='1'}, Area{name='长春市', value='2001', parentValue='2000', num='4'}]

```

#### 方法2：

使用stream流的衍生功能：

```java
ArrayList<Area> parentList = list.stream().collect(Collectors.collectingAndThen(
                Collectors.toCollection(() -> new TreeSet<>(
                        Comparator.comparing(
                                Area::getParentValue))), ArrayList::new));
System.out.println("筛选后");
System.out.println(parentList);
```

输出：

```
筛选前
[Area{name='沈阳市', value='1001', parentValue='1000', num='1'}, Area{name='大连市', value='1002', parentValue='1000', num='2'}, Area{name='鞍山市', value='1003', parentValue='1000', num='3'}, Area{name='长春市', value='2001', parentValue='2000', num='4'}, Area{name='吉林市', value='2002', parentValue='2000', num='5'}]
筛选后
[Area{name='沈阳市', value='1001', parentValue='1000', num='1'}, Area{name='长春市', value='2001', parentValue='2000', num='4'}]

```

如果是多个属性字段或多个条件去重：

```java
ArrayList<PatentDto> collect1 = patentDtoList.stream().collect(Collectors.collectingAndThen(
                Collectors.toCollection(() -> new TreeSet<>(
                        Comparator.comparing(p->p.getPatentName() + ";" + p.getLevel()))), ArrayList::new);
```









<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>