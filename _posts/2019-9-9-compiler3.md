---
title: 来写一个简单的解释器（三）
description: 翻译自 Ruslan's Blog
tags: 翻译
---
# 来写一个简单的解释器（三）

今早起来的时候，我陷入了沉思：“为什么我们学习新知识如此的困难？”

我认为它不仅仅是因为它难。我认为其中的一个原因是因为我们花了大量的时间和精力通过阅读和观看来获取知识，而没有足够的时间通过练习来将这些知识转化成技能。比如学习游泳，你可以花很多时间阅读上百本有关游泳的书，和阅历丰富的游泳运动员和教练谈几个小时，观看所有能找到的训练视频，但是当你第一次跳进水里时候，你仍然会像摇滚一样在水中不知所措。

底线是：你认为自己对这门学科了解多少并不重要————否则你必须把这些知识付诸实践，从而把它变成一种技能。为了帮助你锻炼，我把一些练习题放到了第一章和第二章。是的，你会在今天和以后的文章看到更多更多的练习题，我保证:)

好的，让我们开始今天的素材，不然呢？

目前为止，您已经学会了如何解释两个整数的加减法。比如“7+3”、“12 - 9”。而今天我们讨论的是如何解析（识别）任意数量的加减法运算，比如“7-3+2-1”.

从图像上看，本文中的算术表达式可以用以下的语法图表示：

![](https://ruslanspivak.com/lsbasi-part3/lsbasi_part3_syntax_diagram.png)

什么是语法图呢？语法图是编程语言的语法规则的图形表示。

基本上，语法图可以直观的表示编程语言中允许那些语句，不允许哪些语句。

语法图非常易读：只需要按照箭头所指读下去就行。

一些路径表示选择，一些路径表示循环。

您可以按照如下方式阅读上面的语法图：一个项可选地后面跟着加法或者减法标志，然后跟着一个项，然后又可选地跟着一个加法或者减法，以此类推。

你粗浅的理解了图片的内容。你可能想知道什么是项（term）呢？
在本文中，项只是一个整数。

语法图主要有两个用途：

- 它们以图形方式表示编程语言规范（语法）。

- 它们可以帮助您编写解析器————您可以无脑的把图片内容“复制”到代码上。

您已经学到了识别Token流中短语的过程叫做解析。执行该部分的解释器或编译器的部分叫做解析器。解析也称作语法分析，您猜对了，解析器也恰当地称作语法分析器。

根据上面的语法图，以下所有的算术表达式都是合法的：

- 3

- 3 + 4

- 7 - 3 + 2 - 1

但是表达式“3 +” 不是合法的表达式，因为根据上面的语法图，加号后面必须有一项，否则就是语法错误。

从我们前面的文章（第一二章）可知，`expr`方法是我们解析器和解释器都存在的地方。

同样的，解析器只识别结构，确保它遵循规范，而解释器实际上在解析器成功识别后才会计算表达式。

下面的代码片段显示了相应的解析器代码。

语法图中的矩形框成为解析整数的`term`方法，`expr`方法仅仅遵循刚才的语法图：

```ts
  term = () => {
    this.eat(INTEGER)
  }

  expr = () => {
    this.currentToken = this.getNextToken()

    this.term()
    while (this.currentToken.type in [PLUS, MINUS]) {
      const token = this.currentToken
      if (token.type === PLUS) {
        this.eat(PLUS)
        this.term()
      } else if (token.type === MINUS) {
        this.eat(MINUS)
        this.term()
      }
    }
  }
```

你可以看到`expr`首先调用了`term`方法。然后`expr`方法有一个`while`循环来计算任意次。在循环内部，解析器根据Token来坐出选择。花一些时间给自己证明上面的代码确实遵循刚才的语法图吧。

解析器本身不解释任何东西：如果它识别表达式成功，那么它什么也不做，如果识别失败，就会抛出语法错误。

让我们修改一下`expr`方法并添加解释器代码：

```ts
  term = (): number => {
    const token = this.currentToken
    this.eat(INTEGER)
    return token.value
  }

  expr = () => {
    this.currentToken = this.getNextToken()

    let result = this.term()
    while ([PLUS, MINUS].indexOf(this.currentToken.type) !== -1) {
      const token = this.currentToken
      if (token.type === PLUS) {
        this.eat(PLUS)
        result += this.term()
      } else if (token.type === MINUS) {
        this.eat(MINUS)
        result -= this.term()
      }
    }
    return result
  }
```

因为解释器需要重新识别表达式，所以修改`term`方法返回整数，并修改`expr`在适当的地方执行加减法并返回结果。

即使代码很简单，我还是建议花一些时间研究它。

让我们开始，看看完整的代码。

以下是新版的源代码，可以处理任意数量的合法加减法运算表达式：

```ts
const INTEGER = 'INTEGER'
const PLUS = 'PLUS'
const MINUS = 'MINUS'
const EOF = 'EOF'

class Token {
  type: string
  value: any

  constructor (type: string, value: any) {
    this.type = type
    this.value = value
  }

  toString = () => `Token(${this.type}, ${this.value})`
}

export class Interpreter {
  text: string
  pos: number
  currentToken: Token = new Token(EOF, null)
  currentChar: string | null

  constructor (text: string) {
    this.text = text
    this.pos = 0
    this.currentChar = this.text[this.pos]
  }

  error = (): never => {
    throw Error('Error parsing input')
  }

  advance = () => {
    this.pos++
    if (this.pos > this.text.length - 1) {
      this.currentChar = null
    } else {
      this.currentChar = this.text[this.pos]
    }
  }

  skipWhitespace = () => {
    while (this.currentChar != null && this.currentChar === ' ') {
      this.advance()
    }
  }

  integer = () => {
    let result = ''
    while (this.currentChar != null && /[0-9]/.test(this.currentChar)) {
      result += this.currentChar
      this.advance()
    }
    return Number(result)
  }

  getNextToken = (): Token => {
    while (this.currentChar != null) {
      if (/ /.test(this.currentChar)) {
        this.skipWhitespace()
        continue
      }

      if (/[0-9]/.test(this.currentChar)) {
        return new Token(INTEGER, this.integer())
      } else if (this.currentChar === '+') {
        this.advance()
        return new Token(PLUS, '+')
      } else if (this.currentChar === '-') {
        this.advance()
        return new Token(MINUS, '-')
      }
      return this.error()
    }
    return new Token(EOF, null)
  }

  eat = (tokenType: string) => {
    if (this.currentToken.type === tokenType) {
      this.currentToken = this.getNextToken()
    } else {
      this.error()
    }
  }

  term = (): number => {
    const token = this.currentToken
    this.eat(INTEGER)
    return token.value
  }

  expr = () => {
    this.currentToken = this.getNextToken()

    let result = this.term()
    while ([PLUS, MINUS].indexOf(this.currentToken.type) !== -1) {
      const token = this.currentToken
      if (token.type === PLUS) {
        this.eat(PLUS)
        result += this.term()
      } else if (token.type === MINUS) {
        this.eat(MINUS)
        result -= this.term()
      }
    }
    return result
  }
}
```

我们的测试代码

```ts
import { Interpreter } from '../src'

describe('Unit test', () => {
  it('Interpreter run success', () => {
    expect((new Interpreter('1+2')).expr()).toBe(3)
    expect((new Interpreter('1+9')).expr()).toBe(10)
    expect((new Interpreter('20+20')).expr()).toBe(40)
    expect((new Interpreter('20-20')).expr()).toBe(0)
    expect((new Interpreter('1-2')).expr()).toBe(-1)
    expect((new Interpreter('1+2+3+4')).expr()).toBe(10)
    expect((new Interpreter('10 + 1 + 2 - 3 + 4 + 6 - 15')).expr()).toBe(5)
  })
})
```

还记得我文章开头提到的一堆练习吗？这就是所说的↓

![](https://ruslanspivak.com/lsbasi-part3/lsbasi_part3_exercises.png)

- 画一个语法图，仅包含乘法和除法表达式，比如“7 * 4 / 2 * 3”

- 支持仅包含乘法和除法的算术表达式，比如“7 * 4 / 2 * 3”

- 用其他语言写一个解释器，支持“7 - 3 + 2 - 1”这类的表达式。不要看本文章的例子。考虑一下它需要的东西：词法分析器（它接受把输入转化成Token流）、一个解析器（它从词法分析器提供的Token流中提取，并尝试理解其中的结构），解析器在成功解析后有解释器来解释算术表达式及其结果。将这些碎片串在一起。花一些时间将您获得的知识翻译成算术表达式的解释器吧！

检查你的理解：

- 什么是语法图？

- 什么是语法分析？

- 什么是语法分析器？

敬请关注下一章