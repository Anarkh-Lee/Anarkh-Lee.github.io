---
layout: post
title: 'Browser对象（Window对象、Navigator对象、Screen对象、History对象、Location对象、存储对象）'
date: 2021-03-24
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: JavaScript




---

> Browser对象中Window对象、Navigator对象、Screen对象、History对象、Location对象、存储对象属性与方法介绍与使用

### Browser对象

* 1.概念：

  Browser Object Model浏览器对象模型：将浏览器的各个组成部分封装成对象。

  ![](.\img\JavaScript\1.png)

* 2.组成：
  * Window：窗口对象
  * Navigator：浏览器对象（不常用）
  * Screen：显示器屏幕对象（不常用）
  * History：历史记录对象
  * Location：地址栏对象
  * 存储对象

#### 1.Window 对象



#### 2.Navigator 对象



#### 3.Screen 对象



#### 4.History 对象



#### 5.Location 对象

* Location对象
  * Location 对象包含有关当前 URL 的信息。
  * Location 对象是 window 对象的一部分，可通过 window.Location 属性对其进行访问。
  * 注意：没有应用于Location对象的公开标准，不过所有浏览器都支持该对象。

##### Location 对象属性

| 属性                       | 描述                          |
| -------------------------- | ----------------------------- |
| [hash](#（1）hash)         | 返回一个URL的锚部分           |
| [host](#（2）host)         | 返回一个URL的主机名和端口     |
| [hostname](#（3）hostname) | 返回URL的主机名               |
| [href](#（4）href)         | 返回完整的URL                 |
| [pathname](#（5）pathname) | 返回的URL路径名               |
| [port](#（6）port)         | 返回一个URL服务器使用的端口名 |
| [protocol](#（7）protocol) | 返回一个URL协议               |
| [search](#（8）search)     | 返回一个URL的查询部分         |

###### （1）hash

* 定义和用法：

  hash 属性是一个可读可写的字符串，该字符串是 URL 的锚部分（从 # 号开始的部分）。

* 语法：

  ```javascript
  location.hash
  ```

* 浏览器支持：

  所有主要浏览器都支持 hash 属性。

* 实例：

  返回一个 URL 的主要部分。假设当前的 URL 是 http://www.baidu.com/test.htm＃PART2：

  ```javascript
  document.write(location.hash);
  ```

  以上实例输出结果：

  ```
  #part2
  ```

###### （2）host

* 定义和用法：

  host 属性是一个可读可写的字符串，可设置或返回当前 URL 的主机名称和端口号。

* 语法：

  ```javascript
  location.host
  ```

* 浏览器支持：

  所有主要浏览器都支持 host 属性。

* 实例：

  返回当前URL的主机名和端口（比如当前页面为http://localhost:9090/）：

  ```html
  <script>
  document.write(location.host);
  </script>
  ```

  以上实例输出结果：

  ```
  localhost:9090
  ```

###### （3）hostname

* 定义和用法：

  hostname 属性是一个可读可写的字符串，可设置或返回当前 URL 的主机名。

* 语法：

  ```
  location.hostname
  ```

* 浏览器支持：

  所有主要浏览器都支持 hostname 属性。

* 实例：

  返回当前URL的主机名（比如当前页面为http://localhost:9090/）：

  ```html
  <script>
  document.write(location.hostname);
  </script>
  ```

  以上实例输出结果：

  ```
  localhost
  ```

###### （4）href

* 定义和用法：

  href 属性是一个可读可写的字符串，可设置或返回当前显示的文档的完整 URL。

* 语法：

  ```
  location.href
  ```

* 浏览器支持：

  所有主要浏览器都支持 href 属性。

* 实例：

  返回完整的URL（比如当前页面为https://www.runoob.com/jsref/prop-loc-href.html)：

  ```html
  <script>
  document.write(location.href);
  </script>
  ```

  以上实例输出结果：

  ```
  https://www.runoob.com/jsref/prop-loc-href.html
  ```

###### （5）pathname

* 定义和用法：

  pathname 属性是一个可读可写的字符串，可设置或返回当前 URL 的路径部分。

* 语法：

  ```
  location.pathname
  ```

* 浏览器支持：

  所有主要浏览器都支持 pathname 属性。

* 实例：

  返回当前URL的路径名（比如当前页面为https://www.runoob.com/jsref/prop-loc-pathname.html）：

  ```html
  <script>
  document.write(location.pathname);
  </script>
  ```

  以上实例输出结果：

  ```
  /jsref/prop-loc-pathname.html
  ```

###### （6）port

* 定义和用法：

  port 属性是一个可读可写的字符串，可设置或返回当前 URL 的端口部分。

  **注意：**如果端口号就是80（这是默认的端口号)，无需指定。

* 语法：

  ```
  location.port
  ```

* 浏览器支持：

  所有主要浏览器都支持 port 属性。

* 实例：

  返回当前URL的端口号（比如当前页面为http://localhost:9090/）：

  ```javascript
  <script>
  document.write(location.port);
  </script>
  ```

  以上实例输出结果:

  ```
  9090
  ```

###### （7）protocol

* 定义和用法：

  protocol 属性是一个可读可写的字符串，可设置或返回当前 URL 的协议。

* 语法：

  ```
  location.protocol
  ```

* 浏览器支持：

  所有主要浏览器都支持 protocol 属性。

* 实例：

  返回当前URL的协议部分（比如当前页面为https://www.runoob.com/jsref/prop-loc-pathname.html）：

  ```html
  <script>
  document.write(location.protocol);
  </script>
  ```

  以上实例输出结果:

  ```
  https:
  ```

###### （8）search

* 定义和用法：

  search 属性是一个可读可写的字符串，可设置或返回当前 URL 的查询部分（问号 ? 之后的部分）。

* 语法：

  ```
  location.search
  ```

* 浏览器支持：

  所有主要浏览器都支持 search 属性。

* 实例：

  返回URL的查询部分（比如当前页面为http://www.runoob.com/submit.htm?email=someone@ example.com）：

  ```html
  <script>
  document.write(location.search);
  </script>
  ```

  以上实例输出结果:

  ```
  ?email=someone@example.com
  ```


##### Location 对象方法

| 方法                         | 说明                   |
| ---------------------------- | ---------------------- |
| [assign](#（1）assign())()   | 载入一个新的文档       |
| [reload](#（2）reload())()   | 重新载入当前文档       |
| [replace](#（3）replace())() | 用新的文档替换当前文档 |

###### （1）assign()

* 定义和用法：

  assign()方法加载一个新的文档。

* 语法：

  ```
  location.assign(URL)
  ```

* 浏览器支持：

  所有主要浏览器都支持 assign() 方法。

* 实例：

  使用 assign() 来加载一个新的文档：

  ```html
  <!DOCTYPE html>
  <html>
  <head>
  <meta charset="utf-8">
  <title>菜鸟教程(runoob.com)</title>
  <script>
  function newDoc(){
      window.location.assign("http://www.runoob.com")
  }
  </script>
  </head>
  <body>
  
  <input type="button" value="载入新文档" onclick="newDoc()">
  
  </body>
  </html>
  ```

###### （2）reload()

* 定义和用法：

  reload()方法用于刷新当前文档。

  reload() 方法类似于你浏览器上的刷新页面按钮。

  如果把该方法的参数设置为 true，那么无论文档的最后修改日期是什么，它都会绕过缓存，从服务器上重新下载该文档。这与用户在单击浏览器的刷新按钮时按住 Shift 健的效果是完全一样。

* 语法：

  ```
  location.reload(forceGet)
  ```

* 参数：

  | 参数     | 类型    | 描述                                                         |
  | -------- | ------- | ------------------------------------------------------------ |
  | forceGet | Boolean | 可选。如果把该方法的参数设置为 true，那么无论文档的最后修改日期是什么，它都会绕过缓存，从服务器上重新下载该文档。 |

* 返回值：

  该方法没有返回值。

* 浏览器支持：

  所有主要浏览器都支持 reload() 方法。

* 实例：

  重新载入当前文档:

  ```
  location.reload();
  ```

###### （3）replace()

* 定义和用法：

  replace() 方法可用一个新文档取代当前文档。

* 语法：

  ```
  location.replace(newURL)
  ```

* 浏览器支持：

  所有主要浏览器都支持 replace() 方法。

* 实例：

  使用 replace() 方法来替换当前文档：

  ```html
  <!DOCTYPE html>
  <html>
  <head>
  <meta charset="utf-8">
  <title>菜鸟教程(runoob.com)</title>
  <script>
  function replaceDoc(){
      window.location.replace("https://www.runoob.com")
  }
  </script>
  </head>
  <body>
   
  <input type="button" value="载入新文档替换当前页面" onclick="replaceDoc()">
   
  </body>
  </html>
  ```

  













#### 6.存储对象







<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>