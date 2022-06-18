## [首页](/blog/)

***

### ObjectMapper配置
- SerializationFeature.WRITE_DATE_TIMESTAMPS_AS_NANOSECONDS：序列化时间戳带纳秒
- DeserializationFeature.READ_DATE_TIMESTAMPS_AS_NANOSECONDS：反序列化时间戳带纳秒

### Annotations

- @JsonAlias：属性注解，指定反序列化时属性的别名。

- @JsonUnwrapped：扁平化对象，将对象拆分为属性。

- @JsonAnyGetter：序列化时将map类型属性拆包作为普通属性。

- @JsonAnySetter：反序列化时，使用一个map接受未知属性。

    ```java
    @Data
    public static class TestClass {
        @JsonAnyGetter
        Map<String, Object> map;

        @JsonAnySetter
        void wrap(String k, Object o) {
            if (this.map == null) {
                this.map = new HashMap<>();
            }
            this.map.put(k, o);
        }
    }
    ```

- @JsonAutoDetect：设置属性识别方式，一般通过配置ObjectMapper全局设置。

- @JsonManagedReference、@JsonBackReference：用于解决父子循环依赖，@JsonBackReference标识的属性不会被序列化，但是反序列化时会自动创建循环依赖。

- @JsonIdentityInfo：也可用于解决循环依赖，序列化时自动根据循环关系生成唯一标识，或者指定为已存在的属性，反序列化也会自动创建循环依赖。

- @JsonCreator：指定反序列化调用方法，无改注解则默认为无参构造方法，可以指定为其他构造方法或静态工厂方法，方法参数使用@JsonProperty标识，否则作为整体传入。

- @JsonValue：指定序列化的值，可以注解在方法和字段上。

- @JsonRawValue：序列化时原样输出，适用于属性已经是一个JSON字符串的情况，不会加上转义符和多余的引号。

- @JsonTypeInfo：用于多态场景
    - use：类型信息，JsonTypeInfo.Id枚举值
        - NONE：无
        - CLASS、MINIMAL_CLASS、NAME：全限定类名，最小类型
        - Name：默认为类名，可@JsonSubTypes.Type或@JsonTypeName自定义
        - DEDUCTION：不输出类型信息，反序列化时按照属性匹配
    - include：类型信息输出方式，JsonTypeInfo.As枚举值
    - property：输出类型信息为属性时，指定属性名
    - @JsonSubTypes：标注在父类上，声明所有子类类型，值为@JsonSubTypes.Type数组
    - @JsonTypeName：标注在子类上，指定类型名称

    ```java
    @JsonTypeInfo(use = JsonTypeInfo.Id.NAME)
    @JsonSubTypes({@JsonSubTypes.Type(value = A.class, name = "a"), @JsonSubTypes.Type(B.class)})
    public static abstract class Base {
    }

    public static class A extends Base {
        String a;
    }

    @JsonTypeName("b")
    public static class B extends Base {
        Integer b;
    }
    ```

- @JsonProperty
    - value：指定序列化和反序列化的属性名
    - index：指定序列化的顺序
    - access：字段访问场景，READ_ONLY、WRITE_ONLY、READ_WRITE

- @JsonPropertyOrder：类注解，指定属性排序

- @JsonSerialize
    - as：指定序列化的类型
    - typing：动态根据具体类型序列化，或静态按照声明类型序列化。

- @JsonDeserialize
    - as：指定反序列化的类型
    - builder：使用builder反序列化
    
- @JsonPOJOBuilder：注解在builder上配合@JsonDeserialize.builder

### JsonNode
- findPath(String fieldName)：从整个树中查找指定属性，不存在则返回MissingNode，内部实现是调用findValue
- findValue(String fieldName)：从树中查找指定属性，不存在则返回null
- findParent(String fieldName)：从树中查找包含指定属性的JsonNode，不存在返回null
- require()：要求不能为MissingNode，返回自身。
- with(String propertyName)：如果不存在该属性则添加ObjectNode。
- withArray(String propertyName)：如果不存在该属性则添加ArrayNode。

### ObjectNode
- fields()：如果为ObjectNode，返回所有属性Entry
- fieldNames()：如果为ObjectNode，返回所有属性名
- elements()：如果为ObjectNode，返回所有属性的值
- path(String fieldName)：如果为ObjectNode，则查找属性，不存在返回MissingNode
- get(String fieldName)：如果为ObjectNode，则查找属性，不存在返回null
- required(String fieldName)：如果为ObjectNode，要求必须存在指定属性并返回

### ArrayNode
- elements()：如果为ArrayNode，返回所有元素
- path(int index)：如果为ArrayNode，返回索引为index的元素，不存在返回MissingNode
- get(int index)：如果为ArrayNode，返回索引为index的元素，不存在返回null
- required(int index)：如果为ArrayNode，要求必须存在索引为index的元素并返回