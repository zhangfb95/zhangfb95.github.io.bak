---
layout: post
title:  Jackson实现自定义序列化器
date:   2016-11-03 09:35:01 +0800
categories: json
tag: json
---

* content
{:toc}

jackson能有效地实现序列化操作，将一个java对象序列化成一串字符串。

## @JsonProperty

这个annotation用以修饰java字段，不管此属性是否有getter/setter方法，将此字段序列化和反序列化为指定的`value`，而`value`默认和属性名相同。
包含以下几个可能的属性：
1. value - 将要使用的名称
1. index - 序列化的属性，整数值
1. defaultValue - 默认值，若此值为null时，序列化为字符串时使用
1. access - 访问级别，包含读写、只读、只写

```java
@Data
class AmountReq {
    @JsonProperty("your_name")
    private String yourName;
}
```

## @JsonIgnore

用以忽略java字段

```java
@Data
class AmountReq {
    @JsonIgnore
    private String myName;
}
```

## @JsonFormat

这个属性用于修饰字段或setter方法，仅仅能对Date/Time类型进行反序列化。

```java
@Data
class AmountReq {
    @jsonFormat(shape=JsonFormat.Shape.STRING, pattern="yyyy-MM-dd HH:mm:ss")
    private String herName;
}
```

## @JsonSetter 和 @JsonGetter

可用以替代`JsonProperty`。
`JsonSetter`用以修饰`setter`方法，表示反序列化的字段名和java字段名的映射。
`JsonGetter`用以修饰`getter`方法，表示序列化的字段名和java字段名的映射。

## @JsonSerialize 和 @JsonDeserialize

`@JsonSerialize`用以修饰属性，表示进行序列化时所使用的class，而`@JsonDeserialize`却恰恰相反。
现以人民币的元和分的转换为例。我们java里面的类型为`BigDecimal`，以元为单位，而序列化之后的字符串却使用分为单位。他们之间刚好是100的倍数。

首先，定义一个转换器，用以控制费率。万一以后需要用到其他的参数呢。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface BigDecimal2IntConverter {

    /**
     * 费率，用以数值的转换
     */
    int rate() default 100;
}
```

其次，定义序列化器。这个序列化器可以将自定义annotation中的值传入进来，以便增加其使用灵活性。

```java
public class BigDecimal2IntSerializer extends StdSerializer<BigDecimal> implements ContextualSerializer {

    private int value = 100;

    public BigDecimal2IntSerializer() {
        super(BigDecimal.class);
    }

    public BigDecimal2IntSerializer(int key) {
        super(BigDecimal.class);
        this.value = key;
    }

    @Override
    public void serialize(BigDecimal value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        if (value == null) {
            gen.writeNull();
        } else {
            gen.writeNumber(value.multiply(new BigDecimal(this.value)).intValue());
        }
    }

    @Override
    public JsonSerializer<?> createContextual(SerializerProvider prov, BeanProperty property) throws JsonMappingException {
        int key = 100;
        BigDecimal2IntConverter ann = null;

        if (property != null) {
            ann = property.getAnnotation(BigDecimal2IntConverter.class);
        }

        if (ann != null) {
            key = ann.rate();
        }

        return new BigDecimal2IntSerializer(key);
    }
}
```

再其次，使用反序列化器对其进行处理。

```java
public class BigDecimal2IntDeserializer extends StdDeserializer<BigDecimal> implements ContextualDeserializer {

    private int value = 100;

    public BigDecimal2IntDeserializer() {
        super(BigDecimal.class);
    }

    public BigDecimal2IntDeserializer(int key) {
        super(BigDecimal.class);
        this.value = key;
    }

    @Override
    public BigDecimal deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JsonProcessingException {
        int v = p.getIntValue();
        return new BigDecimal(v).divide(new BigDecimal(value));
    }

    @Override
    public JsonDeserializer<?> createContextual(DeserializationContext ctxt, BeanProperty property) throws JsonMappingException {
        int key = 100;
        BigDecimal2IntConverter ann = null;

        if (property != null) {
            ann = property.getAnnotation(BigDecimal2IntConverter.class);
        }

        if (ann != null) {
            key = ann.rate();
        }

        return new BigDecimal2IntDeserializer(key);
    }
}
```

最后，考虑使用。使用起来就相当简单啦。

```java
@Data
public class AmountReq {

    @BigDecimal2IntConverter
    @JsonSerialize(using = BigDecimal2IntSerializer.class)
    @JsonDeserialize(using = BigDecimal2IntDeserializer.class)
    private BigDecimal amount;
}

public class JacksonTest {

    @Test
    public void test() throws Exception {
        ObjectMapper objectMapper = new ObjectMapper();
        AmountReq req = new AmountReq();
        req.setAmount(new BigDecimal(20.3));
        String s = objectMapper.writeValueAsString(req);
        System.out.println(s);

        AmountReq req2 = objectMapper.readValue("{\"amount\": 29000}", AmountReq.class);
        System.out.println(req2);
    }
}
```