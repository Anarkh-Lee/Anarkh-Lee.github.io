---
layout: post
title: 'Oracle中的BLOB和CLOB字段'
date: 2021-12-30
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Oracle




---

> Oracle中的BLOB和CLOB字段的一些说明

**一般为了更好的管理ORACLE数据库，通常像图片、文件、音乐等信息就用BLOB字段来存储，先将文件转为二进制再存储进去。而像文档或者是较长的文字，就用CLOB存储，这样对以后的查询更新存储等操作都提供很大的方便。**

### 1.BLOB 
BLOB全称为二进制大型对象（Binary Large Object)。它用于存储数据库中的大型二进制对象。可存储的最大大小为4G字节。应用：**通常像图片、文件、音乐等信息就用BLOB字段来存储，先将文件转为二进制再存储进去。**


### 2.CLOB 
CLOB全称为字符大型对象（Character Large Object)。它与LONG数据类型类似，只不过CLOB用于存储数据库中的大型单字节字符数据块，不支持宽度不等的字符集。可存储的最大大小为4G字节。**应用：而像文章或者是较长的文字，就用CLOB存储。CLOB什么时候使用：当某个字段，要保存的长度太长时，varchar2最大4000，但是如果需要的字段长度大于4000应该怎么办，这时候就用clob。**



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>