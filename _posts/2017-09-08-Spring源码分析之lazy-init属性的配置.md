---
layout:     post
title:      Spring源码分析之lazy-init属性的配置
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

AbstractApplicationContext类默认在容器初始化的过程中就会执行依赖注入，即等价于配置lazy-init属性为false，bean的配置如下：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <bean id="studentService" class="com.verify.constant.StudentService" init-method="initMethod" lazy-init="false"/>
</beans>
```
容器的初始化是在AbstractApplicationContext的refresh()方法中执行的，如下代码对lazy-init进行了处理：
```
finishBeanFactoryInitialization(beanFactory);
```
跟踪下去可以找到真正的读取lazy-init属性进行懒加载相关处理的地方
```
@Override
	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							@Override
							public Boolean run() {
								return ((SmartFactoryBean<?>) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged(new PrivilegedAction<Object>() {
						@Override
						public Object run() {
							smartSingleton.afterSingletonsInstantiated();
							return null;
						}
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```
可以看到代码中对所有注册的bean，即this.beanDefinitionNames，对于每个bean都会做如下判断，如果成立就会执行依赖注入：
```
if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit())
```
可以看出，只有单例的bean才有可能在容器初始化的时候就完成依赖注入，当lazy-init属性不配置(默认值)或者配置为false的时候，上述if就会成立，当然这里默认不配置abstract属性，所以它默认也是false。if成立，就会执行getBean从而进行依赖注入，这样在容器初始化的过程中就已经实例化了Bean，当真正的请求bean的时候，其实只是从缓存中读取而已。
而如果lazy-init属性配置为true，那么就会进行懒加载了，这样在容器初始化的过程中不会进行依赖注入，只有当第一个getBean的时候才会实例化Bean。
最后抛出一个我还没有搞明白的问题：
书上说的是容器在初始化的过程中默认情况下其实并没有发生依赖注入，而是在第一次getBean的时候才会进行依赖注入，但是这个说法与上述AbstractApplicationContext的容器初始化过程好像是不一致的？还是说AbstractApplicationContext默认是进行依赖注入，但是BeanFactory默认不进行依赖注入？


