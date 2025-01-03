---
layout: post
title: '使用HttpClient报Going to buffer response body of large or unknown size. Using getResponseBodyAsStream instead is recommended.警告处理'
date: 2021-01-10
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: 网络传输

---

> 使用HttpClient报Going to buffer response body of large or unknown size. Using getResponseBodyAsStream instead is recommended.警告处理

使用如下代码利用HttpClient获取服务器上附件时，报出警告：`WARN  org.apache.commons.httpclient.HttpMethodBase - Going to buffer response body of large or unknown size. Using getResponseBodyAsStream instead is recommended.`的WARN日志。

```java
public String fileToBase64(String fileSrc){
        String externalPath = "";
        String result = "";
        String[] typeArr = fileSrc.split("\\.");
        int num = typeArr.length-1;
        String type = typeArr[num];

        //通过http访问外网电子档案服务
        HttpClient httpClient = new HttpClient();
        httpClient.getHostConfiguration().setHost("127.0.0.1", 8080, "http");//外网dossier服务地址、端口、访问方式
        HttpMethod method;
        byte[] a = null;
        try {
            method = getMethod("/anarkh/dossier/upload/images/"+fileSrc);
            httpClient.executeMethod(method);
            a =  method.getResponseBody();
            BASE64Encoder encoder = new BASE64Encoder();
            if("jpg".equals(type)||"png".equals(type)||"bmp".equals(type)){
                result = encoder.encode(a) ;
            }
            method.releaseConnection();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
}
private static HttpMethod getMethod(String url) throws IOException {
        GetMethod get = new GetMethod(url);
        get.addRequestHeader("Accept","application/json;charset=UTF-8");
        get.releaseConnection();
        return get;
}    
    
```

定位到HttpClient的源码如下：

```java
public abstract class HttpMethodBase
    implements HttpMethod
{
...
    public byte[] getResponseBody() throws IOException {
        if (responseBody == null) {
            InputStream instream = getResponseBodyAsStream();
            if (instream != null) {
                long contentLength = getResponseContentLength();
                if (contentLength > 2147483647L){
                    throw new IOException("Content too large to be buffered: " + contentLength + " bytes");　　　　　　　　　　}
                int limit = getParams().getIntParameter("http.method.response.buffer.warnlimit", 1048576);
                if (contentLength == -1L || contentLength > (long) limit){
                    LOG.warn("Going to buffer response body of large or unknown size. Using getResponseBodyAsStream instead is recommended.");　　　　　　　　　　}
                LOG.debug("Buffering response body");
                ByteArrayOutputStream outstream = new ByteArrayOutputStream(contentLength <= 0L ? 4096: (int) contentLength);
                byte buffer[] = new byte[4096];
                int len;
                while ((len = instream.read(buffer)) > 0){　　　　　　　　　　　　outstream.write(buffer, 0, len);　　　　　　　　　　}
                outstream.close();
                setResponseStream(null);
                responseBody = outstream.toByteArray();
            }
        }
        return responseBody;
    }
...
}
```

报WARN的条件是((contentLength == -1) || (contentLength > limit))，也就是说，或者是返回的HTTP头没有指定contentLength，或者是contentLength大于上限（默认是1M）。如果能确定返回结果的大小对程序没有显著影响，这个WARN就可以忽略，可以在日志的配置中把HttpClient的日志级别调到ERROR，不让它报出来。

当然，这个警告也是有意义的，HttpClient建议使用InputStream getResponseBodyAsStream()代替byte[] getResponseBody()。对于返回结果很大或无法预知的情况，就需要使用InputStreamgetResponseBodyAsStream()，避免byte[] getResponseBody()可能带来的内存的耗尽问题。

修改代码如下解决问题：

```java
public String fileToBase64(String fileSrc){
        String externalPath = "";
        String result = "";
        String[] typeArr = fileSrc.split("\\.");
        int num = typeArr.length-1;
        String type = typeArr[num];
        //获取接口地址
        StringBuffer sb = new StringBuffer("select aaa004 from AA01 where aaa001 = 'SI_External_Dossier'");
        Query q = em.createNativeQuery(sb.toString());
        List list = q.getResultList();
        externalPath =  list.get(0).toString();//应该return
//        result = externalPath + fileSrc;
        String[] externalPathArray = externalPath.split(",");

        //通过http访问外网电子档案服务
        HttpClient httpClient = new HttpClient();
        httpClient.getHostConfiguration().setHost(externalPathArray[0], Integer.parseInt(externalPathArray[1]), externalPathArray[2]);//外网dossier服务地址、端口、访问方式
        HttpMethod method;
        byte[] buffer = new byte[1024*4];
        InputStream input = null;
        ByteArrayOutputStream output = new ByteArrayOutputStream();
        try {
            method = getMethod(externalPathArray[3]+fileSrc);
            httpClient.executeMethod(method);
            input = method.getResponseBodyAsStream();
            int n = 0;
            while (-1 != (n = input.read(buffer))) {
                output.write(buffer, 0, n);
            }
            BASE64Encoder encoder = new BASE64Encoder();
            if("jpg".equals(type)||"png".equals(type)||"bmp".equals(type)){
                result = encoder.encode(output.toByteArray()) ;
            }
            method.releaseConnection();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            try {
                if (output != null) {
                    output.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
            try {
                if (input != null) {
                    input.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return result;
}
private static HttpMethod getMethod(String url) throws IOException {
        GetMethod get = new GetMethod(url);
        get.addRequestHeader("Accept","application/json;charset=UTF-8");
        get.releaseConnection();
        return get;
}  
```



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>