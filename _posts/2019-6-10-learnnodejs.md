---
title: Nodejs的一些奇技淫巧 
tags: nodejs, javascript
---

当复制可枚举元素时候，直接用展开语法(Spread syntax)，效果贼棒

```js
const obj = {}
for (let i = 0; i < 10000; ++i) {
  obj[i] = Math.random()
}
console.time('1')
{
  let temp = {}
  for (let i = 0; i < 10000; ++i) {
    temp = { ...obj }
  }
}
console.timeEnd('1')

console.time('2')
{
  let temp = {}
  for (let i = 0; i < 10000; ++i) {
    for (let k in obj) {
      temp[k] = obj[k]
    }
  }
}
console.timeEnd('2')
```

```bash
> node index.js
1: 104.405ms
2: 3727.528ms
```
