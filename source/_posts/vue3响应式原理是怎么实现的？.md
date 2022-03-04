---
title: vue3响应式原理是怎么实现的？
date: 2022-03-04 23:37:31
tags: vue3
---

在本文中，我们首先会讨论什么是响应式数据和副作用函数，然后尝试实现一个相对完善的响应式系统，在这个过程中，我们会遇到各种各样的问题，例如如何避免无限递归？为什么需要嵌套的副作用函数？两个副作用函数之间会产生哪些影响？以及其他很多需要考虑的细节。接着我们会讨论与响应式数据相关的内容。接下来我们就从认识响应式数据和副作用函数开始，一步步的了解vue3的响应式系统的设计与实现。

# 1、响应式数据与副作用函数

副作用函数指的是会产生副作用的函数，如下的代码所示：

```javascript
 function effect(){
    document.getElementById('div').innerText =obj.text
  }
```

当effect函数执行时，他会设置body的文本内容，但除了effet函数之外的人任何函数都可以读取或者设置body的文本内容，也就是说，effect函数的执行，会直接或间接的影响其他函数的执行，这时我们说effect函数产生了副作用，副作用很容易产生，例如一个函数修改了全局变量，这也是一个副作用，如下面的代码所示：

```javascript
// 全局变量
let val = 1
function effect(){
  val = 2 //修改全局变量 产生副作用
}
```

理解了什么副作用函数，再来说说为什么是响应式函数。假设在一个副作用函数中读取了某个对象的属性：

```javascript
const obj = {text:'hello word'}
function effect(){
  //effect 函数的执行会读取 obj.text
   document.getElementById('div').innerText =obj.text
}
```

如上代码所示，副作用函数effect会设置div元素的innerText属性。其值为obj.text，当obj.text发生改变的时候，我们希望副作用函数

effect也重新执行。

```
obj.text = 'hello vue3' //修改了obj.text的值，同时希望副作用函数也执行。
```

# 2、响应式数据的基本实现

接着上文，如何让obj成为一个响应式数据呢？通过观察我们能发现两个线索：

- 副作用函数effect执行时，会触发obj.text的**读取**操作
- 修改obj.text的值的时候，会触发obj.text的**设置**操作。

当我们能够拦截一个对象的读写操作的时候，事情就会变得简单了，当读取字段obj.text时，我们可以把副作用函数effect存储到一个”**桶**˙“中

接着，当设置obj.text时，再把副作用函数从“桶”中提出来执行即可。
