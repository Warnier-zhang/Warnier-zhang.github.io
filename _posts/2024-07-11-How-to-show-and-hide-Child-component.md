---
layout: post
title: Vue 3：如何显示、隐藏子组件
---

当一个父页面需要打开多个子页面的时候，自然而然地能够想到Vue Router，创建并且切换到一个新页面。

除此之外，也可以基于Vuetify全屏模式的`VDialog`来模拟一个新页面，达到上述目的。后者，以子组件的形式存在，能够更好地保持与父组件之间的联动。

这就产生了一个问题：父组件如何显示、隐藏子组件？有两种方式，一种是通过子组件属性和事件，另一种是通过子组件方法。

## 方式一

```
父组件
---
<template>
    <v-btn @click="showChild1">
    	显示子组件1
    </v-btn>
    
    <v-btn @click="hideChild1">
    	隐藏子组件1
    </v-btn>
    
    <Child1
        :active="active"
        :mode="mode"
        @hide="hideChild1">
    </Child1>
</template>

<script setup>
...

import Child1 from "@/views/Child1.vue";

...

const active = ref(false);
const mode = ref(null);

function showChild1() {
    active.value = true;
    mode.value = '子组件属性';
}

function hideChild1() {
    active.value = false;
    mode.value = null;
}
</script>

子组件
---
<template>
    <v-card
        v-show="active"
        title="子组件1"
        :text="`控制方式：${mode}`">
        <v-card-actions>
            <v-btn @click="$emit('hide', false)">
                隐藏
            </v-btn>
        </v-card-actions>
    </v-card>
</template>

<script setup>
const props = defineProps({
    active: {
        type: Boolean,
        default: false
    },
    mode: {
        type: String,
        default: null
    },
});
</script>
```

父组件通过设置属性`active`的值为`true/false`来控制子组件的显示或者隐藏，其他的参数，例如：控制方式，同样通过额外的属性`mode`传递给子组件。

当子组件需要隐藏的时候，触发`hide`事件，父组件监听该事件，并更新属性`active`的值为`false`，`mode`的值为`null`。

## 方式二

```
父组件
---
<template>
   <v-btn @click="showChild2('子组件方法')">
       显示子组件2
   </v-btn>
   
   <v-btn @click="hideChild2">
       隐藏子组件2
   </v-btn>
   
   <Child2 ref="child2"></Child2>
</template>

<script setup>
...

import Child2 from "@/views/Child2.vue";

...

const child2 = ref(null);

function showChild2(mode) {
    child2.value.show({
        mode
    });
}

function hideChild2() {
    child2.value.hide();
}
</script>

子组件
---
<template>
    <v-card
        v-show="active"
        title="子组件2"
        :text="`控制方式：${mode}`">
        <v-card-actions>
            <v-btn @click="hide">
                隐藏
            </v-btn>
        </v-card-actions>
    </v-card>
</template>

<script setup>
...

defineExpose({
    show,
    hide,
});

const active = ref(false);
const mode = ref(null);

function show(options) {
    active.value = true;
    if(options){
    	mode.value = options.mode;
    }
}

function hide() {
    active.value = false;
    mode.value = null;
}
</script>
```

父组件通过调用`show()/hide()`方法来控制子组件的显示或者隐藏，其他的参数，例如：控制方式，通过`show()`方法的参数`mode`传递给子组件。

当子组件需要隐藏的时候，父组件不用做任何处理。

和方式一相比，方式二的子组件具有**高内聚**，**低耦合**的特点。

