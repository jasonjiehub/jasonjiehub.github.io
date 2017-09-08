---
layout:     post
title:      Spring源码分析之Aop中拦截器,适配器,通知之间的关系
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

首先举一个例子：
```
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {  
  
    private MethodBeforeAdvice advice;  
  
  
    /**  
     * Create a new MethodBeforeAdviceInterceptor for the given advice.  
     * @param advice the MethodBeforeAdvice to wrap  
     */  
    public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {  
        Assert.notNull(advice, "Advice must not be null");  
        this.advice = advice;  
    }  
  
    //这个invoke方法是拦截器的回调方法,会在代理对象的方法被调用时触发回调  
    @Override  
    public Object invoke(MethodInvocation mi) throws Throwable {  
        //首先触发advise的before回调  
        this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );  
        //然后才是MethodInvocation的proceed方法调用  
        return mi.proceed();  
    }  
  
}  
```
上面是MethodBeforeAdviceInterceptor拦截器的源码，MethodBeforeAdvice是对应的通知，另外还有一个角色类是MethodBeforeAdviceAdapter适配器类，这个适配器的源码如下：
```
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {  
  
    @Override  
    public boolean supportsAdvice(Advice advice) {  
        return (advice instanceof MethodBeforeAdvice);  
    }  
  
    @Override  
    public MethodInterceptor getInterceptor(Advisor advisor) {  
        MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();  
        return new MethodBeforeAdviceInterceptor(advice);  
    }  
  
}  
```
可以看到，它的主要作用是，将通知advisor.getAdvice()转换或者说适配成对应的拦截器，有如下三种适配：
```
1. MethodBeforeAdviceAdapter将MethodBeforeAdvice适配成MethodBeforeAdviceInterceptor  
2. AfterReturningAdviceAdapter将AfterReturningAdvice适配成AfterReturningAdviceInterceptor  
3.ThrowsAdviceAdapter将ThrowsAdvice适配成ThrowsAdviceInterceptor  
```
这样适配的目的是什么呢？
还得回到我们最初的MethodBeforeAdviceInterceptor的源码中去看
首先我们得知道通知是什么，通知就是在调用目标方法之前或者之后等要被调用的增强的方法，比如这里的MethodBeforeAdvice的before方法需要在调用目标方法之前被调用，AfterReturningAdvice的afterReturning方法需要在目标方法被调用之后调用，这个调用逻辑（调用顺序）就是由通知对应的拦截器来完成的，可以看到MethodBeforeAdviceInterceptor由于是前置通知的适配器，所以它的invoke方法在执行下一个拦截器的时候先调用了通知的before方法，然后进入到了下一个拦截器。
也就是说，通知只是定义了增强时将要被调用的方法，而方法具体何时调用需要由通知对应的拦截器来进行管理，而适配器的作用就是根据通知得到其对应的拦截器
再可以看看AfterReturningAdviceInterceptor拦截器
```
public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice, Serializable {  
  
    private final AfterReturningAdvice advice;  
  
  
    /**  
     * Create a new AfterReturningAdviceInterceptor for the given advice.  
     * @param advice the AfterReturningAdvice to wrap  
     */  
    public AfterReturningAdviceInterceptor(AfterReturningAdvice advice) {  
        Assert.notNull(advice, "Advice must not be null");  
        this.advice = advice;  
    }  
  
    @Override  
    public Object invoke(MethodInvocation mi) throws Throwable {  
        Object retVal = mi.proceed();  
        this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());  
        return retVal;  
    }  
  
}  
```
这个后置通知拦截器就是在先进入到下一个拦截器后，然后再调用的对应通知AfterReturningAdvice的afterReturning方法
所以我们可以看到，这几个拦截器实现的功能其实非常类似于动态代理设计模式里面实现了InvocationHandler接口的那个类，比如如下：
```
public class ProxyHandler implements InvocationHandler {  
  
    private Object target;  
  
    public ProxyHandler(Object target) {  
        this.target = target;  
    }  
  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
  
        System.out.println("call before target");  
  
        Object proxyObject = method.invoke(target, args);   //这里是要调用目标对象target的目标方法method  
  
        System.out.println("call after target");  
  
        return proxyObject;  
    }  
}  
```
这里目标target的方法被调用前，会被这个类的invoke方法拦截，method.invoke(target, args) 是调用目标方法，这里可以在其前后做一些增强方法
