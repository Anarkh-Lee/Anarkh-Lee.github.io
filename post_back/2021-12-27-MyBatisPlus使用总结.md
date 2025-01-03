---
layout: post
title: 'MyBatisPlus使用总结'
date: 2021-12-27
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: MyBatisPlus














---

> MyBatisPlus使用过程中遇到的一些坑

### 1.mapper中无法识别String(比如`<if test="todoflag == '1'">`无法识别1)

调用：

```java
videoCaseDAO.queryMycaseListNew(“18888888888”,“03”,“1”);
```

mapper:

```xml
<select id="queryMycaseListNew" parameterType="String" resultType="com.neusoft.lemis.video.api.dto.VideoCaseDTO">
		select
		  b.ROOM_ID,
		  b.ROOM_NAME,
		  (SELECT LISTAGG(t.user_name,'、')  WITHIN GROUP (ORDER BY t.user_name) from VIDEO_USER t where t.room_id = b.room_id) DSR,
		  c.APPLICANT,
		  c.respondent,
		  c.STAFF,
		  c.VIDEOTYPE,
		  b.START_TIME,
		  b.END_TIME,
		  b.SOURCE,
		  c.EVENT_CONTENT
		from VIDEO_USER a, VIDEO_ROOM b, VIDEO_CASE c
		where a.MOBILE_PHONE = #{mobile_phone,jdbcType=VARCHAR}
		  and a.room_id = b.room_id and a.room_id = c.room_id
			<if test="todoflag == '1'">
				and b.END_TIME is null
			</if>
			<choose>
				<when test="videotype == '03'">
					and c.videotype = '03'
				</when>
				<otherwise>
					and c.videotype in ('01','02')
				</otherwise>
			</choose>
			order by b.START_TIME desc
	</select>
```

调用中todoflag传入值为“1”，如果写成`<if test="todoflag == '1'">`则无法识别1（如果传入的是01就可以），需要写成以下几种形式才可以：

```xml
//写法1
<if test="todoflag == '1'.toString()">
//写法2
<if test="todoflag == 1">
//写法3
<if test=‘todoflag == “1”’>
```



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>