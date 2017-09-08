---
layout:     post
title:     Spring IOC学习心得之BeanPostProcessor，BeanNameAware，BeanClassLoaderAware，BeanFactoryAware接口是如何起作用的
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

1. 首先说下BeanPostProcessor接口中的两个方法，如下：
```
package org.springframework.beans.factory.config;  
  
import org.springframework.beans.BeansException;  
  
  
public interface BeanPostProcessor {  
  
        //Bean初始化的前置处理器  
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;  
  
    //Bean初始化的后置处理器  
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;  
  
}  
```
应用中自定义的Bean，可以实现这个接口，并覆盖这两个方法来控制Bean的初始化过程，即在Bean的初始化之前做一件事，即调用postProcessBeforeInitialization方法，也可以在Bean的初始化之后做一件事，即调用postProcessAfterInitialization方法。那么这两个方法究竟是如何被Spring调用的呢？


2. 在Bean的初始化过程中，会调用initializeBean方法，该方法的源码如下：
```
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {  
        if (System.getSecurityManager() != null) {  
            AccessController.doPrivileged(new PrivilegedAction<Object>() {  
                @Override  
                public Object run() {  
                    invokeAwareMethods(beanName, bean);  
                    return null;  
                }  
            }, getAccessControlContext());  
        }  
        else {  
            invokeAwareMethods(beanName, bean);  
        }  
  
        Object wrappedBean = bean;  
        if (mbd == null || !mbd.isSynthetic()) {  
            //执行BeanPostProcessor扩展点的PostProcessBeforeInitialization进行修改实例化Bean  
            wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);  
        }  
  
        try {  
            //调用Bean的初始化方法,这个初始化方法是在BeanDefinition中通过定义init-method属性指定的  
            //同时,如果Bean实现了InitializingBean接口,那么这个Bean的afterPropertiesSet实现也不会被调用  
            invokeInitMethods(beanName, wrappedBean, mbd);  
        }  
        catch (Throwable ex) {  
            throw new BeanCreationException(  
                    (mbd != null ? mbd.getResourceDescription() : null),  
                    beanName, "Invocation of init method failed", ex);  
        }  
  
        if (mbd == null || !mbd.isSynthetic()) {  
            //执行BeanPostProcessor扩展点的PostProcessAfterInitialization进行修改实例化Bean  
            wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);  
        }  
        return wrappedBean;  
    }  
```
3. BeanPostProcessor起作用的方式
先说下invokeInitMethods方法，这个是真正的Bean的初始化方法，我们可以看到在该方法之前有一个方法applyBeanPostProcessorsBeforeInitialization，该方法实现Bean初始化的前置处理，可以看下该方法的源码：
```
@Override  
    public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)  
            throws BeansException {  
  
        Object result = existingBean;  
        for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {  
            result = beanProcessor.postProcessBeforeInitialization(result, beanName);  
            if (result == null) {  
                return result;  
            }  
        }  
        return result;  
    }  
```
这个方法中会通过getBeanPostProcessors方法去获取Bean所实现的所有的BeanPostProcessor接口，并调用其postProcessBeforeInitialization方法来实现在Bean的初始化之前做一些预处理
在invokeInitMethods方法之后，有一个applyBeanPostProcessorsAfterInitialization方法，该方法实现Bean的初始化的后置处理，可以看下该方法的源码：
```
@Override  
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)  
            throws BeansException {  
  
        Object result = existingBean;  
        for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {  
            result = beanProcessor.postProcessAfterInitialization(result, beanName);  
            if (result == null) {  
                return result;  
            }  
        }  
        return result;  
    }  
```
与前置处理类似，也是获取Bean实现的所有BeanPostProcessor接口，然后调用所有接口的postProcessAfterInitialization后置处理方法
4.  BeanNameAware，BeanClassLoaderAware，BeanFactoryAware接口起作用的方式
大家注意到initializebean方法中有一个invokeAwareMethods方法，先看下这个方法的源码：
```
private void invokeAwareMethods(final String beanName, final Object bean) {  
        if (bean instanceof Aware) {  
            if (bean instanceof BeanNameAware) {  
                ((BeanNameAware) bean).setBeanName(beanName);  
            }  
            if (bean instanceof BeanClassLoaderAware) {  
                ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());  
            }  
            if (bean instanceof BeanFactoryAware) {  
                ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);  
            }  
        }  
    }  
```
举例来说，当应用自定义的Bean实现了BeanNameAware接口，如下：
```
public class MyBean implements BeanNameAware {  
    private String beanName;  
  
    void setBeanName(String name) {  
        this.beanName = name;  
    }  
  
}  
```
这样就可以获取到该Bean在Spring容器中的名字，原理就是上述invokeAwareMethods方法中，判断了如果bean实现了BeanNameAware接口，就会调用该Bean覆盖的BeanNameAware接口的setBeanName方法，这样MyBean中就获取到了该Bean在Spring容器中的名字。
BeanClassLoaderAware接口和BeanFactoryAware接口同理，可以分别获取Bean的类装载器和bean工厂
5. InitializeBean接口起作用的方式
最后来说下真正的初始化方法invokeInitMethods，该方法的源码如下：
```
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)  
            throws Throwable {  
  
        boolean isInitializingBean = (bean instanceof InitializingBean);  
        if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {  
            if (logger.isDebugEnabled()) {  
                logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");  
            }  
            if (System.getSecurityManager() != null) {  
                try {  
                    AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {  
                        @Override  
                        public Object run() throws Exception {  
                            ((InitializingBean) bean).afterPropertiesSet();  
                            return null;  
                        }  
                    }, getAccessControlContext());  
                }  
                catch (PrivilegedActionException pae) {  
                    throw pae.getException();  
                }  
            }  
            else {  
                ((InitializingBean) bean).afterPropertiesSet(); //启动afterPropertiesSet,afterPropertiesSet是InitializingBean接口的方法  
            }  
        }  
  
        if (mbd != null) {  
            String initMethodName = mbd.getInitMethodName();    //获取用户自定义的初始化方法  
            if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&  
                    !mbd.isExternallyManagedInitMethod(initMethodName)) {  
                invokeCustomInitMethod(beanName, bean, mbd);    //调用自定义的初始化方法  
            }  
        }  
```
可以看到方法中判断了如果该Bean实现了InitializeBean接口，即bean instanceof InitializingBean == true，就会调用该Bean实现的Initializebean接口的afterPropertiesSet方法

6. 自定义的初始化方法起作用的方式
上述代码中有一个方法getInitMethodName，可以获取用户自定义的初始化方法，然后通过调用invokeCustomInitMethod方法来执行用户自定义的初始化方法
