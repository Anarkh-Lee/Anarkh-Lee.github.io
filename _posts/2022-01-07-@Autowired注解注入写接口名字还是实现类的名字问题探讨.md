---
layout: post
title: '@Autowired注解注入写接口名字还是实现类的名字问题探讨'
date: 2022-01-07
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Problem
















---

> 在开发过程中使用@Autowired注解遇到问题产生的一些想法

### 问题

在开发过程中，别人使用了实现类注入，然后一直报错，找了好久原因，现用以下例子说明一下。

准备两个接口（AnarkhService、InvokeService）和两个实现类（AnarkhServiceImpl、InvokeServiceImpl），其中InvokeServiceImpl是要注入AnarkhServiceImpl或AnarkhService的类。

#### 第一种情况：只有自己的方法

AnarkhService：

```java
public interface AnarkhService {
	String testFun();
}
```

AnarkhServiceImpl：

```java
import java.sql.Timestamp;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.neusoft.agileggfw.dao.business.api.s.xmzjjg.b.BanklogDao;
import com.neusoft.agileggfw.entity.xmzjjg.api.support.b.Banklog;
import com.neusoft.agileggfw.service.xmzjjg.api.xmzjjg.service.AnarkhService;

@Service
public class AnarkhServiceImpl implements AnarkhService{

	@Autowired
	private BanklogDao banklogDao;
	
	@Override
	public String testFun() {
		return "sucess";
	}
}

```

InvokeService：

```java
public interface InvokeService {

}
```

InvokeServiceImpl：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.neusoft.agileggfw.service.xmzjjg.api.xmzjjg.service.InvokeService;

@Service
public class InvokeServiceImpl implements InvokeService{
	@Autowired
	private AnarkhServiceImpl anarkhServiceImpl;
}

```

此时，程序启动是ok的。

编写controller对testFun()方法进行调用（此时controller中是使用@Autowired将AnarkhService注入的）：

```java
@Controller
@RequestMapping(value="/a/a/")
public class WwaaAct {
	
	@Autowired
	private AnarkhService anarkhService;
	
	@ResponseBody
	@RequestMapping(value="report/anarkhTest.html", method = RequestMethod.POST, produces = { "text/html;charset=UTF-8" })
	public String anarkhTest(){
		anarkhService.testFun();
		return "sucess";
	}

}
```

**结果：程序启动sucess，调用sucess。**



#### 第二种情况：既有自己的方法也有事务增强（事务增强方法用private）

AnarkhServiceImpl：

```java
@Service
public class AnarkhServiceImpl implements AnarkhService{

	@Autowired
	private BanklogDao banklogDao;
	
	@Override
	public String testFun() {
		insertBanklog("1");
		return "sucess";
	}
	
	@Transactional
	private void insertBanklog(String CCTransCode) {
		Banklog banklog = new Banklog();
		banklog.setCctranscode(CCTransCode);
		banklogDao.save(banklog);
	}

}

```

**结果：程序启动sucess，能进debugger，调用fail。**

报了**No Session found for current thread**的错：

```
org.springframework.web.util.NestedServletException: Request processing failed; nested exception is org.hibernate.HibernateException: No Session found for current thread
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:965)
	at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:855)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:648)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:829)
	at com.neusoft.agileweb2.mvc.internal.BundleDispatcherServlet.service(BundleDispatcherServlet.java:54)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:729)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:291)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at com.neusoft.agileweb2.sensitive.internal.SensitiveCharFilter.doFilter(SensitiveCharFilter.java:36)
	at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:343)
	at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:260)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:330)
	at org.springframework.security.web.access.intercept.FilterSecurityInterceptor.invoke(FilterSecurityInterceptor.java:118)
	at org.springframework.security.web.access.intercept.FilterSecurityInterceptor.doFilter(FilterSecurityInterceptor.java:84)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.access.ExceptionTranslationFilter.doFilter(ExceptionTranslationFilter.java:113)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.session.SessionManagementFilter.doFilter(SessionManagementFilter.java:103)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.AnonymousAuthenticationFilter.doFilter(AnonymousAuthenticationFilter.java:113)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter.doFilter(SecurityContextHolderAwareRequestFilter.java:154)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.savedrequest.RequestCacheAwareFilter.doFilter(RequestCacheAwareFilter.java:45)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.www.BasicAuthenticationFilter.doFilter(BasicAuthenticationFilter.java:150)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter.doFilter(DefaultLoginPageGeneratingFilter.java:155)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:199)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter.doFilterInternal(WebAsyncManagerIntegrationFilter.java:50)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:106)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.context.SecurityContextPersistenceFilter.doFilter(SecurityContextPersistenceFilter.java:87)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.FilterChainProxy.doFilterInternal(FilterChainProxy.java:192)
	at org.springframework.security.web.FilterChainProxy.doFilter(FilterChainProxy.java:160)
	at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:343)
	at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:260)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at com.neusoft.agileweb2.core.api.filter.CorsFilter.doFilter(CorsFilter.java:65)
	at com.neusoft.agileweb2.core.api.filter.CorsFilter.doFilter(CorsFilter.java:38)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:88)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:106)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:212)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:106)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:502)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:141)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:79)
	at org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:616)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:88)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:521)
	at org.apache.coyote.http11.AbstractHttp11Processor.process(AbstractHttp11Processor.java:1096)
	at org.apache.coyote.AbstractProtocol$AbstractConnectionHandler.process(AbstractProtocol.java:674)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1500)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.run(NioEndpoint.java:1456)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:745)
Caused by: org.hibernate.HibernateException: No Session found for current thread
	at org.springframework.orm.hibernate4.SpringSessionContext.currentSession(SpringSessionContext.java:97)
	at org.hibernate.internal.SessionFactoryImpl.getCurrentSession(SessionFactoryImpl.java:1041)
	at com.neusoft.agileweb2.database.api.PlatformDao.save(PlatformDao.java:214)
	at com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.AnarkhServiceImpl.insertBanklog(AnarkhServiceImpl.java:29)
	at com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.AnarkhServiceImpl.testFun(AnarkhServiceImpl.java:21)
	at com.neusoft.agileggfw.ww.internal.a.a.WwaaAct.anarkhTest(WwaaAct.java:172)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:497)
	at org.springframework.web.method.support.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:215)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:132)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:104)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandleMethod(RequestMappingHandlerAdapter.java:745)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:685)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:80)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:919)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:851)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:953)
	... 72 more
```

#### 第三种情况：既有自己的方法也有事务增强（事务增强方法用public）

AnarkhServiceImpl：

```java
@Service
public class AnarkhServiceImpl implements AnarkhService{

	@Autowired
	private BanklogDao banklogDao;
	
	@Override
	public String testFun() {
		insertBanklog("1");
		return "sucess";
	}
	
	@Transactional
	public void insertBanklog(String CCTransCode) {
		Banklog banklog = new Banklog();
		banklog.setCctranscode(CCTransCode);
		banklogDao.save(banklog);
	}

}
```

**结果：程序启动sucess，不能进debugger，调用fail。**

报了**No qualifying bean of type**的错：

```
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'invokeServiceImpl': Injection of autowired dependencies failed; nested exception is org.springframework.beans.factory.BeanCreationException: Could not autowire field: private com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.AnarkhServiceImpl com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.InvokeServiceImpl.anarkhServiceImpl; nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type [com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.AnarkhServiceImpl] found for dependency: expected at least 1 bean which qualifies as autowire candidate for this dependency. Dependency annotations: {@org.springframework.beans.factory.annotation.Autowired(required=true)}
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues(AutowiredAnnotationBeanPostProcessor.java:298)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1148)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:519)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:458)
	at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:293)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:223)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:290)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:191)
	at com.neusoft.agileweb2.core.internal.bundle.BundleBeanFactory.getBean(BundleBeanFactory.java:69)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:636)
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:934)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:479)
	at com.neusoft.agileweb2.core.internal.bundle.BundleBeanFactory.findAutowireCandidates(BundleBeanFactory.java:42)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:864)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:779)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:498)
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:87)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues(AutowiredAnnotationBeanPostProcessor.java:295)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1148)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:519)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:458)
	at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:293)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:223)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:290)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:191)
	at com.neusoft.agileweb2.core.internal.bundle.BundleBeanFactory.getBean(BundleBeanFactory.java:69)
	at org.springframework.context.support.AbstractApplicationContext.getBean(AbstractApplicationContext.java:1119)
	at org.springframework.web.method.HandlerMethod.createWithResolvedBean(HandlerMethod.java:219)
	at org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.getHandlerInternal(AbstractHandlerMethodMapping.java:240)
	at org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.getHandlerInternal(AbstractHandlerMethodMapping.java:56)
	at org.springframework.web.servlet.handler.AbstractHandlerMapping.getHandler(AbstractHandlerMapping.java:297)
	at org.springframework.web.servlet.DispatcherServlet.getHandler(DispatcherServlet.java:1085)
	at org.springframework.web.servlet.DispatcherServlet.getHandler(DispatcherServlet.java:1070)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:891)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:851)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:953)
	at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:855)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:648)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:829)
	at com.neusoft.agileweb2.mvc.internal.BundleDispatcherServlet.service(BundleDispatcherServlet.java:54)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:729)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:291)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at com.neusoft.agileweb2.sensitive.internal.SensitiveCharFilter.doFilter(SensitiveCharFilter.java:36)
	at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:343)
	at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:260)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:330)
	at org.springframework.security.web.access.intercept.FilterSecurityInterceptor.invoke(FilterSecurityInterceptor.java:118)
	at org.springframework.security.web.access.intercept.FilterSecurityInterceptor.doFilter(FilterSecurityInterceptor.java:84)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.access.ExceptionTranslationFilter.doFilter(ExceptionTranslationFilter.java:113)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.session.SessionManagementFilter.doFilter(SessionManagementFilter.java:103)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.AnonymousAuthenticationFilter.doFilter(AnonymousAuthenticationFilter.java:113)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter.doFilter(SecurityContextHolderAwareRequestFilter.java:154)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.savedrequest.RequestCacheAwareFilter.doFilter(RequestCacheAwareFilter.java:45)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.www.BasicAuthenticationFilter.doFilter(BasicAuthenticationFilter.java:150)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter.doFilter(DefaultLoginPageGeneratingFilter.java:155)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:199)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter.doFilterInternal(WebAsyncManagerIntegrationFilter.java:50)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:106)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.context.SecurityContextPersistenceFilter.doFilter(SecurityContextPersistenceFilter.java:87)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.FilterChainProxy.doFilterInternal(FilterChainProxy.java:192)
	at org.springframework.security.web.FilterChainProxy.doFilter(FilterChainProxy.java:160)
	at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:343)
	at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:260)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at com.neusoft.agileweb2.core.api.filter.CorsFilter.doFilter(CorsFilter.java:65)
	at com.neusoft.agileweb2.core.api.filter.CorsFilter.doFilter(CorsFilter.java:38)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:88)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:106)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:212)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:106)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:502)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:141)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:79)
	at org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:616)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:88)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:521)
	at org.apache.coyote.http11.AbstractHttp11Processor.process(AbstractHttp11Processor.java:1096)
	at org.apache.coyote.AbstractProtocol$AbstractConnectionHandler.process(AbstractProtocol.java:674)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1500)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.run(NioEndpoint.java:1456)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:745)
Caused by: org.springframework.beans.factory.BeanCreationException: Could not autowire field: private com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.AnarkhServiceImpl com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.InvokeServiceImpl.anarkhServiceImpl; nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type [com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.AnarkhServiceImpl] found for dependency: expected at least 1 bean which qualifies as autowire candidate for this dependency. Dependency annotations: {@org.springframework.beans.factory.annotation.Autowired(required=true)}
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:526)
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:87)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues(AutowiredAnnotationBeanPostProcessor.java:295)
	... 107 more
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type [com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.AnarkhServiceImpl] found for dependency: expected at least 1 bean which qualifies as autowire candidate for this dependency. Dependency annotations: {@org.springframework.beans.factory.annotation.Autowired(required=true)}
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.raiseNoSuchBeanDefinitionException(DefaultListableBeanFactory.java:997)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:867)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:779)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:498)
	... 109 more
```

#### 第四种情况：既有自己的方法也有事务增强（事务增强方法用private）,在自己方法上加上事务注解或者在类上加事务注解

AnarkhServiceImpl：

```
@Service
public class AnarkhServiceImpl implements AnarkhService{

	@Autowired
	private BanklogDao banklogDao;
	
	@Override
	@Transactional
	public String testFun() {
		insertBanklog("1");
		return "sucess";
	}
	
	@Transactional
	private void insertBanklog(String CCTransCode) {
		Banklog banklog = new Banklog();
		banklog.setCctranscode(CCTransCode);
		banklogDao.save(banklog);
	}

}
```

或：

```
@Service
@Transactional
public class AnarkhServiceImpl implements AnarkhService{

	@Autowired
	private BanklogDao banklogDao;
	
	@Override
	public String testFun() {
		insertBanklog("1");
		return "sucess";
	}
	
	@Transactional
	private void insertBanklog(String CCTransCode) {
		Banklog banklog = new Banklog();
		banklog.setCctranscode(CCTransCode);
		banklogDao.save(banklog);
	}

}
```

**结果与第三种情况一致。**

#### 第五种情况：既有自己的方法也有事务增强（事务增强方法用private），但controller中对AnarkhService没使用@Autowired注入，而是直接new了一个对象

```java
@Controller
@RequestMapping(value="/a/a/")
public class WwaaAct {
	
	@ResponseBody
	@RequestMapping(value="report/anarkhTest.html", method = RequestMethod.POST, produces = { "text/html;charset=UTF-8" })
	public String anarkhTest(){
		AnarkhService anarkhService = new AnarkhServiceImpl();
		anarkhService.testFun();
		return "sucess";
	}

}
```

**结果：程序启动sucess，能进debugger，调用fail。**

报了空指针异常：

```
![new注入2](F:\myNewBlob2\Anarkh-Lee.github.io\_posts\img\Problem\new注入2.png)org.springframework.web.util.NestedServletException: Request processing failed; nested exception is java.lang.NullPointerException
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:965)
	at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:855)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:648)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:829)
	at com.neusoft.agileweb2.mvc.internal.BundleDispatcherServlet.service(BundleDispatcherServlet.java:54)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:729)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:291)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:330)
	at org.springframework.security.web.access.intercept.FilterSecurityInterceptor.invoke(FilterSecurityInterceptor.java:118)
	at org.springframework.security.web.access.intercept.FilterSecurityInterceptor.doFilter(FilterSecurityInterceptor.java:84)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.access.ExceptionTranslationFilter.doFilter(ExceptionTranslationFilter.java:113)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.session.SessionManagementFilter.doFilter(SessionManagementFilter.java:103)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.AnonymousAuthenticationFilter.doFilter(AnonymousAuthenticationFilter.java:113)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter.doFilter(SecurityContextHolderAwareRequestFilter.java:154)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.savedrequest.RequestCacheAwareFilter.doFilter(RequestCacheAwareFilter.java:45)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.www.BasicAuthenticationFilter.doFilter(BasicAuthenticationFilter.java:150)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter.doFilter(DefaultLoginPageGeneratingFilter.java:155)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:199)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:110)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter.doFilterInternal(WebAsyncManagerIntegrationFilter.java:50)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:106)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.context.SecurityContextPersistenceFilter.doFilter(SecurityContextPersistenceFilter.java:87)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:342)
	at org.springframework.security.web.FilterChainProxy.doFilterInternal(FilterChainProxy.java:192)
	at org.springframework.security.web.FilterChainProxy.doFilter(FilterChainProxy.java:160)
	at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:343)
	at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:260)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at com.neusoft.agileweb2.sensitive.internal.SensitiveCharFilter.doFilter(SensitiveCharFilter.java:36)
	at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:343)
	at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:260)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at com.neusoft.agileweb2.core.api.filter.CorsFilter.doFilter(CorsFilter.java:65)
	at com.neusoft.agileweb2.core.api.filter.CorsFilter.doFilter(CorsFilter.java:38)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:88)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:106)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:212)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:106)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:502)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:141)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:79)
	at org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:616)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:88)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:521)
	at org.apache.coyote.http11.AbstractHttp11Processor.process(AbstractHttp11Processor.java:1096)
	at org.apache.coyote.AbstractProtocol$AbstractConnectionHandler.process(AbstractProtocol.java:674)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1500)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.run(NioEndpoint.java:1456)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.NullPointerException
	at com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.AnarkhServiceImpl.insertBanklog(AnarkhServiceImpl.java:29)
	at com.neusoft.agileggfw.service.xmzjjg.internal.xmzjjg.service.impl.AnarkhServiceImpl.testFun(AnarkhServiceImpl.java:21)
	at com.neusoft.agileggfw.ww.internal.a.a.WwaaAct.anarkhTest(WwaaAct.java:202)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:497)
	at org.springframework.web.method.support.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:215)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:132)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:104)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandleMethod(RequestMappingHandlerAdapter.java:745)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:685)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:80)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:919)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:851)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:953)
	... 72 more
```

经过debugger可以看到，这种情况注入的dao层是null：

![](.\img\Problem\new注入1.png)

![](.\img\Problem\new注入2.png)



#### 正确做法：如果不涉及对实现类增强，如事务、日志等，可以用@Autowired对实现类进行注入；但是如果涉及到实现类增强，就必须用@Autowired对接口进行注入。遂为了方便记忆，使用@Autowired注入时统一使用接口进行注入。

controller中不能使用new，第五种情况仍不行。

InvokeServiceImpl：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import com.neusoft.agileggfw.service.xmzjjg.api.xmzjjg.service.AnarkhService;
import com.neusoft.agileggfw.service.xmzjjg.api.xmzjjg.service.InvokeService;

@Service
public class InvokeServiceImpl implements InvokeService{
	@Autowired
	private AnarkhService anarkhService;
}

```

AnarkhServiceImpl类中调用dao层方法的方法也要加上事务注解，不然报**No Session found for current thread**的错

```
@Service
public class AnarkhServiceImpl implements AnarkhService{

	@Autowired
	private BanklogDao banklogDao;
	
	@Override
	@Transactional
	public String testFun() {
		insertBanklog("1");
		return "sucess";
	}
	
	@Transactional
	private void insertBanklog(String CCTransCode) {
		Banklog banklog = new Banklog();
		banklog.setCctranscode(CCTransCode);
		banklogDao.save(banklog);
	}

}
```

### 结论与分析

#### 结论

分两种情况来说：

* 如果只是单纯的数据注入，是可以直接注入实现类的。
* 如果对实现类进行了增强，如事务、日志等，只能注入接口。

#### 分析

首先要明确一点，像事务、日志等这种对实现类的增强都是通过AOP动态代理来实现的。AOP的原理就是动态代理，动态代理分为两种：一种是**基于JDK的动态代理**，这种是基于实现类的实现；一种是**基于Cglib的动态代理**，这种是基于子类的实现。

**Spring默认使用的是JDK动态代理，对实现类对象做增强得到的增强类与实现类是兄弟关系，所以不能用实现类接收增强类对象，只能用接口接收。**

```java
//接口：IAnarkh
 
//实现类：AnarkhImpl
 
//增强类：AnarkhImplProxy
 
AnarkhImpl anarkhImpl = new AnarkhImpl();
 
//通过JDKProxyFactory创建代理对象
JDKProxyFactory factory = new JDKProxyFactory(anarkhImpl);
AnarkhImplProxy anarkhImplProxy = factory.createProxy();//这个增强类对象anarkhImplProxy只能强转为IAnarkh，而不能转为AnarkhImpl，因为JDK代理得到的AnarkhImplProxy类与AnarkhImpl是兄弟关系而非父子
```

因此，如果是直接注入被增强的实现类，其代理对象不会有被增强的内容（事务、日志等），因此会报错。

### 扩展

#### @Autowired注入原理

@Autowired是Spring的注解，@Autowired默认先按照byType，如果发现找到多个bean，则改为按照byName方式比对，如果还有多个bean，就会报出异常。

在controller中获取实例的过程：使用@Autowired，程序在Spring的容器中查找类型是AnarkhService的bean，刚好找到有且只有一个此类型的bean，即anarkhServiceImpl，所以就把anarkhServiceImpl自动装配到controller的实例anarkhService中，anarkhService其实就是增强的（如果实现类有增强的话）AnarkhServiceImpl实例。

**注：**

byName 通过参数名 自动装配，如果一个bean的name 和另外一个bean的 property 相同，就自动装配。

byType 通过参数的数据类型自动自动装配，如果一个bean的数据类型和另外一个bean的property属性的数据类型兼容，就自动装配。

#### 一个接口多个实现类的注入情况

例子代码：

```java
//接口
public interface AnarkhService{
	void test();
}
//实现1
@Service
public class AnarkhServiceImpl1 implements AnarkhService{
    @Override
    public String test() {
        return "AnarkhServiceImpl1";
    }
}
//实现2
@Service
public class AnarkhServiceImpl2 implements AnarkhService{
    @Override
    public String test() {
        return "AnarkhServiceImpl2";
    }
}
```

多个实现类的话可以通过一下两种方式来指定具体使用哪一种实现：

**1.通过指定bean的名字来明确到底要实例哪一个类**

@Autowired 需要结合@Qualifier来使用，如下：

```java
@Autowired
@Qualifier("anarkhServiceImpl1")
//注意此处指定的anarkhServiceImpl1中的a是小写，因为Spring在装配bean的时候，如果没有指定，会把类名的首字母小写作为bean的name
private AnarkhService anarkhService;
```

**2.通过在实现类是添加@Primary注解来指定默认加载类**

```java
@Service
@Primary
public class AnarkhServiceImpl2 implements AnarkhService{
    @Override
    public String test() {
        return "AnarkhServiceImpl2";
    }
}
```

这样如果在使用@Autowired获取实例时，如果不指定bean的名字，就会默认获取AnarkhServiceImpl2的bean，如果指定了bean的名字则以指定的为准。



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>