## [首页](/blog/)

> version: **jdk11**

***

## Type
> Java编程语言中所有类型的公共超接口。

***

## Class
> 没有公共构造函数，当类加载器调用defineClass方法之一时JVM会自动构造类对象。

- Class<?> forName(String className)：native实现，使用该方法，虚拟机会初始化该类，不能用于获取表示基本类型或void的任何类对象。
- isAssignableFrom()：判断当前类是否是给定类的父类。
- TypeVariable<Class<T>>[] getTypeParameters()：按声明的顺序返回当前类的泛型声明。
- Type getGenericSuperclass()：返回带泛型信息的直接父类。
- Type[] getGenericInterfaces()：返回实现的接口数组，带泛型信息。
- getEnumConstants()：如果是枚举类型则相当于values方法，否则返回null。
- getResourceAsStream()：不加'/'，视为相对路径，从该类的包下查找，否则从根目录查找，内部实现是构造出一个绝对路径，最终还是由ClassLoader获取资源。
- getClassLoader().getResourceAsStream：不需要加'/'，默认从根目录获取资源。

### Class中几个方法的比较
``` java
// 数组
System.out.println(String[].class.getTypeName()); // java.lang.String[]
System.out.println(String[].class.getName()); // [Ljava.lang.String;
System.out.println(String[].class.getSimpleName()); // String[]
System.out.println(String[].class.getCanonicalName()); // java.lang.String[]

// 成员内部类
System.out.println(BBB.class.getTypeName()); // lang.reflect.AAA$BBB
System.out.println(BBB.class.getName()); // lang.reflect.AAA$BBB
System.out.println(BBB.class.getSimpleName()); // BBB
System.out.println(BBB.class.getCanonicalName()); // lang.reflect.AAA.BBB

// 匿名内部类
System.out.println(new Object(){}.getClass().getTypeName()); // lang.reflect.AAA$4
System.out.println(new Object(){}.getClass().getName()); // lang.reflect.AAA$1
System.out.println(new Object(){}.getClass().getSimpleName()); // ""
System.out.println(new Object(){}.getClass().getCanonicalName()); // null

// 普通类
System.out.println(AAA.class.getTypeName()); // lang.reflect.AAA
System.out.println(AAA.class.getName()); // lang.reflect.AAA
System.out.println(AAA.class.getSimpleName()); // AAA
System.out.println(AAA.class.getCanonicalName()); // lang.reflect.AAA

// 基本数据类型
System.out.println(int.class.getTypeName()); // int
System.out.println(int.class.getName()); // int
System.out.println(int.class.getSimpleName()); // int
System.out.println(int.class.getCanonicalName()); // int
```

***

## WildcardType
> 通配符型表达，如 ? ， ? extends Number ，或 ? super Integer。

***

## GenericArrayType
> 范型数组，组成数组的元素是 ParameterizedType 或 TypeVariable。

- Type getGenericComponentType()：返回组成泛型数组的实际参数化类型

***

## ParameterizedType
> 参数化类型，例如Map.Entry<String, ,Person>entry。
>> 含有<>则为参数化类型，比如 List<? extends String> 为 ParameterizedType 非 WildcardType。

- Type[] getActualTypeArguments(); // String，Person
- Type getRawType(); // Entry
- Type getOwnerType(); // Map

***

## TypeVariable
> 类型变量，在编译时会被转换为一个特定的类型，用来反映在JVM编译该泛型前的信息。
>> 比如 TypeVariableBean<K extends InputStream & Serializable, V> 中的K ，V 都是属于类型变量。

- Type[] getBounds()：得到上边界的 Type数组。// K ：InputStream、 Serializable、 V：Object
- D getGenericDeclaration(); 返回的是声明这个 TypeVariable 所在的类的 Type // TypeVariableBean
- String getName(); 返回的是这个 TypeVariable 的名称 // K、V

***

## AccessibleObject
> 是字段 ， 方法 ，和构造器（反射对象）的基础类。

- protected AccessibleObject()：构造函数仅供虚拟机使用。
- final boolean trySetAccessible()：jdk9新增，设置失败返回 false 而 setAccessible(true) 失败时抛出InaccessibleObjectException。

***

## Array
> 提供了动态创建和访问Java数组的本地静态方法。

***

## ?（通配符）
> 可以对泛型参数做出某些限制，使代码更安全。

### 与泛型的区别：
> T表示一个特定的类型，？表示一个未知的类型，而不是代表所有的类型。

#### 上边界限定：
> List<? extends Fruit>，一个类型的 List， 这个类型可以是继承了 Fruit 的某种类型。注意，这并不是说这个 List 可以持有 Fruit 的任意类型。通配符代表了一种特定的类型，它表示 “某种特定的类型，但是没有指定”，Fruit 是它的上边界。那么对这样的一个 List 我们能做什么呢？我们不知道这个 List 到底持有什么类型，怎么可能安全的添加一个对象呢？所以向 List 中添加任何对象，无论是 Apple 还是 Orange 甚至是 Fruit 对象，编译器都不允许，唯一可以添加的是 null，在这个 List 中，不管它实际的类型到底是什么，但肯定能转型为 Fruit，所以编译器允许返回 Fruit。

#### 下边界限定：
> List<? super Apple>，它表示某种类型的 List，我们不知道实际类型是什么，但是这个类型肯定是 Apple 的父类型。因此，向这个 List 添加一个 Apple 或者其子类型的对象是安全的，这些对象都可以向上转型为 Apple。但是我们不知道加入 Fruit 对象是否安全，因为那样会使得这个 List 添加跟 Apple 无关的类型。同时编译器只允许返回Object对象，因为只能向上转型为Object。

***