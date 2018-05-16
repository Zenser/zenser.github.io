---
title: Vue源码之旅(2)-nextTick
date: 2017-12-09 14:02:35
thumbnail: /images/2017/next_tick2.png
tags:
  - Vue
categories:
  - Vue
---

* * *
基于vue version2.1.4
* * *

## Vue.nextTick( [callback, context] )
Vue 官方解释用法
> 在下次 DOM 更新循环结束之后执行延迟回调。在修改数据之后立即使用这个方法，获取更新后的 DOM。

## [#开发者调用后执行过程](#开发者调用后执行过程)
![开发者调用后执行过程](/images/2017/next_tick1.png)
  timerFunc是在Vue源码初始化后就已经确定的。
## [#Vue内部初始化timerFunc执行过程](#Vue内部初始化timerFunc执行过程)
![Vue内部初始化timerFunc执行过程](/images/2017/next_tick2.png)
nextTick的实现基于浏览器的microtask队列。浏览器优化了dom的更新，将dom更新划分为宏任务和微任务，js操作dom后并不是立即渲染，而是进入微任务队列中，以队列的形式统一进行更新。所以vue就是基于这个特性，实现了dom循环结束的监听，以便开发者做进一步操作。

但是并不是所有浏览器都暴露出了微任务执行结束事件，所以vue通过功能检查，根据各个浏览器处理能力的不同分出了Promise.then, MutationObserver, setTimeout这一步步的降级方案。

以下是各种处理的源码说明(只截取了部分片段)
##### Promsie.then
```js
if (typeof Promise !== 'undefined' && isNative(Promise)) {
    var p = Promise.resolve()
    var logError = err => { console.error(err) }
    timerFunc = () => {
      p.then(nextTickHandler).catch(logError)
      // UIWebViews, Promise.then并没有完全按规范实现
      // 回调导入了微任务队列，但是DOM并没有刷新
      // 所有通过空定时器hack，手动强制刷新了下
      if (isIOS) setTimeout(noop)
    }
}
```
##### MutationObserver
> MutationObserver给开发者们提供了一种能在某个范围内的DOM树发生变化时作出适当反应的能力.该API设计用来替换掉在DOM3事件规范中引入的Mutation事件.

```js
if (typeof MutationObserver !== 'undefined' && (
      isNative(MutationObserver) ||
      // PhantomJS and iOS 7.x
      MutationObserver.toString() === '[object MutationObserverConstructor]'
    )) {
      // use MutationObserver where native Promise is not available,
      // e.g. PhantomJS IE11, iOS7, Android 4.4
      var counter = 1
      var observer = new MutationObserver(nextTickHandler)
      var textNode = document.createTextNode(String(counter))
      observer.observe(textNode, {
        characterData: true
      })
      timerFunc = () => {
        // 这里巧妙的用了textNode在0,1之间变换，
        // 来进行dom变化后的监听
        counter = (counter + 1) % 2
        textNode.data = String(counter)
      }
}
```
##### setTimeout
微任务没实现，只能用宏任务代替了
```js
  else {
    // fallback to setTimeout
    /* istanbul ignore next */
    timerFunc = () => {
      setTimeout(nextTickHandler, 0)
    }
  }
```

## 附录
- 流程图使用[processon.com](https://processon.com)绘制
- 参考[https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)
- 参考[https://cn.vuejs.org/v2/api/#Vue-nextTick](https://cn.vuejs.org/v2/api/#Vue-nextTick)