---
layout: post
title: 'Oracle自动生成实体sql'
date: 2020-07-14
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Oracle




---

> 可以使用此sql自动生成实体，DTO

```plsql
select rownum as ord,val from (select '@Column(name = "'||a.COLUMN_NAME||'", table = "'||a.TABLE_NAME||'")' as val
  from user_col_comments a,user_tab_columns b
where a.TABLE_NAME = 'TF01_WEB'
  and a.TABLE_NAME = b.TABLE_NAME
  and a.COLUMN_NAME = b.COLUMN_NAME
  ORDER BY TO_NUMBER(B.COLUMN_ID) ASC)
union
select rownum as ord,val from (select 'private '||case when b.DATA_TYPE='NUMBER' then 'BigDecimal' when b.DATA_TYPE='DATE' then 'Date' else 'String' end||' '||lower(a.COLUMN_NAME)||'; //'||a.COMMENTS as val
  from user_col_comments a,user_tab_columns b
where a.TABLE_NAME = 'TF01_WEB'
  and a.TABLE_NAME = b.TABLE_NAME
  and a.COLUMN_NAME = b.COLUMN_NAME
  ORDER BY TO_NUMBER(B.COLUMN_ID) ASC);
```



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>