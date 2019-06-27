---
title: JavaScript的神奇bug
tags: javascript
---

一个例子，猜一下会不会Throw Error

```js
class Father {
  constructor() {
    if (arguments.length === 0) {
      throw Error('1')
    } else {
      this.foo = arguments[0]
    }
  }
}

class Son extends Father {
  constructor(first, second) {
    super(first)
  }
}

const son = new Son()
```

**没有**

为什么呢，因为 `Son` 类型虽然不存在 `first` 参数，但是它是 `undefined`，所以它变成了参数传递给了父类，于是 `this.foo = undefined`

## 正确方法

```js
class Son extends Father {
  constructor(first, second) {
    super(...arguments)
  }
}
```