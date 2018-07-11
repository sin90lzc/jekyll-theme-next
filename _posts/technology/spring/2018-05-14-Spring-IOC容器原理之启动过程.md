众所周知，Spring为IOC容器定义了两类接口：BeanFactory和ApplicationContext。BeanFactory实现了基本依赖反转控制，而ApplicationContext则在BeanFactory的基础上提供了更多高级特性。事实上，ApplicationContext依赖反转控制的实现使用的装饰模式或者是委派模式，装饰和委派的是BeanFactory的实现。这里不再将两类容器接口实现分开讨论，而是只讲述ApplicationContext，原因是ApplicationContext是我们使用得最多的容器，而且其实现已经体现出BeanFactory的实现了。

这里，我们从ApplicationContext的启动过程讲起，而关于ApplicationContext加载BeanDefinition的过程将另起篇章讲述。要去分析源码实现，必须基于一个实现类进行分析，在这里，我不会避重就轻，而去选择较简单的ApplicationContext实现，而是选择功能更强大和复杂的AnnotationConfigWebApplicationContext进行分析。

## Web应用的容器创建

我们选择了AnnotationConfigWebApplicationContext作为ApplicationContext的实现类去分析，从名字中可以看出，这个容器实现版本适用了Web应用环境。在Spring中，Web应用的容器创建是在org.springframework.web.context.ContextLoaderListener中进行的，ContextLoaderListener实现了ServletContextListener的接口，故容器的创建是在contextInitialized()阶段进行。如以下代码清单:

```java
	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}
```

实际的ApplicationContext的创建、配置及触发启动都是交由ContextLoaderListener的父类ContextLoader来处理的。下面将以时序图的方式来表达这个过程：

![](https://sin90lzc.github.io/images/spring/webac_startup.png)

从上图清晰地可以看出，ApplicationContext配置完必要的信息之后，调用AbstractApplicationContext的refresh()方法启动容器。

在refresh()方法中，清晰地定义了各个启动子过程的模板方法。下面就按各子过程进行详细讲解吧。

## refresh()

以下流程图展示了容器启动的完整过程：

![](https://sin90lzc.github.io/images/spring/refresh.png)


### prepareRefresh()

在这里完成容器启动准备工作，包括容器状态初始化，PropertySources（属性源）的初始化，验证环境配置属性，准备早期事件存储，这个过程非常简单，直接看代码清单：

```java
	protected void prepareRefresh() {
		//启动状态初始化
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isInfoEnabled()) {
			logger.info("Refreshing " + this);
		}

		//初始化属性源
		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		//验证环境配置属性
		// Validate that all properties marked as required are resolvable
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		// 准备早期事件存储，这些事件将在multicaster事件发布器可用之后发布
		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
	}
```

### obtainFreshBeanFactory()

这个阶段是容器启动最为核心的阶段，在obtainFreshBeanFactory()中完成了容器的创建及BeanDefinition的加载。

```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}	
```

refreshBeanFactory()方法是在AbstractRefreshableApplicationContext中实现，定义了容器创建及BeanDefinition加载的模块方法：

```java	
	@Override
	protected final void refreshBeanFactory() throws BeansException {
		//需要先把旧有的BeanFactory的资源回收
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();//关闭BeanFactory
		}
		try {
			//创建BeanFactory实例，这里默认使用的实现是DefaultListableBeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			//为BeanFactory的配置提供了扩展模板方法
			customizeBeanFactory(beanFactory);
			//定义了BeanDefinition加载的模板方法
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

从上面的代码清单中可以看出，整个过程非常清晰，先把旧有的BeanFactory回收，再创建新的BeanFactory，并通过子类实现`loadBeanDefinitions(beanFactory);`方法完成BeanDefinition的加载。由于BeanDefinition的加载方式有多种途径，如从XML上加载，或者是从@Configuration配置类中加载等，每种加载途径都会有对应的实现子类，如AnnotationConfigWebApplicationContext就是实现了从@Configuration配置类中加载BeanDefinition。由于BeanDefinition的加载是Spring容器实现的一个非常关键的过程，而且其实现过程是复杂的，为了避免过度深入细化分析，而影响读者对本文关注点容器启动过程的理解，故在此不展开详述，将会在另一篇文章中讲述AnnotationConfigWebApplicationContext的BeanDefinition的加载原理。在这里，读者只需要心中有个概念，`loadBeanDefinitions(beanFactory);`是实现BeanDefinition加载过程的地方，当该方法执行完成，BeanFactory中就已经注册了所有的BeanDefinition了。

### prepareBeanFactory()

在prepareBeanFactory()阶段用于配置BeanFactory。细致的读者可能发现在`obtainFreshBeanFactory()`阶段也有一个机会让我们配置BeanFactory，那就是模块方法`customizeBeanFactory(beanFactory);`了，这两个BeanFactory的模块方法虽然都用于配置BeanFactory，但两者是有区别的，`customizeBeanFactory(beanFactory);`发生在BeanDefinition加载之前，而prepareBeanFactory()则发生在BeanDefinition加载之后及Bean实例创建及依赖注入之前，因此，`customizeBeanFactory(beanFactory);`阶段主要配置的是对BeanDefinition加载有影响的属性，而在prepareBeanFactory()阶段主要配置的是对Bean实例创建或依赖注入过程有影响的属性，如后处理器，PropertiesEditor的注册机等，如代码所示：

```java
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		//Spring EL表达式解析器
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		// PropertiesEditor的注册机配置
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		// 添加ApplicationContextAwareProcessor后处理器，
		// 完成EnvironmentAware，EmbeddedValueResolverAware，ResourceLoaderAware，
		// MessageSourceAware，ApplicationEventPublisherAware，ApplicationContextAware
		// 接口的注入功能
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		
		// 定义应该忽略进行依赖检查和注入的接口，要忽略的原因可能是这些接口在依赖注入之前已经完成注入，
		// 比如在后处理器阶段
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		// 定义了哪些Bean实例是可以用于依赖注入的
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// 往容器中注册默认的Environment实例Bean
		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```
从源码中可以看到，在该阶段完成了以下事情，供后续使用：

* 配置了使用的ClassLoader
* 配置了Spring EL表达式解析器
* 配置了PropertiesEditor的注册机
* 配置了需要忽略的依赖检查和注入的接口
* 配置了ApplicationContextAwareProcessor后处理器，完成部分Aware接口的功能
* 注册了BeanFactory.class/ResourceLoader.class/ApplicationEventPublisher.class/ApplicationContext.class的实例，可用于这些类型的依赖注入
* 注册了Environment的实例Bean

### postProcessBeanFactory()

如果说prepareBeanFactory()阶段是比较常规的BeanFactory配置过程，那么postProcessBeanFactory()阶段则可以认为是用于不太常规的BeanFactory的配置过程。其实这两个过程都是完成BeanFactory的配置，而且配置的时机是完全一致的，因此两者可以说是一样的，只是postProcessBeanFactory()阶段更适合用于配置一些不是常规的配置，正如AnnotationConfigWebApplicationContext的子类AbstractRefreshableWebApplicationContext的实现：

```java
	@Override
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext, this.servletConfig));
		beanFactory.ignoreDependencyInterface(ServletContextAware.class);
		beanFactory.ignoreDependencyInterface(ServletConfigAware.class);

		WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
		WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext, this.servletConfig);
	}
```

在这个实现中配置了与web应用紧密相关的一些配置，因此这些配置并不是常规配置了，因此Spring把这类配置放到了postProcessBeanFactory()阶段。上面代码完成了以下事情：

* 配置了ServletContextAwareProcessor后处理，完成ServletContextAware和ServletConfigAware的注入。
* 配置了需要忽略ServletContextAware和ServletConfigAware依赖检查和注入的接口，因为它们的依赖注入在ServletContextAwareProcessor中完成了
* 注册了与web应该紧密关联的Scope，如SessionScope，RequestScope，ServletScope
* registerEnvironmentBeans()完成了将ServleContext和ServletConfig实例注册到容器，交由容器管理，另外也把ServletContext的配置项也整理为Map，交由容器管理。

### invokeBeanFactoryPostProcessors()

正如其名，该过程用于回调BeanFactoryPostProcessor。该过程的实现委派给了PostProcessorRegistrationDelegate来完成。其实现过程描述如下：

1. 找出所有BeanDefinitionRegistryPostProcessor，并调用。BeanDefinitionRegistryPostProcessor的配置来源可能来自于BeanFactory自身，也可能来源于BeanDefinition，BeanDefinitionRegistryPostProcessor优先调用BeanFactory自身配置的Processor，再调用BeanDefinition中的Processor。实现了PriorityOrdered接口的将得到优先排序并处理，接着是实现了Ordered接口的，最后才是调用无优先级要求的Processor。值得一提的是，@Configuration的配置Bean的处理是由一个BeanDefinitionRegistryPostProcessor来完成的，这将在介绍AnnotationConfigWebApplicationContext的BeanDefinition加载过程中详细讲述。
2. 找出所有BeanFactoryPostProcessor，并调用。其处理优先级与BeanDefinitionRegistryPostProcessor类似，也是先处理BeanFactory自身配置的Processor，再处理实现PriorityOrdered接口，接着是Ordered接口，最后才是其他Processor。

### registerBeanPostProcessors()

正如其名，该过程用于注册BeanPostProcessor。BeanPostProcessor的来源于BeanDefinition配置。BeanPostProcessor的注册顺序取决于PriorityOrdered和Ordered接口，PriorityOrdered永远优先于Ordered接口的处理。

另外，值得注意的是在该阶段，Spring以硬编码的方式注册了ApplicationListenerDetector后处理器，这个后处理器是在Bean实例化之后，探测Bean是否实现了ApplicationListener接口，并注册ApplicationListener接口，以接收容器事件。

### initMessageSource()

该阶段用于初始化MessageSource，MessageSource是用于国际化信息解析的。

### initApplicationEventMulticaster()

该阶段用于初始化ApplicationEventMulticaster，ApplicationEventMulticaster用于发布ApplicationEvent。ApplicationEventMulticaster在Spring的唯一实现类是SimpleApplicationEventMulticaster，该实现类使用了观察者模式，并支持异步监听事件处理，异步处理交由Executor来执行，具体实现请参考SimpleApplicationEventMulticaster。

### onRefresh()



### registerListeners()

该阶段完成向ApplicationEventMulticaster中注册ApplicaitonEventListener，并把在ApplicationEventMulticaster初始化完成之前的事件发布。

### finishBeanFactoryInitialization()

在该阶段完成其余单例Bean的实例化，如配置了`lazy=false`的BeanDefinition的实例化。这个实例化过程其实也就是调用了getBean()方法来完成的。

```java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		//让用户通过BeanDefinition配置的方式定义ConversionService，以覆盖默认实现
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// 这里就是实例化lazy=false的BeanDefinition的地方了
		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
```

从以上代码可以看出，这里提供了一个机会让用户通过BeanDefinition配置的方式定义ConversionService的实例Bean。

### finishRefresh()

这是容器启动过程的最后阶段，它完成了LifecycleProcessor的初始化，并启动实现了SmartLifecycle的Bean，并发布ApplicationEvent[ContextRefreshedEvent]，以通知ApplicationEventListener容器已经启动完成。

```java
	protected void finishRefresh() {
		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

这里可以关注一下LifecycleProcessor的实现。LifecycleProcessor的实现类是DefaultLifecycleProcessor，DefaultLifecycleProcessor.onRefresh()调用会自动执行实现了SmartLifecycle接口Bean实例的start()方法。如果一些组件的启动需要等待容器完全启动完成之后才能启动的时候，比如一些消费组件，这时就可以通过实现SmartLifecycle的start()接口以启动了。

