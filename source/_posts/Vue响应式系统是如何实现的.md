---
title: Vue响应式系统是如何实现的
tags:
  - Vue
  - 响应式
  - Observer
date: 2018-07-07 15:09:02
thumbnail: /images/2018/observe-watcher-flow.png
categories:
  - Vue
---


## 前言
最近深入学习了Vue实现响应式的部分源码，将我的些许收获和思考记录下来，希望能对看到这篇文章的人有所帮助。有什么问题欢迎指出，大家共同进步。

## 什么是响应式系统
一句话概括：数据变更驱动视图更新。这样我们就可以以“数据驱动”的思维来编写我们的代码，更多的关注业务，而不是dom操作。其实Vue响应式的实现是一个变化追踪和变化应用的过程。

## vue响应式原理
以数据劫持方式，拦截数据变化；以依赖收集方式，触发视图更新。利用es5 `Object.defineProperty`拦截数据的setter、getter；getter收集依赖，setter触发依赖更新，而组件render也会变为一个watcher callback被加入相应数据的依赖中。

#### 发布订阅
利用发布订阅设计模式实现，`Observer`作为发布者，`Watcher`作为订阅者，两者无直接交互，通过Dep进行统一调度。
`Observer`负责拦截get, set；get时触发dep添加依赖，set时调度dep发布；添加`Watcher`时会触发订阅数据的get，并加入到dep调度中心的订阅者队列中。

以下的UML类图是Vue实现响应式功能的类，以及他们之间的引用关系。
<small>只包含部分属性方法</small>
![Observer-Component-Dep-Watcher](/images/2018/Observer-Component-Dep-Watcher.png)

上图中的类已经标识的蛮清楚了，但是还是需要一个调用关系图，让调用过程更加清晰，如下图所示。
响应式data对象中，每一项key的劫持get/set函数都闭包了Dep调度实例，这张图显示了一个key更改过程中的数据流转。
![observe-watcher-flow](/images/2018/observe-watcher-flow.png)

#### 部分源码
数据变更过程中的订阅/发布模型上图已经清晰的展示了，从图中我们已经知道了可以通过增加watcher来订阅某一项数据的变更。那么，我们只需要把组件render作为一个watcher订阅的话，数据驱动视图的渲染岂不是水到渠成了。Vue正是这么做的！
<small>以下代码片段来自Vue.prototype._mount函数</small>
```js
callHook(vm, 'beforeMount')
vm._watcher = new Watcher(vm, () => {
    vm._update(vm._render(), hydrating)
}, noop)
hydrating = false
// manually mounted instance, call mounted on self
// mounted is called for render-created child components in its inserted hook
if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
}
```
## 一些问题思考
#### #person赋值新的对象，新对象里的属性是否也是响应式的呢？
[JS Bin在线调试](https://jsbin.com/jogupimigi/edit?html,js,output)
```js
var vm = new Vue({
    el: '#app',
    data: () => ({
        person: null
    })
})
vm.person = {name: 'zs'}
setTimeout(() => {
    // 更改name
    vm.person.name = 'finally zs'
}, 3000)
```
答案：是响应式的。
原因：因为Vue劫持set时，会对value再次做observe，源码如下。
```js
function reactiveSetter (newVal) {
    /* ...省略部分代码 */
    // 这里会再次对新的value做拦截
    childOb = observe(newVal)
    dep.notify()
}
```

#### #当我们监听多层属性时，上层引用变更，是否会触发回调？
```js
var vm = new Vue({
    data: () => ({
        person: {name: '令狐洋葱'}
    }),
    watch: {
        'person.name'(val) {
            console.log('name updated', val)
        }
    }
})
vm.person = {}
```
答案：会。
原因：person.name作为一个表达式传入Watcher时，会被解析成类似这样的函数
```js
() => {this.vm.person.name}
```
这样就会先触发person get, 然后触发name get；所以我们配置的回调函数，不仅仅加入到了name依赖中，person也有。

#### #接着上个问题，person如果被赋值了新的对象，老对象和老对象上的依赖如何垃圾回收的？
- 老对象的回收：由于老对象的直接引用只有vue实例上的person，person切换到了新的引用，所以老对象没有引用了，就会被回收掉。
- 老对象上的依赖dep，watcher的依赖里还存在；但是在run执行时，会调用watcher的get() 获取当前值；get中会执行新的依赖收集，并在收集完毕后，清空老的依赖。
具体源码如下：
```js
/**
* Evaluate the getter, and re-collect dependencies.
*/
get () {
    pushTarget(this)
    const value = this.getter.call(this.vm, this.vm)
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) {
        traverse(value)
    }
    popTarget()
    this.cleanupDeps()
    return value
}
```

#### #当我们多次同步修改name时，回调函数是否会触发多次？
```js
var vm = new Vue({
    data: () => ({
        person: {name: '令狐洋葱'}
    }),
    watch: {
        'person.name': (val) {
            console.log('name updated: ' + val)
        }
    }
})
vm.person = {name: 'zs'}
vm.person.name = '无敌'
```
答案: 不会，因为watch回调函数执行是异步的，且会去重。可以通过sync强制配置成同步run，就会执行2次了。


## 自己实现一个响应式系统
只包含核心功能，具体源码可以看这里[https://github.com/Zenser/z-vue](https://github.com/Zenser/z-vue)，欢迎来star。
实现功能非常基础啦，重在理解，功能不全的。

#### Observer
```js
class Observe {
    constructor(obj) {
        Object.keys(obj).forEach(prop => {
            reactive(obj, prop, obj[prop])
        })
    }
}

function reactive(obj, prop) {
    let value = obj[prop]
    // 闭包绑定依赖
    let dep = new Dep()
    Object.defineProperty(obj, prop, {
        configurable: true,
        enumerable: true,
        get() {
            //利用js单线程，在get时绑定订阅者
            if (Dep.target) {
                // 绑定订阅者
                dep.addSub(Dep.target)
            }
            return value
        },
        set(newVal) {
            value = newVal
            // 更新时，触发订阅者更新
            dep.notify()
        }
    })

    // 对象监听
    if (typeof value === 'object' && value !== null) {
        Object.keys(value).forEach(valueProp => {
            reactive(value, valueProp)
        })
    } 
}
```

#### Dep
```js
class Dep {
    constructor() {
        this.subs = []
    }
    addSub(sub) {
        if (this.subs.indexOf(sub) === -1) {
            this.subs.push(sub)
        }
    }
    notify() {
        this.subs.forEach(sub => {
            const oldVal = sub.value
            sub.cb && sub.cb(sub.get(), oldVal)
        })
    }
}
```

#### Watcher
```js
class Watcher {
    constructor(data, exp, cb) {
        this.data = data
        this.exp = exp
        this.cb = cb
        this.get()
    }
    get() {
        Dep.target = this
        this.value = (function calcValue(data, prop) {
            for (let i = 0, len = prop.length; i < len; i++ ) {
                data = data[prop[i]]
            }
            return data
        })(this.data, this.exp.split('.'))
        Dep.target = null
        return this.value
    }
}
```

## 写在最后
文章和我的实现仅供参考，欢迎留言指正，我们一起探讨。

## 参考文档
- [https://cn.vuejs.org/v2/guide/reactivity.html](https://cn.vuejs.org/v2/guide/reactivity.html)
