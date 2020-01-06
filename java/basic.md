## [首页](https://kingkh1995.github.io/blog/)
> version: **jdk11**

### lang

##### Object
  - registerNatives()
    > native方法，在静态代码块中被调用。
  - getClass()
    > native方法，返回一个运行时Class对象，返回的类对象是被该类static synchronized方法锁住的对象。
  - hashCode()
    > native方法，返回对象的哈希码值，**哈希码值和对象地址有一定关联，但并不一定如此**，具体取决于运行时库和JVM的具体实现。
  - equals()
    > 默认是对象引用对比的结果，通常需要在重写此方法时覆盖hashCode方法，是**为了维护hashCode方法的常规协定（散列集合是基于这些协定设计的）**，equals方法对比相等的对象必须具有相等的哈希码值，哈希码值不相等的对象使用equals方法对比必然不相等（可推算出）。
  - clone()
    > **protected** native方法，默认实现是浅拷贝，如果类未实现Cloneable接口调用clone方法会抛出CloneNotSupportedException，**所有数组都被视为已经实现了Cloneable，并且访问修饰符被修改为了public**。
    >> 注意Object类未实现Cloneable，一个对象想要调用clone方法需要实现Cloneable，并重写clone方法，可能还需要修改访问修饰符。
    
