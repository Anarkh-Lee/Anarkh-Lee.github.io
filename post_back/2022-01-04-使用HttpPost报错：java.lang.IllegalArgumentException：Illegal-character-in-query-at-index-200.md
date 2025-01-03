---
layout: post
title: '使用HttpPost报错：java.lang.IllegalArgumentException：Illegal character in query at index 200'
date: 2022-01-04
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: HttpClient















---

> 使用HttpPost报错问题

**描述：**

本地使用没有问题，打版到服务器上报错如下：

java.lang.IllegalArgumentException：Illegal character in query at index 200

使用HttpPost调用接口时遇到如下报错信息：

![](.\img\HTTP\1.png)

**报错代码：**

```java
public static String sendHttpPost(String httpUrl, Map<String, Object> maps) {
        HttpPost httpPost = new HttpPost(httpUrl);
        String jsonString = JSONObject.toJSONString(maps);
        try {
            httpPost.setHeader("Content-Type", "application/json;charset=UTF-8");//表示客户端发送给服务器端的数据格式
            httpPost.setHeader("Accept", "application/json");                    //表示服务端接口要返回给客户端的数据格式，
            StringEntity entity = new StringEntity(jsonString, ContentType.APPLICATION_JSON);
            httpPost.setEntity(entity);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return sendHttpPost(httpPost);
    }
```

**解决：**

地址中涉及了特殊字符，如‘｜’‘&’等。所以不能直接用String代替**URI**来访问。必须采用%0xXX方式来替代特殊字符。但这种办法不直观。所以只能先把String转成URL，再能过URL生成**URI**的方法来解决问题。

```java
public static String sendHttpPost(String httpUrl, Map<String, Object> maps) throws Exception {
        URL url = new URL(httpUrl);
        URI uri = new URI(url.getProtocol(), url.getHost(), url.getPath(), url.getQuery(), null);

        HttpPost httpPost = new HttpPost(uri);
        String jsonString = JSONObject.toJSONString(maps);
        try {
            httpPost.setHeader("Content-Type", "application/json;charset=UTF-8");//表示客户端发送给服务器端的数据格式
            httpPost.setHeader("Accept", "application/json");                    //表示服务端接口要返回给客户端的数据格式，
            StringEntity entity = new StringEntity(jsonString, ContentType.APPLICATION_JSON);
            httpPost.setEntity(entity);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return sendHttpPost(httpPost);
    }
```





<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>