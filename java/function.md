# [首页](/blog/)

> version: **jdk17**

***

## Lambda表达式

Lambda表达式不会被编译为class文件，**其实是被封装成了主类的一个静态私有方法**；如果是捕获型，则每次都会创建一个对象，否则只会创建一个对象；且无论如何编译器都只会生成一个类对象，生成的代码大概是如下的样子：

```java
// 非捕获型lambda表达式（包括静态方法引用）
public class lambda{

    // 通过invokedynamic指令
    Function<Object, String> f = [dynamic invocation of lambda$1]

    // 生成为静态方法
    static String lambda$1(Object obj){
        return obj -> obj.toString();
    }
}

// 捕获型lambda表达式（包括对象方法引用）
public class lambda{

    // 捕获的参数会被添加到生成方法的参数列表中，如果是对象方法引用则必定捕获该对象。
    A arg = a;

    Function<Object, String> f = [dynamic invocation of lambda$1]

    static String lambda$1(A a, Object obj){
        return obj -> a.toString() + obj.toString();
    }
}
```

Lambda表达式**访问外部变量即是捕获**，必须是final的或者隐式final（不能被修改）。因为Lambda表达式对外部变量的访问实际上是其副本，且Lambda表达式也可能会在其他线程上处理，而局部变量是分配在线程栈上的，所以执行的时候可能已经被回收了，故不允许修改被捕获的外部变量。

***

## SerializedLambda

编译器会为继承了Serializable的lambda表达式（及方法引用）添加writeReplace方法，序列化时将其转换为SerializedLambda对象；在SerializedLambda类中也定义了readResolve方法，反序列化时将SerializedLambda对象转换为lambda表达式。

### 属性

- capturingClass：捕获lambda表达式的主类，即lambda表达式所在的类。
- implClass：lambda表达式的主类，如果是方法引用，则为方法所属的类。
- implMethodName：为编译器生成的私有方法名，以 lambda$main$ 开头，如果是方法引用则为方法名。
- capturedArgs：捕获的参数，如果是对象方法引用，则是引用的对象。

***