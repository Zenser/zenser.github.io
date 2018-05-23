---
title: Vue是如何实现双向绑定的
date: 2018-05-16 10:03:58
tags:
    - Vue
---
* * *
本文主要探讨双向绑定，不对响应式做讨论哈
* * *

## 前言

一直对双向绑定的概念不是很清楚，处于“人云亦云”的状态，想要讲解却无法达到很好讲述水准，不能很清楚的讲明白这个概念。所以就有了写此文的想法。个人觉得，如果讲述者无法清晰、有条理、直白的某件事物，那么就说明他本人对这个事理解的也不是很透彻。

## 双向绑定概念
视图（View）与数据（Model）的同步。

这是我对双向绑定概念的理解。那么既然有双向绑定，那么有些同学可能会想了：有没有单向、三向【手动滑稽】...呢？单向其实就是很多框架主要做的事 <b>从Model -> View 的解决方案</b>，大多主流框架核心功能都是在做这个。

至于三向或更多向绑定，目前我还想不到有哪些需要和视图进行同步的啦...

## 使用Vue的双向绑定
既然大家都用文本输入框(input type=text)来展示，我就使用多选框来 🙋 🌰（举个栗子）啦。

``` html
<div id="app">
  <ol>
    <li v-for="item in options">
      <label>
        <input type="checkbox" v-model="selected" :value="item">
          {{ item }}
      </label>
    </li>
  </ol>
  <p>{{selected}}</p>
</div>
```
``` js
new Vue({
  el: "#app",
  data: {
    options: [ "吃饭", "睡觉", "LOL"],
    selected: ["LOL"]
  }
})
```

例子很简单，初始化时"LOL"会被选中，这其实就是Model -> View 了，当我们选择其他两项时，对应会触发 View -> Model ，selected值会同步更新了，简直不要太爽。当然还没结束，Vue监听到Model变了，会触发组件的render，再进行此组件的vdom对比，p标签的视图值也被同步啦。

## Vue双向绑定魔法是如何实现的

Vue v2开始数据都是单向的了，v-model是为了方便快速开发的语法糖，模板编译后v-model即被改变了。当然，不同的元素和类型会有不同的编译结果哦。

那么，我们把上面的代码片段进行编译，看看结果怎样。
<small>小提示：模板语法最终都会变为render函数！</small>
``` js
// node 脚本
const compiler = require('vue-template-compiler')
const template = `<div id="app">
  <ol>
    <li v-for="item in options">
      <label>
        <input type="checkbox" v-model="selected" :value="item">
          {{ item }}
      </label>
    </li>
  </ol>
  <p>{{selected}}</p>
</div>`

const compileToFunctionsResult = compiler.compileToFunctions(template)
console.log(compileToFunctionsResult.render)
```
以下为输出，已格式化过
``` js
function anonymous() {
    with(this) {
        return _c('div', {
            attrs: {
                "id": "app"
            }
        }, [_c('ol', _l((options), function(item) {
            return _c('li', [_c('label', [_c('input', {
                directives: [{
                    name: "model",
                    rawName: "v-model",
                    value: (selected),
                    expression: "selected"
                }],
                attrs: {
                    "type": "checkbox"
                },
                domProps: {
                    "value": item,
                    "checked": Array.isArray(selected) ? _i(selected, item) > -1 : (selected)
                },
                on: {
                    "change": function($event) {
                        var $$a = selected,
                            $$el = $event.target,
                            $$c = $$el.checked ? (true) : (false);
                        if (Array.isArray($$a)) {
                            var $$v = item,
                                $$i = _i($$a, $$v);
                            if ($$el.checked) {
                                $$i < 0 && (selected = $$a.concat([$$v]))
                            } else {
                                $$i > -1 && (selected = $$a.slice(0, $$i).concat($$a.slice($$i + 1)))
                            }
                        } else {
                            selected = $$c
                        }
                    }
                }
            }), _v("\n          " + _s(item) + "\n      ")])])
        })), _v(" "), _c('p', [_v(_s(selected))])])
    }
}
```

从编译后的代码中，我们就可以明确v-model被编译成了类似这样的模板语法。
``` html
<input :checked="selected.indexOf(item) > -1" @change="change"/>
```
当然，多选项的处理与文本框不同，change触发后多选项会针对数组做增，删等处理。

Vue 屏蔽了各种input底层的不同，抽象出了v-model的语法，减轻了我们开发过程中的代码量，也是很棒哦~

## 更进一步

我们知道了 Vue 对原生标签这么处理，如果是组件的话呢？

组件的话v-model 的等效写法类似以下
``` html
<self-component :value="value" @input="value=arguments[0]"/>
```
传递value属性；监听input事件，触发后更新Model的值，完美！

当然你可以在组件中自定义v-model的映射字段名称，默认就是这样咯
``` js
export default {
    model: {
        prop: 'value',
        event: 'input'
    },
    ...
}
```

## 结语
> 知其然，知其所以然

共勉，继续精进！:-D