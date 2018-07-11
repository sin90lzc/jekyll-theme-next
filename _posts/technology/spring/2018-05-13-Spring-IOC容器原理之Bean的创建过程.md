要去认识Spring-IOC容器的实现原理，需要深入了解IOC容器的三个过程：**BeanDefinition的加载过程**、**Bean实例化过程**、以及**Bean依赖注入过程**。

本文主要介绍的是**Bean实例化过程**、**Bean依赖注入过程**。读者可能会对首先分析这两个过程感到迷惑，大家都知道**BeanDefinition的加载过程**是这两个过程的基础，需要先有*BeanDefinition*数据基础才会有后两个过程的出现。这主要是考虑到这两个过程离开发人员比较近，我们可以对IOC容器的扩展主要来源于这两个过程中。如一系列功能强大的*BeanPostProcessor*的使用也是充分体现在这两个过程中的。这样的分析顺序也说明了*BeanDefinition*这种基础数据虽然重要，但不影响我们对这两个过程的分析，只需要读者心中有一个概念：在进入这两个过程之前，*BeanFactory*已经注册了所有待处理的*BeanDefinition*了。

在进入分析之前，有必要说明一下本文的主要目标。如果你希望在本文中能找到IOC容器实现的所有细节，这可能会令你感到失望。这是因为IOC容器的实现过程是非常复杂的，如果事无具细都进行说明，可能读者会“找不着北”，而迷失于各种复杂细节当中。因此，本文的主要目标是，当你读完本文，可以清晰地知道**Bean实例化过程**、以及**Bean依赖注入过程**这两个过程的关键过程，以及从较高抽象层次去理解这两个过程的实现，而较少地去关注实现细节，具体的实现细节还是需要依赖读者自行阅读源代码，这里只是理清关键过程和概念，为读者深入实现细节进行铺路。另外，我也不会完全抛弃源代码，比如，我依然会引用源代码中的类名和方法名，甚至一部分的代码实现过程，这完全是因为Spring的源代码类命名、方法命名本身有着很强的说明能力，能够说明一种概念，而这些概念也为我们日常沟通的提供了语言。

**Bean实例化过程**、**Bean依赖注入过程**这两个过程的入口是`org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(String, Class<T>, Object[], boolean)`。下面就从这个入口开始分析吧。

## doGetBean关键过程

我把`doGetBean`方法过程人为地划分为四个子过程：**获取已创建单例bean的过程**，**bean实例化的过程**，**bean依赖注入过程**以及**bean的初始化与销毁注册**，而且把`doGetBean`中最关键的过程以时序图的方式表达出来，后面的所有分析都基于这个时序图：

![](https://sin90lzc.github.io/images/spring/doGetBean.png)

时序图看起来相当吓人，但其实已经省略掉了很多实现细节了，而把非常能说明doGetBean的关键过程展示了出来，理解了这些调用就能对doGetBean有非常清晰的理解了。

下面就让我一一道来吧。

### 获取已创建单例Bean的过程

这个过程相对比较简单。它的实现无非就是从一个单例Bean的注册中心里，根据`beanName`来查出完成实例化的Bean(调用**[2]**)，这个注册中心就是`DefaultSingletonBeanRegistry`。但在Spring IOC中还有一种称为`FactoryBean`的工厂Bean，这种Bean是不能直接使用的，因此需要一个组件来完成从`FactoryBean`到单例Bean的转化过程（调用**[6]**)，而且这个组件可能还需要对这个转化结果做一些缓存以提高性能，这个组件就是`FactoryBeanRegistrySupport`了。

#### DefaultSingletonRegistry

`DefaultSingletonBeanRegistry`实现了接口`SingletonBeanRegistry`。该接口声明了以下方法：

```java
	void registerSingleton(String beanName, Object singletonObject);

	Object getSingleton(String beanName);

	boolean containsSingleton(String beanName);

	String[] getSingletonNames();

	int getSingletonCount();

	Object getSingletonMutex();
```
	
从接口声明中可以看出，`DefaultSingletonBeanRegistry`就是一个单例Bean的注册中心。但`DefaultSingletonBeanRegistry`除了实现了基本的单例Bean注册中心的功能之外，还需要为容器提供一些额外的支持，而其中一个很重要的容器特性是允许Bean之间的双向引用，而这个特性则需要`DefaultSingletonBeanRegistry`能够为已实例化但还未进行依赖注册的bean提供注册和查询功能。这种已实例化但还未进行依赖注入的bean在Spring中称为*EarlyBeanReference*。

```java
	//这里就是DefaultSingletonBeanRegistry注册EarlyBeanReference工厂类的地方
	//而ObjectFactory是early reference bean的工厂类
	protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				this.singletonFactories.put(beanName, singletonFactory);
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
	}
	
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
```
	
在上面的代码片段中，`singletonFactories`是持有EarlyBeanReference的工厂类的Map，earlySingletonObjects是持有由工厂类生成的EarlyBeanReference的Map，singletonObjects则是持有已完成依赖注入的Bean的Map了。

从getSingleton方法中可以看到，bean的获取过程是先从singletonObjects中获取bean，若没有再从earlySingletonObjects中获取EarlyReference Bean，以实现Bean双向引用的。

EarlyBeanReference注册应该发生在Bean实例化之后，正如时序图中**[30]**调用所示，发生的时刻正是在Bean实例化之后，但这里并不是简单地注册实例化后的Bean，而是一个`ObjectFactory`，这个`ObjectFactory`会进一步调用`SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference(beanInstance,beanName)`对原始Bean进行增强(调用**[3]**)，通过`ObjectFactory`增强的EarlyBeanReference会被注册到earlySingletonObjects中去。正如下面代码片段展示的一样：

```java
	//[18]调用代码片段，注册EarlyBeanReference工厂类
	addSingletonFactory(beanName, new ObjectFactory<Object>() {
		@Override
		public Object getObject() throws BeansException {
			return getEarlyBeanReference(beanName, mbd, bean);
		}
	});
	
	//使用SmartInstantiationAwareBeanPostProcessor对EarlyBeanReference进行增强
	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					//使用SmartInstantiationAwareBeanPostProcessor对Early Reference Bean进行增强
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
					if (exposedObject == null) {
						return null;
					}
				}
			}
		}
		return exposedObject;
	}
```

对于`SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference(beanInstance,beanName)`的其中一个实现是`org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator`，它在getEarlyBeanReference阶段创建一个代理对象。关于`org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator`的实现，若有机会将在另一篇文章中说明。

[//]:TODO写关于org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator自动代理BeanPostProcessor的文章



#### FactoryBeanRegistrySupport

这个组件非常简单，没什么好讲的。只是提供了从`FactoryBean`到Bean的转换方法，并且缓存转换结果以提高性能。`FactoryBean`到Bean的转换方法实际上就是调用了`FactoryBean.getObject()`来生成Bean的。

### bean实例化的过程

Bean实例化过程的入口是**[14]createBeanInstance**。从时序图中可以看出，有两种实例化Bean的方式：工厂方法和构造函数（其实这也正是我们手动编码时实例化对象的主要方式了）。下面以工厂方法方式，从较高抽象层面去分析这个实例化过程，使用构造函数方式实例化Bean的方式是非常类似的。

假如给我一个`BeanDefinition`，如何利用它来实例化一个Bean呢？首先，如果在`BeanDefinition`中给出了工厂方法名称，那么就应该使用工厂方法进行实例化，否则使用构造函数方式实例化bean。

`BeanDefinition`中给出了工厂方法名称，而且工厂方法是静态方法，则直接调用静态工厂方法实例化Bean，否则应先创建工厂实例，再调用动态工厂方法，而这里的工厂实例的创建应该交由另一个`BeanDefinition`去创建的，因此这里的`BeanDefinition`只需要提供工厂实例的beanName引用，由`BeanFactory.getBean(beanName)`来生成工厂Bean。

这时我们有了工厂类或工厂实例，以及工厂方法名称了。但我们还缺少方法调用的参数。我们应该意识到方法调用参数的来源有三种，一种是直接通过getBean()方法传参进来，一种是从`BeanDefinition`中`ConstructorArgumentValues`中获取，`ConstructorArgumentValues`保存了xml配置中构造函数参数的定义，另一种则是通过参数名或参数类型自动解析出参数对象（这种方式在spring中称为**自动装载autowired**)。`ConstructorArgumentValues`保存的只是xml配置信息，因此需要将这些配置信息解析为真实对象，我们解析的可能是一个beanName引用，或者只是简单的字符串值，甚至可能是一些集合对象。虽然我们解析出了对象实例，但可能这些对象实例与方法参数类型是不匹配的，这时候就需要有一个类型转换器，能把不匹配的类型转为适当的方法参数类型，如配置的对象是一个字符串，但方法参数类型却是一个Date，这时就需要将String转换为Date对象了。

完成以上过程就可以调用工厂方法来生成Bean实例了吗？事情并没有如此简单。因为同一个类中可能存在多个相同方法名称。这就需要找出最适合的工厂方法才行。如何确定一个工厂方法是最合适的呢？这就需要先找出每个方法的所有参数对象，并需要一种算法策略来确定谁才是最合适的。

这时候，已经确定了调用的工厂类或实例，工厂方法也确定了，方法参数也解析出来了，这时候就可以通过反射调用相应的方法生成Bean实例了。

以上的分析过程也正是Spring实现Bean实例化的过程。回顾一下以上过程，不难发现有几个关键的概念和过程：自动装载、类型转换器、确定工厂方法。

#### 自动装载<span id="autowire"></span>

在spring中，自动装载有两种方式：autowiredByName和autowiredByType。

autowiredByName的实现比较简单，只需要获取到需要依赖注入的属性名或方法参数名作为beanName，从`BeanFactory.getBean(beanName)`中获取即可。细节实现请看`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.autowireByName(String, AbstractBeanDefinition, BeanWrapper, MutablePropertyValues)`。

autowiredByType的实现就稍微复杂一些了。它把需要进行依赖注入的属性或方法参数封装成`DependencyDescriptor`，进而调用`org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DependencyDescriptor, String, Set<String>, TypeConverter)`，从`BeanFactory`中解析出类型相匹配的Bean。`resolveDependency`方法的过程描述如下：

1. 如果依赖注入的类型是`java.util.Optional`，`ObjectFactory`，或`javax.inject.Provider`，那么解析出这些类型的泛型参数类型，再递归调用`resolveDependency`方法解析出真实的依赖bean，并重新封装成`java.util.Optional`，`ObjectFactory`，或`javax.inject.Provider`。这里解析泛型参数类型的实现已经封装在`DependencyDescriptor`中了，直接调用`DependencyDescriptor.increaseNestingLevel()`就把需要解析的依赖类型指向下一层泛型参数了。
2. 当属性或方法参数上带有`@Lazy`注解，则使用`AutowireCandidateResolver.getLazyResolutionProxyIfNecessary(DependencyDescriptor, String)`创建一个代理对象，代理对象在被调用时将回调`resolveDependency`方法获取真实调用对象，以达到延迟装载的目的。`AutowireCandidateResolver`的一个实现参考是`ContextAnnotationAutowireCandidateResolver`。
3. 当属性或方法参数上带有`@Value`注解，则使用`AutowireCandidateResolver.getSuggestedValue(DependencyDescriptor)`解析出由程序提供的值作为自动装载对象。`getSuggestedValue`方法的具体实现可参考`QualifierAnnotationAutowireCandidateResolver.getSuggestedValue(DependencyDescriptor)`
4. 当以上都找不到装载对象时，在BeanFactory中查找出所有满足自动装载属性或方法参数类型的beanName，进一步调用`AutowireCandidateResolver.isAutowireCandidate(BeanDefinitionHolder, DependencyDescriptor)`以确定是否满足自动装载的要求，类型是否完全匹配以及是否满足注解`@Qualifier`。另外，还需要从满足的装载对象中分析@Primary和@Order注解，以找出最适合的装载对象。

#### 类型转换器

在这个过程中使用的类型转换器是`org.springframework.beans.BeanWrapperImpl`。我们粗略地过一下`BeanWrapperImpl`实现的主要功能接口，以及这些接口的描述如下：

* org.springframework.beans.TypeConverter：类型转换器
* org.springframework.beans.PropertyEditorRegistry：PropertyEditor注册器接口
* org.springframework.beans.BeanWrapper：封装Bean实例，以及获取Bean属性的PropertyDescriptor的能力。
* org.springframework.beans.PropertyAccessor：属性访问器，定义了按属性名访问属性值/PropertyDescriptor，或者设置属性值的能力

由些可见，`BeanWrapperImpl`具有类型转换的能力，而且`PropertyEditorRegistry`定义了`PropertyEditor`注册功能接口，`PropertyEditor`其实也是一个类型转换器，只是它的功能比较有局限性，只能转换String<->Object这种双向转换，而不能支持任意Object<->Object的转换能力。其实Spring也提供了任意Object<->Object的转换能力的接口`org.springframework.core.convert.ConversionService`。由于Spring旧版本中提供了大量的`PropertyEditor`的实现，而这些实现也是类型转换能力的一部分，因此Spring通过组合`PropertyEditor`和`ConversionService`的实现来完成完整的类型转换功能，这个组合这两类实现的类是`org.springframework.beans.TypeConverterDelegate`。

那么，现在还有一个问题，`PropertyEditor`和`ConversionService`是如何配置到`BeanWrapperImpl`的呢？答案是在`BeanWrapperImpl`初始化时，从BeanFactory中获取并配置进来的：

	BeanWrapperImpl bw = new BeanWrapperImpl();
	this.beanFactory.initBeanWrapper(bw);
	
再看看initBeanWrapper方法的实现：

	protected void initBeanWrapper(BeanWrapper bw) {
		//ConversionService直接来源于容器的conversionService属性
		bw.setConversionService(getConversionService());
		registerCustomEditors(bw);
	}
	
	protected void registerCustomEditors(PropertyEditorRegistry registry) {
		PropertyEditorRegistrySupport registrySupport =
				(registry instanceof PropertyEditorRegistrySupport ? (PropertyEditorRegistrySupport) registry : null);
		if (registrySupport != null) {
			registrySupport.useConfigValueEditors();
		}
		
		//PropertyEditor的来源一：this.propertyEditorRegistrars
		if (!this.propertyEditorRegistrars.isEmpty()) {
			for (PropertyEditorRegistrar registrar : this.propertyEditorRegistrars) {
				try {
					registrar.registerCustomEditors(registry);
				}
				catch (BeanCreationException ex) {
					Throwable rootCause = ex.getMostSpecificCause();
					if (rootCause instanceof BeanCurrentlyInCreationException) {
						BeanCreationException bce = (BeanCreationException) rootCause;
						if (isCurrentlyInCreation(bce.getBeanName())) {
							if (logger.isDebugEnabled()) {
								logger.debug("PropertyEditorRegistrar [" + registrar.getClass().getName() +
										"] failed because it tried to obtain currently created bean '" +
										ex.getBeanName() + "': " + ex.getMessage());
							}
							onSuppressedException(ex);
							continue;
						}
					}
					throw ex;
				}
			}
		}
		
		//PropertyEditor的来源二：this.customEditors
		if (!this.customEditors.isEmpty()) {
			for (Map.Entry<Class<?>, Class<? extends PropertyEditor>> entry : this.customEditors.entrySet()) {
				Class<?> requiredType = entry.getKey();
				Class<? extends PropertyEditor> editorClass = entry.getValue();
				registry.registerCustomEditor(requiredType, BeanUtils.instantiateClass(editorClass));
			}
		}
	}

从以上代码可以看出，`BeanWrapperImpl`的`PropertyEditor`和`ConversionService`来自于容器的的三个属性：conversionService，propertyEditorRegistrars，customEditors。那么容器的这三个属性值是如何配置进来的呢？

其中一个`org.springframework.beans.support.ResourceEditorRegistrar`是在容器的refresh启动阶段添加进来的，其他则通过`org.springframework.beans.factory.config.CustomEditorConfigurer`这个`BeanFactoryPostProcessor`添加进来的。

还有另一个地方会注册默认的`PropertyEditor`，这个过程是在`org.springframework.beans.PropertyEditorRegistrySupport.createDefaultEditors()`，由于`BeanWrapperImpl`也实现了`PropertyEditorRegistrySupport`，因此这些默认的`PropertyEditor`也注册到`BeanWrapperImpl`中去了。

#### 确定工厂方法

当具有多个相同工厂方法名称的时候，就需要确定使用哪一个工厂方法了。在Spring中会对每一个工厂方法参数类型匹配情况进行计分，按计分排序，从而获取最优的工厂方法。而到底Spring是如何计分的，可参考实现`org.springframework.beans.factory.support.ConstructorResolver.ArgumentsHolder.getTypeDifferenceWeight(Class<?>[])`实现，这里不再详述。

构造函数的确定过程与工厂方法类似，但Spring为构造函数的确定提供了一个后处理器方法`SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors()`，让用户有机会自己去确定使用哪个构造函数作为候选构造函数。这个后处理器方法的调用时机如**[20]**调用所示，发生在autowireConstructor方法调用之前。

`determineCandidateConstructors`方法的其中一个实现类是`AutowiredAnnotationBeanPostProcessor`，它把带有@Autowired,@Value,@Inject注解的构造函数作为候选构造函数，并优先使用带有@Required注解的构造函数。

### Bean依赖注入过程

Bean依赖注入关键过程入口在**[32]**调用方法`populateBean()`。依赖注入完成了以下过程：

1. **[33]**调用`InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation`，在这里可对实例化后的Bean进行增强，如创建代理对象。
2. 根据autowire类型（byName或byType）调用对应的autowireByName()或autowireByType()方法。值得注意的是，这里并不是处理@Autowire/@Resource注解的地方，而是处理所有复杂对象类型的属性的依赖解析工作。并把解析后的对象放到BeanDefinateion的PropertyValue中去，在applyPropertyValues方法调用**[39]**中注入到属性中去。而关于autowireByName()或autowireByType()方法的实现在[自动装载](#autowire)中讨论过了，这里就不再讨论了。
3. 接着回调`InstantiationAwareBeanPostProcessor:postProcessPropertyValues()`**[37]**后处理器，以增强PropertyValues。这些后处理器的典型实现是完成@Autowire/@Resource注解的依赖注入，实现细节请参考`AutowiredAnnotationBeanPostProcessor`。
4. 最后调用applyPropertyValues方法**[39]**，将PropertyValues中的值注入到bean实例中去。但此时，可能有些未经过解析的PropertyValues是不能直接注入的（这些未经解析的PropertyValues可能是来自于Xml配置或其他配置过程），此时需要BeanDefinitionValueResolver对未解析的PropertyValue进行解析**[40]**再完成注入了。注入的过程是在`BeanWrapper.setPropertyValues()`方法**[42]**中完成的。

### Bean初始化与销毁注册

Bean初始化及销毁注册过程都比较简单，这里只是简单描述一下这些过程都做了什么。

Bean初始化过程：

1. 完成BeanNameAware,BeanClassLoaderAware,BeanFactoryAware三个Aware接口的注入**[45]**
2. 回调BeanPostProcessor:postProcessBeforeInitialization()**[46]**
3. 调用Bean的初始化方法，完成Bean的初始化**[47]**
4. 回调BeanPostProcessor:postProcessAfterInitialization()**[48]**

Bean的销毁注册**[49]**非常简单，只是简单地把可销毁Bean注册到容器中去。但哪些Bean是可销毁的Bean呢？满足下面其中一个条件的Bean将是可销毁Bean：

1. Bean实现了DisposableBean接口
2. Bean实现了java.lang.AutoCloseable接口
3. BeanDefinition中定义的destroyMethodName为“(inferred)”，则尝试推断destroyMethodName，可推断的方法名是"close"或"shutdown"

另外，值得一提的是，可销毁Bean是经过`DisposableBeanAdapter`适配器封装之后才能注册到容器中去的。`DisposableBeanAdapter`适配了`DisposableBean`接口和`Runnable`接口，适配器负责确定调用的销毁方法和调用，并为后处理器`DestructionAwareBeanPostProcessor`的执行提供了场所。

这里我们已经看到Bean是如何销毁的了，Bean的销毁应该是跟随容器的销毁而销毁的。但容器又是如何销毁的呢？触发容器销毁的条件可能是以下几种情况：

* 程序正常结束或调用ApplicationContext的close()方法
* 控制台中ctrl+c结束程序

容器是通过使用`Runtime.getRuntime().addShutdownHook()`方法来注册进程关闭回调的，以优雅地在关闭容器。这是在`AbstractApplicationContext.registerShutdownHook()`方法中进行注册的：

```java
	@Override
	public void registerShutdownHook() {
		if (this.shutdownHook == null) {
			// No shutdown hook registered yet.
			this.shutdownHook = new Thread() {
				@Override
				public void run() {
					synchronized (startupShutdownMonitor) {
						doClose();
					}
				}
			};
			Runtime.getRuntime().addShutdownHook(this.shutdownHook);
		}
	}
```

## BeanPostProcessor归纳总结

BeanPostProcessor为容器的实现提供了很多强大的功能，也为容器的扩展提供了无限可能性。如果完全抛开BeanPostProcessor的实现将无法看到Spring容器的一些强大特性，而且各种BeanPostProcessor接口的使用分布在容器实现的各个阶段，容易使人感到迷惘，故此特别为BeanPostProcessor做一些总结。

### InstantiationAwareBeanPostProcessor:postProcessBeforeInstantiation

调用编号：**[11]**

调用时机：在Bean实例化阶段之前，即**[14]**createBeanInstance()之前。

|实现类|实现描述|
|-----------|-----|------|---|----|
|AbstractAutoProxyCreator|实现自动代理，基本思路是从容器中找出所有满足要创建bean的Advisor，并使用ProxyFactory创建代理Bean|

### SmartInstantiationAwareBeanPostProcessor:determineCandidateConstructors

调用编号：**[20]**

调用时机：在使用构造函数实例化Bean对象之前，用于决定候选的构造函数

|实现类|实现描述|
|-----------|-----|------|---|----|
|AutowiredAnnotationBeanPostProcessor|找出候选构造函数，具有@Autowired,@Value,@Inject注解的构造函数作为候选构造函数，并且优先使用带有@Required注解的构造函数|

### MergedBeanDefinitionPostProcessor:postProcessMergedBeanDefinition

调用编号：**[28]**

调用时机：在Bean实例化（即**[14]**createBeanInstance()）之后，并在依赖注入（即**[32]**populateBean()）之前

|实现类|实现描述|
|-----------|-----|------|---|----|
|AutowiredAnnotationBeanPostProcessor|对属性或方法参数中的@AutoWired/@Value注解做预处理，即找出@AutoWired/@Value注解的属性和方法参数并缓存起来供后续使用|
|CommonAnnotationBeanPostProcessor|找出@Resource/@javax.xml.ws.WebServiceRef/@javax.ejb.EJB注解的属性和方法参数并缓存起来供后续使用|
|InitDestroyAnnotationBeanPostProcessor|初始化方法或销毁方法注解的后处理器，用于找出带有指定注解的方法作为初始化方法或销毁方法。在该阶段只是预处理，即先找出待处理的方法|

### InstantiationAwareBeanPostProcessor:postProcessAfterInstantiation

调用编号：**[33]**

调用时机：在Bean依赖注入之前，即**[32]**populateBean()之前。

|实现类|实现描述|
|-----------|-----|------|---|----|
|暂无|暂无|

### SmartInstantiationAwareBeanPostProcessor:getEarlyBeanReference

调用编号：**[3]**

调用时机：getSingleton()时，而且Bean已经完成实例化，但还未完成依赖注入过程

|实现类|实现描述|
|-----------|-----|------|---|----|
|AbstractAutoProxyCreator|当Bean满足Advisor时，创建经过代理后的EarlyBeanReference|



### InstantiationAwareBeanPostProcessor:postProcessPropertyValues

调用编号：**[39]**

调用时机：在调用BeanWrapperImpl.setPropertyValues()之前，即**[42]**调用之前

|实现类|实现描述|
|-----------|-----|------|---|----|
|AutowiredAnnotationBeanPostProcessor|对@AutoWired/@Value注解的属性和方法参数执行注入操作|
|CommonAnnotationBeanPostProcessor|对@Resource/@javax.xml.ws.WebServiceRef/@javax.ejb.EJB注解的属性和方法参数执行注入操作|
|RequiredAnnotationBeanPostProcessor|对@Required注解的属性进行验证|


### BeanPostProcessor:postProcessBeforeInitialization()

调用编号：**[46]**
调用时机：执行Bean初始化方法之前

|实现类|实现描述|
|-----------|-----|------|---|----|
|ApplicationContextAwareProcessor|完成对EnvironmentAware，EmbeddedValueResolverAware，ResourceLoaderAware，ApplicationEventPublisherAware，MessageSourceAware，ApplicationContextAware接口的注入|
|BeanValidationPostProcessor|支持JSR303规范，用于自动对Bean的属性，方法参数等进行约束验证，可参考[JSR303规范的使用](https://www.ibm.com/developerworks/cn/java/j-lo-beanvalid/index.html)|
|BootstrapContextAwareProcessor|注入BootstrapContext|
|ImportAwareBeanPostProcessor|注入ImportRegistry|
|ServletContextAwareProcessor|完成ServletContext和ServletConfig的注入|
|InitDestroyAnnotationBeanPostProcessor|初始化方法或销毁方法注解的后处理器，用于找出带有指定注解的方法作为初始化方法或销毁方法。在该阶段调用注解的初始化方法|

### BeanPostProcessor:postProcessAfterInitialization()

调用编号：**[49]**
调用时机：执行Bean初始化方法之后

|实现类|实现描述|
|-----------|-----|------|---|----|
|AbstractAdvisingBeanPostProcessor|通过配置指定的Advisor，使用ProxyFactory创建特定功能的代理对象，如@Async注解的实现|
|AdvisorAdapterRegistrationManager|用于AdvisorAdapter的注册，AdvisorAdapter用于将Advisor转换为MethodInterceptor|
|BeanValidationPostProcessor|支持JSR303规范，用于自动对Bean的属性，方法参数等进行约束验证，可参考[JSR303规范的使用](https://www.ibm.com/developerworks/cn/java/j-lo-beanvalid/index.html)|
|ScheduledAnnotationBeanPostProcessor|实现@Scheduled和@Schedules的地方，原理就是把注解的方法作为定时任务，但其实现过程却是有其特殊的地方，值得借签，有机会的话专门写一篇文章来介绍|
|ApplicationListenerDetector|将实现了ApplicationListener的Bean向容器注册，由于注册的时机是在Bean初始化方法之后，因此Bean初始化方法之前的事件都无法监听|
|SimpleServletPostProcessor|调用受容器管理的servlet的init()方法|
|AbstractAutoProxyCreator|对于在之前阶段没有为Advisor创建代理的Bean，在此刻创建代理。在之前阶段没有创建代理的原因，可能是有些Advisor是在AdvisorAdapterRegistrationManager这个后处理中加入的|

### DestructionAwareBeanPostProcessor:postProcessBeforeDestruction()

调用时机：在容器关闭时，在调用普通Bean的销毁方法之前

|实现类|实现描述|
|-----------|-----|------|---|----|
|ApplicationListenerDetector|移出已注册的ApplicationListener，但只是移除受容器管理的bean的ApplicationListener|
|InitDestroyAnnotationBeanPostProcessor|初始化方法或销毁方法注解的后处理器，用于找出带有指定注解的方法作为初始化方法或销毁方法。在该阶段调用注解的销毁方法|

### DestructionAwareBeanPostProcessor:postProcessAfterInitialization()

调用时机：在容器关闭时，在调用普通Bean的销毁方法之后

|实现类|实现描述|
|-----------|-----|------|---|----|
|SimpleServletPostProcessor|调用受容器管理的servlet的destroy()方法|


[//]:TODO写一篇关于ScheduledAnnotationBeanPostProcessor的文章






