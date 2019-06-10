---
title: 基于 Decorator 实现对象校验
date: 2019-06-05 15:36:24
thumbnail: /images/2019/dvalidator_poster.png
tags:
  - Decorator
  - js
---

**Decorator 已经提案很久了，已经有过很大的改动。本文基于老的提案实现。**

## 前言

有了 Decorator，我认为表单校验方式会有更多的玩法。所以基于 Decorator 实现了一个纯净的对象校验的库 [dvalidator](https://github.com/Zenser/dvalidator)。
在无任何校验库的帮助下，我们可能会写出这样的代码

```js
let form = {
  nickname: '',
  password: ''
}

function submit() {
  if (!checkNickName(form.nickname)) {
    alert('昵称格式不正确')
    return
  }
  if (!checkPassword(form.password)) {
    alert('密码格式不正确')
    return
  }

  remoteValid(form.nickname)
    .then(() => {
      // do next
    })
    .catch(() => {
      alert('昵称已被注册')
    })
}
```

使用 [dvalidator](https://github.com/Zenser/dvalidator) 我们可以这样写

```js
import dvalidator from 'dvalidator'

let form = {
  @dvalidator(remoteValid)('昵称已被注册')
  @dvalidator(checkNickName)('昵称格式不正确')
  nickname: '',
  @dvalidator(checkPassword)('密码格式不正确')
  password: ''
}

function submit() {
  form
    .$validate()
    .then(() => {
      // do next
    })
    .catch(error => alert(error[0].message))
}
```

## Decorator 基础

> “装饰者模式：在不改变原对象的基础上，通过对其进行包装拓展（添加属性或者方法）使原有对象可以满足用户的更复杂需求。”
> --《JavaScript 设计模式》

1. 许多面向对象的语言都有修饰器（Decorator）函数，用来修改类的行为
2. 可作用于类或对象中的属性和方法
3. 初始化时从上至下运行，执行时从内向外

**代码来源： [http://es6.ruanyifeng.com/#docs/decorator#%E6%96%B9%E6%B3%95%E7%9A%84%E4%BF%AE%E9%A5%B0](http://es6.ruanyifeng.com/#docs/decorator#%E6%96%B9%E6%B3%95%E7%9A%84%E4%BF%AE%E9%A5%B0)**
```js
function dec(id){
  console.log('evaluated', id);
  return (target, property, descriptor) => console.log('executed', id);
}

class Example {
    @dec(1)
    @dec(2)
    method(){}
}
// evaluated 1
// evaluated 2
// executed 2
// executed 1
```


## 使用 dvalidator

使用 dvalidator 校验对象有这些优点

1. 代码更加直观，优雅，便于后续维护
2. 支持异步校验：传递的校验函数返回 Promise 即可实现
3. 按顺序校验：根据 decorator 执行的先后顺序

#### 安装
```bash
npm install dvalidator --save
```

```bash
npm install @babel/plugin-proposal-decorators --save-dev
```

#### 使用

配置 babel

```js
plugins: [
  [
    '@babel/plugin-proposal-decorators',
    {
      legacy: true
    }
  ]
]
```

## 原理

1. 为对象增加 `__rules` 属性，并不可枚举，配置，写
2. rules的属性与obj属性一一对应
3. 每申明一个Decorator，其实都是更新 `__rules` 属性
4. 调用 `$validate` 时，会根据 rules 对整个对象进行校验，返回 Promise，校验失败会返回所有失败信息

## 参考
- http://es6.ruanyifeng.com/#docs/decorator
- https://github.com/yiminghe/async-validator
