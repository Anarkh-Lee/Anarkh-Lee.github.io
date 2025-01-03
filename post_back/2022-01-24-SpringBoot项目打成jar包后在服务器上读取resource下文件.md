---
layout: post
title: 'SpringBoot项目打成jar包后在服务器上读取resource下文件'
date: 2022-01-24
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Problem




---

> SpringBoot项目读取resource下文件，使用File路径读取只在本地好用，打成jar包后部署在服务器上，报空指针找不到文件；必须使用流的方式读取才能保证本地开发环境与部署jar环境都好用

最近在使用poi-tl导出word的一个功能中，需要对word模板进行读取，遂将word模板文件放到resource下，使用下面方法进行读取：

```java
String path = ClassUtils.getDefaultClassLoader().getResource("").getPath();
        XWPFTemplate template = XWPFTemplate.compile(path+"/file/bweam.docx",config)
                .render(data);
        try (OutputStream os = response.getOutputStream();
             BufferedOutputStream bos = new BufferedOutputStream(os);){
            template.write(bos);
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            PoitlIOUtils.closeQuietlyMulti(template);
        }
```

这种写法在本地运行没什么问题，但是达成jar包放到服务器上就读不到这个文件了。应该采用以下方式进行编写：

```java
ClassPathResource classPathResource = new ClassPathResource("file/bweam.docx");
        InputStream inputStream = null;
        try {
            inputStream = classPathResource.getInputStream();
        } catch (IOException e) {
            e.printStackTrace();
        }
        XWPFTemplate template = XWPFTemplate.compile(inputStream,config)
                .render(data);

        try (OutputStream os = response.getOutputStream();
             BufferedOutputStream bos = new BufferedOutputStream(os);){
            template.write(bos);
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            PoitlIOUtils.closeQuietlyMulti(template);
        }
```

注：

不管是用ClassPathResource还是ClassLoader，读取jar里面的文件，我们只能用流去读取，不能用file，文件肯定要牵扯路径。

```java
BufferedReader in = new BufferedReader(new InputStreamReader(AuditConstant.class.getClassLoader().getResourceAsStream("xxx")));
StringBuffer buffer = new StringBuffer();
String line = "";
while ((line = in.readLine()) != null){
    buffer.append(line);
}
```



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>