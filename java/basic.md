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
    > **protected** native方法，默认实现是浅拷贝，如果类未实现Cloneable接口调用clone方法会抛出CloneNotSupportedException，**注意Object类未实现Cloneable**。
    >> 所有数组都被视为已经实现了Cloneable，并且访问修饰符被修改为了public，clone方法会返回一个利用原数组内元素构造的新数组，使用equals方法会返回false，因为实际调用的是Object的equals方法。
  - notify()、notifyAll()
    > **final** native方法，notify唤醒正在此对象监视器上等待的任意的一个线程，notifyAll唤醒正在此对象监视器上等待的所有线程。被唤醒的线程仍然无法继续，直到当前线程放弃对对象的锁定。
  - wait()、wait(long timeoutMillis)、wait(long timeoutMillis, int nanos)
    > **final** native方法，调用的均是wait(long timeoutMillis)方法，使当前线程放弃对该对象的监视器锁，等待被唤醒或者时间达到超时时间，之后与其他等待中的线程公平竞争该对象的锁。
    >> **wait() == wait(0L)，并不是只放弃锁之后立刻进入对象锁等待池，而是一直等待直到被notify方法唤醒**。
    
##### String
  - private final byte[] value
    
