PropertyResolver抽象了对key-value这种键值对数据结构的解析行为。

## PropertyResolver类图

![](https://sin90lzc.github.io/images/uml/spring/PropertyResolver.png)

### PropertyResolver接口

PropertyResolver抽象了对key-value这种键值对数据结构的解析行为，包括：

1. 判断属性key是否存在
2. 用key查找value，可提供默认值
3. 用key查找value，并转换为对应的实例
4. 用key查找value，并转换为对应的Class
5. 可对文本中的变量进行解析

### ConfigurablePropertyResolver接口

ConfigurablePropertyResolver为PropertyResolver的实现提供了可配置的支持。

为了让PropertyResolver具有类型转换的能力，为其提供了`void setConversionService(ConfigurableConversionService conversionService);`的接口方法，令PropertyResolver可配置类型转换的策略。

为了让PropertyResolver可自定义变量解析的前后缀，默认值的分隔符，是否可忽略不可解析的变量，ConfigurablePropertyResolver定义以下接口方法：

	void setPlaceholderPrefix(String placeholderPrefix);

	void setPlaceholderSuffix(String placeholderSuffix);

	void setValueSeparator(String valueSeparator);
	
	void setIgnoreUnresolvableNestedPlaceholders(boolean ignoreUnresolvableNestedPlaceholders);

为了约束PropertyResolver必须解析某些属性，ConfigurablePropertyResolver又定义了：

	void setRequiredProperties(String... requiredProperties);

	void validateRequiredProperties() throws MissingRequiredPropertiesException;

### AbstractPropertyResolver

AbstractPropertyResolver是一个abstract的类，为PropertyResolver引用了必要的功能类和数据结构，这些在AbstractPropertyResolver的属性定义中可以看出来：

	// 类型转换的策略
	protected ConfigurableConversionService conversionService = new DefaultConversionService();

	// 非约束的变量解析工具类，变量解析都是由工具类PropertyPlaceholderHelper来完成
	private PropertyPlaceholderHelper nonStrictHelper;

	// 约束的变量解析工具类，变量解析都是由工具类PropertyPlaceholderHelper来完成
	private PropertyPlaceholderHelper strictHelper;

	// 是否忽略不能解析的变量
	private boolean ignoreUnresolvableNestedPlaceholders = false;

	// 变量解析前缀，默认为${
	private String placeholderPrefix = SystemPropertyUtils.PLACEHOLDER_PREFIX;

	// 变量解析后缀，默认为}
	private String placeholderSuffix = SystemPropertyUtils.PLACEHOLDER_SUFFIX;

	// 默认值的分隔符
	private String valueSeparator = SystemPropertyUtils.VALUE_SEPARATOR;

	// 必须可解析的属性集合
	private final Set<String> requiredProperties = new LinkedHashSet<String>();

另外，**AbstractPropertyResolver为PropertyResolver的实现定义了一套模板，这里应用的就是模板方法设计模式。**

### PropertySourcesPropertyResolver
 
PropertySourcesPropertyResolver是Spring对PropertyResolver的唯一实现类。Environment虽然也实现了PropertyResolver接口，但Environment的内部实现是使用了装饰模式，通过装饰PropertySourcesPropertyResolver来实现PropertyResolver的功能的。

PropertySourcesPropertyResolver在AbstractPropertyResolver的基础上新增了一个域propertySources，该域指定了PropertyResolver要解析的属性源：

	private final PropertySources propertySources;

下面针对PropertyResolver接口中抽象的行为，逐个进行分析。

#### 判断属性key是否存在

	//from PropertySourcesPropertyResolver.java
	@Override
	public boolean containsProperty(String key) {
		if (this.propertySources != null) {
			for (PropertySource<?> propertySource : this.propertySources) {
				if (propertySource.containsProperty(key)) {
					return true;
				}
			}
		}
		return false;
	}

这个实现就非常简单了，只是迭代每个属性源，判断属性是否存在。

#### key查找value

PropertyResolver的`getProperty()`的实现核心代码段为：

	//from PropertySourcesPropertyResolver.java
	protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
		boolean debugEnabled = logger.isDebugEnabled();
		if (logger.isTraceEnabled()) {
			logger.trace(String.format("getProperty(\"%s\", %s)", key, targetValueType.getSimpleName()));
		}
		if (this.propertySources != null) {
			for (PropertySource<?> propertySource : this.propertySources) {
				if (debugEnabled) {
					logger.debug(String.format("Searching for key '%s' in [%s]", key, propertySource.getName()));
				}
				// 从属性源中解析出key对应的value
				Object value = propertySource.getProperty(key);
				if (value != null) {
					Class<?> valueType = value.getClass();
					if (resolveNestedPlaceholders && value instanceof String) {
						value = resolveNestedPlaceholders((String) value);
					}
					if (debugEnabled) {
						logger.debug(String.format("Found key '%s' in [%s] with type [%s] and value '%s'",
								key, propertySource.getName(), valueType.getSimpleName(), value));
					}
					if (!this.conversionService.canConvert(valueType, targetValueType)) {
						throw new IllegalArgumentException(String.format(
								"Cannot convert value [%s] from source type [%s] to target type [%s]",
								value, valueType.getSimpleName(), targetValueType.getSimpleName()));
					}
					return this.conversionService.convert(value, targetValueType);
				}
			}
		}
		if (debugEnabled) {
			logger.debug(String.format("Could not find key '%s' in any property source. Returning [null]", key));
		}
		return null;
	}

上面的实现逻辑如下：

1. 从属性源中获取key对应的value值
2. value值为字符串，而且需要解析变量时，交由`AbstractPropertyResolver.resolveNestedPlaceholders()`方法进行变量解析
3. 利用ConversionService策略进行类型转换，关于ConversionService的源码分析，请看

跟踪`AbstractPropertyResolver.resolveNestedPlaceholders()`的代码实现，变量解析的核心实现交由PropertyPlaceholderHelper来完成。

	private String doResolvePlaceholders(String text, PropertyPlaceholderHelper helper) {
		return helper.replacePlaceholders(text, new PropertyPlaceholderHelper.PlaceholderResolver() {
			@Override
			public String resolvePlaceholder(String placeholderName) {
				return getPropertyAsRawString(placeholderName);
			}
		});
	}
PropertyPlaceholderHelper的核心实现代码如下：
	
	protected String parseStringValue(
			String strVal, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {

		StringBuilder result = new StringBuilder(strVal);

		int startIndex = strVal.indexOf(this.placeholderPrefix);
		while (startIndex != -1) {
		    // 查找变量结束符的索引
			int endIndex = findPlaceholderEndIndex(result, startIndex);
			if (endIndex != -1) {
				// 取出变量部分
				String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
				String originalPlaceholder = placeholder;
				// 防止变量解析嵌套，解析死循环
				if (!visitedPlaceholders.add(originalPlaceholder)) {
					throw new IllegalArgumentException(
							"Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
				}
				// Recursive invocation, parsing placeholders contained in the placeholder key.
				placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
				// 交由变量解析器去进行变量的解析
				String propVal = placeholderResolver.resolvePlaceholder(placeholder);
				if (propVal == null && this.valueSeparator != null) {
					int separatorIndex = placeholder.indexOf(this.valueSeparator);
					if (separatorIndex != -1) {
						String actualPlaceholder = placeholder.substring(0, separatorIndex);
						String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
						propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
						if (propVal == null) {
							propVal = defaultValue;
						}
					}
				}
				if (propVal != null) {
					// Recursive invocation, parsing placeholders contained in the
					// previously resolved placeholder value.
					propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
					result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
					if (logger.isTraceEnabled()) {
						logger.trace("Resolved placeholder '" + placeholder + "'");
					}
					startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
				}
				else if (this.ignoreUnresolvablePlaceholders) {
					// Proceed with unprocessed value.
					startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
				}
				else {
					throw new IllegalArgumentException("Could not resolve placeholder '" +
							placeholder + "'" + " in string value \"" + strVal + "\"");
				}
				visitedPlaceholders.remove(originalPlaceholder);
			}
			else {
				startIndex = -1;
			}
		}

		return result.toString();
	}

变量解析的过程描述如下：

1. 如果文本中包含有变量前缀，执行步聚2。否则跳到步聚3
2. 提取变量前缀与后缀之间的变量名，变量名作为文本执行步聚1
3. 由PlaceholderResolver对变量名进行解析，解析成功，则使用解析后的值替换掉变量占位符;解析失败，按默认分隔符分离出真实的变量名与默认值，再次对真实的变量名进行解析，如果再次解析失败，则使用默认值作为解析结果

