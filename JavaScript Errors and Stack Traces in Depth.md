---
title: 【翻译】深入了解JavaScript错误和堆栈追踪
date: 2017-03-06
tags: ["JavaScript", "translation"]
---

原文地址：http://lucasfcosta.com/2017/02/17/JavaScript-Errors-and-Stack-Traces.html

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
    at repl:1:1 // <-- 这个指针下面的东西都是Nodejs的内部实现，无视就好
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
-  `message` - 一条错误信息。
-  `name` - 错误的名称

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
        throw new Error('Nested error.'); // 这里抛出的错误会被他自身的catch子句所捕获
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

同样的，当抛出值不是 `Error`  对象时，你不需要去访问其他重要的数据，比如它的`stack`，一个在一些环境中 `Error` 对象所包含的字段。

错误同样可以被当作其他（一般）的对象来使用，你并不一定要把他们抛出。这就是为什么它们经常被当初回调函数的第一个参数的原因。比如，在 `fs.readdir` 方法种：

```javascript
const fs = require('fs');

fs.readdir('/example/i-do-not-exist', function callback(err, dirs) {
  if (err instanceof Error) {
    // `readdir` 将会抛出一个错误，因为这个文件根本不存在    
    // 现在我们能够使用回调函数中的错误对象了
    console.log('Error Message:' + err.message);
    console.log('See? We can use Errors without using try statements.');
  } else {
    console.log(dirs);
  }
})
```

最后但并非不重要， `Error` 对象在 promise  reject 时被使用。这使得控制promise的rejections变得容易：

```javascript
new Promise(function(resolve, reject) {
  reject(new Error('The promise was rejected.'));
}).then(function() {
  console.log('I am an error.');
}).catch(function(err) {
  if (err instanceof Error) {
    console.log('The promise was rejected with an error.');
    console.log('Error Message:' + err.message);
  }
})
```

### 操作堆栈追踪

现在就是你所期待的部分了：如何去操作堆栈追踪信息。

这个章节只针对一些支持 `Error.captureStackTrace` 的特殊环境，比如 NodeJS。

这个 `Error.captureStackTrace` 方法将一个 `object` 作为它的一个参数，一个可选的 `function` 作为它的第二个参数。这个 captureStackTrace 做的呢就是捕获当前的堆栈信息（废话）并且在一个大的对象中创建一个 `stack` 参数来保存它。如果提供了第二个参数，这个被传递的方法将会被认为是调用堆栈的重点。因此堆栈跟踪将仅显示在调用此函数之前发生的调用。

让我们给一些例子来让这一切变得更清晰。首先，我们将会捕获当前的堆栈信息，并且将它保存在一个普通的对象中。

```javascript
const myObj = {};

function c() {
  
}

function b() { 
  // 这里将会讲当前的堆栈信息储存到 myObj 中
  Error.captureStackTrace(myObj);
  c();
}

function a() {
  b();
}

// 首先我们会调用这些方法
a();

// 现在让我们看看什么堆栈信息被存入了 myObj.stack
console.log(myObj.stack);

// 这将会在控制台中打印出如下信息：
//    at b (repl:3:7) <-- 因为它在B内被调用，所以B是堆栈中的最后一个条目
//    at a (repl:2:1)
//    at repl:1:1 <-- 下面是 node 的内部实现
//    at realRunInThisContextScript (vm.js:22:35)
//    at sigintHandlersWrap (vm.js:98:12)
//    at ContextifyScript.Script.runInThisContext (vm.js:24:12)
//    at REPLServer.defaultEval (repl.js:313:29)
//    at bound (domain.js:280:14)
//    at REPLServer.runBound [as eval] (domain.js:293:12)
//    at REPLServer.onLine (repl.js:513:10)
```

正如你在上述例子中看到的，我们首先调用了 `a` （被压入了栈内）然后在 `a` 内调用了 `b` （被 push 在 `a` 上面）。然后，在 `b` 内，我们捕获到了当前的堆栈信息，并且存入了 `myObj`。 这就是为什么我们在控制台中只获得了 `a` 和 `b`。 

现在，让我们传递一个方法作为第二个参数给`Error.captureStackTrace` 方法，来看会发生什么：

```javascript
const myObj = {};

function d() {
  // 这里我们将会储存当前的堆栈信息到 myObj 中
  // 这一次我么将会隐藏 `b` 之后以及它自身的栈帧 
  Error.captureStackTrace(myObj, b);
}

function c() {
  d(); 
}

function b() {
  c();  
}

function a() {
  b();
}

// 首先我们会调用这些方法
a();

// 现在让我们看看什么堆栈信息被存入了 myObj.stack
console.log(myObj.stack);

// 这将会在控制台中打印出如下信息：
//    at a (repl:2:1) <-- 如你所见在这里我们只能获得 `b` 之前的被调用的栈帧
//    at repl:1:1 <-- 下面是 node 的内部实现
//    at realRunInThisContextScript (vm.js:22:35)
//    at sigintHandlersWrap (vm.js:98:12)
//    at ContextifyScript.Script.runInThisContext (vm.js:24:12)
//    at REPLServer.defaultEval (repl.js:313:29)
//    at bound (domain.js:280:14)
//    at REPLServer.runBound [as eval] (domain.js:293:12)
//    at REPLServer.onLine (repl.js:513:10)
//    at emitOne (events.js:101:20)
```

当我们传递 `b` 给`Error.captureStackTrace` 函数时，它隐藏了 `b` 本身以及在它之上的所有栈帧。这就是为什么我们在堆栈追踪中只看到了`a`。

现在你或许会问你自己： “为什么这东西有用？”。这个东西在当你试图对非你的用户隐藏内部实现细节时非常有用。在 Chai 内，举个例子， 们使用它来避免向我们的用户显示与我们实现检查和断言自身的方式无关的细节。

### 真实环境中的堆栈追踪操作

正如我在上一个小节提到的，Chai 使用堆栈操作技术来使得堆栈追踪与我们的用户（的操作）更加关联。下面是我们如何做的。

首先，让我们看一看当断言失败时， `AssertionError` 构造函数会抛出什么：

```javascript
// `ssfi` 代表 “start stack function”. 它指向堆栈追踪中删除不相关帧的起点
function AssertionError (message, _props, ssf) {
  var extend = exclude('name', 'message', 'stack', 'constructor', 'toJSON')
    , props = extend(_props || {});
  
  // 默认值
  this.message = message || 'Unspecified AssertionError';
  this.showDiff = false;
  
  // 从参数中拷贝
  for (var key in props) {
    this[key] = props[key];
  }
  
  // 这里就是与我们相关的部分：
  // 如果一个start stack function 被提供了，我们捕获了当前堆栈的追踪信息，并且将其传递给了 `captureStackTrace` 方法，那样我们移除在这个之后的栈帧了。
  ssf = ssf || arguments.callee;
  if (ssf && Error.captureStackTrace) {
    Error.captureStackTrace(this, ssf);
  } else {
    // 如果没有提供 start stack function 我们就用原来的 stack 属性。
    try {
      throw new Error();
    } catch(e) {
      this.stack = e.stack;
    }
  }
}
```

如你所见，在上面的代码中我们使用 `Error.captureStackTrace` 来捕获堆栈信息，并且将其储存在我们所生成的 `AssertionError` 实例中，（当它存在时）我们传递了一个 start stack function 给它来将不相干的栈帧从栈列内移除。这些仅仅展示了Chai的内部实现细节并且在最后污染了栈列。

现在让我们看看现在由 [@meeber](https://github.com/meeber)在 [这个碉堡的PR内](https://github.com/chaijs/chai/pull/922)的代码是怎么写的.

在我们看下面的代码之前，我必须告诉你 `addChainableMethod` 方法做了什么。它将传递给它的可链接方法添加到断言，并且还使用包含断言的方法标记断言本身。它以 `ssfi` 作为名称保存（代表了起始栈方法指示器）。这基本上意味着当前断言将是堆栈中的最后一帧，因此我们不会在堆栈中显示Chai中的任何进一步的内部方法。我避免添加整个代码，因为它有很多东西，而且有点棘手，但如果你想读它，[这里是它的链接](https://github.com/meeber/chai/blob/42ff3c012b8a5978e7381b17d712521299ced341/lib/chai/utils/addChainableMethod.js).。

在下面的代码中，我们有一个 `lengthOf` 断言的逻辑，它检查对象是否具有一个明确的 `长度`。我们希望我们的用户像这么用它：`expect(['foo', 'bar']).to.have.lengthOf(2)`

```javascript
function assertLength (n, msg) {
    if (msg) flag(this, 'message', msg);
    var obj = flag(this, 'object')
        , ssfi = flag(this, 'ssfi');

    // 注意这一行
    new Assertion(obj, msg, ssfi, true).to.have.property('length');
    var len = obj.length;

    // 这一行也同样相关
    this.assert(
            len == n
        , 'expected #{this} to have a length of #{exp} but got #{act}'
        , 'expected #{this} to not have a length of #{act}'
        , n
        , len
    );
}

Assertion.addChainableMethod('lengthOf', assertLength, assertLengthChain);
```

在上面的代码中，我突出强调了与我们现在相关的代码段。我们先来看看 `this.assert` 的调用。

下面是 `this.assert` 方法的代码：

```javascript
Assertion.prototype.assert = function (expr, msg, negateMsg, expected, _actual, showDiff) {
    var ok = util.test(this, arguments);
    if (false !== showDiff) showDiff = true;
    if (undefined === expected && undefined === _actual) showDiff = false;
    if (true !== config.showDiff) showDiff = false;

    if (!ok) {
        msg = util.getMessage(this, arguments);
        var actual = util.getActual(this, arguments);

        // 这里是我们所要关注的行
        throw new AssertionError(msg, {
                actual: actual
            , expected: expected
            , showDiff: showDiff
        }, (config.includeStack) ? this.assert : flag(this, 'ssfi'));
    }
};
```

基本上，`assert`方法负责检查是否通过了布尔表达式的断言。如果没有，我们必须实例化一个`AssertionError`。请注意，当实例化这个新的`AssertionError`时，我们也向其传递一个堆栈跟踪功能指示符（`ssfi`）。如果配置标志`includeStack`被打开，我们通过将`this.assert`本身传递给它来显示整个堆栈跟踪，这真的是堆栈中的最后一帧。但是，如果`includeStack`配置标志被启用，我们必须从堆栈跟踪中隐藏更多的内部实现细节，所以我们使用什么存储到`ssfi`标志。

现在，我们来谈谈另一个相关的行：

```javascript
new Assertion(obj, msg, ssfi, true).to.have.property('length');
```

正如你可以看到的，我们在创建我们的嵌套断言时传递了我们从`ssfi`标志获得的内容。这意味着当创建新的断言时，它将使用此函数作为从堆栈跟踪中删除无用框架的起点。顺便说一下，这是`Assertion`构造函数：

```javascript
function Assertion (obj, msg, ssfi, lockSsfi) {
    // This is the line that matters to us
    flag(this, 'ssfi', ssfi || Assertion);
    flag(this, 'lockSsfi', lockSsfi);
    flag(this, 'object', obj);
    flag(this, 'message', msg);

    return util.proxify(this);
}
```

你可以记住从我对`addChainableMethod`的说法，它设置`ssfi`标志与自己的包装方法，这意味着这是堆栈跟踪中最低的内部帧，所以我们可以删除所有上面的帧。

通过将`ssfi`传递给嵌套断言，它只检查我们的对象是否具有属性长度，我们避免重置我们将用作起点指示符的帧，然后在堆栈中使得之前的`addChainableMethod`保持可见。

这可能看起来有点复杂，所以让我们回顾一下Chai发生的事情，我们想从堆栈中删除无用的帧：

1. 当我们运行断言时，我们设置自己的方法作为删除堆栈中的下一个帧的参考
2. 断言运行，如果它失败，我们删除我们存储的引用后的所有内部帧
3. 如果我们有嵌套断言，我们仍然必须使用当前的断言包装方法作为删除堆栈中的下一个帧的参考点，所以我们将当前的`ssfi`（启动堆栈函数指示符）传递给我们正在创建的断言，以便它可以保留它