---
title: 来写一个简单的解释器（二）
description: 翻译自 Ruslan's Blog
tags: 翻译
---
# 来写一个简单的解释器（二）

在一本有趣的书————《有用思维的五要素》中，作者Burger和 Starbird分享了他们如何观察Tony Plig，一个国际知名的小号演奏家，为优秀的小号演奏者开了一个大师班的故事。学生们首先演奏复杂的音乐短句，并且他们演奏的非常好。但后来他们被要求演奏非常基础、简单的音符。当他们演奏音符时候，与先前的复杂短句相比，这些音符显得十分幼稚。他们完成演奏后，老师也演奏了相同的音符，但是当他演奏的时候，却不显得幼稚。差异让人惊讶。Tony解释说，掌握简单的音符能够让人更好的控制复杂的曲子。这个例子很明确————想要掌握精湛的技术，必须注重简单、基础的思想。

故事中的启示不仅仅适用于音乐，也适用于软件开发。这个故事提醒我们不要忽视在基础、简单的想法上埋工夫的重要性。即使有时候干这些事情像是在退步。虽然熟练使用您使用的工具或者框架非常重要，但是其背后的原理也非常重要。

> “如果你只学习方法，那么你只会方法；但如果你学习原理，那么你可以定义你自己的方法。”

话题转回来，我们继续深入研究解释器和编译器。

今天，我将给你展示第一章中计算器的新版本，这个版本更够：

1. 处理字符串中任何位置的空白字符

2. 从输入中处理多位整数

3. 支持减法（之前目前只能加法）

以下是新版的源代码，可以执行以上所有操作

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

  expr = () => {
    this.currentToken = this.getNextToken()

    const left = this.currentToken
    this.eat(INTEGER)

    const op = this.currentToken
    if (op.type === PLUS) {
      this.eat(PLUS)
    } else {
      this.eat(MINUS)
    }

    const right = this.currentToken
    this.eat(INTEGER)

    if (op.type === PLUS) {
      return left.value + right.value
    } else {
      return left.value - right.value
    }
  }
}
```

然后我们的测试变成了这样

```ts
import { Interpreter } from '../src'

describe('Unit test', () => {
  it('Interpreter run success', () => {
    expect((new Interpreter('1+2')).expr()).toBe(3)
    expect((new Interpreter('1+9')).expr()).toBe(10)
    expect((new Interpreter('20+20')).expr()).toBe(40)
    expect((new Interpreter('20-20')).expr()).toBe(0)
    expect((new Interpreter('1-2')).expr()).toBe(-1)
  })
})
```

与第一版相比，主要的代码变成了：

1. 方法`getNextToken`方法稍微重构了一下，增加pos指针的逻辑被抽象到了新的方法`advance`

2. 增添了两个新方法，`skipWhitespace`用于忽略空格字符，`integer`用于处理多位整数

3. 除了 INTEGER -> PLUS -> INTEGER 之外，我们还增加了 INTEGER -> MINUS -> INREGER 短语。现在还能成功的识别加法或者减法

在第一部分，您学习了两个重要的概念，Token和词法分析器。

今天我想谈谈Lexemes、parsing和parsers。

你已经知道了Token。但是为了让我们完成对Token的讨论，我们需要谈一谈Lexemes。

什么是Lexeme？

Lexeme是一系列形成Token的字符。在下面的图片中，您可以看到Token和lexemes的一些例子，希望它能使它们之间的关系变得清晰一点：

![](https://ruslanspivak.com/lsbasi-part2/lsbasi_part2_lexemes.png)

现在，还记得我们的老朋友，`expr`方法吗？

我之前说过，这就是算术表达式实际发生的地方。但在你解析一个表达式之前，你首先要识别出它们是什么类型的短语，例如，它是加法，还是减法。

这就是`expr`方法的本质工作：他从`getNextToken`方法中获取标记流中的结构，然后解释已经识别的短语，生成算术表达式。

在Token流查找结构的过程，或换句话说，识别Token流的短语的过程称作解析（Parsing）。执行该工作的解释器或编译器的一部分乘坐解析器（Parser）。

所以，你现在知道了`expr`方法是你的解释器的一部分，它同时解析和解释句子。

`expr`方法首先尝试解析 INTEGER - > PLUS - > INTEGER或INTEGER - > MINUS - > INTEGER短语的Token流，并在识别成功后将加法或减法的结果返回给调用者。


然后，又到了练习时间。

![](https://ruslanspivak.com/lsbasi-part2/lsbasi_part2_exercises.png)

1. 扩展两位数的乘法

2. 扩展两位数的除法

3. 修改代码，支持任意数量的加法或减法。例如“9-5+3 + 11”

检查你的理解：

1. 啥是Lexeme？

2. Token流中查找结构的进程名称是什么？

3. 解析Token流的部分叫什么？

下一节，我们会扩展更复杂的算术表达式。敬请期待。