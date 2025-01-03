---
layout: post
title: 'HttpClient使用总结'
date: 2021-01-09
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: 网络传输
---

> HttpClient的简单介绍与基本使用

# 一、HttpClient功能简介

* 实现了所有HTTP 的方法（GET,POST,PUT ,HEAD,OPTIONS,TRACE ）
* 支持自动转向
* 支持HTTPS 协议
* 透明地穿过HTTP代理建立连接
* 通过CONNECT方法，利用通过建立穿过HTTP 代理的HTTPS连接
* 利用本地Java socket，透明地穿过SOCKS( 版本5 和4）代理建立连接
* 支持利用Basic、Digest 和NTLM 加密的认证
* 支持用于上传大文件的Multi-Part 表单POST方法
* 插件式安全socket 实现，易于使用第三方的解决方案
* 连接管理，支持多线程应用，支持设定单个主机总连接和最高连接数量,自动检测和关闭失效的连接
* 直接将请求信息流送到服务器的端口
* 直接读取从服务器的端口送出的应答信息
* 支持HTTP/1.0 中用KeepAlive 和HTTP/1.1 中用persistance 设置的持久连接
* 直接访问由服务器送出的应答代码和头部信息
* 可设置连接超时时间
* HttpMethods 实现Command Pattern ，以允许并行请求或高效连接复用
* 遵循the Apache Software License 协议，源码免费可得

# 二、环境搭建

## 1.HttpClient 3.1 所需的基本jar 包：

commons-httpclient-3.1.jar ，下载地址：
http://archive.apache.org/dist/httpcomponents/commons-httpclient/binary/ ；
commons-logging.jar ，下载地址：
http://commons.apache.org/logging/download_logging.cgi ；
commons-codec.jar ，下载地址：
http://commons.apache.org/codec/download_codec.cgi  ；

## 2.HttpClient 4 所需的基本jar 包：

下载地址：http://hc.apache.org/downloads.cgi ；
最新版本为4.1.2，且官方不再升级HttpClient3 。

# 三、HttpClient 3.x 基本功能的使用

## 1.使用HttpClient 需要以下6 个步骤：

* 创建HttpClient 的实例
* 创建某种连接方法的实例，在这里是GetMethod 。在GetMethod 的构造函数中传入待连接的地址
* 调用第一步中创建好的实例的execute 方法来执行第二步中创建好的method 实例
* 读response
* 释放连接。无论执行方法是否成功，都必须释放连接
* 对得到后的内容进行处理

## 2.使用Get 方式提交请求

* 2.1 创建HttpClient 实例。大部分情况下HttpClient 默认的构造函数已经足够使用。

  ```java
  HttpClient httpClient = new HttpClient();
  ```

* 2.2 创建GET方法的实例。在GET方法的构造函数中传入待连接的地址即可。

  ```java
  GetMethod getMethod = new GetMethod("http://www.baidu.com/");
  ```

* 2.3 调用实例httpClient 的executeMethod 方法来执行getMethod 。由于是执行在网络上的程序，在运行executeMethod 方法的时候，需要处理两个异常，分别是HttpException 和IOException。引起第一种异常的原因主要可能是在构造getMethod 的时候传入的协议不对，比如不小心将"http" 写成"htp" ，或者服务器端返回的内容不正常等，并且该异常发生是不可恢复的；第二种异常一般是由于网络原因引起的异常，对于这种异常（IOException），HttpClient 会根据你指定的恢复策略自动试着重新执行executeMethod方法。HttpClient 的恢复策略可以自定义（通过实现接口HttpMethodRetryHandler 来实现）。通过httpClient 的方法setParameter 设置你实现的恢复策略。executeMethod 返回值是一个整数，表示了执行该方法后服务器返回的状态码，该状态码能表示出该方法执行是否成功、需要认证或者页面发生了跳转（默认状态下GetMethod 的实例是自动处理跳转的）等。

  ```java
  // 设置成了默认的恢复策略，在发生异常时候将自动重试3 次，在这里你也可以设置成自定义的恢复策略
  getMethod.getParams().setParameter(HttpMethodParams.RETRY_HANDLER,
  new DefaultHttpMethodRetryHandler());
  // 执行getMethod
  int statusCode = client.executeMethod(getMethod);
  if (statusCode != HttpStatus.SC_OK) {
  System.err.println("Method failed: " + getMethod.getStatusLine());
  }
  ```

* 2.4 在返回的状态码正确后，即可取得内容。取得目标地址的内容有三种方法：

  * 第一种，`getResponseBody() `，该方法返回的是目标的二进制的byte 流；
  * 第二种，`getResponseBodyAsString() `，这个方法返回的是String 类型；
  * 第三种，`getResponseBodyAsStream()`，这个方法对于目标地址中有大量数据需要传输是最佳的。

  注意：使用``getResponseBody()`时，HttpClient有时会报`WARN  org.apache.commons.httpclient.HttpMethodBase - Going to buffer response body of large or unknown size. Using getResponseBodyAsStream instead is recommended.`警告。报WARN的条件是((contentLength == -1) || (contentLength > limit))，也就是说，或者是返回的HTTP头没有指定contentLength，或者是contentLength大于上限（默认是1M）。如果能确定返回结果的大小对程序没有显著影响，这个WARN就可以忽略，可以在日志的配置中把HttpClient的日志级别调到ERROR，不让它报出来。 当然，这个警告也是有意义的，HttpClient建议使用`InputStream getResponseBodyAsStream()`代替`byte[] getResponseBody()`。对于返回结果很大或无法预知的情况，就需要使用`InputStream getResponseBodyAsStream()`，避免`byte[] getResponseBody()`可能带来的内存的耗尽问题。

  在这里我们使用了最简单的getResponseBody 方法。

  ```java
  byte[] responseBody = method.getResponseBody();
  ```

* 2.5 释放连接。无论执行方法是否成功，都必须释放连接。

  ```java
  method.releaseConnection();
  ```

* 2.6 处理内容。

  ```java
  System.out.println(new String(responseBody));
  ```

## 3.使用Post 方式提交请求

* 3.1 使用post 的方式提交请求，步骤与get 方式类似，只是声明方法实例的时候，使用的是PostMethod 。

  ```java
  PostMethod postMethod = new PostMethod (url);
  ```

* 3.2 传递参数
  当我们需要使用post 方式提交表单时，可以使用类NameValuePair 表示表单中的域；该类的构造函数第一个参数是域名，第二参数是该域的值；将表单所有的值设置到PostMethod 中用方法setRequestBody。例如：

  ```java
  NameValuePair nameValuePair = new NameValuePair( "mob" , "1330227" );
  // 将参数放入post 方法中去
  post.setRequestBody( new NameValuePair[] { nameValuePair} );
  ```

  如果传递多个参数，可以使用逗号隔开，即可。

## 4.使用HttpClient3 遇到的一些问题及解决方法

* 4.1 字符编码问题。

  * 4.1.1 有时候，当我们传递的参数中含有中文的时候，使用httpclient 发送，web 服务器无法识别，如果返回的http 响应头中含有中文字符，也会出现乱码。解决的方法是修改httpclient 的参数。

    ```java
    httpClient.getParams().setHttpElementCharset("UTF-8");
    ```

  * 4.1.2 有时候，服务器返回的页面内容出现中文乱码，可以设置如下参数：

    ```java
    httpClient.getParams().setParameter(HttpMethodParams.HTTP_CONTENT_CHARSET,
    "UTF-8");
    ```

* 4.2 超时的设置

  * 4.2.1 连接超时设置

    ```java
    httpClient.getHttpConnectionManager().getParams().setConnectionTimeout(5000);
    ```

  * 4.2.2 读取信息超时设置

    ```java
    httpClient.getHttpConnectionManager().getParams().setSoTimeout(1000);
    ```

* 4.3 自定义恢复策略
  正如前文所说，当出现IOException 异常时，httpclient 可以根据我们指定的恢复策略自动试着重新执行executeMethod 方法。如果我们不想让httpclient 重新执行executeMethod(), 或是不想让它重新执行3 次，可以如下设置：

  ```java
  // 不重试
  getMethod.getParams().setParameter(HttpMethodParams.RETRY_HANDLER,new
  DefaultHttpMethodRetryHandler( 0,false ));
  // 重试1次
  getMethod.getParams().setParameter(HttpMethodParams.RETRY_HANDLER,
  new DefaultHttpMethodRetryHandler(1,true));
  ```

  我们也可以自定义恢复策略，通过实现HttpRequestRetryHandler 接口，执行自定义的行为。

* 4.4 重定向问题
  Httpclient 的get 的方法支持自动转向处理，而post 方法不支持。当然我们也可以取消掉httpclient 的自动转向处理，可以调用方法：

  ```java
  getMethod.setFollowRedirects(false);
  ```

  我们可以根据执行过http 方法后，根据得到的返回状态进行自定义处理。如果得到301代码，这时服务器返回的头信息中location 的值就是sendRedirect 转向的目标地址。可以通过以下方法读取新的url：

  ```java
  String newurl = method.getResponseHeader("location").getValue();
  ```

  一些常见的http 状态码及含义：

  ​	200	获取到页面. 	HttpStatus.SC_OK
  ​	301	永久移动. 	    HttpStatus.SC_MOVED_PERMANENTLY
  ​	302	临时移动. 	    HttpStatus.SC_MOVED_TEMPORARILY
  ​	303	See Other. 	 HttpStatus.SC_SEE_OTHER
  ​	307	临时重定向. 	HttpStatus.SC_TEMPORARY_REDIRECT

* 4.5 设置代理
  有些网站要浏览器才可以访问，但程序可以仿浏览器，主要是设置http 头。

  ```java
  // 设置代理服务器的ip 地址和端口
  httpClient.getHostConfiguration().setProxy(ip,port);
  // 设置http 头
  List headers = new ArrayList();
  headers.add(new Header("User-Agent",Mozilla/4.0 (compatible; MSIE 7.0; Windows NT
  5.1)"));
  httpClient.getHostConfiguration().getParams().setParameter("http.default-headers",
  headers);
  ```

# 四、HttpClient 4.x 基本功能的使用

## 1.使用Get 方式提交请求

* 1.1 创建httpclient 实例

  ```java
  HttpClient httpclient = new DefaultHttpClient();
  ```

* 1.2 构建get 方法实例

  ```java
  HttpGet httpget = new HttpGet("http://www.baidu.com/");
  ```

* 1.3 执行方法，获得响应

  ```java
  HttpResponse response = httpclient.execute(httpget);
  HttpEntity entity = response.getEntity();
  ```

* 1.4 处理得到的http 实体

  ```java
  System.out.println(EntityUtils.toString(entity));// 以String 形式获取内容
  EntityUtils. toByteArray(entity)// 以二进制byte 流形式
  entity.getContent()// 流的形式
  ```

* 1.5 释放连接资源（必须执行的）

  ```java
  httpClient.getConnectionManager().shutdown();
  ```

## 2.使用Post 方式提交请求

* 2.1 除了构建方法使用的是：

  ```java
  HttpPost httppost = new HttpPost("http://www.baidu.com/");
  ```

  其他的步骤与使用get 方式相同。

* 2.2 传递参数
  许多应用程序需要频繁模拟提交一个HTML 表单的过程，比如，为了来记录一个Web应用程序或提交输出数据。HttpClient 提供了特殊的实体类UrlEncodedFormEntity 来这个满足过程。

  ```java
  List<NameValuePair> formparams = new ArrayList<NameValuePair>();
  formparams.add(new BasicNameValuePair("param1", "value1"));
  formparams.add(new BasicNameValuePair("param2", "value2"));
  UrlEncodedFormEntity entity = new UrlEncodedFormEntity(formparams, "UTF-8");
  HttpPost httppost = new HttpPost("http://localhost/handler.do");
  httppost.setEntity(entity);
  ```

## 3.使用HttpClient4 遇到的一些问题及解决方法

* 3.1 字符编码

  * 3.1.1 参数中文乱码解决：
    Get 方式提交的乱码处理：

    * 对中文参数使用`URLEncoder.encode(src);`来编码；
    * 设置GetMethod 编码格式为utf-8 ：`get_method.addRequestHeader("Content-type" ,
      "text/html; charset=utf-8");`

    Post 方式提交的乱码处理：
    `UrlEncodedFormEntity entity = new UrlEncodedFormEntity(formparams, "UTF-8");`

  * 3.1.2 内容中文乱码处理：
    `EntityUtils.toString(entity,"utf-8");`

* 3.2 超时设置

  ```java
  httpClient.getParams().setIntParameter(CoreConnectionPNames.CONNECTION_TIMEOUT,5
  000);// 连接超时
  httpclient.getParams().setIntParameter(CoreConnectionPNames.SO_TIMEOUT,5000);/ 读取
  超时
  ```

* 3.3 自定义请求重试设置

## 4.附1：使用get 方式，获取百度首页内容

```java
public static String gestDemos() throws HttpException, IOException {
	String url = "http://www.baidu.com/" ;
	HttpClient httpClient = new HttpClient();
	// 创建方法实例
	HttpMethod get = new GetMethod(url);
	// 执行get 方法
	httpClient.executeMethod(get);
	// 获取返回的页面内容
	String content = get.getResponseBodyAsString();
	// 释放链接资源
	get.releaseConnection();
	return content;
}
```

## 5.附2：使用post 方式，提交参数，查询手机归属地

```java
public static String post() {
	HttpClient httpClient = new DefaultHttpClient();
	String url = "http://haoma.imobile.com.cn/index.php" ;
	List nameValuePair = new ArrayList();
	nameValuePair.add( new BasicNameValuePair( "mob" , "1330227" ));
	try {
		// 声明一个url-encoded 实体
		UrlEncodedFormEntity urlEntity = new
		UrlEncodedFormEntity(nameValuePair, "utf-8" );
		// 声明使用post 的方法提交请求
		HttpPost post = new HttpPost(url);
		post.setEntity(urlEntity);
		// 执行post 请求
		HttpResponse response = httpClient.execute(post);
		// 获取响应实体
		HttpEntity entity = response.getEntity();
		String content = "" ;
		if (entity != null ) {
			System. out .println( "======== 服务器返回内容======" );
			content = EntityUtils. toString (entity, "utf-8" );
		}
		return content;
	} catch (Exception e) {
		System. out .println( "------Exception-------" );
		e.printStackTrace();
		return null ;
	} finally {
		// 关闭连接资源
		httpClient.getConnectionManager().shutdown();
}
```



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>