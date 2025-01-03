---
layout: post
title: 'JVM如何保证static成员变量只实例化一次？'
date: 2021-08-30
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Java








---

> 在一个JVM中，static修饰的成员变量只会实例化一次

使用static修饰的成员变量，会在类初始化的过程中被收集进类构造器即`<clinit>`方法中，在多线程场景下，JVM会保证只有一个线程能执行该类的`<clinit>`方法，其他线程将会被阻塞等待，等到唯一的一次`<clinit>`方法执行完成，其他线程将不会再执行`<clinit>`方法，转而执行自己的代码。也就是说，static修饰的成员变量，在多线程的情况下能保证只实例化一次。

注：

`<init>`：是instance实例构造器，对非静态变量进行初始化。

`<clinit>`：是class类构造器，对静态变量、静态代码块进行初始化。











<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>