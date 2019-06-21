---
title: Vue基于snabbdom做了哪些事
date: 2019-06-20 10:47:13
thumbnail: /images/2019/vue_snabbdom_poster.png
tags:
  - Vue
categories:
  - Vue
---

## 前言

之前有简单看过 Vue patch 部分的源码，了解了是基于 [Snabbdom](https://github.com/snabbdom/snabbdom) 库实现的。最近想详细了解下 Vue 处理 vnode patch 的整个过程，想知道它在 Snabbdom 之上做了哪些事情？所以带着这个问题，写了这篇文章来记录。

## Snabbdom 做了哪些事？

> A virtual DOM library with focus on simplicity, modularity, powerful features and performance. 
(一个虚拟 DOM 库，专注于简单性，模块化，强大的功能和性能。)

Snabbdom 核心代码大约只有 200 行。它提供了模块化架构，具有丰富的功能，可通过自定义模块进行扩展。在了解核心 patch 前，需要先了解 snabbdom 的模块化架构思想。

#### modules

在节点的生命周期里做一些任务，来扩展 Snabbdom ，就可以称之为模块。
Snabbdom 在 patch 的过程中会注入很多钩子(hooks)。模块实现就是基于这些钩子，钩子可以理解为 vnode 节点的生命周期。

比如 eventlisteners 模块：
1. create： 节点创建时添加时间监听(addEventListener)
2. update： 节点更新时移除老节点事件，添加新事件
3. destroy： 节点销毁时，移除老节点事件

#### Hooks

`Hooks` 是一种挂载 vnode 生命周期的方式。软件开发领域有类似这样的设计思想，比如版本管理工具 `git`。Snabbdom 中的模块就是基于此来实现扩展的。当然，也可以传递 hook 配置，实现在 vnode 生命周期里做一些事（这种只针对单个节点，而模块针对所有节点），例如：

```js
h('div.row', {
  key: movie.rank,
  hook: {
    insert: vnode => {
      movie.elmHeight = vnode.elm.offsetHeight
    }
  }
})
```

Vue 中的指令就是基于此实现的。具体的生命周期可以参考官方文档：[https://github.com/snabbdom/snabbdom#hooks](https://github.com/snabbdom/snabbdom#hooks)

#### patch(oldVnode, newVnode) 函数

Snabbdom 的核心部分，patch 函数由 `snabbdom.init` 创建，初始化时提供绑定 hook 的模块。
如果 `oldVnode` 是具有父节点的 DOM 元素，则 `newVnode` 将变为 DOM 节点，并且传递的元素将被创建的 DOM 节点替换。如果传递的是个 `vnode` 实例，则会比对此节点和递归对比它的子节点，并做 dom 更新。这一块网上的源码解析文章也比较多，这里就不多介绍了。

#### snabbdom/h

`h` 函数用来创建 vnode 实例，类比 Vue render 函数中的参数 `h`，Vue 中扩展了入参。比如第一个参数 `tag`，Vue 可以是对象或函数；第二个参数 `data`，Vue 增加了一些特有的功能比如：`scopedSlots`, `slot`, `directive`, `ref`... 等。

## Vue 相比 Snabbdom，patch 过程中有哪些不一样的地方？

#### 组件

- Vue 中有组件的概念，组件同样是可以嵌套的，所有就存在父组件和子组件。父组件 render 函数执行后，子组件并没有被初始化，仅仅是创建了一个特殊的 vnode 节点，而这个节点上会绑定一些 hooks: `init`, `prepatch`, `insert`, `destroy`。当父组件 patch 执行的过程中，遇到子组件的话，会调用 hook: init 方法，此方法中才会初始化组件实例。后续 insert hook 触发时，相应的会调用组件 `mounted` 生命周期钩子。这一块源码可以查看 [https://github.com/vuejs/vue/blob/dev/src/core/vdom/create-component.js#L36-L97](https://github.com/vuejs/vue/blob/dev/src/core/vdom/create-component.js#L36-L97)

- `setScope` 也是在 patch 执行过程中调用的，它会往 dom 节点上增加 `${scopedId}` 属性，用来实现 `scoped` 样式功能。

- patch 过程中会更新 `ref` 到上下文实例上。

- 因为有组件概念，所以 Vue 中创建 vnode 第一个参数 tag 可以是对象或函数，Snabbdom 上只能是字符串。

#### 生命周期

- Vue 去掉了 `pre` 和 `post` 2 个钩子，这 2 个钩子分别用来在 patch 执行前后调用的
- Vue 中 `init` 钩子只对组件节点生效，而 Snabbdom 中所有节点都可定义 `init` hooks
- Vue 新增了 `active` 钩子，用来处理 `<keep-alive>` 嵌套 `<transition>` 组件产生的[边界问题](https://github.com/vuejs/vue/issues/4339)

#### 静态节点优化

Vue 会对静态节点做性能优化。

- 编译时：Vue compiler 会对 template 中的静态节点特殊处理：创建静态节点的方法会存放在 `staticRenderFns` 属性里。除了纯 html 语法产生的静态节点外，`v-once`, `v-pre` 也会产生静态节点。
- 运行时：当组件初始化时，静态节点会被保存到实例 `_staticTrees` 数组中。组件更新重新触发 render 时，不会重新创建 vnode 节点，直接使用之前已有的静态节点。进而不会触发 `patchVnode` 操作。

#### hydrate

对于 Vue 服务端渲染输出的 html，客户端初始化挂载节点时，会把已经渲染好的 dom 和 vnode 一一绑定，以达到同构的效果，`hydrate` 函数就做了这个任务。

## patch 整体流程

我整理了一份比较详细的 `patch` 流程图。
一些细节没有写入，比如：`updateChildren` diff 过程、异步组件、keep-alive、hydrate...

![Vue Patch Process](/images/2019/vue_patch_process.png)

## Vue 中有哪些功能是基于 patch 中的 Hooks 实现的？

#### directives

Vue 2 中的指令就是基于 hooks 实现的，从 directive 的生命周期来看：

1. bind：节点上绑定新的指令时调用 -> hooks: create, update
2. inserted：节点上绑定新的指令，若 hooks == create ，则在节点被 insert 时调用；若 hooks == update，则直接调用
3. update：节点上已存在指令时调用 -> hooks: update
4. componentUpdated： 节点上已存在指令，且在 hooks: postpatch 触发时调用，此时指令所在组件的 VNode 及其子 VNode 已全部更新。
5. unbind：节点被销毁时调用 -> hooks: destroy

#### ref

vnode 创建、更新、销毁时都会更新所在组件上的 `$refs` 属性。
patch 过程中处理了子组件为空时，父组件指向的 ref 为 undefined 的[边界情况](https://github.com/vuejs/vue/issues/3455)

#### style

对于动态绑定 style，Vue 还会智能的往不支持的属性前加厂商前缀。

```js
const normalize = cached(function(prop) {
  emptyStyle = emptyStyle || document.createElement('div').style
  prop = camelize(prop)
  if (prop !== 'filter' && prop in emptyStyle) {
    return prop
  }
  const capName = prop.charAt(0).toUpperCase() + prop.slice(1)
  for (let i = 0; i < vendorNames.length; i++) {
    const name = vendorNames[i] + capName
    if (name in emptyStyle) {
      return name
    }
  }
})
```

Vue 还支持 value 数组写法。比如 `{display: ["-webkit-box", "-ms-flexbox", "flex"]}`

Snabbdom 中对 style 增加了 hooks: remove，这个是为了实现节点被移除时的过渡动效，而 Vue 对过渡动效的处理封装在了 `<transition>` 组件中。

## 写在后面

其实 Vue 中还有很多功能都依赖 vnode 节点 patch 的过程，`transition` 的功能也比较多，这里暂时不深入了。
由于本人理解有限，文中如有任何问题欢迎留言指正。

## 参考

- [https://github.com/snabbdom/snabbdom](https://github.com/snabbdom/snabbdom)
- [https://github.com/vuejs/vue](https://github.com/vuejs/vue)
