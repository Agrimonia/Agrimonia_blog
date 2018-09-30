+++
title = "Vue 封装表单类组件时的数据绑定方式"
date = 2018-09-09T10:13:55+08:00
tags = ["Vue"]
categories = ["前端"]
draft = false
+++

# Vue 封装表单类组件时的数据绑定方式

对于表单类组件，数据双向绑定可以大大简化数据流的复杂度，所以用 vue 的时候我倾向于尽可能使用 v-model 来减少工作量。

v-model 是一个语法糖，它的本质是用 v:bind 传入 props 加上用 v:on 监听子组件中的事件，所以当我们写一个 my-input.vue 组件，在其中给 `<input/>` 加上双向数据绑定`<input v-model="value">` 的时候，实际上是 `<input :value="value" @input="value = $event.target.value"`。

这些在官方文档的[组件基础](https://cn.vuejs.org/v2/guide/components.html#%E5%9C%A8%E7%BB%84%E4%BB%B6%E4%B8%8A%E4%BD%BF%E7%94%A8-v-model)里说的很清楚了。

如果想要对 input 进行封装，比如 `<my-input />`，并在自定义组件上使用 v-model，官方文档的[自定义事件](https://cn.vuejs.org/v2/guide/components-custom-events.html#%E8%87%AA%E5%AE%9A%E4%B9%89%E7%BB%84%E4%BB%B6%E7%9A%84-v-model)同样有详细的描述，你需要用到 vm.$emit 触发一个自定义事件，把 value 从子组件传递到父组件。 如果是别的事件类型，比如 select 等，可以自定义 model 选项的 props 和 event。

如果去阅读各种 Vue 组件库的 form 类组件的源码，基本上都是这么做的。

---

但现在我需要创建一个 custom-input.vue 组件来进一步封装上文提到的 my-input.vue，实际场景中它可能是 el-input、md-select 等第三方组件。

这时如法炮制行不通了，在 custom-input 中跨过 my-input 对 input 事件的监听将静默失效。

尤大大在 [issue#6846](https://github.com/vuejs/vue/issues/6846#issuecomment-339665335) 中解释了：

> It only **happened to work** previously because the native event was emitted first and then overwritten by the custom event.

官方文档的[将原生事件绑定到组件](https://cn.vuejs.org/v2/guide/components-custom-events.html#%E5%B0%86%E5%8E%9F%E7%94%9F%E4%BA%8B%E4%BB%B6%E7%BB%91%E5%AE%9A%E5%88%B0%E7%BB%84%E4%BB%B6)中使用 vm.$listeners 属性来解决这个问题。

我们将 my-input 透明化，官方文档讲其形容为 **a fully transparent wrapper**，此时 my-input 可以当成一个 input 组件来使用。

为了写的更简洁，我使用 v-model 的写法

<iframe src="https://codesandbox.io/embed/9lln79qmxo?module=%2Fsrc%2Fcomponents%2Fcustom-input.vue" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

