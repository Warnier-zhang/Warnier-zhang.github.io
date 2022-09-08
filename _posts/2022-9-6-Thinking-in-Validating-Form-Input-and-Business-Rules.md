---
layout: post
title: 关于表单输入和业务规则验证的一点思考
---

后端代码如何优雅地验证表单输入和校验业务规则？

验证表单输入的做法最成熟，既有Bean Validation 1.0（JSR-303）规范，及其实现Hibernate Validator，又有靠Spring背书的Spring Validation框架。

至于如何校验业务规则，好像还没有业界标准可供参考。

什么是业务规则？就是执行某个操作的前提条件。举个例子：用户提交一个购买蓝色、L码、全棉衬衫的订单，后台代码首先需要判断是否有库存、是否能使用优惠券、是否能派送等等，然后才能创建订单。

与此段逻辑相关的代码，要么是一些惹人厌的**嵌套的If-Else**，要么就抛出各种类型的**Exception**。

有没有更好的做法呢？Vuetify 2和Spring Validation可以提供一点灵感。

Vuetify 2的特点：

- 表单输入控件都有一个`rules`属性，用来配置一组自定义的验证规则；
- 规则，是一种特殊的**函数**，**参数是用户输入的值**，**返回值是**`true / false`，或者**一个包含错误信息的字符串**；
- 当用户输入完成时，**循环**应用**一组规则**，如果验证不通过，就立马显示错误信息；
- `<v-form>`控件能够**显式调用`validate()`**方法来检查用户输入是否合法；

Spring Validation的特点：

- **以领域模型（Domain Model）为中心**；
- 基于注解；
- **支持`@Validated`自动触发验证，也支持调用`Validator.validate()`手动触发验证**；
- 若验证不通过，则返回**一个`ConstraintViolation（包含错误信息和无效的值）`集合**，否则返回**一个空集合**；

结合Vuetify 2和Spring Validation的特点，尝试提出如下的一套解决方案：

- `Rule`接口：

  ```
  public interface Rule<T> extends Function<T, String> {
  }
  ```
  `Rule`，顾名思义，即验证规则，是`java.util.function.Function`的子接口，接收一些必要的输入条件，如果验证通过，就返回`null`，否则返回**一个包含错误信息的字符串**；
  
  > 最佳实践：
  > 1. <u>常用的</u>、<u>通用的</u>、<u>共享的</u>规则最好提供Rule接口的实现类；
  > 2. <u>特殊的</u>、<u>独享的</u>、<u>与功能模块紧耦合的</u>规则可以直接在Service层定义一个Lambda表达式；

- `RuleValidator`类

  ```
  public class RuleValidator {
      public static <T> List<String> validate(T t, Rule<T>... rules) {
          return Stream.of(rules)
                  .map(rule -> rule.apply(t))
                  .filter(Objects::nonNull)
                  .collect(Collectors.toList());
      }
  }
  ```

  `RuleValidator`是一个工具类，角色类似Spring Validation中的`javax.validation.Validator`接口，用来对输入条件应用一组验证规则。只有一个静态方法`validate()`，接收一些必要的输入条件（同`Rule接口apply()方法`传参）和**一组`Rule`实例或`Lambda`表达式**，若验证不通过，则返回一个包含错误信息的字符串集合，否则，返回一个空集合；

下面用一个最简单、最直白的例子来演示`Rule`和`RuleValidator`的用法。

## 案例

类ABC有a、b、c等3个属性，需要判断对象abc1、abc2是否符合规则`a="1", b="2", c="3"`？

```
@Data
@AllArgsConstructor
public class ABC {
    private String a;
    private String b;
    private String c;
}
```
假设规则`a="1"`被多个地方共用，如上所述，此时需要单独提供一个Rule实现类：
```
public class R1 implements Rule<ABC> {
    @Override
    public String apply(ABC abc) {
        if ("1".equals(abc.getA())) {
            return null;
        } else {
            return "a的值是" + abc.getA() + "，不是1！";
        }
    }
}
```
假设规则`b="2"`、`c="3"`只在`TestService`中用到，就只需要在`TestService`内定义即可：
```
public class TestService {
    public void valicateABC() {
        ABC abc1 = new ABC("1", "2", "3");
        ABC abc2 = new ABC("11", "22", "33");
        List<String> validationResults = RuleValidator.validate(abc2, new R1(), getR2(), getR3());
        System.out.println(validationResults);
    }

    private Rule<ABC> getR2() {
        return (abc) -> {
            if ("2".equals(abc.getB())) {
                return null;
            } else {
                return "b的值是" + abc.getB() + "，不是2！";
            }
        };
    }

    private Rule<ABC> getR3() {
        return (abc) -> {
            if ("3".equals(abc.getC())) {
                return null;
            } else {
                return "c的值是" + abc.getC() + "，不是3！";
            }
        };
    }
}
```

最后调用`RuleValidator.validate()`方法，并传入参数`abc2`、`R1`、`R2`、`R3`。
