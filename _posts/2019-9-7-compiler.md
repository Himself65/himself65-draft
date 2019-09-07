---
title: 来写一个简单的解释器（一）
description: 翻译自 Ruslan's Blog
tags: 翻译
---
# 来写一个简单的解释器（一）

> “如果你不知道编译器的原理，那么你就不知道电脑如何运行。如果你不是100%确定你知道编译器的原理，那么你就是不知道他们的原理”
>
> — Steve Yegge

你读了这句话。然后仔细想一想。它真的与你是否是一个电脑小白或者经验丰富的开发者没有关系：如果你不知道编译器和解释器的原理，那么你就真的不知道电脑如何运行。就是这么简单。

所以说，你知道解释器和编译器的工作原理吗？我的意思是，你是否百分比确信你知道他们的原理。

如果你说你不知道

[](https://ruslanspivak.com/lsbasi-part1/lsbasi_part1_i_dont_know.png)

或者你说你不知道并且对此十分的激动

[](https://ruslanspivak.com/lsbasi-part1/lsbasi_part1_omg.png)

不要担心。如果你坚持完成这个系列，并和我一起搭建一个解释器和编译器，你最终就会知道他们的原理的。然后你也就会变成一个高兴又自信的人。至少我是这么觉得。

[](https://ruslanspivak.com/lsbasi-part1/lsbasi_part1_i_know.png)

为啥你会去学编译器和解释器呢？我给你三个理由

1. 要写一个解释器/编译器，那么你必须一起使用许多的技术。写一个编译器或解释器将帮助你提升这些技能并且变成一个更牛逼的软件开发者。同样的，这些你将学到的技能是在写任何程序时候都是很有用的，而不仅仅局限于解释器或编译器。

2. 你真的想知道计算机时如何工作的。解释器和编译器通常看起来和魔术一样。而且你不会对这些魔术感到满意。您想要揭开解释器和编译器的神秘面纱，理解它们如何工作，以及控制事务的过程。

3. 你想要编写属于你自己的编程语言或DSL（领域特殊语言）。如果你设计了一个语言，那么你也得去给它写一个解释器或编译器。最近掀起了一股创造编程语言之风。你也可以看到一个个新语言涌现：Elixir、Go、Rust等等。

Ok，那么啥是解释器，啥是编译器呢？？？

解释器或编译器的目标是把更高级的源代码翻译成另外一种形式。很模糊吧？带着疑问，之后的系列你将会学会源代码到底翻译成什么的细节。

此刻，你可能也想知道解释器和编译器的区别是什么。出于这一系列的目的只是为了简单起见，我们认为，如果翻译器将源代码翻译成机器语言，那他就是编译器。如果一个翻译器处理并且运行了源代码，而且没有翻译它到机器语言，那他就是解释器。

如图所示：

![](https://ruslanspivak.com/lsbasi-part1/lsbasi_part1_compiler_interpreter.png)
![](https://ruslanspivak.com/lsbasi-part1/lsbasi_part1_compiler_interpreter.png)

我希望目前为止你真的想学习、构建一个解释器和编译器。你对本系列的解释器有什么期待呢？

答案是这样的。你和我将给Pascal语言的大子集写一个简单的解释器。本系列的最后，你将有一个高效的Pascal解释器和一个像Python的pdb一样的代码级别的调试器。

你可能会问，为啥是Pascal呢？一方面，Pascal不是仅仅为了写这篇文章而虚构出的语言：真的存在这个语言，它还有许多重要的语言结构。并且有一些古老但是有用的计算机科学书籍，用着Pascal语言写着Example（我知道这不是一个令人信服的理由，来让人选择给它写一个解释器，但是学习一个非流行语言至少会很好）

接下来是一段Pascal中的阶乘函数的例子。你将用自己的解释器来运行它，并且在此过程中，你将写一个交互式的源代码调试器来调试它。

```pascal
program factorial;

function factorial(n: integer): longint;
begin
    if n = 0 then
        factorial := 1
    else
        factorial := n * factorial(n - 1);
end;

var
    n: integer;

begin
    for n := 0 to 16 do
        writeln(n, '! = ', factorial(n));
end.
```

实现Pascal解释器的语言将会是TypeScript（原文是Python，但是译者我不喜欢Python），但是你可以用任何你喜欢的语言来写，因为这不依赖于任何特定的语言。

OK，让我们开始！

你将通过编写一个简单的算数表达式解释器（计算器），开始首次涉足解释器和编译器。今天的目标非常简约：让你的计算器能够处理两位整数相加，比如 3+5。这是你的计算器的源代码，不对，是解释器：

```ts
const INTEGER = 'INTEGER'
const PLUS = 'PLUS'
const EOF = 'EOF'

class Token {
  type: string
  value: string

  constructor (type: string, value: any) {
    this.type = type
    this.value = value
  }

  toString = () => `Token(${this.type}, ${this.value})`
}

export class Interpreter {
  text: string
  pos = 0
  currentToken: Token = new Token(EOF, null)

  constructor (text: string) {
    this.text = text
  }

  error = (): never => {
    throw Error('Error parsing input')
  }

  getNextToken = (): Token => {
    const text = this.text
    if (this.pos > text.length - 1) {
      return new Token(EOF, null)
    }

    const currentChar = text[this.pos]

    if (/[0-9]/.test(currentChar)) {
      this.pos++
      return new Token(INTEGER, Number(currentChar))
    } else if (currentChar === '+') {
      this.pos++
      return new Token(PLUS, Number(currentChar))
    } else {
      return this.error()
    }
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
    this.eat(PLUS)

    const right = this.currentToken
    this.eat(INTEGER)

    return left.value + right.value
  }
}
```

然后我们写一个测试（使用jest）

```ts
import { Interpreter } from '../src'

describe('Unit test', () => {
  it('Interpreter run success', () => {
    expect((new Interpreter('1+2')).expr()).toBe(3)
    expect((new Interpreter('1+9')).expr()).toBe(10)
  })
})
```

为了使您的计算器正常工作，您还必须遵守以下规则：

- 仅仅输入个位数相加

- 目前唯一支持的操作只有加法

- 输入中不允许有空格

为了保持计算器简单，这些限制是有必要的。不要担心，代码很快就会变得很复杂。

现在，我们深入了解你的解释器的工作原理，以及它如何鉴别算术表达式。

当你在控制行里输入表达式“3+5”时候，你的解释器获取了一个字符串“3+5”，为了让解释器真正理解如何处理该字符串，首先需要将输入“3+5”分解成为称作**Token**的组件。一个**Token**是一个带有类型和值的对象。比如，对于字符“3”，它的类型会是INTEGER，对应的值将是整数3。

将输入字符串分解为标记的过程称为词法分析（lexical analysis）。所以，你的解释器第一步就是读取输入的字符并且将他们转换成一串Token。执行此操作的解释器部分称为词法分析器（lexical analyzer），或者简称 Lexer。你可能还会遇到同一组件的其他说法，比如Scanner或者Tokenizer。但他们都是一个意思：用于将字符输入转换成Token流，作为解释器或者编译器的一部分。

`Interpreter`类的方法`getNextToken`是你的词法分析器。每次调用它时候，都会从传给解释器的字符串中创建下一个Token。我们细看一下方法本身是如何将字符转换成Token的。输入存储在变量text中，pos是一个字符串的索引（把字符串视为字符数组）。

最初pos指向字符“3”。该方法首先检查其是否是数字，如果是，则返回值为3的INTEGER的Token实例并pos递增。

![](https://ruslanspivak.com/lsbasi-part1/lsbasi_part1_lexer1.png)

然后pos现在指向了“+”，下一次调用这个方法，他检测了字符是否为数字，然后检测字符是否是加号，他的确是。
最后返回了一个新的“+”值的PLUS类型的Token，然后pos递增。

![](https://ruslanspivak.com/lsbasi-part1/lsbasi_part1_lexer2.png)

同理（译者不想翻译这段了）

![](https://ruslanspivak.com/lsbasi-part1/lsbasi_part1_lexer3.png)

因为pos现在已经指过了“3+5”，之后每次调用`getNextToken`时会返回EOF

![](https://ruslanspivak.com/lsbasi-part1/lsbasi_part1_lexer4.png)

既然你的解释器可以访问由输入字符组成的Token流，那么解释器还需要对它做一点事情：需要在lexer`getNextToken`的Token流中找到结构。

至少您希望解释器找到如下结构

`INTEGER -> PLUS -> INTEGER`

也就是说，他试图找到一些列Token：整数后面跟一个加号，紧接着一个整数

赋值查找和解释该结构的方法应该是`expr`。这个方法验证了Token序列确实是想象中的序列，即`INTEGER -> PLUS -> INTEGER`。
在成功确认结构后，它通过将`+`左右两边的Token相加得到结果。从而成功的传递给解释器结果。

`expr` 方法本身使用方法`eat`来验证传递给其的参数与当前Token是否一致。

匹配传递的令牌类型后，`eat`方法获取下一个Token并将其赋值给`currentToken`，从而有效的“吃掉”当前匹配的Token并且推进了Token流中的虚拟指针（pos）。如果不对应，`eat`方法会抛出异常。

让我们回顾一下您的解释器为计算一个算术表达式所作的事情：

- 解释器接受一个输入的字符串，比如“3+5”

- 解释器调用了`expr`方法在词法分析器`getNextToken`来找到Token流中的结构。他试图找到INTEGER、PLUS、INTEGER的形式。确认结构后，它相加两个INTEGER的Token的值来解释输入，因为这时解释器清楚它需要做的就是将3和5相加。

祝贺自己。你刚刚学会了如何构建您的第一个解释器！

---

现在是练习的时候了。

![](https://ruslanspivak.com/lsbasi-part1/lsbasi_exercises2.png)

你不会认为读这篇文章就够了吧？对吧？

接下来是一些练习

1. 修改代码来允许输入多位整数，例如“12+3”

2. 添加一个跳过空格的办法，让您的计算器可以处理带有空格的字符串输入，比如“   12 + 3”

3. 支持“-”运算符，比如“7-5”这样的减法。

检查你的理解：

1. 什么是解释器？

2. 什么是编译器？

3. 解释器和编译器的区别？

4. Token是什么？

5. 将输入分解为Token的过程叫什么？

6. 词法分析所称的解释器部分叫啥？

7. 解释器或者编译器的那一部分的其他常用名字是什么？

---

（跳过了一大堆废话翻译）

下一章是扩展计算器已经更多的算术表达式。敬请关注

---

推荐阅读（如果你等不到第二篇）

[Language Implementation Patterns: Create Your Own Domain-Specific and General Programming Languages (Pragmatic Programmers)](http://www.amazon.com/gp/product/193435645X/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=193435645X&linkCode=as2&tag=russblo0b-20&linkId=MP4DCXDV6DJMEJBL)

[Writing Compilers and Interpreters: A Software Engineering Approach](http://www.amazon.com/gp/product/0470177071/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0470177071&linkCode=as2&tag=russblo0b-20&linkId=UCLGQTPIYSWYKRRM)

[Modern Compiler Implementation in Java](http://www.amazon.com/gp/product/052182060X/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=052182060X&linkCode=as2&tag=russblo0b-20&linkId=ZSKKZMV7YWR22NMW)

[Modern Compiler Design](http://www.amazon.com/gp/product/1461446988/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1461446988&linkCode=as2&tag=russblo0b-20&linkId=PAXWJP5WCPZ7RKRD)

[Compilers: Principles, Techniques, and Tools (2nd Edition)](http://www.amazon.com/gp/product/0321486811/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0321486811&linkCode=as2&tag=russblo0b-20&linkId=GOEGDQG4HIHU56FQ)

---

原文地址：https://ruslanspivak.com/lsbasi-part1