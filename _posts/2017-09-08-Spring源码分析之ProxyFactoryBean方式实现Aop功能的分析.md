---
layout:     post
title:      Spring源码分析之ProxyFactoryBean方式实现Aop功能的分析
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

实现Aop功能有两种方式，
1. ProxyFactoryBean方式： 这种方式是通过配置实现
2. ProxyFactory方式：这种方式是通过编程实现
这里只说ProxyFactoryBean方式
首先说下具体的配置，一个例子如下：

```
<bean id="testAdvisor" class="com.abc.TestAdvisor"
	<property name="pointcut"  ref="bookPointcut"/> 
        <property name="advice" ref="aroundMethod"></property>
</bean>
<bean id="testAop" class="org.springframeword.aop.ProxyFactoryBean">
      <property name="proxyInterfaces">
              <value>com.test.AbcInterface</value>
      </property>
      <property name="target">
              <bean class="com.abc.TestTarget"/>
      </property>
      <property name="interceptorNames">
              <list>
                        <value>testAdvisor</value>  
              </list>
      </property>
</bean>
```
上述配置中，testAdvisor是配置了一个通知器，该通知器配置了pointcut，即执行该通知需要满足的条件，还配置了匹配条件时要执行的方法，target配置的是要被增强的目标对象，interceptorNames配置的是一些通知，用来增强目标对象。proxyInterfaces配置的是需要代理的接口名的字符串数组。如果没有提供，将为目标类使用一个CGLIB代理，即这个接口的配置将会影响是用JDK还是CGLIB来创建目标对象的代理对象。
首先看下ProxyFactoryBean的getObject方法
```
@Override
	public Object getObject() throws BeansException {
		initializeAdvisorChain();
		//生成代理对象时,因为Spring中有singleton类型和prototype类型这两种不同的Bean,所以要对代理对象的生成做一个区分
		if (isSingleton()) {
			//生成singleton的代理对象,这个方法是ProxyFactoryBean生成AOPProxy代理对象的调用入口
			//代理对象会封装对target目标对象的调用,也就是说针对target对象的方法调用行为会被这里生成的代理对象所拦截
			return getSingletonInstance();
		}
		else {
			if (this.targetName == null) {
				logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
						"Enable prototype proxies by setting the 'targetName' property.");
			}
			return newPrototypeInstance();
		}
	}
```
这个方法其实就是用来为目标对象生成代理对象的
initializeAdvisorChain是初始化通知器链，即从上述配置中读取interceptorNames参数的值就可以拿到所有为目标对象配置的通知器，该方法的代码如下：
```
/**
	 * Create the advisor (interceptor) chain. Advisors that are sourced
	 * from a BeanFactory will be refreshed each time a new prototype instance
	 * is added. Interceptors added programmatically through the factory API
	 * are unaffected by such changes.
	 * 初始化通知器链,通知器链封装了一系列的拦截器,这些拦截器都需要从配置中读取,然后为代理对象的生成做好准备
	 */
	private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
		//这个标志位是用来表示通知器链是否已经初始化,初始化的工作发生在应用第一次通过ProxyFactoryBean去获取代理对象的时候
		if (this.advisorChainInitialized) {
			return;
		}

		if (!ObjectUtils.isEmpty(this.interceptorNames)) {
			if (this.beanFactory == null) {
				throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) " +
						"- cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
			}

			// Globals can't be last unless we specified a targetSource using the property...
			if (this.interceptorNames[this.interceptorNames.length - 1].endsWith(GLOBAL_SUFFIX) &&
					this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
				throw new AopConfigException("Target required after globals");
			}

			// Materialize interceptor chain from bean names.
			//这里是添加Advisor链的调用,是通过interceptorNames属性进行配置的
			//this.interceptorNames就是配置中配置的所有通知器
			for (String name : this.interceptorNames) {
				if (logger.isTraceEnabled()) {
					logger.trace("Configuring advisor or advice '" + name + "'");
				}

				if (name.endsWith(GLOBAL_SUFFIX)) {
					if (!(this.beanFactory instanceof ListableBeanFactory)) {
						throw new AopConfigException(
								"Can only use global advisors or interceptors with a ListableBeanFactory");
					}
					addGlobalAdvisor((ListableBeanFactory) this.beanFactory,
							name.substring(0, name.length() - GLOBAL_SUFFIX.length()));
				}

				else {
					// If we get here, we need to add a named interceptor.
					// We must check if it's a singleton or prototype.
					//如果程序在这里被调用,那么需要加入命名的拦截器advice,并且需要检查这个Bean是singleton还是prototype
					Object advice;
					if (this.singleton || this.beanFactory.isSingleton(name)) {
						// Add the real Advisor/Advice to the chain.
						//取得advisor的地方,是通过beanFactory取得的,把intercepNames这个List中的interceptor的名字交给BeanFactory,然后通过getBean去获取
						advice = this.beanFactory.getBean(name);
					}
					else {
						// It's a prototype Advice or Advisor: replace with a prototype.
						// Avoid unnecessary creation of prototype bean just for advisor chain initialization.
						advice = new PrototypePlaceholderAdvisor(name);
					}
					addAdvisorOnChainCreation(advice, name);
				}
			}
		}

		this.advisorChainInitialized = true;
	}
```
其他的getSingletonInstance方法和newPrototypeInstance类其实就是构造代理对象
其中getSingletonInstance方法的代码如下：
```
private synchronized Object getSingletonInstance() {
		if (this.singletonInstance == null) {
			this.targetSource = freshTargetSource();
			if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
				// Rely on AOP infrastructure to tell us what interfaces to proxy.
				//根据AOP框架来判断需要代理的接口
				Class<?> targetClass = getTargetClass();
				if (targetClass == null) {
					throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
				}
				//设置代理对象的接口
				setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
			}
			// Initialize the shared singleton instance.
			super.setFrozen(this.freezeProxy);
			//createAopProxy()方法可能会返回ObjenesisCglibAopProxy对象,也可能会返回JdkDynamicAopProxy对象
			//然后getProxy方法会根据ObjenesisCglibAopProxy或者JdkDynamicAopProxy对象的getProxy方法来生成最终的代理对象
			//这就是所谓的,Spring生成代理对象的两种方式,一种是CGLIB,一种是JDK
			this.singletonInstance = getProxy(createAopProxy());	//这里的方法会使用ProxyFactory来生成需要的Proxy,通过createAopProxy返回的AopProxy来得到代理对象
		}
		return this.singletonInstance;
	}
```
注意this.singletonInstance = getProxy(createAopProxy());这行代码
createAopProxy()方法可能会返回ObjenesisCglibAopProxy对象,也可能会返回JdkDynamicAopProxy对象，这个逻辑是在DefaultAopProxyFactory
类中实现的，逻辑如下：
```
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	//config里面封装了想要生成的代理对象的信息
	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();	//首先要从AdvisedSupport对象中取得配置的目标对象,如果目标对象为空,则直接抛出异常,因为连目标对象都没有,还为谁创建代理对象
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			//关于AopProxy代理对象的生成,需要考虑使用哪种生成方式,如果目标对象是接口类,那么适合使用JDK来生成代理对象,否则spring会使用CGLIB来生成目标对象的代理对象
			//对于具体的AopProxy代理对象的生成,最终并不是由DefaultAopProxyFactory来完成,而是分别由JdkDynamicAopProxy和ObjenesisCglibAopProxy完成
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);	//使用JDK来生成AOPProxy代理对象
			}
			return new ObjenesisCglibAopProxy(config);	//使用第三方CGLIB来生成AOPProxy代理对象
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```
可以看到，它会根据目标类是不是接口等信息来判定使用ObjenesisCglibAopProxy还是JdkDynamicAopProxy

然后getProxy方法会根据ObjenesisCglibAopProxy或者JdkDynamicAopProxy对象的getProxy方法来生成最终的代理对象
这就是所谓的,Spring生成代理对象的两种方式,一种是CGLIB,一种是JDK
下面说说用JdkDynamicAopProxy方式生成的代理对象的拦截方式，它实际用的就是JDK的动态代理
我们知道，动态代理拦截的入口是实现了InvocationHandler接口后的invoke方法，即所有对目标方法的调用首先会被invoke方法拦截
而JdkDynamicAopProxy方式实现的动态代理的拦截入口也是该类的invoke方法，该类的部分方法如下：
```
/**
 * JDK-based {@link AopProxy} implementation for the Spring AOP framework
 *InvocationHandler接口的invoke方法就是拦截回调的入口,即对目标方法的调用会先被invoke方法拦截,并在invoke方法里面来调用目标方法
 */
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

	public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
		Assert.notNull(config, "AdvisedSupport must not be null");
		if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("No advisors and no TargetSource specified");
		}
		this.advised = config;
	}


	@Override
	public Object getProxy() {
		return getProxy(ClassUtils.getDefaultClassLoader());
	}

	@Override
	public Object getProxy(ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		//首先从advised对象中取得代理对象的代理接口配置
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		//第三个参数需要实现InvocationHandler接口和invoke方法,这个invoke方法是Proxy代理对象的回调方法
		//这种方式其实就是用JDK的动态代理来为目标对象创建代理对象,对目标对象方法的调用就是由这个代理对象来调用的
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}

	/**
	 * Implementation of {@code InvocationHandler.invoke}.
	 * <p>Callers will see exactly the exception thrown by the target,
	 * unless a hook method throws an exception.
	 * 拦截回调入口
	 */
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Class<?> targetClass = null;
		Object target = null;

		try {
			//如果目标对象没有实现Object类的基本方法:equals
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
				return equals(args[0]);
			}
			//如果目标对象没有实现Object类的基本方法:hashcode
			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
				return hashCode();
			}
			else if (method.getDeclaringClass() == DecoratingProxy.class) {
				// There is only getDecoratedClass() declared -> dispatch to proxy config.
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}
			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				//根据代理对象的配置来调用服务
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			// May be null. Get as late as possible to minimize the time we "own" the target,
			// in case it comes from a pool.
			//得到目标对象
			target = targetSource.getTarget();
			if (target != null) {
				targetClass = target.getClass();
			}

			// Get the interception chain for this method.	获取方法method的拦截器链
			// 拦截器链实际就是由一系列的Advice通知对象组成的
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			// Check whether we have any advice. If we don't, we can fallback on direct
			// reflective invocation of the target, and avoid creating a MethodInvocation.
			//如果没有定义拦截器链,就直接调用target对象的对应方法
			if (chain.isEmpty()) {
				// We can skip creating a MethodInvocation: just invoke the target directly
				// Note that the final invoker must be an InvokerInterceptor so we know it does
				// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);	//适配参数
				//调用target对象的对应方法
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				// We need to create a method invocation...
				//如果有拦截器的设定,那么需要调用拦截器之后才调用目标对象的相应方法,通过构造一个ReflectiveMethodInvocation来实现
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
				//沿着拦截器链继续前进
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				// Special case: it returned "this" and the return type of the method
				// is type-compatible. Note that we can't help if the target sets
				// a reference to itself in another returned object.
				retVal = proxy;
			}
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			if (target != null && !targetSource.isStatic()) {
				// Must have come from TargetSource.
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}

}
```
上述getProxy方法其实就是JdkDynamicAopProxy用来给目标对象生成代码对象的方法
而invoke就是对目标方法调用时的拦截入口
其中的
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
这行代码就是获取到目标对象所有的拦截器，为什么这里是获取拦截器？其实在上面初始化通知器链的时候拿到的都是配置的通知器，这个方法是要将这些通知器用对应的适配器
适配成对应的拦截器，至于为什么要做这个步骤，在我的另外一篇博客中说的很清楚了，地址如下：
http://blog.csdn.net/u011734144/article/details/73436539
这里转换成拦截器后，也并不是直接就要将该拦截器加入最终要执行的拦截器链中，还需要判断对应的通知是否应该执行，对应的代码片段如下：
```
//对配置的advisor通知器进行逐个遍历,这个通知器链都是配置在interceptorNames中的
		for (Advisor advisor : config.getAdvisors()) {
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					//registry.getInterceptors(advisor)是对从ProxyFactoryBean配置中得到的通知进行适配,从而得到相应的拦截器,再把它加入到前面设置好的list中去
					//从而完成所谓的拦截器注册过程
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
						if (mm.isRuntime()) {
							// Creating a new object instance in the getInterceptors() method
							// isn't a problem as we normally cache created chains.
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			else if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			else {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}
 ```
这里需要判断通知器中配置的切入点是否匹配当前要被调用的方法，即MethodMatchers.matches是否为true，只有匹配的通知才会将对应的拦截器加入到最终待执行的拦截器链中
接下来invoke方法中比较核心的就是如下代码：
retVal = invocation.proceed();
这个方法其实就是启动拦截器链的执行，依次执行每一个拦截器链，在每一个拦截器里面都会根据通知的类型来决定是先执行通知的方法还是先继续执行下一个拦截器，
拦截器中具体的执行逻辑也请参考我上面说的我的另外一篇文章
