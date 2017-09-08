---
layout:     post
title:      Spring源码分析之SpringMVC的DispatcherServlet是如何处理Http请求的
subtitle:   
date:       2017-09-08
author:     Jason
header-img: 
catalog: true
tags:
    - Spring
---

一般我们会在web.xml文件中配置DispatcherServlet，比如如下配置方式：
```
<servlet>  
        <servlet-name>dispatcherServlet</servlet-name>  
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
        <init-param>  
            <param-name>contextConfigLocation</param-name>  
            <param-value>classpath:dispatcher-servlet.xml</param-value>  
        </init-param>  
        <load-on-startup>1</load-on-startup>  
    </servlet>  
      
    <servlet-mapping>  
        <servlet-name>dispatcherServlet</servlet-name>  
        <url-pattern>/</url-pattern>  
    </servlet-mapping>  
```
上述代码配置了所有路径的http请求都会交给DispatcherServlet来处理
那么我们来看下DispatcherServlet的具体代码：
```
@SuppressWarnings("serial")  
public class DispatcherServlet extends FrameworkServlet {  
    public DispatcherServlet(WebApplicationContext webApplicationContext) {  
        super(webApplicationContext);  
        setDispatchOptionsRequest(true);  
    }  
  
    @Override  
    protected void onRefresh(ApplicationContext context) {  
        initStrategies(context);  
    }  
  
    /**  
     * Initialize the strategy objects that this servlet uses.  
     * <p>May be overridden in subclasses in order to initialize further strategy objects.  
     */  
    protected void initStrategies(ApplicationContext context) {  
        initMultipartResolver(context);  
        initLocaleResolver(context);  
        initThemeResolver(context);  
        initHandlerMappings(context);  
        initHandlerAdapters(context);  
        initHandlerExceptionResolvers(context);  
        initRequestToViewNameTranslator(context);  
        initViewResolvers(context);  
        initFlashMapManager(context);  
    }  
  
  
    @Override  
    protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {  
          
    }  
  
  
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {  
          
    }  
```
可以看到这个类继承了FrameworkServlet类，再来看下这个类的代码：
```
@SuppressWarnings("serial")  
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {  
  
    @Override  
    protected final void initServletBean() throws ServletException {  
          
    }  
  
    protected WebApplicationContext initWebApplicationContext() {  
  
        if (!this.refreshEventReceived) {  
            // Either the context is not a ConfigurableApplicationContext with refresh  
            // support or the context injected at construction time had already been  
            // refreshed -> trigger initial onRefresh manually here.  
            onRefresh(wac);  
        }  
  
          
    }  
    @Override  
    public void destroy() {  
        getServletContext().log("Destroying Spring FrameworkServlet '" + getServletName() + "'");  
        // Only call close() on WebApplicationContext if locally managed...  
        if (this.webApplicationContext instanceof ConfigurableApplicationContext && !this.webApplicationContextInjected) {  
            ((ConfigurableApplicationContext) this.webApplicationContext).close();  
        }  
    }  
  
  
    /**  
     * Override the parent class implementation in order to intercept PATCH requests.  
     */  
    @Override  
    protected void service(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {  
  
        HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());  
        if (HttpMethod.PATCH == httpMethod || httpMethod == null) {  
            processRequest(request, response);  
        }  
        else {  
            super.service(request, response);  
        }  
    }  
  
      
    @Override  
    protected final void doGet(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {  
  
        processRequest(request, response);  
    }  
  
      
    @Override  
    protected final void doPost(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {  
  
        processRequest(request, response);  
    }  
  
    protected final void processRequest(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {  
    }  
  
}  
```
可以看到这个类继承了HttpServletBean类，而HttpServletBean类又继承了HttpServlet类，其中HttpServlet类的init，destroy，service三个方法负责管理Servlet的生命周期。
init方法：负责初始化Servlet对象
destroy方法：负责销毁Servlet对象
service方法：负责处理http请求
由于HttpServletBean实现了HttpServlet的init方法，可以看到该方法的代码如下：
```
public final void init() throws ServletException {  
        if (logger.isDebugEnabled()) {  
            logger.debug("Initializing servlet '" + getServletName() + "'");  
        }  
  
        // Set bean properties from init parameters.  
        // 获取servlet初始化的参数,对Bean属性进行配置  
        try {  
            PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);  
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);  
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());  
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));  
            initBeanWrapper(bw);  
            bw.setPropertyValues(pvs, true);  
        }  
        catch (BeansException ex) {  
            logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);  
            throw ex;  
        }  
  
        // Let subclasses do whatever initialization they like.  
        // 调用子类的initServletBean进行具体的初始化  
        initServletBean();  
  
        if (logger.isDebugEnabled()) {  
            logger.debug("Servlet '" + getServletName() + "' configured successfully");  
        }  
    }  
```
这里会读取配置在ServletContext中的Bean属性参数，这里可以看到对PropertyValues和BeanWrapper的使用，这里就完成了对Servlet的初始化
接着调用了initServletBean方法来初始化DispatcherServlet持有的IOC容器，当然底层其实是调用initWebApplicationContext方法来初始化，其代码如下：
```
protected WebApplicationContext initWebApplicationContext() {  
        // 得到根上下文,这个根上下文是保存在ServletContext中的,使用这个根上下文作为当前MVC上下文的双亲上下文,当前的上下文就是wac  
        // cwac.setParent(rootContext)就是设置当前上下文的根上下文  
        WebApplicationContext rootContext =  
                WebApplicationContextUtils.getWebApplicationContext(getServletContext());  
        WebApplicationContext wac = null;  
  
  
        if (wac == null) {  
            // No context instance is defined for this servlet -> create a local one  
            wac = createWebApplicationContext(rootContext);  
        }  
  
        if (!this.refreshEventReceived) {  
            // Either the context is not a ConfigurableApplicationContext with refresh  
            // support or the context injected at construction time had already been  
            // refreshed -> trigger initial onRefresh manually here.  
            onRefresh(wac);  
        }  
    }  
```
可以看到createWebApplicationContext方法就是在创建并初始化IOC容器，代码如下：
```
protected WebApplicationContext createWebApplicationContext(ApplicationContext parent) {  
          
        //设置IOC容器的各个参数  
        wac.setEnvironment(getEnvironment());  
        wac.setParent(parent);  
        wac.setConfigLocation(getContextConfigLocation());  
  
        configureAndRefreshWebApplicationContext(wac);  
  
        return wac;  
    } 
```
这里设置了当前IOC容器上下文的双亲上下文，双亲上下文是如下方法获取到的：
```
WebApplicationContext rootContext =  
                WebApplicationContextUtils.getWebApplicationContext(getServletContext());  
```
这样当从IOC容器中getBean的时候会先从其双亲上下文中获取bean,configureAndRefreshWebApplicationContext方法里面调用了wac.refresh()，也就是refresh方法来真正启动对IOC容器的初始化，这里就跟IOC容器的初始化保持一致了

到这里，对DispatcherServlet所持有的IOC容器的初始化就完成了，初始化完成后，DispatcherServlet就持有一个以自己的Servlet名称命名的IOC容器了，这个IOC容器是一个WebApplicationContext对象。
我们还可以看到initWebApplicationContext方法中还调用了onrefresh方法，该方法是由DispatcherServlet来实现的，并最终调用了DispatcherServlet的如下方法来进行初始化：

```
protected void initStrategies(ApplicationContext context) {  
        initMultipartResolver(context);  
        initLocaleResolver(context);  
        initThemeResolver(context);  
        initHandlerMappings(context);  
        initHandlerAdapters(context);  
        initHandlerExceptionResolvers(context);  
        initRequestToViewNameTranslator(context);  
        initViewResolvers(context);  
        initFlashMapManager(context);  
    }  
```
在这个方法里，DispatcherServlet对MVC模块的其他部分进行了初始化，比如handlerMapping，ViewResolver等，它是启动整个Spring MVC框架的初始化
比如，支持国际化的LocalResolver，支持request映射的HandlerMapping，以及视图生成的ViewResolver等的初始化
这里的initHandlerMapping是为HTTP请求找到相应的Controller控制器。
DispatcherServlet的整个初始化过程就到这里结束了，下面说下DispatcherServlet是如何处理http请求的

首先http请求是由doService来处理的，那么我们可以看下HttpServlet类的该方法：
```
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {  
        String method = req.getMethod();  
        long errMsg;  
        if(method.equals("GET")) {  
            errMsg = this.getLastModified(req);  
            if(errMsg == -1L) {  
                this.doGet(req, resp);  
            } else {  
                long ifModifiedSince = req.getDateHeader("If-Modified-Since");  
                if(ifModifiedSince < errMsg) {  
                    this.maybeSetLastModified(resp, errMsg);  
                    this.doGet(req, resp);  
                } else {  
                    resp.setStatus(304);  
                }  
            }  
        } else if(method.equals("HEAD")) {  
            errMsg = this.getLastModified(req);  
            this.maybeSetLastModified(resp, errMsg);  
            this.doHead(req, resp);  
        } else if(method.equals("POST")) {  
            this.doPost(req, resp);  
        } else if(method.equals("PUT")) {  
            this.doPut(req, resp);  
        } else if(method.equals("DELETE")) {  
            this.doDelete(req, resp);  
        } else if(method.equals("OPTIONS")) {  
            this.doOptions(req, resp);  
        } else if(method.equals("TRACE")) {  
            this.doTrace(req, resp);  
        } else {  
            String errMsg1 = lStrings.getString("http.method_not_implemented");  
            Object[] errArgs = new Object[]{method};  
            errMsg1 = MessageFormat.format(errMsg1, errArgs);  
            resp.sendError(501, errMsg1);  
        }  
  
    }  
```
这个方法里面会根据不同的请求类型，来调用不同的处理方法，比如会调用doGet或者doPost方法来分别处理get请求和post请求，跟踪这些方法，可以发现，最终是调用了DispatcherServlet的doService方法，代码如下：
```
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {  
        try {  
            this.doDispatch(request, response);  
        } finally {  
            if(!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted() && attributesSnapshot1 != null) {  
                this.restoreAttributesAfterInclude(request, attributesSnapshot1);  
            }  
  
        }  
  
    }  
```
可以看到该方法调用了doDispatch来进行处理，doDispatch是MVC模式的主要部分
