---
layout: post
title: 'Oracle中WM_CONCAT函数的替代写法'
date: 2020-06-30
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Oracle




---

> Oracle数据库12c之后不支持WM_CONCAT函数，工作过程中使用到了，需要替代写法。

**1.如果不考虑去重的话，以下两种方式等同**

```plsql
select TO_CHAR(wm_concat(AAC003)) from TF10_WEB order by AAC003;
```

![](.\img\wm_concat1.png)

```plsql
select listagg(t.AAC003,',') within group(order by t.AAC003) from TF10_WEB t;
```

![](.\img\wm_concat2.png)

**2.工作中的一个视图改写**

原视图：

```plsql
create or replace view v_vp01_cp04_unit as
SELECT AAB001, AWM0A3,ACP017,AWM011,AAB004_S, TO_CHAR(WM_CONCAT(DISTINCT(AAC003))) XM
  FROM NNJY.V_CP01_CP04 T
 GROUP BY AAB001, AWM0A3,ACP017,AWM011,AAB004_S;
 comment on column v_vp01_cp04_unit.AAB001 is '单位编号';
 comment on column v_vp01_cp04_unit.AWM0A3 is '业务表主键';
 comment on column v_vp01_cp04_unit.ACP017 is '表WM0A中AWM0A0字段';
 comment on column v_vp01_cp04_unit.AWM011 is '业务类别';
 comment on column v_vp01_cp04_unit.AAB004_S is '申报单位名称';
 comment on column v_vp01_cp04_unit.XM is '已分配专家';
```



改写后：

```plsql
CREATE OR REPLACE VIEW V_VP01_CP04_UNIT AS
SELECT AAB001, AWM0A3,ACP017,AWM011,AAB004_S,f_get_pro(t.aab001,t.AWM0A3) XM
  FROM NNJY.V_CP01_CP04 T
 GROUP BY AAB001, AWM0A3,ACP017,AWM011,AAB004_S;
comment on column V_VP01_CP04_UNIT.AAB001 is '单位编号';
comment on column V_VP01_CP04_UNIT.AWM0A3 is '业务表主键';
comment on column V_VP01_CP04_UNIT.ACP017 is '表WM0A中AWM0A0字段';
comment on column V_VP01_CP04_UNIT.AWM011 is '业务类别';
comment on column V_VP01_CP04_UNIT.AAB004_S is '申报单位名称';
comment on column V_VP01_CP04_UNIT.XM is '已分配专家';
```

```plsql
create or replace function f_get_pro(p_AAB001 in varchar2, P_AWM0A3 in varchar2)
  return varchar2 is FunctionResult varchar(500);
begin
for item in (SELECT distinct(aac003) FROM NNJY.V_CP01_CP04 T where aab001 = p_AAB001 and AWM0A3 = P_AWM0A3) loop 
  FunctionResult := FunctionResult || ',' || item.aac003;
end loop; 
   FunctionResult := LTRIM(FunctionResult,',');
return(FunctionResult);
end f_get_pro;

```

<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>