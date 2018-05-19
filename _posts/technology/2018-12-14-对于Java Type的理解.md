之前一直对Java Type与Class的概念感到模糊，通过上网翻阅资料，算是有个深刻的理解了。

> 术语说明 

类型参数：指的是类型声明中的< >部分描述的参数列表
类型变量：如Map<K,V extends Number>，类型变量指的就是K，V。

--------

由于Java的类型擦除机制，在运行时无法获取类型参数所表达的真实对象类型。但是，在编译期，类字节码中已经保存了源代码中的类型声明，包括类型参数。而这些源代码中的类型声明，就是使用`Type`这种数据类型来表达了。

Java中的`Type`描述的是源代码中各种类型的声明：

`java.lang.reflect.Type`是对源代码中所有类型声明的抽象，它只有一个抽象方法，用于描述类型名称：

	default String getTypeName() {
	    return toString();
	}

`java.lang.Class`也实现了`java.lang.reflect.Type`，`java.lang.Class`在原有对字节码描述的基础上，加上了对源代码中普通类型声明描述，如对于String来说明，就是`java.lang.String`。`java.lang.Class`同时实现了`java.lang.reflect.GenericDeclaration`接口，即`java.lang.Class`对象可以获取自身类型声明中参数类型。如：

	@Test
	public void testForGenericDeclaration1() {
		TypeVariable<Class<Map>>[] typeParameters = Map.class.getTypeParameters();//获取到的就是Map接口定义中的<K,V>
		Assert.assertEquals("K",typeParameters[0].getTypeName());
		Assert.assertEquals("V", typeParameters[1].getTypeName());
	}
	
`java.lang.reflect.ParameterizedType`是对源代码中使用泛型声明的类型抽象（如：`Map<String,String>`或`List<T>`）,因此它的接口声明中使用了三个抽象方法来描述这种类型的特征：

	//用来描述泛型类型声明中的<>部分，如对于Map<String,String>来说，返回就是[Class<String>,Class<String>]，对于List<T>来说，返回的就是[TypeVariable<Class<T>>]
	Type[] getActualTypeArguments();
	
	//获取泛型声明中的原始类型，对于`Map<String,String>`来说明就是Class<Map>，对于List<T>来说就是Class<List>
	Type getRawType();
	
	// 获取泛型声明是一个内部类，那么获取主类的类型声明。
	Type getOwnerType();
	
`java.lang.reflect.TypeVariable<D>`描述的是类型声明中的类型变量部分。它的接口定义描述了它的特征：

	//类型变量声明的上界，如T extends Number，T的上界就是Number
	Type[] getBounds();
	
	//获取类型变量的声明类
	D getGenericDeclaration();
	
	String getName();
	
`java.lang.reflect.GenericArrayType`描述的是泛型数组，它的组成元素是`java.lang.reflect.ParameterizedType`或`java.lang.reflect.TypeVariable<D>`

`WildcardType`描述的是`? extends Number`和 `? super Integer`这种通配符类型。



