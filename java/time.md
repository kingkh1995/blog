## [首页](https://kingkh1995.github.io/blog/)
> version: **jdk11**

#### 日期时间类
- **所有类均为线程安全，不可变类**
- Temporal 接口：所有日期，时间和偏移对象的基本接口类型，定义了对时态对象的读写访问，继承自 TemporalAccessor 接口，该接口只定义了只读访问。
- Instant 类：时间戳，等同于 java.util.Date & java.sql.Timestamp，通过 from(Instant instant) 和 toInstant() 互相转换。
- ZonedDateTime 类：带时区信息的日期时间类，本地时间日期类和 Instant 之间转换的中介，需要指定 ZoneId。
  - toLocalTime() & toLocalDateTime() & toLocalDate() & toInstant()
  - Instant 和 LocalDateTime 的 atZone(ZoneId zone) 方法、LocalDate 的 atStartOfDay(ZoneId zone) 方法。

#### 时间量
- TemporalAmount 接口：时间量基本接口
- Duration 类：基于时间的时间量，以秒和纳秒为单位，between() 方法计算两个时间对象之间的时间量，如果两个对象类型不同，将会把第二个对象转换为第一个对象类型。
- Period 类：基于日期的时间量，以年月日为单位，between() 方法计算两个LocalDate之间的时间量。

#### 日期时间类的操作
- ChronoUnit 枚举：TemporalUnit接口的实现，提供了一组标准日期时间单位 。
- ChronoField 枚举：TemporalField接口的实现，提供一组标准日期时间字段字段。
##### 查询
- 使用时间日期类对应的 getXXX() 方法以及 int get(TemporalField field)。

##### 增加减少
- 使用时间日期类对应的 plusXXX()方法和minusXXX() 以及 minus(TemporalAmount amountToAdd)和plus(TemporalAmount amountToAdd)。
- **要知道每次调用方法都会创建一个新的对象，因为均为不可变类**，所以要减少调用次数。

##### 调整
- 使用时间日期类对应的 withXXX() 方法、 with(TemporalAdjuster adjuster)方法和 with(TemporalField field, long newValue)方法。
- TemporalAdjusters 类：提供了各种静态方法来获取各种实用的TemporalAdjuster。