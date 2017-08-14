*本文所引用的代码为JDK 1.8版本*

[请尊重博主劳动成果，转载请标明原文链接。](http://blog.csdn.net/hwliu51/article/details/76945255)

## 反射

Java反射中最常使用到的几个类：Class，Constructor，Method，Field。

* Class：用于获取类的字节码对象，获取构造方法，普通方法，以及属性。
* Constructor；用于创建对象。
* Method：调用对象的方法和调用类的静态方法。获取方法上配置的注解信息。
* Field：修改对象的属性或类的静态属性，获取对象或类的属性值。获取属性上配置的注解信息。

在某些被限定不能访问的类，方法或属性，当我需要创建对象，调用方法或获取值时，则可以使用反射来绕过这些非public修饰符的限制。

### Class

常用到的几个方法：

#### 获取字节码对象

**forName(String className)**

该方法是静态方法，用于获取类的字节码对象，className为类的路径。使用方式

```java
Class<?> clazz = Class.forName("xx.xx.Xxx");
```
’xx.xx.Xxx‘是类的路径。

#### 获取构造器

**getConstructors()**

获取字节码对象中所有public类型的构造方法，返回值为Constructor<?>[]。

**getConstructor(Class<?>... parameterTypes)**

获取指定参数类型的public的构造方法，parameterTypes为需要传入的参数的class，返回值为Constructor<?>。
例如，类A的构造方法如下：
```java
public A(int i, String s){
	...
}
```

则获取A的该构造方法的方式（clazz为类A的字节码对象）
```java
Constructor<?> con = clazzA.getConstructor(int.class, String.class);
```

**getDeclaredMethods()**
获取字节码对象中所有类型的构造方法，返回值为Constructor<?>[]。

**getDeclaredConstructor(Class<?>... parameterTypes)**
获取指定参数类型的构造方法。private，protected和缺省的构造方法都能够获取。parameterTypes为需要传入的参数的class，返回值为Constructor<?>。使用的方式与**getConstructor()**一致。

#### 获取方法

**getMethods()**

获取字节码对象中public类型的方法，包含public类型的静态方法，返回值为Method[]。

**getDeclaredConstructors()**

获取字节码对象中所有类型的方法，返回值为Method[]。

**getMethod(String name, Class<?>... parameterTypes)**
获取值得名称和参数类型的public的方法。name为方法名称，parameterTypes为参数的class。因为Java类中可能会有重载，所以需要指定参数的类型。返回值为Method。

**getDeclaredMethod(String name, Class<?>... parameterTypes)**
获取值得名称和参数类型的方法。private，protected和缺省类型的方法都能够获取。返回值为Method。

#### 获取属性

**getFields()**

获取字节码对象中public类型的属性，包含public类型的静态属性，返回值为Field[]。

**getDeclaredFields()**

获取字节码对象中所有类型的属性，返回值为Field[]。

**getField(String name)**

获取指定名称的public属性，返回值为Field。

**getDeclaredField(String name)**
获取指定名称的属性，private，protected和缺省类型的都能够获取，返回值为Field。

#### 小结

* 获取所有public类型的构造器，方法或属性的方式：getXxxs()，返回数组。
* 获取所有类型的构造器，方法或属性的方式：getDeclaredXxxs()，返回数组。
* 获取指定的public类型的构造器，方法或属性的方式：getXxx()
* 获取指定的构造器，方法或属性的方式：getDeclaredXxx()

获取的方式都比较相似。

### Constructor

#### 创建对象

**newInstance(Object ... initargs)**

构造方法，使用该方法来创建对象，其中initargs为初始化的参数值。Java中的另外一种创建对象的方式是使用**new**关键字。

如果获取到的构造方法为public类型，则可以直接调用newInstance方法创建。如果为private，protected和缺省，则需要先调用`setAccessible(true)`，设置其为可访问，然后在调用newInstance方法。

#### 获取注解

**getAnnotation(Class<T> annotationClass)**

用于获取构造器上的注解信息，annotationClass为注解类的class。返回的为注解类的对象，通过该对象可以获取注解上的配置信息。

**getParameterAnnotations()**

用于获取构造方法中的注解信息。返回值为Annotation[][]。

### Method

#### 调用方法

**invoke(Object obj, Object... args)**

obj为创建或获取到的对象，args为调用该方法时需要传入的参数值。如果该Method对象为非public的则需要先调用`setAccessible(true)`，设置为可访问，才能使用invoke方法。如果不清楚method访问限定类型，可以先使用`getModifiers()`获取方法修饰符类型，再使用`Modifier.isPublic()`判断是否为public类型。如果Method为静态方法，则obj可以为null。

#### 获取注解

**getAnnotation(Class<T> annotationClass)**

获取方法上配置的注解，annotationClass为注解类的class。返回值为注解对象。

### Field

#### 设置或获取属性

**set(Object obj, Object value)**
设置Field属性值。obj为创建或获取到的对象，value为设置到该属性的值。如果该Field对象为非public的则需要先调用`setAccessible(true)`，设置为可访问，才能使用invoke方法。如果不清楚method访问限定类型，可以先使用`getModifiers()`获取方法修饰符类型，再使用`Modifier.isPublic()`判断是否为public类型。如果Field为静态方法，则obj可以为null。

**get(Object obj)**
获取Field属性值。使用方式同上。

#### 获取注解

**getAnnotation(Class<T> annotationClass)**

获取属性上配置的注解，annotationClass为注解类的class。返回值为注解对象。

### 小结

反射使用步骤：

1，通过Class.forName()获取类的字节对象。
2，通过字节对象获取到构造器Constructor<?>，如果为非public，则调用`setAccessible(true)`，并调用构造器创建对象。
3，通过字节对象获取需要操作的方法Method，调用invoke()执行方法。如果需要获取方法上的注解信息，则先调用getAnnotation()获取注解对象，然后从注解对象获取配置信息。
4，通过字节对象获取需要操作的属性Field，赋值则调用set()，获取值则调用get()。获取注解方法同上。


## 代理

代理分为两种：静态代理，动态代理。
静态代理是直接写在代码里，通过一个类去操作需要代理的类。
动态代理则是通过反射和Proxy及相关的类在代码运行时动态实现。

动态代理大致可以分为3个步骤：
1，建立委托，即创建Interface（接口）和完成实现类；
2，实现InvocationHandler，在invoke方法内执行代理操作；
3，通过Proxy.newProxyInstance()方法创建代理对象，将代理对象传递给调用者。


### Proxy

#### 创建代理对象

**newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)**

该方法是一个静态方法，返回值为代理对象。三个参数信息：

* loader：加载代理对象的字节码的ClassLoader。
* interfaces：代理对象的类实现的接口，或者是我们需要代理的方法的接口的class数组。
* h：执行代理操作类的对象

执行该方法，VM会动态生成一个名为$Proxy0的class文件。这个class文件实现了对interfaces中的方法的动态代理。如果代理的对象方法比较多，生成class文件的时间会比较长，且使用时会降低运行速度。


### InvocationHandler
这是一个接口，生成代理对象需要传入该接口实现类的对象。

#### 执行代理

**invoke(Object proxy, Method method, Object[] args)**
代理对象会调用这个方法来执行被代理的方法，也就是说我们可以在此根据需求对参数args进行修改或添加，或者执行其它的方法。

参数说明：
proxy：为代理的对象，即通过Proxy.newProxyInstance()生成的。
method：代理对象需要执行的方法。
args：被调用的方法需要传入的参数值。

## 测试

接口和实现类：

```java
package com.reflect.test;

public interface Interface {
	
	void method0(String s);
	
	void method1(int i);
}

//修饰符缺省，只能包内访问
class InterfaceImpl implements Interface{
	
	public InterfaceImpl(){}

	@Override
	public void method0(String s) {
		System.out.println(" s = " + s);
	}

	@Override
	public void method1(int i) {
		System.out.println(" i = " + i);
	}
	
}
```

InvocationHandler实现类和测试代码
```java
package com.reflect;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

import com.reflect.test.Interface;

public class ProxyTest {

	public static void main(String[] args) {
		try {
			//获取InterfaceImpl字节码对象
			Class<?> clazz = Class.forName("com.reflect.test.InterfaceImpl");
			Constructor<?> cons = clazz.getConstructor();
			//设置为可访问
			cons.setAccessible(true);
			//创建对象
			Object obj = cons.newInstance();
			//创建代理对象
			Interface interf = (Interface) Proxy.newProxyInstance(
					clazz.getClassLoader(), 
					new Class<?>[]{Interface.class}, 
					new InterfaceIH(obj));
			
			//代理对象调用方法
			interf.method0("call method0");
			interf.method1(1);
			
			clazz = Proxy.getProxyClass(clazz.getClassLoader(), new Class<?>[]{Interface.class});
			System.out.println(clazz.getCanonicalName());
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}

//InvocationHandler实现类
class InterfaceIH implements InvocationHandler{
	//目标对象
	Object target;
	
	public InterfaceIH(Object obj){
		target = obj;
	}
	
	@Override
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		String name = method.getName();
		//被调用的方法名称
		System.out.println("call method: " + name);
		//传入的参数信息
		if(args != null && args.length > 0){
			System.out.println("arg = " + args[0]);
		}
		
		//修改参数
		if("method0".equals(name)){
			args[0] = "proxy " + args[0];
		}else if("method1".equals(name)){
			args[0] = 10 + (Integer)args[0];
		}
		return method.invoke(target, args);
	}
	
}
```

测试结果：
```
call method: method0
arg = call method0
 s = proxy call method0
call method: method1
arg = 1
 i = 11
com.sun.proxy.$Proxy0
```

参考博客：

公共技术点之 Java 动态代理
http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86

深度剖析JDK动态代理机制
http://www.cnblogs.com/MOBIN/p/5597215.html


