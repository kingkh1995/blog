## [首页](/blog/)
> **DDD 之 Domain Primitive**
>> 不是讲架构，而是一种比较好用的开发思想。

***

## Java Bean

* 所有属性为private
* 提供默认构造方法
* 提供getter和setter
* 实现serializable接口

### 问题

以前是被告知好处是使用getter和setter可以自行添加业务逻辑，但是慢慢共识已经变成不允许添加任何逻辑了，如果添加了可能造成不必要的bug，唯一的作用可能只是说控制某些属性的可见性和可修改性。

---

## Data Object (DO)

* 首先是一个Java Bean
* 数据库模型的映射
* **日常开发使用的数据模型**，一般放置在 entity 或 domian 包下
```java
// 一个典型的DO类
@Data
@TableName("tb_user")
public class User implements Serializable {
    @TableId
    private Long id;
    private String name;
    private String phone;
    private String address;
}
```

### DO应该如何设计和使用

* 是一个java bean
* 字段与数据库字段一一对应
* 将其活动范围限制在基础设施层

### DTO Assembler & Data Converter

* DTO Assembler：DTO 和 entity互相转化
* Data Converter：DO 和 entity互相转化

---

## 简单的web三层架构

UI层（controller）、业务层（service、facade等）、基础设施层（mapper、repository等）

- 问题：
    * 上层对下层直接依赖，耦合度过高（本次不涉及）
    * 业务层没有自己的数据模型，将DO当成了Entity（领域）使用。

- 正确的数据模型对应关系：
    * UI层：VO、DTO、Query、Command
    * 业务层：Entity (domain)
    * 基础设施层：DO、DTO(rpc)

### 将DO当作entity使用带来的问题

**一段用户注册的代码示例：**

```java
// 业务层
public class RegistrationServiceImpl implements RegistrationService {

    private UserRepository userRepo;

    public User register(String name, String phone, String address) {

        // 校验逻辑
        if (name == null || name.length() == 0) {
            throw new ValidationException("name");
        }
        if (phone == null || !isValidPhoneNumber(phone)) {
            throw new ValidationException("phone");
        }
        // 省略其他字段校验逻辑

        // 业务逻辑
        if(userRepo.findByName(name) != null){
            throw new ValidationException("姓名不能重复");
        }
        if(userRepo.findByPhone(phone) != null){
            throw new ValidationException("手机号不能重复");
        }
        // 省略其他业务逻辑

        // 创建用户DO
        User user = new User();
        user.setName(name);
        user.setPhone(phone);
        user.setAddress(address);

        // 调用基础设施层方法持久化 返回结果
        return userRepo.save(user);
    }
}
```
* 问题1：清晰度不明确，属性使用的是没有特殊意义的原始类型，并不是一个具体的概念，只能通过变量名，无法通过变量类型去区分。
```java
    User register(String name, String phone, String address) 

    // 上述注册方法在运行时 参数全是String类型
    User register(String, String, String);

    // 在controller层如此调用不会出现问题，因为是完全可以运行的，甚至如果遗漏了参数校验，甚至可能直接落库，导致出现脏数据
    service.register("张三", "浙江省杭州市", "13211110000");

    // 因为name和phone类型都是String 所以只能在方法名加上ByXXX区分
    User findByName(String name);

    User findByPhone(String phone);

    // 多个参数也只能通过变量名区分
    User findByNameOrPhone(String name, String phone);

    // 参数弄错可以直接运行，运行后出现问题不好排查
    findByNameOrPhone(user.getPhone(), user.getName());
```
* 问题2：数据验证和错误处理代码四散，每次使用Name，Phone这种有业务意义的参数时，都需要添加相应的校验代码，会有大量的重复代码，维护成本非常高，而且也是不可控的，因为无法保证是否会被调用以及调用的方式是否正确。
* 问题3：业务代码的清晰性，如果需要解析参数，如从address参数解析出省市区信息，我们还得额外添加一段代码。
* 问题4：可测试性，如果我想测试校验address参数是否合法的代码？我们如何编写测试用例？
```java
    // 弊端：不能保证调用方使用相同的调用方式
    @Test
    void testAddress() {
        String address = " ";
        //
        if(address==null){
            throw new ValidationException();
        }
        if(!address.contains("省")){
            throw new ValidationException();
        }
        // 其他省略

    }

    // 抽离出静态工具类 弊端业务逻辑分散
    @Test
    void testAddress() {
        String address = " ";
        //
        if(AddressUtils.validate(address)){
            throw new ValidationException();
        }
    }
```

#### 现有的解决方案

- 抽离成公共代码
  - 不在同一个类里也需要粘贴复制，以后修改得多处修改。
- 抽离成静态工具类
    * 工具类的基本原则就是封闭修改的，而静态代码是一段业务逻辑，不可避免的会被修改；
    * 大量的静态工具类也会造成业务逻辑代码的分散，增加维护难度；
    * 无法控制其他开发人员如何使用静态工具类，或者还需要去了解使用的方法。

***根本问题就是：name、address、phone都是具有业务意义的概念，并不能简单的用原始类型去描述。***

---

## Java Primitive 原始类型

八大原始类型，以及对应的包装类、String 、BigDecimal、BigInteger、以及枚举等都可以视作 Java 语言和 Java Bean 的基础。

* 不从任何事物发展而来
* 初级的形成或生长的早期阶段

## Domain Primitive 领域的基础

* DP 是一个传统意义上的 Value Object （表示一个有含义的值的对象），拥有 Immutable 的特性。
* DP 是一个完整的概念整体，拥有精准定义。
* DP 使用业务域中的原生语言。
* DP 可以是业务域的最小组成部分、也可以构建复杂组合。

## 如何创建一个DP？

* 隐形概念显性化
    * 创建一个 Type（数据类型）去显性的表示一个概念。
    * 将这个概念相关的逻辑完整的收集到一个 Class（类）里。
```java
// 参考Integer String 等原始类型去设计DP
public class Name {
    // 用final修饰属性 创建完即不可变
    @Getter
    private final String name;

    // 构造方法不对外暴露
    private Name(String name) {
        this.name = name;
    }

    // 使用静态方法创建对象
    public static Name valueOf(String name) throws ValidationException {
        // 参数校验
        if (name == null || name.isBlank()) {
            throw new ValidationException("不能为空");
        }
        if (!Pattern.compile("[\\u4e00-\\u9fa5]+").matcher(name).matches()) {
            throw new ValidationException("姓名只能为中文！");
        }
        // 其他省略

        // 创建对象并返回
        return new Name(name);
    }
}

// DP类应该定义行为方法
public class Address {

    private final String address;

    // 其他方法省略

    // 一个获取省份的行为方法
    public String getProvince(){
        return this.address.split("省")[0];
    }
}
```
* 隐性上下文显性化
    * 拓展一个简单概念的上下文，将多个概念组合成一个独立的完整概念。
```java
// 钱这个概念其实隐性包含有币种这个概念，只是我们会将币种默认为人民币，但不代表其不存在
// 金额和币种才能完整的构成money这个概念，可以先将其定义，方便以后拓展。
public class Money {

private BigDecimal amount;

// 币种可以是一个DP或者一个枚举
private Currency currency;
}
```
* 封装多对象行为（不再拓展）
    * 一个概念涉及多个对象之间的复杂业务逻辑，将其封装为 DP，简化原始代码。

## DP的使用

```java
// DP的使用

// 接口方法
User find(Name name);

User find(Phone phone);

User find(Name name, Phone phone);

// 类型不匹配，编译出错，及时发现问题
find(user.getPhone(), user.getName());

//测试用例
@Test
void testAddress() {
    String address = " ";
    // 创建一个Address对象即可。
    Address.valueOf(address);
}
```
** 一旦创建必然合法，定义的行为方法能保证安全使用，即完全可控，所有业务逻辑全部封装在类中。**

## DP适合的场景

* 有格式限制的字符串：比如 Name，PhoneNumber，OrderNumber，ZipCode， Address 等。
* 有限制的整数：比如 OrderId（>0），Percentage（0-100%），Quantity（>=0） 等。
* 浮点数：一般用到的 Double 或 BigDecimal 都是有业务含义的，比如 Temperature、Money、Amount、ExchangeRate、Rating 等。
* 复杂的数据结构：比如 Map 等，尽量能把 Map 的所有操作包装掉，仅暴露必要行为。
### 完整示例：

```java
/**
 * 有范围长整型DP类基类 <br>
 *
 * @author kaikoo
 */
@EqualsAndHashCode(callSuper = true)
public abstract class RangedLong extends Number implements Type {

  private final long value;

  /**
   * @param value 数值
   * @param fieldName 字段名称
   * @param min 最小值
   * @param minInclusive 是否包含最小值
   * @param max 最大值
   * @param maxInclusive 是否包含最大值
   */
  protected RangedLong(
      long value,
      String fieldName,
      Long min,
      Boolean minInclusive,
      Long max,
      Boolean maxInclusive) {
    if (min != null) {
      var cmp = Long.compare(value, min);
      if ((minInclusive && cmp < 0) || (!minInclusive && cmp <= 0)) {
        throw IllegalArgumentExceptions.forMinValue(fieldName, min, minInclusive);
      }
    }
    if (max != null) {
      var cmp = Long.compare(value, max);
      if ((maxInclusive && cmp > 0) || (!maxInclusive && cmp >= 0)) {
        throw IllegalArgumentExceptions.forMaxValue(fieldName, max, maxInclusive);
      }
    }
    this.value = value;
  }

  @JsonValue
  public long getValue() {
    return this.value;
  }

  protected static long parseLong(Object o, String fieldName) {
    if (o == null) {
      throw IllegalArgumentExceptions.forIsNull(fieldName);
    } else if (o instanceof Long l) {
      return l;
    } else if (o instanceof String s) {
      try {
        return Long.parseLong(s);
      } catch (NumberFormatException e) {
        throw IllegalArgumentExceptions.forWrongPattern(fieldName);
      }
    }
    throw IllegalArgumentExceptions.forWrongClass(fieldName);
  }

  @Override
  public int intValue() {
    return (int) this.value;
  }

  @Override
  public long longValue() {
    return this.value;
  }

  @Override
  public float floatValue() {
    return (float) this.value;
  }

  @Override
  public double doubleValue() {
    return (double) this.value;
  }
}

/**
 * long类型Id
 *
 * @author kaikoo
 */
@EqualsAndHashCode(callSuper = true) // 重写EqualsAndHashCode
public class LongId extends RangedLong implements Identifier {

  protected LongId(long value, String fieldName) {
    super(value, fieldName, 0L, false, null, null);
  }

  @JsonCreator
  public static LongId of(long l) {
    return new LongId(l, "LongId");
  }

  public static LongId valueOf(Object o, String fieldName) {
    return new LongId(parseLong(o, fieldName), fieldName);
  }

  @Override
  public String identifier() {
    return String.valueOf(getValue());
  }
}
```

---

## 项目改造

1. 确定概念，创建DP类，收集概念的所有相关业务逻辑和行为。
2. 替换所有创建和使用
3. 创建新接口
4. 修改外部调用
```java
    // controller层
    @PostMapping("/user")
    public Boolean register(@RequestBody @Valid UserCreateCommand command) {
        // 做一些参数的非空校验

        // 参数封装为DP对象
        return registerService.register(Name.valueOf(command.getName()), Phone.valueOf(command.getPhone), Address.valueOf(command.getAddress()));
    }

    // service层
    public User register(Name name, Phone phone, Address address) {

        // 不再需要参数校验逻辑，DP对象创建出来后就必然是合法的

        // 业务逻辑 （其实可以移到 entity 的行为方法中）
        if(userRepo.find(name) != null){
            throw new ValidationException("姓名不能重复");
        }
        if(userRepo.find(phone) != null){
            throw new ValidationException("手机号不能重复");
        }

        // 省略其他业务逻辑

        // 创建entity
        User user = User.builder().name(name).phone(phone).address(address).build();

        // 调用持久层方法持久化 返回结果
        return userRepo.save(user);
    }
```

---

## Entity 实体 业务模型

* 属于领域对象，业务层使用
* 由DP组成
* 生命周期存在内存中，不需要序列化
* 拥有业务行为
* 不对外开放属性的修改
* 字段和方法与业务语言一致

DO作为数据模型的可靠性无法保障，创建了一个对象，这个对象无法被可靠的创建，且在整个周期内无法可靠的运行。因为暴露了构造方法和set方法，可以随意创建一个不符合业务规范的对象，且生存周期内可以随意通过set方法修改属性。

***下面是一个entity的案例：***

```java
/**
 * 用户账户
 *
 * @author KaiKoo
 */
@JsonDeserialize(builder = Account.AccountBuilder.class) // 设置反序列化使用Builder
@EqualsAndHashCode(callSuper = true)
@Getter
@Builder
public class Account extends Entity<AccountId> {

  @Setter(AccessLevel.PROTECTED)
  private AccountId id;
  @DiffIgnore private Instant createTime;
  @DiffIgnore private Instant updateTime;
  private AccountState state;

  // 行为方式使用依赖反转
  public void save(AccountService accountService) {
    // handle
    if (this.id == null) {
      // 新增逻辑
      // 设置初始状态
      this.state = AccountState.of(AccountStateEnum.INIT);
    } else {
      // 更新逻辑
      if (!accountService.allowModify(this)) {
        throw new BusinessException("不允许修改");
      }
    }
    // save
    accountService.save(this);
  }

  // 行为方法，只能在entity内部修改属性
  public void invalidate() {
    // 领域都是合法的，可以忽略空指针问题。
    if (AccountStateEnum.ACTIVE.equals(this.state.getValue())) {
      this.state = AccountState.of(AccountStateEnum.TERMINATED);
    } else {
      throw new BusinessException("当前状态无法失效。");
    }
  }
}
```

---

## 总结

DP就是为一个有业务意义的字段创建一个类，并把相关的所有代码全写在这个类中，其实平常大家应该都有过这样的设计想法，只是Java Bean的影响太深了。

entity由DP组成，字段不需要和数据库一一对应，不需要持久化。 
