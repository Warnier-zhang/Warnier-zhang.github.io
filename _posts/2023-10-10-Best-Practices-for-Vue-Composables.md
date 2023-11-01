---
layout: post
title: Vue 3组合式函数（Composable）最佳实践
---

组合式函数（Composable）**是一个**封装、复用**有状态的代码逻辑**的**函数**。类似Vue 2中的混入（Mixin）。组合式函数只能在 `<script setup>`中被调用。

既然组合式函数也是一个函数，那么它的**签名——函数名、参数列表、返回值**和普通函数有什么区别呢？

```
export function useA(p, options) {
    const {
        immediate = true,
    } = options || {};

    const a = ref(null);

    async function getA() {
        a.value = await new Promise((resolve, reject) => {
            setTimeout(() => {
                resolve(
                    Array.from({length: 10}, (v, i) => i).map((i) => {
                        return `${toValue(p)}-a-${i + 1}`;
                    })
                );
            }, 2000);
        });
    }

    if (immediate) {
        getA();
    }

    return {
        a,
        getA
    };
}
```

**函数名**

函数名约定以“use”作为开头。

**参数列表**

参数列表分成必选参数（例如：`p`）和可选参数（例如：`options`）2个部分，其中：

- 必选参数：最好是**ref引用类型**，当参数值变化时，可以捕获到新旧值；可搭配`toValue()`使用。
- 可选参数：推荐用**Object类型**，和长的参数列表相比，能忽略参数顺序，方便添加新参数；

**返回值**

上述定义提到组合式函数封装了**状态**。状态就是一个**ref引用类型**的变量（例如：`a`），之所以把状态定义成ref，是因为可以用于template中。

通常，约定**返回值**是一个包含多个状态的（ref）**普通对象**。也可以通过可选参数（例如：`options`）动态决定是返回一个值，还是返回一个普通对象。

## 异步编程

组合式函数是处理、复用API调用的最优解。借助组合式函数，可忽略async、await等API调用过程，只关心API返回结果。常见的API调用场景如下：

1. 单个API调用，例如：API `useA()`返回`a`，侦听器监控`a`，若`a`的值非空，则格式化`a`，最终，页面显示`a`；
2. 多个API组合，例如：API `useA()`返回`a`，侦听器监控`a`，若`a`的值非空，则把`a`作为输入参数调用API `useB()`，API `useB()`返回`b`，侦听器监控`b`，若`b`的值非空，则格式化`b`，最终，页面显示`b`；

第1个场景非常简单，不用多说。

第2个场景相对复杂。充斥着大量的侦听器，一方面用来处理API返回结果，另一方面起到串联后继API的作用。侦听器不但嵌套，而且可能会冲突、失效，尤其是<u>把API `useA()`和API `useB()`作为某个按钮单击事件的回调函数</u>时。侦听器嵌套尚且可以通过由后继API支持接收异步参数来解决。侦听器冲突、失效就棘手了！！！

当然，也不是完全无解，一种解决方案是：让**侦听器**只负责**处理API返回结果**，由**触发器**来负责**<u>显式</u>调用后继API**。具体的做法就是，组合式函数返回值**除了**包含**状态**，**还要**包含**触发器——该状态对应的更新函数**（例如：`getA()`），在监控到API `useA()`返回值`a`变化时，显式调用API `useB()`的触发器`getB()`来获取`b`！

完整的示例如下：

```
export function useA(p, options) {
    const {
        immediate = true,
    } = options || {};

    const a = ref(null);

    async function getA() {
        a.value = await new Promise((resolve, reject) => {
            setTimeout(() => {
                resolve(
                    Array.from({length: 10}, (v, i) => i).map((i) => {
                        return `${toValue(p)}-a-${i + 1}`;
                    })
                );
            }, 2000);
        });
    }

    if (immediate) {
        getA();
    }

    return {
        a,
        getA
    };
}
```

```
export function useB(a, options) {
    const {
        immediate = true,
    } = options || {};

    const b = ref(null);

    async function getB() {
        b.value = await new Promise((resolve, reject) => {
            setTimeout(() => {
                resolve(
                    toValue(a).map(((a2) => {
                        return Array.from({length: 10}, (v, i) => i).map((i) => {
                            return `${a2}-b-${i + 1}`;
                        })
                    }))
                )
            }, 2000);
        });
    }

    if (immediate) {
        getB();
    }

    return {
        b,
        getB
    };
}
```

```
<template>
    <input type="text" v-model="p"/>
    <input type="button" value="Test" @click="onBtnClick"/>
    <div>a: {{ a }}</div>
    <div>b: {{ b }}</div>
</template>

<script setup>
import {ref, watch, toValue} from 'vue'
import {useA} from '@/apis/a';
import {useB} from '@/apis/b';

const p = ref(null);

const {a, getA} = useA(p, {immediate: false});
const {b, getB} = useB(a, {immediate: false});
watch(
    a,
    (value) => {
        if (value) {
            getB();
        }
    },
    {
        immediate: true,
        deep: true
    }
);

function onBtnClick() {
    getA();
}
</script>
```
