---
title: Vue源码之旅(1)-vue实例
date: 2016-12-24 10:07:34
tags:
  - Vue
categories:
  - Vue
---

> vue version 1.0.28。后续会有与2.x版本的比较。原创，欢迎转载（请注明链接）。未完待续。
* * *

## 入口文件
导入Vue类，添加全局api（例如：编译器，工具方法，解析器，指令，过滤器....），这些之后文章会单独介绍。
后面是对chrome扩展程序Vue devtool的处理，在开发环境中会唤醒Vue devtool，没安装的话在chrome环境下回提示用户进行安装已获得更好的开发体验。开发者工具主要用来组件数据可视化的。
关键代码：
```js
installGlobalAPI(Vue)

Vue.version = '1.0.28'

export default Vue

// devtools global hook
/* istanbul ignore next */
setTimeout(() => {
  if (config.devtools) {
    if (devtools) {
      devtools.emit('init', Vue)
    } else if (
      process.env.NODE_ENV !== 'production' &&
      inBrowser && /Chrome\/\d+/.test(window.navigator.userAgent)
    ) {
      console.log(
        'Download the Vue Devtools for a better development experience:\n' +
        'https://github.com/vuejs/vue-devtools'
      )
    }
  }
}, 0)
```
## Vue实例
主要分为2大部分：内部和外部api
*  公共方法/属性以* $ *开头
*  私有方法/属性以* _ *开头
*  其他属性为用户自定义

入口文件：

```js

function Vue (options) {
  this._init(options)
}

// install internals
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
miscMixin(Vue)

// install instance APIs
dataAPI(Vue)
domAPI(Vue)
eventsAPI(Vue)
lifecycleAPI(Vue)

export default Vue
```

### Vue初始化
initMixin(Vue)做了什么？要从internal/init.js看起。
绑定_init方法到Vue原型上。初始化一堆变量。
-  绑定上下文_content
-  给每个实例都绑定上递增的uid
-  打通构造器options和用户自定义options和实例，共享数据。
-  触发钩子函数init&created。可以看到* init *是在除state和events初始化前进行触发的，* created *是在其之后触发。
```js
let uid = 0

export default function (Vue) {
  /**
   * The main init sequence. This is called for every
   * instance, including ones that are created from extended
   * constructors.
   *
   * @param {Object} options - this options object should be
   *                           the result of merging class
   *                           options and the options passed
   *                           in to the constructor.
   */

  Vue.prototype._init = function (options) {
    options = options || {}

    this.$el = null
    this.$parent = options.parent
    this.$root = this.$parent
      ? this.$parent.$root
      : this
    this.$children = []
    this.$refs = {}       // child vm references
    this.$els = {}        // element references
    this._watchers = []   // all watchers as an array
    this._directives = [] // all directives

    // a uid
    this._uid = uid++

    // a flag to avoid this being observed
    this._isVue = true

    // events bookkeeping
    this._events = {}            // registered callbacks
    this._eventsCount = {}       // for $broadcast optimization

    // fragment instance properties
    this._isFragment = false
    this._fragment =         // @type {DocumentFragment}
    this._fragmentStart =    // @type {Text|Comment}
    this._fragmentEnd = null // @type {Text|Comment}

    // lifecycle state
    this._isCompiled =
    this._isDestroyed =
    this._isReady =
    this._isAttached =
    this._isBeingDestroyed =
    this._vForRemoving = false
    this._unlinkFn = null

    // context:
    // if this is a transcluded component, context
    // will be the common parent vm of this instance
    // and its host.
    this._context = options._context || this.$parent

    // scope:
    // if this is inside an inline v-for, the scope
    // will be the intermediate scope created for this
    // repeat fragment. this is used for linking props
    // and container directives.
    this._scope = options._scope

    // fragment:
    // if this instance is compiled inside a Fragment, it
    // needs to register itself as a child of that fragment
    // for attach/detach to work properly.
    this._frag = options._frag
    if (this._frag) {
      this._frag.children.push(this)
    }

    // push self into parent / transclusion host
    if (this.$parent) {
      this.$parent.$children.push(this)
    }

    // merge options.
    options = this.$options = mergeOptions(
      this.constructor.options,
      options,
      this
    )

    //更新父组件$refs属性，绑定到当前组件
    this._updateRef()

    // initialize data as empty object.
    // it will be filled up in _initData().
    this._data = {}

    // call init hook
    this._callHook('init')

    // 启动实例作用域包括对数据observe，计算属性computed，自定义methods
    // 主要是对数据流进行处理，监听数据变化，变化后对绑定的计算属性进行计算
    this._initState()

    // 绑定实例的events & watchers，用于组件间同行，和数据变动后的watch钩子函数触发
    this._initEvents()

    // call created hook
    this._callHook('created')

    // if `el` option is passed, start compilation.
    if (options.el) {
    	//el: {Element|DocumentFragment|string}
    	//编译处理dom
      this.$mount(options.el)
    }
  }
}
```
## Vue api
这部分是暴露给框架使用者的api
包括data的操作：
1. $get, $set, $delete 解析表达式
```js
Vue.prototype.$get = function (exp, asStatement) {
  var res = parseExpression(exp)
  if (res) {
    if (asStatement) {
      var self = this
      return function statementHandler () {
        self.$arguments = toArray(arguments)
        var result = res.get.call(self, self)
        self.$arguments = null
        return result
      }
    } else {
      try {
        return res.get.call(this, this)
      } catch (e) {}
    }
  }
}
function parseExpression (exp, needSet) {
  exp = exp.trim()
  // 内存已存放直接使用
  var hit = expressionCache.get(exp)
  if (hit) {
    if (needSet && !hit.set) {
      hit.set = compileSetter(hit.exp)
    }
    return hit
  }
  var res = { exp: exp }
  //简单路径，例如{{user.name}}{{path + '/lala'}}
  //复杂路径，例如{{user[nam + 'e']}}
  res.get = isSimplePath(exp) && exp.indexOf('[') < 0
    // 例如exp='user.name'
    // 返回function(scope){return scope.user.name}
    ? makeGetterFn('scope.' + exp)
    // dynamic getter
    : compileGetter(exp)
  if (needSet) {
    res.set = compileSetter(exp)
  }
  expressionCache.put(exp, res)
  return res
}
```
2, $watch, $eval, $interpolate就暂时不展开说了，感觉每个都可以当做一篇来写。

## 结语
大家有什么看法欢迎各种渠道沟通，谢谢~