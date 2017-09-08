---
layout:     post
title:     Spring IOC学习心得之源码级分析ContextLoaderListener的作用(IOC容器初始化入口)
subtitle:   
date:       2017-09-08
author:     Jason
header-img: 
catalog: true
tags:
    - Spring
    - Java
    - Linux
---

ContextLoaderListener类是负责初始化IOC容器，即在我们的web项目中，这里就是IOC容器初始化的入口，由这个类启动IOC容器的初始化。
它配置在web.xml中，比如如下配置：
```
<context-param>  
        <param-name>contextConfigLocation</param-name>  
        <param-value>classpath:context/applicationContext.xml</param-value>  
    </context-param>  
  
    <!-- Spring监听器 -->  
    <listener>  
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
    </listener>  
```
跟踪ContextLoaderListener类，可以看到有一个如下方法
```
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {  
        String configLocationParam;  
        if(ObjectUtils.identityToString(wac).equals(wac.getId())) {  
            configLocationParam = sc.getInitParameter("contextId");  
            if(configLocationParam != null) {  
                wac.setId(configLocationParam);  
            } else {  
                wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX + ObjectUtils.getDisplayString(sc.getContextPath()));  
            }  
        }  
  
        wac.setServletContext(sc);  
        configLocationParam = sc.getInitParameter("contextConfigLocation"); //很显然这里就是读取web.xml中配置的资源文件的路径，也就是上述context-param  
标签配置的参数  
        if(configLocationParam != null) {  
            wac.setConfigLocation(configLocationParam);  
        }  
  
        ConfigurableEnvironment env = wac.getEnvironment();  
        if(env instanceof ConfigurableWebEnvironment) {  
            ((ConfigurableWebEnvironment)env).initPropertySources(sc, (ServletConfig)null);  
        }  
  
        this.customizeContext(sc, wac);  
        wac.refresh();  //有了资源文件后，在这里开启IOC容器的初始化  
    }  
```
refresh方法的实现是在AbstractApplicationContext类中，代码如下：
```
public void refresh() throws BeansException, IllegalStateException {  
        Object var1 = this.startupShutdownMonitor;  
        synchronized(this.startupShutdownMonitor) {  
            this.prepareRefresh();  
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();  
            this.prepareBeanFactory(beanFactory);  
  
            try {  
                this.postProcessBeanFactory(beanFactory);  
                this.invokeBeanFactoryPostProcessors(beanFactory);  
                this.registerBeanPostProcessors(beanFactory);  
                this.initMessageSource();  
                this.initApplicationEventMulticaster();  
                this.onRefresh();  
                this.registerListeners();  
                this.finishBeanFactoryInitialization(beanFactory);  
                this.finishRefresh();  
            } catch (BeansException var9) {  
                if(this.logger.isWarnEnabled()) {  
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);  
                }  
  
                this.destroyBeans();  
                this.cancelRefresh(var9);  
                throw var9;  
            } finally {  
                this.resetCommonCaches();  
            }  
  
        }  
    }  
```
上述代码就会进行完整的IOC容器的初始化
