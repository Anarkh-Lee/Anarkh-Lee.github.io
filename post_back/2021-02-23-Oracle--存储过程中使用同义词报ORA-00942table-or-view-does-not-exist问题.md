---
layout: post
title: 'Oracle--存储过程中使用同义词报ORA-00942:table or view does not exist问题'
date: 2021-02-23
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Oracle


---

> 同一实例下的两个Oracle用户，B用户中的存储过程使用了指向A用户实表的同义词，报ORA-00942:table or view does not exist问题解决

场景：同一实例下有两个Oracle用户：A用户和B用户。A用户有一实表：ZJ10。B用户下建立指向A用户实表ZJ10的同义词ZJ10，B用户下一存储过程中部分内容如下：

```sql
SELECT COUNT(1)
      INTO N_SUM
      FROM ZJ10
     WHERE AAB998 = PRM_AAB998
       AND ISDEL = '0'
       AND AAE100 = '1';
```

![](.\img\Oracle\2.png)

编译时报错：

```sql
ORA-00942:table or view does not exist
```

![](.\img\Oracle\1.png)

解决方案：

在A用户下给B用户授予表ZJ10的对应权限（此处授予读的权限即可，也可以授予全部权限）：

```sql
grant select on ZJ10 TO B;
--或
grant all on ZJ10 TO B;
```



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>