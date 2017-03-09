---
title: 【翻译】深入了解JavaScript错误和堆栈追踪
date: 2017-03-06
tags: ["JavaScript", "translation"]
---

原文地址：http://lucasfcosta.com/2017/02/17/JavaScript-Errors-and-Stack-Traces.html?utm_source=javascriptweekly&utm_medium=email

(｡･∀･)ﾉﾞ嗨，大家好！鉴于我几个星期没有写些什么关于JavaScript的东西了，是时候让我们回到正轨了。

这一次，我们将会来探讨一下 errors 和 stack traces，并且熟练的掌握它们。

有些时候人们的确不太注意这些细节，但是这些细节知识在当你写一个库，并且需要测试和调错时会非常有用。举个例子，这周在 Chai 时，我们有一个很棒的pull-request，关于如何提升我们在堆栈追踪的处理能力上，从而能够使我们的用户能够在 assert 测试失败时，能够获得更多的信息。

熟练的操控堆栈追踪能偶让你清理掉一些不必要的干扰信息，从而能够关注于真正的问题上。此外，当你理解什么是错误及其属性，你会感到更有信心利用它。

**这篇博文在开头可能看起来太浅显了，但是当我们开始操作堆栈追踪时，它变得相当复杂，因此在我们进入那个章节之前，请确保您对以前的内容有了很好的理解。**

### 调用堆栈是如何工作的

在我们讨论errors之前，我们必须理解调用堆栈是如何工作的。（的确）这很单调，不过在深入之前理解这些是很有必要的。如果你已经知道了这些，请随意跳过这节。

**当一个方法被调用时，它会被push到栈顶。在它执行完成后，它会从栈顶被移除。**

这种数据结构有趣的地方在于 **最后进来的元素会最先出去**。同样这被称作 LIFO (后入先出) 原则。

给你看另一个例子，假设你有如下代码：

```javascript
function c() {
    console.log('c');
}

function b() {
    console.log('b');
    c();
}

function a() {
    console.log('a');
    b();
}

a();
```

在上面的例子中，当运行方法a时，它被添加到我们栈的顶部。然后，当方法 b 在方法 a 内被调用时，它也被添加到了栈顶。同样的事也发生在方法 c 在方法 b 内被调用时。

当运行方法 c 时，我们的堆栈追踪内顺序包含 a, b, c 三个方法。

一旦方法 c 结束运行，它从栈顶被移除，控制权重新交回给方法 b 。当方法 b 完成时，它也从栈顶被移除，现在控制权被交回到了方法 a 手中。最终，当方法 a 结束运行后，它同样也从栈顶被移除。

为了更好的演示这些行为，我们将会使用`console.trace()`方法。它能够在控制台种将当前的堆栈信息打印出来。同样，你应该从上到下来阅读这些信息。仔细想想下面每一行代码被调用时都发生了什么。

```javascript
function c() {
    console.log('c');
    console.trace();
}

function b() {
    console.log('b');
    c();
}

function a() {
    console.log('a');
    b();
}

a();
```

当代码在 node REPL 运行时，我们得到下面一些信息。

```javascript
Trace
    at c (repl:3:9)
    at b (repl:3:1)
    at a (repl:3:1)
    at repl:1:1 // <-- For now feel free to ignore anything below this point, these are Node's internals
    at realRunInThisContextScript (vm.js:22:35)
    at sigintHandlersWrap (vm.js:98:12)
    at ContextifyScript.Script.runInThisContext (vm.js:24:12)
    at REPLServer.defaultEval (repl.js:313:29)
    at bound (domain.js:280:14)
    at REPLServer.runBound [as eval] (domain.js:293:12)
```

简单的讲：你调用了一个东西，它被压入栈顶。当它完成了它就被弹出。就是这么简单。

### 错误对象和错误处理

当错误发生时，通常一个 `Error` 对象被抛出。`Error` 对象同样也被当作原型来使用，来拓展或创建自己的错误。

`Error.prototype` 对象通常包含下面属性：

-  `constructor` - 构造函数负责这个实例的原型。
- `message` - 一条错误信息。
- `name` - 错误的名称

上述这些是标准的属性，有些时候不同的环境会有它们自己特定参数。在一些环境下，比如 Node, Firefox, Chrome, Edge, IE 10+, Opera 和 Safari 6+，我们甚至会有 `stack` 参数，它包含了一个错误的堆栈追踪信息。

**一个错误的堆栈追踪信息包含所有到它自身的结构函数为止的栈帧信息**。

如果你希望了解更多的`Error`对象的参数，我非常推荐你去看看[MDN上的这篇文章](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/prototype).

为了抛出一个错误你必须使用`throw` 关键词。为了`catch` 一个被抛出的错误，你必须用`try catch`将那些可能会抛出错误的代码包裹起来。Catch 同样可以接收一个被抛出的错误作为参数。

如同在 java 种发生的一样， JavaScript 同样允许你在`try/catch`之后添加一个 `finally` 区块而不需要去关系 `try`区块内是否发生了错误。使用 `finally` 来做好一些善后工作，而不用关心你的操作是否正常工作。

到目前为止的所有东西对于大多数人而言都很基础，所有让我们来看一些不太注意的细节。（译者： indeed 😭）

你可以使用 `try`区块而不在后面带上 `catch`区块，但是这时必须带上 `finally`。这意味着你可以使用三种不同的try表达式结构：

- `try...catch`
- `try...finally`
- `try...catch...finally`

Try表达式能够签到在其他的 `try` 表达式内，比如：

```javascript
try {
    try {
        throw new Error('Nested error.'); // The error thrown here will be caught by its own `catch` clause
    } catch (nestedErr) {
        console.log('Nested catch'); // This runs
    }
} catch (err) {
    console.log('This will not run.');
}
```

你同样可以将 `try`嵌入 `catch` 和 `finally` 区块内：

```javascript
try {
    throw new Error('First error');
} catch (err) {
    console.log('First catch running');
    try {
        throw new Error('Second error');
    } catch (nestedErr) {
        console.log('Second catch running.');
    }
}
```

```
try {
    console.log('The try block is running...');
} finally {
    try {
        throw new Error('Error inside finally.');
    } catch (err) {
        console.log('Caught an error inside the finally block.');
    }
}
```

同样重要的是，**你要知道throw 同样可以抛出非 `Error` 对象**。尽管这看起来很cool，但是实际上真的不好，特别是那些需要在开发时使用其他库的开发者们，他们不得不去处理别人的代码，因为这之前并没有标准，你永远不会知道用户会给你什么东西。你不能信任他们而单纯的只是抛出一个`Error` 对象，因为他们可能选择不这么做，取而代之，而是抛出一个字符串或者数字。这使得你在处理堆栈追踪和其他一些有价值的元数据时变得困难。

假设你有如下代码：

```javascript
function runWithoutThrowing(func) {
    try {
        func();
    } catch (e) {
        console.log('There was an error, but I will not throw it.');
        console.log('The error\'s message was: ' + e.message)
    }
}

function funcThatThrowsError() {
    throw new TypeError('I am a TypeError.');
}

runWithoutThrowing(funcThatThrowsError);
```

当使用者传递一个含有错误抛出的方法到你的 `runWithoutThrowing`函数时，一切都正常工作。但是如果他们抛了一个 `String` 给你时，那你就有麻烦了：

```javascript
function runWithoutThrowing(func) {
    try {
        func();
    } catch (e) {
        console.log('There was an error, but I will not throw it.');
        console.log('The error\'s message was: ' + e.message)
    }
}

function funcThatThrowsString() {
    throw 'I am a String.';
}

runWithoutThrowing(funcThatThrowsString);
```

现在你的第二行 `console.log` 将告诉你 error 的 message 是 `undefined` 。这看起来在当前似乎不是很重要，不过如果你需要确认`Error` 对象内存在的一个特定的属性后者需要从用一种方法上处理 `Error ` 特定属性时（比如  [Chai’s`throws` 断言文档](https://github.com/chaijs/chai/blob/a7e1200db4c144263599e5dd7a3f7d1893467160/lib/chai/core/assertions.js#L1506)），你需要做更多的工作。

