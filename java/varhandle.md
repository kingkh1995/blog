# [首页](/blog/)

> VarHandle since 9
>> Reference: <https://www.lenshood.dev/2021/01/27/java-varhandle/>

## VarHandle

使用 java.lang.invoke.Varhandle 来替代 Unsafe 大部分功能，对变量或参数定义的变量系列的动态强类型引用，包括静态字段，非静态字段，数组元素或堆外数据结构的组件。

### MethodHandles

- MethodHandles.lookup()

    创建一个默认的 Lookup 对象，用于查找**当前类或包**中的方法、字段等成员。

    ```java
    public class VarHandleTest {
        private int i;
        private static VarHandle VH;

        static {
            try {
                VH = MethodHandles.lookup().findVarHandle(VarHandleTest.class, "i", int.class);
            } catch (Exception e) {
            throw new RuntimeException(e);
            }
        }

        public static void main(String[] args) {
            var test = new VarHandleTest();
            VH.set(test, 5);
            System.out.println(VH.get(test));
        }
    }
    ```

- MethodHandles.privateLookupIn(Class<?> targetClass, Lookup caller)

    创建一个可以访问指定类的 private 成员的 Lookup 对象。

- MethodHandles.publicLookup()

    创建一个可以访问公共成员成员的 Lookup 对象。

### Lookup

- findVarHandle(): 获取非静态字段

- findStaticVarHandle()：获取静态字段

- unreflectVarHandle(Field f)：通过反射获取字段

### 访问模式

支持四种共享内存的访问模式

#### Plain

普通访问模式，不保证内存可见性及执行顺序；对应方法为get()、set() ；**在不涉及并发的场景下使用**。

#### Opaque

保证执行顺序，不保证内存可见性（**最终会可见**），保证位原子性（bitwise）；对应方法为setOpaque()、getOpaque()。

#### Release/Acquire (RA)

保证执行顺序，setRelease确保前面的load和store不会被重排序到后面，但不确保后面的load和store重排序到前面；getAcquire确保后面的load和store不会被重排序到前面，但不确保前面的load和store被重排序；对应方法为setRelease()、getAcquire()；RA 模式更多的用于 ownership 模型，即只有 owner 才能写，其他线程只能读。

- Release模式的写操作之前所有读/写操作，一定会发生在Release写之前；
- Acquire模式的读操作之后所有读/写操作，一定会发生在Acquire读之后。

#### Volatile

标准的volatile语义，成本最高的操作；对应方法为setVolatile()、getVolatile()。

#### 示例

```java
volatile int x, y; // initially zero, with VarHandles X and Y

Thread 1               |  Thread 2
X.setM(this, 1);       |  Y.setM(this, 1);
int ry = Y.getM(this); |  int rx = X.getM(this);
```
如果是Volatile模式，则ry和rx至少有一个为1，因为在get之前set一定会先发生；如果是RA模式，则rx和ry都可能为0，因为只能保证set之前所有的get先发生，不能保证set之后的get一定会发生。

### 内存屏障

支持五种内存屏障。

- fullFence：保证方法调用之前的所有读写操作不会被方法后的读写操作重排
- acquireFence：保证方法调用之前的所有读操作不会被方法后的读写操作重排
- releaseFence：保证方法调用之前的所有读写操作不会被方法后的写操作重排
- loadLoadFence：保证方法调用之前的所有读操作不会被方法后的读操作重排
- storeStoreFence：保证方法调用之前的所有写操不会被方法后的写操作重排

### VarHandle.AccessType

```java
enum AccessType {
    GET(Object.class),
    SET(void.class),
    COMPARE_AND_SET(boolean.class),
    COMPARE_AND_EXCHANGE(Object.class),
    GET_AND_UPDATE(Object.class);
}
```

### VarHandle.AccessMode

支持的访问模式，每个枚举对应一个方法。
