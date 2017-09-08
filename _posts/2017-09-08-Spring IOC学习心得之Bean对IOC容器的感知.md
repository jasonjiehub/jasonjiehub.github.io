---
layout:     post
title:     Spring IOC学习心得之Bean对IOC容器的感知
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

容器管理的Bean一般不需要了解容器的状态和直接使用Bean，但是在某些情况下，是需要在Bean中直接对IOC容器进行操作的，这时候，就需要在Bean中设定对容器的感知。Spring IOC也提供了该功能，它是通过特定的aware接口来完成的。aware接口有如下这些：
BeanNameAware：可以在Bean中得到它在IOC容器中的Bean的实例名称
BeanFactoryAware：可以在Bean中得到Bean所在的IOC容器，从而直接在Bean中使用IOC容器的服务
ApplicationContextAware：可以在Bean中得到Bean所在的应用上下文，从而直接在Bean中使用应用上下文的服务
MessageSourceAware：在Bean中可以得到消息源
ApplicationEventPublisherAware：在Bean中可以得到应用上下文的事件发布器，从而可以在Bean中发布应用上下文的事件
ResourceLoaderAware：在Bean中可以得到resourceloader，从而在Bean中使用ResourceLoader加载外部对应的Resource资源
要在自定义的Bean中获取上述资源，只需要让Bean实现对应的接口，并覆写set方法，如下：
```
public class TestSpring implements ApplicationContextAware{  
    private ApplicationContext applicationContext;  
  
    @Override  
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {  
        this.applicationContext = applicationContext;  
    }  
}  
```
这样这个Bean实现了ApplicationContextAware接口，并覆写setApplicationContext方法，就可以获取到ApplicationContext对象。
这个setApplicationContext方法的回调是容器自动完成的，容器调用该方法的时候，我们就可以将容器传入的参数applicationContext保存起来以供使用
setApplicationContext方法被容器的自动调用是在BeanPostProcessor接口的方法postProcessBeforeInitialization中完成的，实现是在ApplicationContextAwareProcessor类中，具体代码如下：
```
class ApplicationContextAwareProcessor implements BeanPostProcessor {  
  
    private final ConfigurableApplicationContext applicationContext;  
  
    private final StringValueResolver embeddedValueResolver;  
  
  
    /**  
     * Create a new ApplicationContextAwareProcessor for the given context.  
     */  
    public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {  
        this.applicationContext = applicationContext;  
        this.embeddedValueResolver = new EmbeddedValueResolver(applicationContext.getBeanFactory());  
    }  
  
  
    @Override  
    public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {  
        AccessControlContext acc = null;  
  
        if (System.getSecurityManager() != null &&  
                (bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||  
                        bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||  
                        bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {  
            acc = this.applicationContext.getBeanFactory().getAccessControlContext();  
        }  
  
        if (acc != null) {  
            AccessController.doPrivileged(new PrivilegedAction<Object>() {  
                @Override  
                public Object run() {  
                    invokeAwareInterfaces(bean);  
                    return null;  
                }  
            }, acc);  
        }  
        else {  
            invokeAwareInterfaces(bean);  
        }  
  
        return bean;  
    }  
//这里就是容器自动调用set方法的地方，其实就是调用自定义的TestSpring类实现的setApplicationContext方法，这样TestSpring类就获取到了applicationContext  
    private void invokeAwareInterfaces(Object bean) {  
        if (bean instanceof Aware) {  
            if (bean instanceof EnvironmentAware) {  
                ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());  
            }  
            if (bean instanceof EmbeddedValueResolverAware) {  
                ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);  
            }  
            if (bean instanceof ResourceLoaderAware) {  
                ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);  
            }  
            if (bean instanceof ApplicationEventPublisherAware) {  
                ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);  
            }  
            if (bean instanceof MessageSourceAware) {  
                ((MessageSourceAware) bean).setMessageSource(this.applicationContext);  
            }  
            if (bean instanceof ApplicationContextAware) {  
                ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);  
            }  
        }  
    }  
  
    @Override  
    public Object postProcessAfterInitialization(Object bean, String beanName) {  
        return bean;  
    }  
  
}  
```
