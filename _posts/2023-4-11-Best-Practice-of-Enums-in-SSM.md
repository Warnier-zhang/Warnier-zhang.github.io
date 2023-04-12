---
layout: post
title: SSM中的枚举最佳实践
---

枚举是Java 5开始引入的一种新特性，用来定义常量。也可以在枚举类内可以定义属性、方法，通常会定义2个属性及其`getter`，一个表示文本（或名称），另一个表示值（或代码），如下面的`Sex`类。一般情况下，常量名是英文全称，方便编码和理解；文本是中文描述，方便阅读和展示；值是英文字母或缩写，方便存储；

```
@Getter
public enum Sex implements BaseEnum {
    BOY("B", "男"),
    GIRL("G", "女");

    private String value;

    private String text;

    Sex(String value, String text) {
        this.value = value;
        this.text = text;
    }
}
```
枚举不支持继承，但可以实现接口。因此，抽象出一个接口`BaseEnum`，用来统一描述那些包含`text`和`value`2个属性的枚举类。
```
public interface BaseEnum {
    String getText();

    String getValue();
}
```

众所周知，SSM，即Spring MVC（也有说Struts 2）、Spring、MyBatis，是一套实现分层（MVC）架构的主流框架。在实践中，分层架构包含控制层（Controller）、服务层（Service）、数据库访问层（Dao）、模型层（Model）、视图层（View）等等。在SSM中使用枚举时，针对实现`BaseEnum`的枚举类，我们希望达到如下效果：

> 1. 在保存时，把枚举常量的`value`属性保存到数据库中；在查询时，可以根据查到的`value`值解析出对应的枚举常量。
> 2. 在后台把枚举常量返回给前端时，前端得到的是该常量的`text`属性；在前端向后台提交`value`参数时，后台解析出对应的枚举常量。

## MyBatis

MyBatis默认提供了`EnumTypeHandler`和`EnumOrdinalTypeHandler`等2个`TypeHandler`处理枚举类型，分别根据枚举常量的名称（`Enum.name()`）和索引（`Enum.ordinal()`）来序列化、实例化枚举类型对象。

显然两者都无法直接满足我们的要求。针对`EnumTypeHandler`，我们还可以通过把<u>枚举常量的名称</u>和<u>`value`属性的值</u>改成相同的来达到类似的效果。

最好是提供一个特殊的`TypeHandler`，专门用来处理实现`BaseEnum`的枚举类，如下：

```
public class BaseEnumTypeHandler<T extends Enum<T> & BaseEnum> extends BaseTypeHandler<T> {
    private final Class<T> type;

    public BaseEnumTypeHandler(Class<T> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
        if (jdbcType == null) {
            ps.setString(i, parameter.getValue());
        } else {
            ps.setObject(i, parameter.getValue(), jdbcType.TYPE_CODE);
        }
    }

    @Override
    public T getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String s = rs.getString(columnName);
        return s == null ? null : BaseEnum.parseEnum(this.type, s);
    }

    @Override
    public T getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String s = rs.getString(columnIndex);
        return s == null ? null : BaseEnum.parseEnum(this.type, s);
    }

    @Override
    public T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String s = cs.getString(columnIndex);
        return s == null ? null : BaseEnum.parseEnum(this.type, s);
    }
}
```

静态方法`parseEnum()`类似`Enum.valueOf()`。

此外，MyBatis需要配置`default-enum-type-handler`参数：

```
mybatis:
  configuration:
    default-enum-type-handler: xxx.yyy.zzz.BaseEnumTypeHandler
```

## Spring MVC

在Spring MVC中，枚举常量的序列化和实例化是由[**Jackson**](https://github.com/FasterXML/jackson)实现的，序列化、实例化类分别是`EnumSerializer`和`EnumDeserializer`，两者默认也是根据`Enum.name()`、`Enum.toString()`（默认的返回值同`Enum.name()`）、`Enum.ordinal()`等方法的返回值来转换字符串和枚举常量。

针对实现`BaseEnum`的枚举类，如果想要Jackson把`text`或者`value`属性的值作为序列化、实例化的输出、输入，就要用`@JsonValue`注解来标注某个属性。例如：如果用`@JsonValue`注解标注`value`属性，`Sex`类中的常量`BOY`在序列化之后输出字符串`B`，反之，把字符串`B`作为实例化的输入才能解析出常量`BOY`。

这和我们的需求有点不一样，我们希望是：

> 1. 常量`BOY`在序列化之后输出`text`的值`男`；
> 2. 把`value`的值`B`作为实例化的输入后得到常量`BOY`；

理想很丰满，现实很骨感。Jackson默认是不支持上述转换的，但是，可以通过扩展`JsonSerializer`和`JsonDeserializer`来达到目的。

```
public class BaseEnumSerializer extends JsonSerializer<BaseEnum> {
    @Override
    public void serialize(BaseEnum value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeString(value.getText());
    }
}
```

```
public class BaseEnumDeserializer extends JsonDeserializer<BaseEnum> implements ContextualDeserializer {
    private final JavaType valueType;

    public BaseEnumDeserializer() {
        this(null);
    }

    public BaseEnumDeserializer(JavaType valueType) {
        this.valueType = valueType;
    }

    @Override
    public JsonDeserializer<?> createContextual(DeserializationContext ctxt, BeanProperty property) throws JsonMappingException {
        JavaType valueType = ctxt.getContextualType() != null ? ctxt.getContextualType() : property.getType();
        return new BaseEnumDeserializer(valueType);
    }


    @Override
    public BaseEnum deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JsonProcessingException {
        String value = p.getValueAsString();
        return Arrays.stream(valueType.getRawClass().getEnumConstants())
                .map(t -> (BaseEnum) t)
                .filter(t -> Objects.equals(t.getValue(), value))
                .findAny()
                .orElse(null);
    }
}
```

有了`BaseEnumSerializer`和`BaseEnumDeserializer`之后，还需要按照如下方式配置才行：

```
@JsonSerialize(using = BaseEnumSerializer.class)
@JsonDeserialize(using = BaseEnumDeserializer.class)
public interface BaseEnum {
    ...
}
```

