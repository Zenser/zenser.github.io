---
title: 使用Dvalidator重新定义对象校验
date: 2019-09-12 18:43:32
tags:
  - Decorator
  - js
  - 校验
categories:
  - js
---

> 道，自然也。自然即是道。自然者，自，自己。然，如此，这样，那样。

## 前言

世间万物，变幻莫测。现代社会发展迅速，”效率“这个话题一直围绕着我们。互联网提升了传统行业的运行效率，开发行业中各种框架/工具提升了程序员的开发效率。如果你是个前端，并且你会有开发表单的需求；如果你是个 node.js 后端，并且你需要做一些参数校验的工作。那么 [Dvalidator](https://github.com/Zenser/dvalidator) 也许值得一试。

## 简介

Dvalidator 是一个用于对象校验的工具，可以使用在任何 js 运行环境中。它推荐使用装饰器语法来制定校验规则，目前装饰器提案处于 Stage2(Draft) 阶段了，是一个初步标准，这意味着它有可能还会有一些变动。不过 Decorator 目前已经非常泛滥了，我认为可以无所顾虑的使用它。

你可能会有这样的疑问 “为什么要写 Dvalidator ，市面上不是有很多校验工具么？”。我几年前使用过 `angular.js` 中自带的校验，也使用过 `element-ui` 中的校验。是的，都做得非常好，这些作者也许比我水平高出几个珠穆朗玛峰。但是我认为都不够简洁（功能太强），不独立（集成特定框架），我想做一个纯 js 的对象校验，而且要非常简单，非常易用。我可能考虑的应用场景还太少，但是我认为还是能覆盖大部分场景的。Dvalidator 的使用者只需要制定规则，然后发起校验，就完了。对！只有 2 个 Api。

也许我只是想造个非常简单的轮子。

## Dvalidator

> Dvalidator 是一个纯净、可扩展、非常有用的校验工具，它基于 Promise 和 Decorator 实现。

它有以下特性：

- <b>Compatibility</b> : 同时支持最新版 Decorator 用法和老版用法
- <b>Asynchronous</b> : 支持传递异步函数
- <b>Ordered</b> : 根据你定义的顺序，有序校验
- <b>Small</b> : 小巧，源码不超过 200 行
- <b>Easy</b> : 使用简单，仅仅只有 2 个 Api

## 起步

```bash
npm install dvalidator --save
```

```bash
npm install @babel/plugin-proposal-decorators --save-dev
```

配置 babel.config.js

```js
plugins: [
  [
    '@babel/plugin-proposal-decorators',
    {
      // Dvalidator 支持最新的 Decorator 提案（legacy: false）
      // 同样也支持旧版的 (legacy: true)，Decorator 可以作用于字面量对象
      // 按照你的喜好设置，推荐使用最新的提案
    }
  ]
];
```

## 使用

假设我们有这样一个需求，我们将校验一个 user 对象，昵称和手机号是必选的，并且手机号需要发起一个服务端远程校验。

使用 Dvalidator，我们可以这样写：

```js
// common.js
// 可以把校验规则做一下封装，写在单独的文件里，这样业务代码会非常简洁。
import dvalidator from 'dvalidator';

const requiredRule = {
  validator: val => val != null && val !== '',
  message: 'required'
};
const required = dvalidator(requiredRule);
const checkPhone = dvalidator(value => fetch(`xxx/${value}`));

// user-signup.js
// 业务逻辑
class User {
  @required('nickname is required')
  nickname = '';
  @checkPhone('phone valid fail')
  @required
  phone = '';
}
const user = new User();

user
  .$validate()
  .then(/* success */)
  .catch(({ errors, fields }) => {
    /* fail */
    alert(errors[0].message);
    // errors 包含每个属性的错误信息，结构一致，嵌套对象会拍平
    // fields 以对象形式获取错误信息，一般用于展示表单中每一栏的错误信息
  });
```

你也可以实际操作一下 [https://jsfiddle.net/zeusgo/oy1xv0za/7](https://jsfiddle.net/zeusgo/oy1xv0za/7)

## Api

#### dvalidator(rule: string | Function | Rule): Dvalidator

<small>一个类柯里化函数，你可以无限次调用去丰富规则或者覆盖规则。</small>

你需要传递规则进来，规则可以是一个函数（校验方法），字符串（错误信息），或者对象（包含以上两者的集合）。

例如：

```js
dvalidator({
  validator: val => {
    // 你的校验部分代码
    // 可以返回 Boolean（同步校验） 或者 Promise（异步校验）
  },
  // 校验出错时会返回给你
  message: ''
});
```

一个校验规则想要返回不同错误信息：

```js
// 传递不同的 message
dvalidator(checkPhone)('msg1');
dvalidator(checkPhone)('msg2');
```

```js
// 也可以动态返回错误信息
dvalidator(() => {
  return Promise.reject(x ? 'msg1' : 'msg2');
});
```

#### \$validate(filter?: Function): Promise<ValidateError | void>

把装饰器加入到对象上后，对象就是属于 “可校验对象”，你可以此方法进行数据校验。filter 是一个用来过滤属性的方法，我们可以用它做一些动态校验。

```js
// 返回 Promise
user
  .$validate(fieldKey => {
    // 这里可以定义你的过滤逻辑
    // 如果是嵌套的对象，那么 fieldKey 会做拼接
    // 例如 user: { like: { game: 'lol' } }，只想校验 like.game 的时候，你可以这样写
    return /^like\.game/.test(fieldKey);
  })
  .catch(({ errors, fields }) => {
    // xxx
  });
```

## 接口声明文件

从这里可以看到更详细的结构信息！
[index.d.ts](https://github.com/Zenser/dvalidator/blob/master/lib/index.d.ts)

## And More

Enjoy it!
