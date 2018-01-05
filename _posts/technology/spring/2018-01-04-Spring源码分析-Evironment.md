Environment抽象了一个环境的所有配置，比如在dev和在online环境中数据源的配置就肯定是不相同的。另外，Environment实现了PropertyResolver接口，可对属性进行解析操作，如属性值是一个类全限定名，则可以为该属性解析为一个Class对象，也可对属性值中的变量使用另外一个属性值去替换。

## Environment类图

![](https://sin90lzc.github.io/images/uml/spring/Environment.png)

## Environment接口分析

从上图中可以看到一个Environment实例实现了三个接口，分别是PropertyResolver,Environment，ConfigurablePropertyResolver，ConfigurableEnvironment。

### PropertyResolver接口

定义了属性解析的所有行为，包括：

1. 是否包含该属性
2. 根据属性key获取原始的属性值
3. 将属性值转换为指定类型的实例
4. 将属性值转换为指定类型的Class
5. 解析属性值中的变量，并返回解析后的属性值

因此，PropertyResolver赋予了Environment属性解析的能力。

### ConfigurablePropertyResolver接口

定义了针对PropertyResolver的可配置行为。包括：

1. 配置属性值转换为指定类型的策略，这个策略由ConfigurableConversionService定义。
2. 定义变量解析的前后缀
3. 定义变量解析失败后的默认值分隔符

### Environment接口

Environment接口正如其名字声明的那样，它只包含与环境相关的profiles。profile可以认为是一套配置的集合的命名，Environment接口只需要知道哪些profile是激活状态的，就知道当前环境可用的配置有哪些了。

### ConfigurableEnvironment接口

正如接口命名，它是用于配置Environment的。

由于Environment实现了PropertyResolver接口，有属性解析的能力，但属性解析的前提是要有属性可解析啊，因此，ConfigurableEnvironment提供了获取属性源的能力，为属性解析提供了属性源，由接口方法`getPropertySources()`声明。

同样地，Environment能获取profile，但应该暴露方法可以让用户配置当前环境使用的profile。因此有了配置profile的接口方法。

另外，接口`void merge(ConfigurableEnvironment parent);`定义了从父Evironment中合并数据进来。

## Environment实现分析

从上面的接口分析中，可以看出，一个Environment的实现必须实现四大要素：

1. 实现profiles的管理
2. 实现属性源的管理
3. 实现属性解析能力
4. 实现Environment合并

AbstractEnvironment是Environment的抽象实现类，它实现了Environment的核心功能。

### 实现profiles的管理

在AbstractEnvironment实现中，使用了两个属性进行维护profiles，activeProfiles用于保存激活的profile集合，而defaultProfiles则用于保存默认的profile集合，两个属性使用的数据结构是`LinkedHashSet`,代码声明如下：

	private final Set<String> activeProfiles = new LinkedHashSet<String>();

	private final Set<String> defaultProfiles = new LinkedHashSet<String>(getReservedDefaultProfiles());

下面看看活跃profiles和默认profiles的核心获取逻辑：

	protected Set<String> doGetActiveProfiles() {
		synchronized (this.activeProfiles) {
			if (this.activeProfiles.isEmpty()) {
				String profiles = getProperty(ACTIVE_PROFILES_PROPERTY_NAME);
				if (StringUtils.hasText(profiles)) {
					setActiveProfiles(StringUtils.commaDelimitedListToStringArray(
							StringUtils.trimAllWhitespace(profiles)));
				}
			}
			return this.activeProfiles;
		}
	}
	
	protected Set<String> doGetDefaultProfiles() {
		synchronized (this.defaultProfiles) {
			if (this.defaultProfiles.equals(getReservedDefaultProfiles())) {
				String profiles = getProperty(DEFAULT_PROFILES_PROPERTY_NAME);
				if (StringUtils.hasText(profiles)) {
					setDefaultProfiles(StringUtils.commaDelimitedListToStringArray(
							StringUtils.trimAllWhitespace(profiles)));
				}
			}
			return this.defaultProfiles;
		}
	}

从上面代码中可以看出，**活跃profiles和默认profiles是分别通过系统属性spring.profiles.active,spring.profiles.default定义了。而且，这两个方法都是protected的，即可以由子类去扩展定义profiles的获取途径。**

下面看看，spring是如何判断一个profile是否可用的：

	protected boolean isProfileActive(String profile) {
			validateProfile(profile);
			Set<String> currentActiveProfiles = doGetActiveProfiles();
			return (currentActiveProfiles.contains(profile) ||
					(currentActiveProfiles.isEmpty() && doGetDefaultProfiles().contains(profile)));
	}

上面代码逻辑说明，**优先从活跃profiles中去查找，只有当没有配置任何活跃的profiles时，才从默认的profiles中去查找**。

### 实现属性源的管理

AbstractEnvironment通过`MutablePropertySources`进行属性源管理的。
	
	private final MutablePropertySources propertySources = new MutablePropertySources(this.logger);


从其类图中可以看出，MutablePropertySources其实是一个Iteratable对象，如：

![](https://sin90lzc.github.io/images/uml/spring/PropertySources.png)

MutablePropertySources实现Iteratable<PropertySource<?>>是通过内部聚合了List<PropertySource<?>>来实现的，并支持PropertySource<?>的增删改查，**MutablePropertySources使用了组合设计模式**。

	private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<PropertySource<?>>();

`PropertySource<?>`抽象了一个属性源的所有行为。由于属性源是多种多样的，有来自Map数据结构，有来自Properties，甚至来自数据库都有可能，因此，Spring提供了多种多样的`PropertySource<?>`：

![](https://sin90lzc.github.io/images/uml/spring/property_source_arch.jpg)

`PropertySource<?>`的具体实现比较简单，这里就不一一详述了，但有一点需要特别说明，**PropertySource是由name属性作为唯一标识，意味着往MutablePropertySources添加两个相同name的PropertySource，后添加的将被忽略！**

AbstractEnvironment暴露了一个protected接口，给子类进行属性源的配置：

	protected void customizePropertySources(MutablePropertySources propertySources) {
	}



### 实现属性解析能力

AbstractEnvironment通过组合一个`ConfigurablePropertyResolver`的实现类`PropertySourcesPropertyResolver`来提供属性解析的能力的，**这里也是用到了组合设计模式**。

	private final ConfigurablePropertyResolver propertyResolver =
			new PropertySourcesPropertyResolver(this.propertySources);



关于`PropertySourcesPropertyResolver`的源码分析请看TODO


### Environment合并

这个实现比较简单，就直接贴代码了：

	@Override
	public void merge(ConfigurableEnvironment parent) {
		for (PropertySource<?> ps : parent.getPropertySources()) {
			if (!this.propertySources.contains(ps.getName())) {
				this.propertySources.addLast(ps);
			}
		}
		String[] parentActiveProfiles = parent.getActiveProfiles();
		if (!ObjectUtils.isEmpty(parentActiveProfiles)) {
			synchronized (this.activeProfiles) {
				for (String profile : parentActiveProfiles) {
					this.activeProfiles.add(profile);
				}
			}
		}
		String[] parentDefaultProfiles = parent.getDefaultProfiles();
		if (!ObjectUtils.isEmpty(parentDefaultProfiles)) {
			synchronized (this.defaultProfiles) {
				this.defaultProfiles.remove(RESERVED_DEFAULT_PROFILE_NAME);
				for (String profile : parentDefaultProfiles) {
					this.defaultProfiles.add(profile);
				}
			}
		}
	}

该实现逻辑：

1. 把父Environment中定义的属性源添加到子Environment中去，重复的属性源被丢弃。
2. 把父Environment中的活跃profile和默认profile添加到子profile中来。

## 深思

在分析源码的过程中，发现一个值得思考的问题，为什么`MutablePropertySources getPropertySources();`该接口方法由`ConfigurableEnvironment`定义，而不是由`ConfigurablePropertyResolver`来定义呢？

首先，如果放在`ConfigurablePropertyResolver`中定义，PropertyResolver解析对象就是属性源，因此，由PropertyResolver去维护属性源也是合理的。但如果从另一个角度看，属性源其实是会随着环境的变更而不同，这样就是在`ConfigurableEnvironment`定义更合理些了，而Environment的设计者应该也是考虑到这一点吧。

## Environment的使用示例

