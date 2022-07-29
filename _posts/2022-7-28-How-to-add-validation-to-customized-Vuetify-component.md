---
layout: post
title: Vuetify自定义组件实现表单验证
---

虽然Vuetify有一个大而全的组件库，但是为了能够更好地贴合应用场景，还是要开发者自己动手定制一些组件，比如：把几个现有的Vuetify组件组合成一个新组件等等。

定制新组件，尤其是新的表单组件，实现表单验证是重中之重。Vuetify内置了一套自己独有的表单验证体系，如实时输入提示、提交之前调用`this.$refs.form.validate()`判断等等。如何让新组件融入到这个体系中是一个难点。

`<v-input>`是所有表单组件的基础，官方推荐通过扩展它来定制新的表单组件。

## Vuetify表单验证源码分析

从`<v-input>`的源码来看，**用于显示表单验证结果的`<v-messages>`组件**的值来源于计算属性`messagesToDisplay`。

```
...
genMessages () {
  if (!this.showDetails) return null

  return this.$createElement(VMessages, {
    props: {
      color: this.hasHint ? '' : this.validationState,
      dark: this.dark,
      light: this.light,
      value: this.messagesToDisplay,
    },
    attrs: {
      role: this.hasMessages ? 'alert' : null,
    },
    scopedSlots: {
      default: props => getSlot(this, 'message', props),
    },
  })
},
...
```

计算属性`messagesToDisplay`的值是由计算属性`validations`决定的，`validations`在`<v-input>`的源码中找不到，而是定义在混入（Mixin） `validatable`中，混入`validatable`被所有的组件共享，承担了表单验证的绝大部分工作。

```
computed: {
	...
    messagesToDisplay (): string[] {
      if (this.hasHint) return [this.hint]
    
      if (!this.hasMessages) return []
    
      return this.validations.map((validation: string | InputValidationRule) => {
        if (typeof validation === 'string') return validation
    
        const validationResult = validation(this.internalValue)
    
        return typeof validationResult === 'string' ? validationResult : ''
      }).filter(message => message !== '')
    },
    ...
},
```

`validations`和`validationTarget`息息相关，

```
computed: {
    ...
    validations (): InputValidationRules {
      return this.validationTarget.slice(0, Number(this.errorCount))
    },
    ...
    validationTarget (): InputValidationRules {
      if (this.internalErrorMessages.length > 0) {
        return this.internalErrorMessages
      } else if (this.successMessages && this.successMessages.length > 0) {
        return this.internalSuccessMessages
      } else if (this.messages && this.messages.length > 0) {
        return this.internalMessages
      } else if (this.shouldValidate) {
        return this.errorBucket
      } else return []
    },
    ...
},
```

计算属性`validationTarget`的影响因素有`internalErrorMessages`和`errorBucket`。

`internalErrorMessages`最终关联到`errorMessages`，`errorMessages`是允许用户自行设置的属性，也是<u>供集成第三方表单验证用的（本篇文章不涉及）</u>，可以忽略。

`errorBucket`是`data`对象中的一个属性，给它**赋值**的地方可以追溯到`validate` 方法，**`validate`方法就是表单验证这个动作实际发生的地方！**`validate` 方法根据用户设置的校验规则逐条验证。

```
methods: {
    ...
    validate (force = false, value?: any): boolean {
      const errorBucket = []
      value = value || this.internalValue

      if (force) this.hasInput = this.hasFocused = true

      for (let index = 0; index < this.rules.length; index++) {
        const rule = this.rules[index]
        const valid = typeof rule === 'function' ? rule(value) : rule

        if (valid === false || typeof valid === 'string') {
          errorBucket.push(valid || '')
        } else if (typeof valid !== 'boolean') {
          consoleError(`Rules should return a string or boolean, received '${typeof valid}' instead`, this)
        }
      }

      this.errorBucket = errorBucket
      this.valid = errorBucket.length === 0

      return this.valid
    },
    ...
},
```

那么，在什么条件下会触发运行`validate` 方法呢？

`value = value || this.internalValue`这里透露了一点线索。`internalValue`即`lazyValue`，**`lazyValue`是`data`对象中的一个属性，用来存放用户输入，也就是`v-model`指令绑定的内容**。**`internalValue`受到监控，当它的值发生变化时，就会触发运行`validate`方法（如果设置了`validateOnBlur`属性，就会等到`focus`变化之后再调用）。**

```
watch: {
    ...
    rules: {
      handler (newVal, oldVal) {
        if (deepEqual(newVal, oldVal)) return
        this.validate()
      },
      deep: true,
    },
    internalValue () {
      // If it's the first time we're setting input,
      // mark it with hasInput
      this.hasInput = true
      this.validateOnBlur || this.$nextTick(this.validate)
    },
    isFocused (val) {
      // Should not check validation
      // if disabled
      if (
        !val &&
        !this.isDisabled
      ) {
        this.hasFocused = true
        this.validateOnBlur && this.$nextTick(this.validate)
      }
    },
    ...
},
```

这下，因果关系、逻辑链就理清楚了。因此，如果要让新组件支持表单验证，就得实现上述提到的要点。

## 定制新表单组件要点

总结一下：

1. 扩展`<v-input>`组件;

2. 把用于显示表单验证结果的组件或标签的值设置为`messagesToDisplay`；

3. 当新表单组件的值发生变化时，需要更新计算属性`internalValue`的值；

   > 注意：计算属性`internalValue`的`set()`方法中调用了`$emit()`抛出`input`事件，如果用户自行处理`v-model`绑定等，就需要覆写该方法。
