在对Spring AOP原理进行解剖之前，先看看如何使用Spring AOP的API创建代理的过程，让大家有个感性的认识：

```java
public class AopTest {

	public static void main(String[] args) {
		// 1. AOP代理配置
		AdvisedSupport aopConfig=new AdvisedSupport();
		aopConfig.setInterfaces(AopInterface.class);// 1.1 配置需要代理的接口
		aopConfig.setTargetSource(new SingletonTargetSource(new Tim())); // 1.2 代理的目标对象，这里使用TargetSource去抽象目标对象
		aopConfig.addAdvisor(new DefaultPointcutAdvisor(Pointcut.TRUE, new MethodInterceptor() {
			@Override
			public Object invoke(MethodInvocation invocation) throws Throwable {
				String method=invocation.getMethod().getName();
				System.out.println(String.format("invoke method [%s] before!",method));
				Object result=invocation.proceed();
				System.out.println(String.format("invoke method [%s] after ,with result [%s]!",method,result));
				return result;
			}
		})); // 1.3 1个或多个Advisor，Advisor由Pointcut和Advice组成，Pointcut定义了切入点，而Advice描述了切点功能
		
		// 2. 通过AopProxyFactory创建AopProxy
		AopProxyFactory aopProxyFactory=new DefaultAopProxyFactory();
		AopInterface proxy = (AopInterface)aopProxyFactory.createAopProxy(aopConfig).getProxy();
		proxy.name();
		proxy.age();
	}
	
	public static class Tim implements AopInterface{
		@Override
		public String name() {
			return "Tim";
		}
		@Override
		public int age() {
			return 10;
		}
	}
	
	public interface AopInterface{
		public String name() ;
		public int age();
	}
}
```

从上面的注释中也可以看出，要使用Spring AOP创建一个代理对象只需要2步：1. AOP的配置信息，包括需要代理的接口、目标对象和切面。AOP配置信息由AdvisedSupport进行管理。2. 使用AopProxyFactory抽象工厂创建AopProxy，由AopProxy负责创建代理对象（在这里AopProxyFactory使用的是抽象工厂设计模式，而AopProxy则是工厂方法设计模式）。

由此可见，只要通过AdvisedSupport提供必要的代理配置信息，就能由DefaultAopProxyFactory为我们创建所需的代理对象了，这是多么简单的API使用啊。

下面就从AdvisedSupport和DefaultAopProxyFactory为切入口，探索Spring AOP的奥秘吧！

## AdvisedSupport

我们先来看看AdvisedSupport的继承关系：

![](https://sin90lzc.github.io/images/spring/AdvisedSupport.jpg)

AdvisedSupport主要实现的是两个接口Advised以及TargetClassAware，以及继承了ProxyConfig，下面看看这三个类都定义了什么：

```java
/**
	TargetClassAware定义了获取目标对象的class接口
*/
public interface TargetClassAware {
	Class<?> getTargetClass();
}

public interface Advised extends TargetClassAware {

	/**
	 * 是否冻结Advised的配置
	 */
	boolean isFrozen();

	/**
	 * 使用代理目标类的方式还是使用代理接口。
	 * 如果为true，则使用CGLib代理，如果为false，则使用Jdk代理
	 */
	boolean isProxyTargetClass();

	/**
	 * 返回需要代理的接口列表
	 */
	Class<?>[] getProxiedInterfaces();

	/**
	 * 判断intf是否有被代理
	 */
	boolean isInterfaceProxied(Class<?> intf);

	/**
	 * 获取目标源，TargetSource是对目标对象的封装
	 */
	TargetSource getTargetSource();
	
	/**
	 * 是否在调用代理对象方法期间通过ThreadLocal来暴露该代理对象，
	 * 目标对象可通过AopContext.currentProxy()来访问代理后的对象
	 */
	boolean isExposeProxy();

	/**
	 * 用于标识Advisor是否在应用前已经对Pointcut做过校验，
	 * 可避免在代理调用期间重复校验
	 * true: 已做校验，false：未做校验
	 */
	boolean isPreFiltered();

	Advisor[] getAdvisors();

	/**
	 * 添加切面
	 */
	void addAdvisor(Advisor advisor) throws AopConfigException;

	/**
	 * 添加切面
	 */
	void addAdvice(Advice advice) throws AopConfigException;
}

/**
 ProxyConfig主要实现了以下5个代理配置项的管理
*/
public class ProxyConfig implements Serializable {

	/**
	 * 代理目标类，而非代理接口，true意味着使用CGLIB代理
	 */
	private boolean proxyTargetClass = false;

	/**
	 * 
	 */
	private boolean optimize = false;

	/**
	 * 是否暴露Advised接口
	 */
	boolean opaque = false;
	/**
	 * 是否在调用代理对象方法期间通过ThreadLocal来暴露该代理对象，
	 * 目标对象可通过AopContext.currentProxy()来访问代理后的对象
	 */
	boolean exposeProxy = false;

	/**
	 * 是否冻结Advisor的配置，一旦frozen为true就不能修改Advisor列表
	 */
	private boolean frozen = false;
}
```

从上面代码的注释中可以看到每个配置项的作用了，它们都会影响生成的代理对象。

在AdvisedSupport配置中，有一个称为Advisor的概念，可以认为这是Spring提供的一个org.aopalliance.aop.Advice提供者，Advisor和Advice都可以认为是切面功能的抽象概念。从Advisor的继承关系可以看出它有两条主线，一个是IntroductionAdvisor，另一个是PointcutAdvisor。

![](https://sin90lzc.github.io/images/spring/Advisor.jpg)

两者的接口定义如下：

```java
public interface IntroductionAdvisor extends Advisor, IntroductionInfo {

	/**
	 * ClassFilter定义了类型匹配规则
	 */
	ClassFilter getClassFilter();

	/**
	 * 用于校验该Advisor是否适用于所有AdvisedSupport中配置的代理接口
	 */
	void validateInterfaces() throws IllegalArgumentException;

}

public interface PointcutAdvisor extends Advisor {
	
	/**
	 * Pointcut不旦定义了类型匹配规则，还定义了方法匹配规则
	 */
	Pointcut getPointcut();

}

/**
 * Pointcut不旦定义了类型匹配规则，还定义了方法匹配规则
 */
public interface Pointcut {

	ClassFilter getClassFilter();

	MethodMatcher getMethodMatcher();

	Pointcut TRUE = TruePointcut.INSTANCE;

}
```

如果说Advisor抽象了切面功能，那么IntroductionAdvisor和PointcutAdvisor在Advisor的基础上抽象了切入点规则，IntroductionAdvisor和PointcutAdvisor才是完整的切面抽象。但是IntroductionAdvisor只能对类型进行约束，而PointcutAdvisor可以对类型及方法进行约束。所以PointcutAdvisor更适用于用户的使用习惯。

## DefaultAopProxyFactory

DefaultAopProxyFactory实现了AopProxyFactory接口，从接口定义中可以看出AopProxyFactory使用了抽象工厂设计模式，它的接口定义返回了另一个工厂类AopProxy：

```java
public interface AopProxyFactory {

	AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException;

}

public interface AopProxy {

	Object getProxy();

	Object getProxy(ClassLoader classLoader);

}
```

由此可见，AopProxy才是真正的代理工厂，而AopProxyFactory只是根据AdvisedSupport的配置信息决定使用哪种代理生成策略（在这里只有JDK和CGLIB两种代理策略）。正如DefaultAopProxyFactory实现的那样：

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
	
	private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
		Class<?>[] ifcs = config.getProxiedInterfaces();
		return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
	}

}
```

DefaultAopProxyFactory的实现逻辑很简单，只是简单地根据AdvisedSupport中的配置信息来决定代理策略：

1. 当AdvisedSupport配置了optimize或ProxyTargetClass或AdvisedSupport中并没有需要代理的用户接口时，并且目标类型不是接口和Proxy实现时，就使用CGLIB代理策略。
2. 其他情况都使用JDK代理策略

### JdkDynamicAopProxy

下面看看JDK代理策略JdkDynamicAopProxy的实现，

```java
	@Override
	public Object getProxy(ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
```

AopProxyUtils.completeProxiedInterfaces()方法用于补全所有需要代理的接口列表，包括：

1. 代理类型（ProxyClass）是一个接口时，需要把该接口添加到代理接口列表
2. 代理类型是一个Proxy实例时，需要把代理类型实现的所有接口都添加到代理接口列表
3. 另外需要把SpringProxy这个标识接口添加到代理接口列表，用于标识生成的代理对象是由Spring AOP生成的。
4. 如果AdvisedSupport配置了是非透明代理时，需要添加Advised接口到代理接口列表，方便在生成的代理对象中查看代理配置信息。

`findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);`调用用于确定代理的接口列表中是否定义了equals或hashCode接口方法。

从`Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);`中看出JdkDynamicAopProxy同时也实现了InvocationHandler接口，这是Jdk代理策略最关键的实现了：

```java
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Class<?> targetClass = null;
		Object target = null;

		try {
			// 接口中没有定义equals方法，就使用JdkAopProxy的equals实现
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				return equals(args[0]);
			}
			// 接口中没有定义hashCode方法，就使用JdkAopProxy的hashCode实现
			if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				return hashCode();
			}
			// 如果调用的是Advised中的接口，使用反射调用this.advised中对应的方法
			if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			// 获取代理目标
			target = targetSource.getTarget();
			if (target != null) {
				targetClass = target.getClass();
			}

			// 这是很关键的步骤：生成拦截器链
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			// 没有拦截器链的时候，则直接反射调用对应的方法
			if (chain.isEmpty()) {
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				// MethodInvocation使用了命令模式，调用拦截器链并最终调用目标对象方法
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				retVal = invocation.proceed();
			}

			// 处理目标对象方法使用return this返回值时的处理，此时应返回代理对象
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
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
```

在这段代码实现中，有两个最关键的步骤，一个是从AdvisedSupport中获取拦截器链，另一个是ReflectiveMethodInvocation如何通过命令模式调用拦截器链并最终调用目标对象方法的实现原理。

通过阅读`AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice()`源码不难发现，获取拦截器链的职责最终委派给`DefaultAdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice()`来完成。该方法的主要是实现了把AdvisedSupport中配置的Advisors转换为MethodInterceptor或InterceptorAndDynamicMethodMatcher，并以列表的形式返回。InterceptorAndDynamicMethodMatcher是由MethodInterceptor和MethodMatcher组装而成，用于在运行时动态切入点校验。至于如何把Advisor转换为MethodInterceptor，是通过AdvisorAdapter来完成的，任何Advice都必须有对应AdvisorAdapter才能实现这个转换过程，而AdvisorAdapter是通过GlobalAdvisorAdapterRegistry完成全局注册的。如下代码所示：

```java
	@Override
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, Class<?> targetClass) {

		List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

		for (Advisor advisor : config.getAdvisors()) {
			// PointcutAdvisor需要进行Pointcut检验才能作为拦截器返回
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					// advisor -> MethodInterceptor的转换过程
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
						if (mm.isRuntime()) {
							// Pointcut需要运行时校验时，拦截器需要保留MethodMatcher以供运行时检查
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

		return interceptorList;
	}

```

下面再来分析ReflectiveMethodInvocation的实现。ReflectiveMethodInvocation使用了命令模式，它维护着所有代理方法调用的所有信息和状态，如目标对象，调用的方法，方法参数列表，以及拦截器链，并维护着当前调用的拦截器链的索引。ReflectiveMethodInvocation和MethodInterceptor使用了双分派模式，使得可以重复执行ReflectiveMethodInvocation的调用命令以实现递归调用，直到拦截器链调用完毕，并反射调用目标方法以结束命令的执行。下面贴出其实现源码：

```java
	public Object proceed() throws Throwable {
		//	拦截器链执行完毕时，反射调用目标对象方法
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// 运行时切入点动态校验
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// 动态校验失败，跳过该拦截器
				return proceed();
			}
		}
		else {
			// 简单地执行拦截器即可
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```

> 在本文中只是介绍了Jdk的代理策略，CGLIB的代理策略与此实现类似。另外，Spring还有另一种代理方式ASPECTJ，基于时间关系，有机会再补上这两种策略吧。



