---
title: 【译】typeof null渊源
id: 38
categories:
  - js
date: 2016-10-14 17:15:00
tags:
---

> 原文：[http://www.2ality.com/2013/10/typeof-null.html](http://www.2ality.com/2013/10/typeof-null.html)

* * *

**更新于**2013-11-05：我看了下C语言typeof规范，它对于typeof null为何结果是'object'有更好的解释

* * *

在javascript中，typeof null等于'object'，这是不正确的。这是个bug，不幸的是这并不能解决，因为这将破坏已有规范。接下来解释下这个bug的历史。

"typeof null"的问题从JavaScripts第一个版本开始已经存在了。这个版本，值使用32bit存储，由一个type tag（1~3bits）和实际数据组成。这个标志位存储在较小单元中。共有5种标志：

*   000：object。对象。
*   1: int。占31bit的整型数据
*   010：double。双精度浮点数
*   100：string.字符串
*   110：boolean。
有2个值是特殊的：

*   undefined(JSVAL_VOID) 为integer -2^30(整型长度边界值)
*   null(JSVAL_NULL) 为机器码NULL的指针，或者说：为零的object类型标签
现在，关于为什么typeof null是一个object应该已经呼之欲出了；它检测一个他的type tag并且返回"object"。下面列出typeof执行规则。
```js
    JS_PUBLIC_API(JSType)
    JS_TypeOfValue(JSContext *cx, JSVAL v){
        JSType type = JSTYPE_VOID;
        JSObject *obj;
        JSObjectOps *ops;
        JSClass *clasp;

        CHECK_REQUEST(cx);
        if (JSVAL_IS_VOID(v)) {  // (1)
            type = JSTYPE_VOID;
        } else if (JSVAL_IS_OBJECT(v)) {  // (2)
            obj = JSVAL_TO_OBJECT(v);
            if (obj &amp;&amp;
                (ops = obj-&gt;map-&gt;ops,
                 ops == &amp;js_ObjectOps
                 ? (clasp = OBJ_GET_CLASS(cx, obj),
                    clasp-&gt;call || clasp == &amp;js_FunctionClass) // (3,4)
                 : ops-&gt;call != 0)) {  // (3)
                type = JSTYPE_FUNCTION;
            } else {
                type = JSTYPE_OBJECT;
            }
        } else if (JSVAL_IS_NUMBER(v)) {
            type = JSTYPE_NUMBER;
        } else if (JSVAL_IS_STRING(v)) {
            type = JSTYPE_STRING;
        } else if (JSVAL_IS_BOOLEAN(v)) {
            type = JSTYPE_BOOLEAN;
        }
        return type;
    }
```
解释下以上代码：

*   （1），引擎首先检测值是否是undefined.他做了这样的比较：
```js       
#define JSVAL_IS_VOID(v)  ((v) == JSVAL_VOID)
```

*   下一个（2）是检测该值是否具有object type。 如果它可使用call被调用（3）或其存在内部属性[[Class]]标记为函数（4），则v是函数。 否则，它是一个对象。 这是由typeof null生成的结果。
*   后续检查是针对number，string和boolean。 甚至没有明确检查null。
```js
   #define JSVAL_IS_NULL(v)  ((v) == JSVAL_NULL)
```

这似乎是一个非常明显的bug，但不要忘记，第一个版本的JavaScript完成只用了极少的时间。

* * *

致谢：感谢Tom Schuster（@evilpies）指导我经典JavaScript源代码。