---
layout:     post
title:      Spring源码分析之doDispatch分发请求逻辑
subtitle:   
date:       2017-09-08
author:     Jason
header-img: 
catalog: true
tags:
    - Spring
    - Java
---


首先，我的另外一篇博客已经讲述了DispatcherServlet的整个初始化过程，地址如下：
http://blog.csdn.net/u011734144/article/details/74136168
下面说说DispatcherServlet是如何分发请求的
分发请求是由该类的doDispatch方法来完成的，先看下具体代码
```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			// 这里为视图准备好一个ModelAndView,这个ModelAndView持有handler处理请求的结果
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				// 根据请求得到对应的Handler
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				// 这里是实际调用handler的地方,在执行handler之前,用handlerAdapter先先检查一下handler的合法性: 即是不是按spring的要求编写的handler
				// handler处理的结果封装到ModelAndView中,为视图提供展现数据
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (logger.isDebugEnabled()) {
						logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
					}
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				// 通过调用handlerAdapter的handle方法,实际上触发对Controller的handleRequest方法的调用
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
	
	}
```
首先可以看到，这里为视图准备好一个ModelAndView,这个ModelAndView持有handler处理请求的结果
然后如下代码：
mappedHandler = getHandler(processedRequest)
可以根据请求得到对应的handler，如果获取不到对应的handler，就会调用noHandlerFound方法来进行处理，表示找不到对应的处理器，从而抛出异常
在执行handler之前,用handlerAdapter先先检查一下handler的合法性: 即是不是按spring的要求编写的handler，即需要判断这个handler是不是Controller接口的实现
这个判断是通过HandlerAdapter来判断的，以SimpleControllerHandlerAdapter的实现为例来了解这个判断是如何起作用的，这个判断是通过support方法来实现，
判断当前的handler是不是Controller对象，具体代码如下：

```
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

	// 判断将要调用handler是不是controller
	@Override
	public boolean supports(Object handler) {
		return (handler instanceof Controller);
	}

	@Override
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		//对http请求响应的处理.生成各种需要的数据,并把这些数据封装到ModelAndView中去
		return ((Controller) handler).handleRequest(request, response);
	}

	@Override
	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}

}
```
接下来mappedHandler.applyPreHandle方法，代码如下：
```
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = 0; i < interceptors.length; i++) {
				HandlerInterceptor interceptor = interceptors[i];
				if (!interceptor.preHandle(request, response, this.handler)) {
					triggerAfterCompletion(request, response, null);
					return false;
				}
				this.interceptorIndex = i;
			}
		}
		return true;
	}
```
这是调用为handler配置的拦截器，从HanlderExecutionChain中取出所有的拦截器Interceptor进行前置处理
mv = ha.handle(processedRequest, response, mappedHandler.getHandler()); 是真正调用handler进行处理的地方，实际是触发对Controller的
handleRequest方法的调用来进行http请求的处理
mappedHandler.applyPostHandle方法与前置处理方法applyPreHandle类似，是handle的拦截器的后置处理，具体代码如下：
```
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, ModelAndView mv) throws Exception {
		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = interceptors.length - 1; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				interceptor.postHandle(request, response, this.handler, mv);
			}
		}
	}
```
最后的processDispatchResult方法就是使用视图对ModelAndView数据的展现
到这里，主流程我们就分析完了。
接下来我们看下具体的获取handler的过程，也就是getHandler方法的代码
```
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		// 这里是从HandlerMapping中去取handler的调用
		for (HandlerMapping hm : this.handlerMappings) {
			if (logger.isTraceEnabled()) {
				logger.trace(
						"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
			}
			HandlerExecutionChain handler = hm.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
		return null;
	}
```
可以看到它其实就是从handlerMappings中获取对应的处理器，也就是遍历当前所持有的所有的HanlderMapping，因为在DispatcherServlet中可能定义了不止一个
handlerMapping，在这一系列handlerMapping中只要找到了一个需要的handler，就会停止查找，在找到了handler以后，通过hanler返回的是一个HandlerExecutionChain
对象，其中包含了最终的Controller和定义的一个拦截器链，拦截器链是为了对handler其实也就是目标Controller进行增强。这样就会按照上面的步骤，先来调用拦截器链的前置处理方法，然后调用Controller来进行请求处理，然后调用
拦截器链的后置处理方法。
那么接下来我们想知道的就是handlerMappings中的拦截器链和Controller是何时注册进去的
下面我们以HandlerMapping的其中一个实现SimpleUrlHandlerMapping来进行讲解，根据名字，可以看出他是根据Url来进行映射，并注册Handler和Interceptor，从而
维护一个反映这种url到handler和Interceptor的映射关系的handlerMap，当需要匹配Http请求的时候就需要查询这个handlerMap中的信息来得到对应的
HandlerExecutionChain，而这个对象里面就封装了url对应的handler和Interceptor，这个过程就是在SimpleUrlHandlerMapping的如下方法中完成的：
```
public void initApplicationContext() throws BeansException {
   super.initApplicationContext();
   registerHandlers(this.urlMap);
}
```
```
public abstract class ApplicationObjectSupport implements ApplicationContextAware {


	@Override
	public final void setApplicationContext(ApplicationContext context) throws BeansException {
		if (context == null && !isContextRequired()) {
			// Reset internal context state.
			this.applicationContext = null;
			this.messageSourceAccessor = null;
		}
		else if (this.applicationContext == null) {
			// Initialize with passed-in context.
			if (!requiredContextClass().isInstance(context)) {
				throw new ApplicationContextException(
						"Invalid application context: needs to be of type [" + requiredContextClass().getName() + "]");
			}
			this.applicationContext = context;
			this.messageSourceAccessor = new MessageSourceAccessor(context);
			initApplicationContext(context);
		}
		else {
			// Ignore reinitialization if same context passed in.
			if (this.applicationContext != context) {
				throw new ApplicationContextException(
						"Cannot reinitialize with different application context: current one is [" +
						this.applicationContext + "], passed-in one is [" + context + "]");
			}
		}
	}
```
可以看到这个类实现了ApplicationContextAware接口，我们知道在类进行初始化的时候，系统会自动帮我们调用ApplicationContextAware类的
setApplicationContext方法，来让bean获取对IOC容器的感知，这个调用在我的其他文章中也有具体的介绍，这里不做过多的说明。我们可以看到这个方法里面就是调用
initApplicationContext的地方，即这里来完成启动对映射关系的注册

接着说上面的初始化方法，其中的registerHandlers方法其实是调用的基类AbstractUrlHandlerMapping的registerHandlers方法来进行的注册，具体代码如下：
```
protected void registerHandler(String urlPath, Object handler) throws BeansException, IllegalStateException {
		Assert.notNull(urlPath, "URL path must not be null");
		Assert.notNull(handler, "Handler object must not be null");
		Object resolvedHandler = handler;

		// Eagerly resolve handler if referencing singleton via name.
		// 如果直接使用bean名称进行映射,那就直接从容器中获取handler
		if (!this.lazyInitHandlers && handler instanceof String) {
			String handlerName = (String) handler;
			if (getApplicationContext().isSingleton(handlerName)) {
				resolvedHandler = getApplicationContext().getBean(handlerName);
			}
		}

		Object mappedHandler = this.handlerMap.get(urlPath);
		if (mappedHandler != null) {
			if (mappedHandler != resolvedHandler) {
				throw new IllegalStateException(
						"Cannot map " + getHandlerDescription(handler) + " to URL path [" + urlPath +
						"]: There is already " + getHandlerDescription(mappedHandler) + " mapped.");
			}
		}
		else {
			// 处理URL是"/"的映射,把这个"/"映射的controller设置到rootHandler中
			if (urlPath.equals("/")) {
				if (logger.isInfoEnabled()) {
					logger.info("Root mapping to " + getHandlerDescription(handler));
				}
				setRootHandler(resolvedHandler);
			}
			// 处理URL是"/*"的映射,把这个"/"映射的controller设置到defaultHandler中
			else if (urlPath.equals("/*")) {
				if (logger.isInfoEnabled()) {
					logger.info("Default mapping to " + getHandlerDescription(handler));
				}
				setDefaultHandler(resolvedHandler);
			}
			// 处理正常的URL映射,设置handlerMap的key和value,分别对应于URL和映射的controller
			else {
				this.handlerMap.put(urlPath, resolvedHandler);
				if (logger.isInfoEnabled()) {
					logger.info("Mapped URL path [" + urlPath + "] onto " + getHandlerDescription(handler));
				}
			}
		}
	}
```
在这个处理过程中，如果使用Bean的名称作为映射，那么直接从容器中获取这个Http映射对应的Bean，然后还要对不同的URL配置进行解析处理，比如在Http请求中配置
成“/”和通配符“/*”的URL，以及正常的URL请求，完成这个解析处理过程以后，会把URL和handler作为键值对放到上面说到的handlerMap中去，handlerMap其实就是
一个hashMap，其中保存了URL请求到Controller的映射关系，这个handlerMap是在AbstractUrlHandlerMapping中定义的
