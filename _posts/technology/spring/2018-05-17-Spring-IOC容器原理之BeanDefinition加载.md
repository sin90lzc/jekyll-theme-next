在《Spring-IOC容器原理之启动过程》一文中讲到，负责BeanDefinition加载过程都封装在模板方法`loadBeanDefinitions(beanFactory);`中，下面就以AnnotationConfigWebApplicationContext的这个较为复杂的容器实现，从这个方法为起点，开始BeanDefinition加载过程的分析之旅。

## BeanDefinition的加载过程

直接上源码：

```java
	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) {
		// 使用的BeanDefinitionReader是AnnotatedBeanDefinitionReader		
		AnnotatedBeanDefinitionReader reader = getAnnotatedBeanDefinitionReader(beanFactory);
		// 使用ClassPathBeanDefinitionScanner去扫描指定package下的BeanDefinition，并注册到容器
		ClassPathBeanDefinitionScanner scanner = getClassPathBeanDefinitionScanner(beanFactory);

		// BeanNameGenerator定义了BeanName生成策略
		BeanNameGenerator beanNameGenerator = getBeanNameGenerator();
		if (beanNameGenerator != null) {
			reader.setBeanNameGenerator(beanNameGenerator);
			scanner.setBeanNameGenerator(beanNameGenerator);
			beanFactory.registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
		}

		// ScopeMetadataResolver用于解析@Scope注解
		ScopeMetadataResolver scopeMetadataResolver = getScopeMetadataResolver();
		if (scopeMetadataResolver != null) {
			reader.setScopeMetadataResolver(scopeMetadataResolver);
			scanner.setScopeMetadataResolver(scopeMetadataResolver);
		}

		if (!this.annotatedClasses.isEmpty()) {
			if (logger.isInfoEnabled()) {
				logger.info("Registering annotated classes: [" +
						StringUtils.collectionToCommaDelimitedString(this.annotatedClasses) + "]");
			}
			//AnnotatedBeanDefinitionReader加载带有BeanDefinition注解的类
			reader.register(this.annotatedClasses.toArray(new Class<?>[this.annotatedClasses.size()]));
		}

		if (!this.basePackages.isEmpty()) {
			if (logger.isInfoEnabled()) {
				logger.info("Scanning base packages: [" +
						StringUtils.collectionToCommaDelimitedString(this.basePackages) + "]");
			}
			// ClassPathBeanDefinitionScanner扫描指定package下的类文件
			scanner.scan(this.basePackages.toArray(new String[this.basePackages.size()]));
		}

		// 对configLocations配置的处理，优先使用AnnotatedBeanDefinitionReader去解析，
		// 解析失败将使用ClassPathBeanDefinitionScanner去解析
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			for (String configLocation : configLocations) {
				try {
					Class<?> clazz = getClassLoader().loadClass(configLocation);
					if (logger.isInfoEnabled()) {
						logger.info("Successfully resolved class for [" + configLocation + "]");
					}
					reader.register(clazz);
				}
				catch (ClassNotFoundException ex) {
					if (logger.isDebugEnabled()) {
						logger.debug("Could not load class for config location [" + configLocation +
								"] - trying package scan. " + ex);
					}
					int count = scanner.scan(configLocation);
					if (logger.isInfoEnabled()) {
						if (count == 0) {
							logger.info("No annotated classes found for specified class/package [" + configLocation + "]");
						}
						else {
							logger.info("Found " + count + " annotated classes in package [" + configLocation + "]");
						}
					}
				}
			}
		}
	}
```

从以上源码可以看出，实现BeanDefinition加载的类有两个：AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner。

AnnotatedBeanDefinitionReader通过解析类信息来加载BeanDefinition，这些类是BeanDefinition配置的载体，可能带有一些BeanDefinition相关的注解，如@Configuration,@Bean等，甚至没有任何注解，这时加载的将是最普通的BeanDefinition，它将通过构造函数完成实例化（请参考Spring-IOC容器原理之Bean创建过程）。

ClassPathBeanDefinitionScanner通过扫描指定package下的class文件，把满足筛选条件的class作为BeanDefinition配置的载体，并把class信息转换为BeanDefinition后，注册到容器。

下面分别看看这两个类的实现。

### AnnotatedBeanDefinitionReader实现

AnnotatedBeanDefinitionReader的核心方法是`registerBean(Class<?> annotatedClass, String name, Class<? extends Annotation>... qualifiers)`:

```java
	public void registerBean(Class<?> annotatedClass, String name, Class<? extends Annotation>... qualifiers) {
	
		// 创建BeanDefinition，这里的实现类是AnnotatedGenericBeanDefinition
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
		
		// conditionEvaluator用于获取@Conditional注解中配置的Condition实例，
		// 并通过Condition判断该类是否满足条件约束，只有满足条件约束，才能对其进行解析
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}

		// 解析@Scope注解，解析出ScopeName和ScopeProxyMode
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
		// 配置BeanDefinition的scope
		abd.setScope(scopeMetadata.getScopeName());
		
		// beanName生成
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
		
		// 完成通用注解的配置
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				if (Primary.class == qualifier) {
					abd.setPrimary(true);
				}
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true);
				}
				else {
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}

		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		// ScopeProxyMode不是ScopeProxyMode.NO的时候
		// 对BeanDefinition进行增强，
		// 使用ScopeProxyFactoryBean作为实现类，用于生成一个代理Bean
		// 该代理Bean主要增加ScopedObject接口的实现，
		// 代理委派给DefaultScopedObject来完成
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		
		//向容器注册BeanDefinition
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}
```

在这里，向容器注册的BeanDefinition的具体实现是AnnotatedGenericBeanDefinition，它继承了GenericBeanDefinition，并在GenericBeanDefinition的基础上新增了AnnotatedBeanDefinition接口的实现：

```java
public interface AnnotatedBeanDefinition extends BeanDefinition {

	/**
	 * Obtain the annotation metadata (as well as basic class metadata)
	 * for this bean definition's bean class.
	 * @return the annotation metadata object (never {@code null})
	 */
	AnnotationMetadata getMetadata();

	/**
	 * Obtain metadata for this bean definition's factory method, if any.
	 * @return the factory method metadata, or {@code null} if none
	 * @since 4.1.1
	 */
	MethodMetadata getFactoryMethodMetadata();
}
```

由于这里我们使用class类信息作为BeanDefinition配置的载体，因此class中的注解元数据，类型信息，方法名称等信息都应该能成为BeanDefinition配置的一部分，这些元数据信息通过AnnotationMetadata和MethodMetadata来表达，正如AnnotatedBeanDefinition接口声明所示。这种接口扩展真是顺势而为啊!

AnnotationMetadata和MethodMetadata这两个接口很重要，可以说是解析BeanDefinition配置的基础数据。下面分别看看这两个接口的定义：

```java
public interface AnnotationMetadata extends ClassMetadata, AnnotatedTypeMetadata {

	//获取class上所有标注的类型名称
	Set<String> getAnnotationTypes();

	//获取class上名字为annotationName的标注类型名称，包括标注上的其他标注
	Set<String> getMetaAnnotationTypes(String annotationName);

	//判断class上是否有标注名称为annotationName的标注
	boolean hasAnnotation(String annotationName);

	//判断class上是否有标注名称为annotationName的标注，包括标注上的其他标注
	boolean hasMetaAnnotation(String metaAnnotationName);
	
	//判断class中是否带有标注的方法
	boolean hasAnnotatedMethods(String annotationName);

	//获取带有指定标注名称annotationName的方法元数据
	Set<MethodMetadata> getAnnotatedMethods(String annotationName);

}
```

AnnotationMetadata定义了解析class上的注解相关方法。另外，AnnotationMetadata继承了ClassMetadata和AnnotatedTypeMetadata接口：

```java
public interface ClassMetadata {
	String getClassName();
	boolean isInterface();
	boolean isAnnotation();
	boolean isAbstract();
	boolean isConcrete();
	boolean isFinal();
	boolean isIndependent();
	boolean hasEnclosingClass();
	String getEnclosingClassName();
	boolean hasSuperClass();
	String getSuperClassName();
	String[] getInterfaceNames();
	String[] getMemberClassNames();
}

public interface AnnotatedTypeMetadata {
	boolean isAnnotated(String annotationName);
	Map<String, Object> getAnnotationAttributes(String annotationName);
	Map<String, Object> getAnnotationAttributes(String annotationName, boolean classValuesAsString);
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName);
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName, boolean classValuesAsString);
}
```
从ClassMetadata和AnnotatedTypeMetadata接口声明中可以看出，ClassMetadata定义了与class定义相关信息的获取接口，而AnnotatedTypeMetadata则定义了通用可注解元素（类/属性/方法）对注解的通用获取接口。

最后，MethodMetadata则定义了与方法定义相关信息的获取接口：

```java
public interface MethodMetadata extends AnnotatedTypeMetadata {
	String getMethodName();
	String getDeclaringClassName();
	String getReturnTypeName();
	boolean isAbstract();
	boolean isStatic();
	boolean isFinal();
	boolean isOverridable();
}
```

由此可见，AnnotatedGenericBeanDefinition已经具备了所有用于从class中读取所有BeanDefinition配置元数据的功能了，这为后面通过元数据生成BeanDefinition提供了数据基础。

接下来可以看到，所有分析配置过程都依赖于ClassMetadata和AnnotatedTypeMetadata元数据。

接着是`conditionEvaluator.shouldSkip()`，该方法用于获取@Conditional注解中配置的Condition实例，并通过Condition判断该BeanDefintion是否满足约束条件，这个过程并不复杂，具体实现可参考源代码。

`AnnotationConfigUtils.processCommonDefinitionAnnotations()`方法就是对一般注解的处理了，包括@Lazy/@Primary/@DependsOn/@Role/@Description注解。

最后则调用`BeanDefinitionReaderUtils.registerBeanDefinition()`向容器注册BeanDefinition了。到此，AnnotatedBeanDefinitionReader的register()方法也就执行完成了。

### ClassPathBeanDefinitionScanner实现

ClassPathBeanDefinitionScanner的核心方法是`scan(String... basePackages)`：

```java
	public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

		doScan(basePackages);

		// Register annotation config processors, if necessary.
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}
```

doScan()方法中实现了核心逻辑：

```java
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
		for (String basePackage : basePackages) {
			// 找出候选的BeanDefinition
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				// 这里的过程与AnnotatedBeanDefinitionReader.registerBean()异常相似了
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```

可以看到，doScan()的实现与AnnotatedBeanDefinitionReader.registerBean()异常相似了，不同的是，在doScan()中需要实现查找候选的BeanDefinition的功能，这在父类ClassPathScanningCandidateComponentProvider.findCandidateComponents()方法中实现：

```java
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
		try {
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + "/" + this.resourcePattern;
			Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource);
				}
				if (resource.isReadable()) {
					try {
						MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
						if (isCandidateComponent(metadataReader)) {
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							sbd.setResource(resource);
							sbd.setSource(resource);
							if (isCandidateComponent(sbd)) {
								if (debugEnabled) {
									logger.debug("Identified candidate component class: " + resource);
								}
								candidates.add(sbd);
							}
							else {
								if (debugEnabled) {
									logger.debug("Ignored because not a concrete top-level class: " + resource);
								}
							}
						}
						else {
							if (traceEnabled) {
								logger.trace("Ignored because not matching any filter: " + resource);
							}
						}
					}
					catch (Throwable ex) {
						throw new BeanDefinitionStoreException(
								"Failed to read candidate component class: " + resource, ex);
					}
				}
				else {
					if (traceEnabled) {
						logger.trace("Ignored because not readable: " + resource);
					}
				}
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
		}
		return candidates;
	}
```

`findCandidateComponents()`的实现有以下几个关键步骤：

1. PathMatchingResourcePatternResolver找到package下的所有class文件，并以Resource返回。
2. 需要一个解析器，把Resource解析为AnnotationMetadata，这由MetadataReader来完成
3. 分析AnnotationMetadata元数据，找出满足筛选条件的AnnotationMetadata，默认的筛选条件是类定义中含有@Component注解
4. 将AnnotationMetadata封装成ScannedGenericBeanDefinition作为候选BeanDefinition

> @Service/@Repository/@Configuration注解定义上都添加了@Component注解的，因此这里扫描出来的类自然包括了这些注解所标注的类了。

这个过程涉及几个重要的类型，下面简单说明一下它们的特性，至于这些类的实现原理请自行翻阅源码。

* PathMatchingResourcePatternResolver可以按给定的路径模式找到对应的资源Resource列表
* MetadataReader可以解析基于Resource的class文件为AnnotationMetadata，解析过程是基于ASM框架。
* ScannedGenericBeanDefinition与AnnotatedGenericBeanDefinition的实现类似，只是并没有实现MethodMetadata

至此，AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner的核心实现已经分析完了。但相信读者可以发现，这个过程非常简单，似乎少了一些注解的实现，比如@ComponentScan，@Bean等等，那么这些注解的处理是在哪里实现的呢？其实这是在一个BeanDefinitionRegistryPostProcessor实现中处理的，这个BeanDefinitionRegistryPostProcessor实现类是ConfigurationClassPostProcessor。

那这个ConfigurationClassPostProcessor又是在哪里注册进容器的呢？其实在AnnotatedBeanDefinitionReader的构造函数及ClassPathBeanDefinitionScanner的scan()方法中都有这个后处理器的注册调用。如以下代码清单：

```java
	//AnnotatedBeanDefinitionReader的构造函数
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		// 这里就是注册ConfigurationClassPostProcessor的入口了
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}

	// ClassPathBeanDefinitionScanner的scan()方法实现
	public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

		doScan(basePackages);

		// 这里就是注册ConfigurationClassPostProcessor的入口了
		// Register annotation config processors, if necessary.
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}
	
	//AnnotationConfigUtils.registerAnnotationConfigProcessors()
	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, Object source) {

		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
		if (beanFactory != null) {
			if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
				beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
			}
			if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
				beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
			}
		}

		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<BeanDefinitionHolder>(4);

		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			//在这里看到ConfigurationClassPostProcessor的注册了
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
		if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition();
			try {
				def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
						AnnotationConfigUtils.class.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
			}
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
		}
		if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
		}

		return beanDefs;
	}
```

在AnnotationConfigUtils.registerAnnotationConfigProcessors()中，除了注册了ConfigurationClassPostProcessor后处理器，还有其他一些常用的后处理器在《Spring-IOC容器原理之Bean创建过程》中有介绍过了。

### ConfigurationClassPostProcessor

ConfigurationClassPostProcessor实现了BeanDefinitionRegistryPostProcessor接口，根据《Spring-IOC容器原理之启动过程》中关于容器启动过程的说明，容器会先进行postProcessBeanDefinitionRegistry()方法的回调，接着回调BeanFactoryPostProcessor接口。因此，我们先看ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()的实现：

```java
	@Override
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
		// 注册了ImportAwareBeanPostProcessor后处理器，完成对ImportAware接口的注入
		RootBeanDefinition iabpp = new RootBeanDefinition(ImportAwareBeanPostProcessor.class);
		iabpp.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(IMPORT_AWARE_PROCESSOR_BEAN_NAME, iabpp);

		// 注册了EnhancedConfigurationBeanPostProcessor后处理器，完成对EnhancedConfiguration的处理
		RootBeanDefinition ecbpp = new RootBeanDefinition(EnhancedConfigurationBeanPostProcessor.class);
		ecbpp.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(ENHANCED_CONFIGURATION_PROCESSOR_BEAN_NAME, ecbpp);

		int registryId = System.identityHashCode(registry);
		if (this.registriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException(
					"postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
		}
		if (this.factoriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + registry);
		}
		this.registriesPostProcessed.add(registryId);
		// 这里是处理配置类的核心入口
		processConfigBeanDefinitions(registry);
	}
```

在此处，注册了两个后处理器ImportAwareBeanPostProcessor和EnhancedConfigurationBeanPostProcessor，分别用于完成对ImportAware接口的注入以及对EnhancedConfiguration的处理。之后processConfigBeanDefinitions()调用对容器中注册的BeanDefinition进行解析处理，以找到更多隐藏在配置类中BeanDefinition，并把这些BeanDefinition注册到容器中去。

processConfigBeanDefinitions()的处理可以分解为如下图所示的三个过程，这三个过程循环执行，直至再也找不到未处理的*ConfigurationBeanDefinition*：

![](https://sin90lzc.github.io/images/spring/processConfigBeanDefinitions.png?d)

上图中包含了几个新的概念：

1. *ConfigurationBeanDefinition*是指这样的BeanDefinition，它的class使用了@Configuration,@Component,@ComponentScan,@Import,@ImportResource注解，或者带有@Bean注解方法。*ConfigurationBeanDefinition*这个概念是自创的，只在本文中使用到，创造这个概念的原因是为了能更简单明了地表达这种类型的BeanDefinition。
2. ConfigurationClass，是Spring中定义的一个类，一个ConfigurationClass对应着一个*ConfigurationBeanDefinition*，它表达了它是一个配置类的概念。ConfigurationClass用于存储解析*ConfigurationBeanDefinition*的中间结果（这个解析过程还是依赖于最基础的元数据AnnotationMetadata），这些中间解析结果包括@Bean注解的方法列表、通过@Import注解导入的ImportBeanDefinitionRegistrar列表、通过@ImportResource注解导入的BeanDefinition配置资源列表，以及ConfigurationClass是被哪些ConfigurationClass通过@Import导入进来的。这些中间解析结果是为了生成新的BeanDefinition做准备的。
3. ConfigurationClassParser，也是Spring中定义的一个类，它的作用是把*ConfigurationBeanDefinition*解析成为ConfigurationClass。下面会详细讲解这些解析过程。
4. ConfigurationClassBeanDefinitionReader，是Spring中定义的类，用于读取ConfigurationClass的中间解析结果，生成供容器使用的BeanDefinition，并注册到容器。

相信大家在认识了这四个新概念之后，对Spring是如何实现对*ConfigurationBeanDefinition*解析处理有了一个概念性的理解了，大家是否会觉得这种概念性的分析过程会更易记忆和理解呢？相信大家的回答都是YES!YES!YES!这就是挖掘高层次抽象性概念的好处了，让我们可以站在更高的高度去看待问题，会使我们的视野更开阔更明朗，而不至于出现“盲人摸象”的囧况。

有了高层次概念性理解之后，我们就可以深入到细节中去，深入探索内部奥秘，而不至于迷失方向。下面就先分析一下ConfigurationClassParser是如何解析*ConfigurationBeanDefinition*的。

ConfigurationClassParser解析*ConfigurationBeanDefinition*的过程封装在parse()方法中，然而核心的解析过程是在`processConfigurationClass(ConfigurationClass)`中进行，此时的入参ConfigurationClass已经封装了基础的元数据AnnotationMetadata，后续的解析过程是围绕AnnotationMetadata元数据进行的。而至于AnnotationMetadata是如何生成的呢？看看三个parse()的重载方法：

```java
	protected final void parse(String className, String beanName) throws IOException {
		MetadataReader reader = this.metadataReaderFactory.getMetadataReader(className);
		processConfigurationClass(new ConfigurationClass(reader, beanName));
	}

	protected final void parse(Class<?> clazz, String beanName) throws IOException {
		processConfigurationClass(new ConfigurationClass(clazz, beanName));
	}

	protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
		processConfigurationClass(new ConfigurationClass(metadata, beanName));
	}
```

可以看出，AnnotationMetadata的生成有三种方式，一种是*ConfigurationBeanDefinition*中已经含有AnnotationMetadata了，如AnnotatedGenericBeanDefinition，可以直接拿来使用，另一种是使用StandardAnnotationMetadata(Class)构造函数创建AnnotationMetadata，最后则直接通过MetadataReader（实现类是SimpleMetadataReader）解析class文件生成，这种方式的实现基于ASM框架，有兴趣的读者自行深入分析。

下面将关注点回到核心解析过程`processConfigurationClass(ConfigurationClass)`中来：

```java
	protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
		// 对@Conditional注解中定义的Condition进行条件判断，判断通过才需要进行解析
		if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
			return;
		}

		ConfigurationClass existingClass = this.configurationClasses.get(configClass);
		// 对ConfigurationClass重复解析的处理
		// 处理逻辑：
		//		只保留不是通过@Import进来的ConfigurationClass的解析结果;
		// 		如果重复解析的ConfigurationClass都是通过@Import进来的，只需维护导入关系
		if (existingClass != null) {
			if (configClass.isImported()) {
				if (existingClass.isImported()) {
					existingClass.mergeImportedBy(configClass);
				}
				// Otherwise ignore new imported config class; existing non-imported class overrides it.
				return;
			}
			else {
				// Explicit bean definition found, probably replacing an import.
				// Let's remove the old one and go with the new one.
				this.configurationClasses.remove(configClass);
				for (Iterator<ConfigurationClass> it = this.knownSuperclasses.values().iterator(); it.hasNext(); ) {
					if (configClass.equals(it.next())) {
						it.remove();
					}
				}
			}
		}

		// SourceClass是对AnnotationMetadata的另一种封装形式
		// Recursively process the configuration class and its superclass hierarchy.
		SourceClass sourceClass = asSourceClass(configClass);
		do {
			// configClass描述的是一个ConfigurationBeanDefinition的配置类，而sourceClass描述的可能是configClass自身，也可能是configClass的父类
			sourceClass = doProcessConfigurationClass(configClass, sourceClass);
		}
		while (sourceClass != null);

		//记录下已经解析的ConfigurationClass
		this.configurationClasses.put(configClass, configClass);
	}
```

从以上源码可以看出，这里还未深入到AnnotationMetadata中去解析，而只是对ConfigurationClass进行@Conditional注解的条件判断，以及重复解析ConfigurationClass的处理逻辑：

1. 只保留不是通过@Import进来的ConfigurationClass的解析结果
2. 如果重复解析的ConfigurationClass都是通过@Import进来的，只需记录下被哪个ConfigurationClass导入进来的

其后循环调用doProcessConfigurationClass(configClass, sourceClass)，以解析配置类的各继承层级，每个继承层级所对应的AnnotationMetadata，用SourceClass来表达，直至父类并不是java自身的类。doProcessConfigurationClass(configClass, sourceClass)的过程用以下活动图来表达其实现原理：

![](https://sin90lzc.github.io/images/spring/doProcessConfigurationClass.png?)

如图中黑色加粗方框所示，doProcessConfigurationClass()可以分解为以下子过程：解析内部类 -> 解析@PropertySource注解 -> 解析@ComponentScan注解 -> 解析@Import注解 -> 解析@ImportResource注解 -> 解析@Bean注解 -> 递归doProcessConfigurationClass()处理父类。每个方框内都表达了每个子过程的实现原理，理解了上图就理解了解析*ConfigurationBeanDefinition*的原理了。

在上图中，有一个比较唐突的新概念：importStack。它是一个ImportRegistry接口的实现类，它主要用于注册import关系的以及关系的获取，即ConfigurationClass是被哪个ConfigurationClass导入进来的。

从上图中也可以看出，在整个过程中，除了在解析@ComponentScan注解的过程中会向容器注册新的BeanDefinition之外，其他过程并没有向容器注册新的BeanDefinition。向容器注册新的BeanDefinition是交由ConfigurationClassBeanDefinitionReader完成的。

> 另外也应注意到，通过@Import导入以及解析内部类而得到的ConfigurationClass，并没有在容器中注册成BeanDefinition的形式。

### ConfigurationClassBeanDefinitionReader

在上面的分析中，ConfigurationClassPostProcessor已经完成了对ConfigurationBeanDefinition的预解析，解析结果保存在ConfigurationClass中。下面看看ConfigurationClassBeanDefinitionReader是如何读取ConfigurationClass中的中间解析结果，完成BeanDefinition的生成和注册的。

ConfigurationClassBeanDefinitionReader读取ConfigurationClass的核心方法是loadBeanDefinitionsForConfigurationClass：

```java
	private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass,
			TrackedConditionEvaluator trackedConditionEvaluator) {
		// trackedConditionEvaluator与conditionEvaluator类似，只不过
		// trackedConditionEvaluator是对整个导入链进行条件约束校验，而非单
		// 而非单个ConfigurationClass
		if (trackedConditionEvaluator.shouldSkip(configClass)) {
			// 当确定要跳过对configClass的处理之后，要从容器中移出注册的BeanDefinition
			// 还要从importRegistry移除注册
			String beanName = configClass.getBeanName();
			if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
				this.registry.removeBeanDefinition(beanName);
			}
			this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
			return;
		}
		// 如果configClass是通过Import进来的，还需要把它作为BeanDefinition
		// 注册到容器中
		if (configClass.isImported()) {
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}
		// 把BeanMethod解析为BeanDefinition的处理过程，后续详解
		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
			loadBeanDefinitionsForBeanMethod(beanMethod);
		}
		// 处理@ImportResource解析的中间结果，调用BeanDefinitionReader加载BeanDefintion配置的地方
		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
		//回调ImportBeanDefinitionRegistrar的接口完成BeanDefinition的注册
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}
```

从以上源码中看到，上述过程并不复杂。

trackedConditionEvaluator用于校验configClass的导入方ConfigurationClass，甚至导入方的导入方ConfigurationClass，直到导入方并不是通过其他导入方导入，校验这整个导入链是否都满足条件约束，只有都满足的时候才需要继续处理。当导入链不满足条件约束时，将从容器中移除它的BeanDefinition，而且还需要从导入关系注册ImportRegistry中移除。

registerBeanDefinitionForImportedConfigurationClass()完成的是对import进来的configClass注册到容器中去，过程与AnnotatedBeanDefinitionReader.registerBean()非常类似，这里不再罗嗦。

loadBeanDefinitionsForBeanMethod()则是把BeanMethod解析为BeanDefinition的处理过程，如代码清单所示:

```java
	private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
		ConfigurationClass configClass = beanMethod.getConfigurationClass();
		MethodMetadata metadata = beanMethod.getMetadata();
		String methodName = metadata.getMethodName();

		// Do we need to mark the bean as skipped by its condition?
		//这里还需要对BeanMethod上的Condition进行条件约束校验
		if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
			configClass.skippedBeanMethods.add(methodName);
			return;
		}
		if (configClass.skippedBeanMethods.contains(methodName)) {
			return;
		}

		// Consider name and any aliases
		// 从BeanMethod中解析出beanName以及aliase
		AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class);
		List<String> names = new ArrayList<String>(Arrays.asList(bean.getStringArray("name")));
		String beanName = (names.size() > 0 ? names.remove(0) : methodName);

		// Register aliases even when overridden
		// 注册aliases
		for (String alias : names) {
			this.registry.registerAlias(beanName, alias);
		}

		// 判断容器中是否存在重复的BeanDefintion，存在着跳过
		// Has this effectively been overridden before (e.g. via XML)?
		if (isOverriddenByExistingDefinition(beanMethod, beanName)) {
			return;
		}

		// 下面就开始创建BeanDefinition了
		ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);
		beanDef.setResource(configClass.getResource());
		beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));
		
		// @Bean注解的创建的Bean是使用工厂方法创建的，
		// 这里就是确定使用的是静态工厂方法还是动态工厂方法，
		// 以及工厂方法的名称
		if (metadata.isStatic()) {
			// static @Bean method
			beanDef.setBeanClassName(configClass.getMetadata().getClassName());
			beanDef.setFactoryMethodName(methodName);
		}
		else {
			// instance @Bean method
			beanDef.setFactoryBeanName(configClass.getBeanName());
			beanDef.setUniqueFactoryMethodName(methodName);
		}
		// 默认使用AUTOWIRE_CONSTRUCTOR的方式完成依赖注入
		beanDef.setAutowireMode(RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
		beanDef.setAttribute(RequiredAnnotationBeanPostProcessor.SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

		AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);

		Autowire autowire = bean.getEnum("autowire");
		if (autowire.isAutowire()) {
			beanDef.setAutowireMode(autowire.value());
		}
		// 从BeanMethod中解析出初始化方法
		String initMethodName = bean.getString("initMethod");
		if (StringUtils.hasText(initMethodName)) {
			beanDef.setInitMethodName(initMethodName);
		}
		// 从BeanMethod中解析出销毁方法
		String destroyMethodName = bean.getString("destroyMethod");
		if (destroyMethodName != null) {
			beanDef.setDestroyMethodName(destroyMethodName);
		}

		// Consider scoping
		ScopedProxyMode proxyMode = ScopedProxyMode.NO;
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
		if (attributes != null) {
			beanDef.setScope(attributes.getAliasedString("value", Scope.class, configClass.getResource()));
			proxyMode = attributes.getEnum("proxyMode");
			if (proxyMode == ScopedProxyMode.DEFAULT) {
				proxyMode = ScopedProxyMode.NO;
			}
		}

		// Replace the original bean definition with the target one, if necessary
		BeanDefinition beanDefToRegister = beanDef;
		if (proxyMode != ScopedProxyMode.NO) {
			BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
					new BeanDefinitionHolder(beanDef, beanName), this.registry, proxyMode == ScopedProxyMode.TARGET_CLASS);
			beanDefToRegister = new ConfigurationClassBeanDefinition(
					(RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata);
		}

		if (logger.isDebugEnabled()) {
			logger.debug(String.format("Registering bean definition for @Bean method %s.%s()",
					configClass.getMetadata().getClassName(), beanName));
		}
		// 向容器中注册Bean定义
		this.registry.registerBeanDefinition(beanName, beanDefToRegister);
	}
```

上述过程也是很简单明了的，若有不理解，通过中文的注释也应该能明白的，这里不再细致说明了。

其后loadBeanDefinitionsFromImportedResources()则是处理@ImportResource解析的中间结果的，实现方式就是调用BeanDefinitionReader加载BeanDefintion了，这里的BeanDefinitionReader可能是XmlBeanDefinitionReader，也可能是GroovyBeanDefinitionReader，甚至是自定义的BeanDefinitionReader。但默认情况是根据配置文件的后缀来决定使用XmlBeanDefinitionReader还是GroovyBeanDefinitionReader。

loadBeanDefinitionsFromRegistrars()就更简单了，只是回调ImportBeanDefinitionRegistrar的接口完成BeanDefinition的注册。

至此，关于ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()的实现已经解析完毕了。但ConfigurationClassPostProcessor还实现了postProcessBeanFactory()接口，下面不妨也看看这个接口的实现吧。

### ConfigurationClassPostProcessor.postProcessBeanFactory()

直接上postProcessBeanFactory()接口实现：

```java
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		int factoryId = System.identityHashCode(beanFactory);
		if (this.factoriesPostProcessed.contains(factoryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + beanFactory);
		}
		this.factoriesPostProcessed.add(factoryId);
		if (!this.registriesPostProcessed.contains(factoryId)) {
			// BeanDefinitionRegistryPostProcessor hook apparently not supported...
			// Simply call processConfigurationClasses lazily at this point then.
			processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
		}

		enhanceConfigurationClasses(beanFactory);
	}
```

在上面源码中，比较陌生的是enhanceConfigurationClasses()方法实现，其目的是使用CGLIB代理加强使用了@Configuration注解的BeanDefinition，在其原来的基础上实现了EnhancedConfiguration接口，EnhancedConfiguration接口只是实现BeanFactoryAware接口而已。

在EnhancedConfigurationBeanPostProcessor后处理器中看到对EnhancedConfiguration接口的处理：

```java
		public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) {
			// Inject the BeanFactory before AutowiredAnnotationBeanPostProcessor's
			// postProcessPropertyValues method attempts to auto-wire other configuration beans.
			if (bean instanceof EnhancedConfiguration) {
				((EnhancedConfiguration) bean).setBeanFactory(this.beanFactory);
			}
			return pvs;
		}
```

至此，AnnotationConfigWebApplicationContext对BeanDefinition加载原理已经分析完毕，你是否已经掌握了呢？