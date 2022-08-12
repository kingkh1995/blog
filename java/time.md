# [首页](/blog/)

> version: **jdk17**

***

## java.util.Date

时间戳，**为可变对象**，只建议作为PO类的属性使用，对应数据库的TIMESTAMP和DATETIME类型。

-   ```java
    public Date() {
        this(System.currentTimeMillis());
    }

    public Date(long date) {
        fastTime = date;
    }

    // @since 1.8
    public static Date from(Instant instant) {
        try {
            return new Date(instant.toEpochMilli());
        } catch (ArithmeticException ex) { // 可能的异常来自instant.toEpochMilli()
            throw new IllegalArgumentException(ex);
        }
    }
    ```
    其他创建Date对象的方法均已被标为过时，不建议再使用。

- ```java
  public Instant toInstant() {
      return Instant.ofEpochMilli(getTime());
  }
  ```
  通过毫秒时间戳与Instant互相转换。

***

## Temporal Object（时间对象）

**java.time包下所有日期时间类均实现了Temporal、TemporalAdjuster、Comparable和Serializable，均是不可变的，即线程安全的。**

### TemporalField

时间字段接口，实现为**ChronoField枚举**，表示以何种方式对时间对象进行描述，如month-of-year、minute-of-hour等。

### **TemporalAccessor**

时间对象只读访问接口。

```java
// 是否支持给定TemporalField
boolean isSupported(TemporalField field);c

// 返回使用给定TemporalField描述的值
int get(TemporalField field); // 默认使用getLong方法实现
long getLong(TemporalField field);
```

### TemporalUnit

时间单位接口，实现为**ChronoUnit枚举**，表示以何种方式对事件对象进行测量，如年、月、日、小时、分、秒。

### Temporal extends TemporalAccessor

时间对象读写访问接口。

```java
// 是否支持给定TemporalUnit
boolean isSupported(TemporalUnit unit);

// 将按给定TemporalField描述的值调整为给定值，不是修改原对象而是返回新对象。
Temporal with(TemporalField field, long newValue);

// 使用给定的TemporalAdjuster调整当前时间
default Temporal with(TemporalAdjuster adjuster) {
    return adjuster.adjustInto(this);
}

// 按给定TemporalUnit计算直到另一个时间（不包含）的时间量，正数则表示是之后的时间。
long until(Temporal endExclusive, TemporalUnit unit);
```

### TemporalAdjuster

表示调整时间的策略，**所有时间类都实现了该接口，即时间对象自身也可以作为时间调整策略**。

```java
// 调整给定的时间，谨慎使用，如果字段不支持会抛出异常。
Temporal adjustInto(Temporal temporal);

// 实现类都是用with方法实现，故都是返回新对象。
default Temporal adjustInto(Temporal temporal) { // ChronoLocalDate
    return temporal.with(EPOCH_DAY, toEpochDay());
}
```

### TemporalAdjusters

时间调整策略工具类，返回公共的常用TemporalAdjuster对象。

### TemporalAmount

时间量接口，表示一段以特定时间单位测量的时间量，如‘6小时’、‘8天’、‘两年零三个月’等。

```java
// 当前时间量以给定时间单位测量出的值
long get(TemporalUnit unit);

// 当前时间量使用的所有时间单位
List<TemporalUnit> getUnits();
```

### final class **Duration** implements TemporalAmount

以秒和纳秒为单位测算的时间量。

```java
public static Duration between(Temporal startInclusive, Temporal endExclusive) {
    // 使用Temporal接口的until方法
    try {
        // 首先使用纳秒单位计算
        return ofNanos(startInclusive.until(endExclusive, NANOS));
    } catch (DateTimeException | ArithmeticException ex) {
        // 抛出异常后再使用秒单位计算
        long secs = startInclusive.until(endExclusive, SECONDS);
        long nanos; // 使用TemporalAccessor接口的getLong方法获取纳秒值
        try {
            nanos = endExclusive.getLong(NANO_OF_SECOND) - startInclusive.getLong(NANO_OF_SECOND);
            ...
        } catch (DateTimeException ex2) {
            nanos = 0;
        }
        return ofSeconds(secs, nanos); // 构造Duration对象
    }
}
```
返回两个**时间对象**之间以Duration表示的时间量。

### final class **Period** implements ChronoPeriod

以年月日为单位测算的时间量。

```java
public static Period between(LocalDate startDateInclusive, LocalDate endDateExclusive) {
    return startDateInclusive.until(endDateExclusive);
}
```
返回两个**LocalDate**之间以Period表示的时间量。

***

## abstract class Clock

特定时区的**不可变的**时钟对象，用于获取Instant，需要通过静态工厂方法创建。

- ```java
    // 获取时钟的时区信息
    public abstract ZoneId getZone();

    // 返回一个新时钟，将时区调整到给定时区。
    public abstract Clock withZone(ZoneId zone)

    // 获取当前瞬间
    public abstract Instant instant();

    // 获取当前毫秒时间戳
    public long millis() {
        return instant().toEpochMilli();
    }
    ```

- ```java
    public static Clock system(ZoneId zone) {
        Objects.requireNonNull(zone, "zone");
        if (zone == ZoneOffset.UTC) {
            return SystemClock.UTC;
        }
        return new SystemClock(zone);
    }
    ```
    需要指定时区，使用系统时钟获取瞬间，**故等于System.currentTimeMillis()**。

- ```java
    public static Clock tick(Clock baseClock, Duration tickDuration) {
        ...
        return new TickClock(baseClock, tickNanos);
    }
    ```
    需要指定基础时钟，此时钟每隔指定时间才会更新一次瞬间值。

- ```java
    public static Clock fixed(Instant fixedInstant, ZoneId zone) {
        Objects.requireNonNull(fixedInstant, "fixedInstant");
        Objects.requireNonNull(zone, "zone");
        return new FixedClock(fixedInstant, zone);
    }
    ```
    此时钟永远返回创建时设置的固定瞬间值。

- ```java
    public static Clock offset(Clock baseClock, Duration offsetDuration) {
        Objects.requireNonNull(baseClock, "baseClock");
        Objects.requireNonNull(offsetDuration, "offsetDuration");
        if (offsetDuration.equals(Duration.ZERO)) {
            return baseClock;
        }
        return new OffsetClock(baseClock, offsetDuration);
    }
    ```
    需要指定基础时钟，此时钟返回的瞬间与基础时钟返回的瞬间有固定偏移。

***

### Instant

精确到纳秒的瞬间，**用于替代TimeStamp**，是java.util.Date与java.time包日期时间类之间转换的桥梁（**toInstant方法和ofInstant方法**）。

- ```java
    // 获取系统当前瞬间
    public static Instant now() {
        return Clock.currentInstant();
    }

    // 使用时钟对象获取当前瞬间
    public static Instant now(Clock clock) {
        return clock.instant();
    }
    ```

- ```java
    public static Instant from(TemporalAccessor temporal) {
        if (temporal instanceof Instant) {
            return (Instant) temporal;
        }
        // 时间对象需要支持INSTANT_SECONDS和NANO_OF_SECOND，否则抛出DateTimeException。
        try {
            long instantSecs = temporal.getLong(INSTANT_SECONDS);
            int nanoOfSecond = temporal.get(NANO_OF_SECOND);
            return Instant.ofEpochSecond(instantSecs, nanoOfSecond);
        } catch (DateTimeException ex) {
            throw new DateTimeException( ... , ex);
        }
    }
    ```

- ```java
    public String toString() {
        return DateTimeFormatter.ISO_INSTANT.format(this);
    }
    ```
    默认输出为UTC时区ISO-8601格式（2011-12-03T10:15:30Z）。

***

## time-zone

### abstract class ZoneId implements Serializable

国际上使用的时区信息，无法直接创建对象。

- ```java
    public static final Map<String, String> SHORT_IDS = Map.ofEntries( ... );
    ```
    支持的的短时区ID（CTT=Asia/Shanghai）。

- ```java
    public static Set<String> getAvailableZoneIds() {
        return new HashSet<String>(ZoneRulesProvider.getAvailableZoneIds());
    }
    ```
    所有当前支持的时区ID，通过ServiceLoader加载。

- ```java
    public static ZoneId systemDefault() {
        return TimeZone.getDefault().toZoneId();
    }
    ```
    获取系统默认时区，如果其被修改，则方法返回值也会更新。

### final class ZoneRegion extends ZoneId

**非公共类，包含地区信息以及支持夏令时调整，只能通过ZoneId的静态方法返回。**

### final class **ZoneOffset** extends ZoneId implements TemporalAccessor, TemporalAdjuster, Comparable<ZoneOffset>, Serializable

与UTC的时区偏移，如+02:00，可以指定任意的偏移量（-18:00 ~ +18:00）。

- ```java
    public static ZoneOffset of(String offsetId) { ... }
    ```
    不同于ZoneId的of方法（参数为时区ID），参数要求是偏移量，Z（UTC）、+h、-hh、+hh:mm、+hh:mm:ss等。

- ```java
    public static ZoneOffset ofTotalSeconds(int totalSeconds) { ... }
    ```
    应该使用ofHours、ofHoursMinutes、ofHoursMinutesSeconds和ofTotalSeconds，其他方法都是使用ofTotalSeconds方法实现。

### OffsetDateTime & OffsetTime

可以指定偏移量的DateTime和Time，**应该用于日期存储或网络通信**。

```java
// OffsetDateTime
private final LocalDateTime dateTime;
private final ZoneOffset offset;

public static OffsetDateTime of(LocalDateTime dateTime, ZoneOffset offset) {
    return new OffsetDateTime(dateTime, offset);
}

public static OffsetDateTime of(LocalDate date, LocalTime time, ZoneOffset offset) {
    LocalDateTime dt = LocalDateTime.of(date, time);
    return new OffsetDateTime(dt, offset);
}
```

```java
// OffsetTime
private final LocalTime time;
private final ZoneOffset offset;

public static OffsetTime of(LocalTime time, ZoneOffset offset) {
    return new OffsetTime(time, offset);
}
```

### final class **ZonedDateTime** implements Temporal, ChronoZonedDateTime<LocalDate>, Serializable

**ZonedDateTime拥有时区信息并支持夏令时调整**，偏移量是由时区信息控制，适合用于用户显示日期。

```java
private final LocalDateTime dateTime; // 本地时间
private final ZoneOffset offset; // 由时区控制的偏移量
private final ZoneId zone; // 时区信息
```

- ```java
    // preferredOffset表示首选偏移量，可以为null。
    public static ZonedDateTime ofLocal(LocalDateTime localDateTime, ZoneId zone, ZoneOffset preferredOffset) { ... }

    // offset不能为null，会严格验证localDateTime、offset、zone的组合是否合法。
    public static ZonedDateTime ofStrict(LocalDateTime localDateTime, ZoneOffset offset, ZoneId zone) { ... }
    ```
    因为夏令时的特殊性，**一个具体时刻可能有两个偏移量**，所以还需要ZoneOffset去辅助表示。

- ```java
    // OffsetDateTime转ZonedDateTime
    public ZonedDateTime toZonedDateTime() {
        return ZonedDateTime.of(dateTime, offset); // offset作为zone
    }

    // ZonedDateTime
    public static ZonedDateTime of(LocalDateTime localDateTime, ZoneId zone) {
        return ofLocal(localDateTime, zone, null);
    }

    // ZonedDateTime转OffsetDateTime
    public OffsetDateTime toOffsetDateTime() {
        return OffsetDateTime.of(dateTime, offset);
    }
    ```

***

## local

### LocalTime

### LocalDate

### LocalDateTime

- 查询： 使用时间日期类对应的 getXXX() 方法以及 int get(TemporalField field)。

- 增减： 使用时间日期类对应的 plusXXX()方法和minusXXX() 以及参数为 TemporalAmount 类型的 minus 和 plus 方法。

- 调整：使用时间日期类对应的 withXXX() 方法、 with(TemporalAdjuster adjuster)方法和 with(TemporalField field, long newValue)方法。

- **注意：增减或调整方法均会返回一个新的对象，所以要减少调用次数。***