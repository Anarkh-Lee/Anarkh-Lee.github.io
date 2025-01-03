---
layout: post
title: 'HTML中表单校验（正则）的小总结（onkeyup、onafterpaste、ng-pattern）'
date: 2020-07-10
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: HTML



---

> 校验失败表单自动清空、校验信息及时显示在HTML页面上等。

##### 1、限制只能输入数字，非数字清空（粘贴非数字也清空）

```html
<div class="col-md-3" align="right">
	<label style="color: red;">获奖年度</label>
</div>
<div class="col-md-8">
	<input type="text" class="form-control" ng-model="tf0fadd.atf0f4" maxlength="4" 			onkeyup="this.value=this.value.replace(/[^\d]/g,'') " 						      		  onafterpaste="this.value=this.value.replace(/[^\d]/g,'') ">
</div>
```

```
注:
onafterpaste事件是指限制文本框只能输入数字 onkeyup+onafterpaste。
“onkeyup”是指按键抬起触发，“onafterpaste”是指粘贴后打开。
整体的意思是用于限制只能输入数字。
/[^\d]/g:全局查找非数字 str是否符合非数字
```

##### 2、在html上使用正则校验年度提醒

```html
<form name="infoForm" class="form-horizontal formCustomSetting normalForm">
	<div class="col-md-2">
		<label><font color="red">申报年度</font></label>
	</div>
	<div class="col-md-3">
		<input type="text" class="form-control" ng-model="tf50web.aae001"  name="aae001" 			ng-pattern="/^(19\d{2}|20\d{2}|2100)$/" maxlength="4" />
		<span style="color: #ce0000;" class="warning" ng-							    			show="infoForm.aae001.$error.pattern">年度格式如2018</span>
	</div>
</form>
```

效果：

![](.\img\HTMLcheck.gif)

```
注：
使用ng-pattern进行验证表单时（实际上使用ng进行表单验证时），需要给<form>、<input>等name属性赋值。
```

<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>