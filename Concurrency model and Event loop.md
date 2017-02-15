---
title: 【翻译】并发模型和事件循环
date: 2017-01-31 12:21:33
tags: ["JavaScript", "translation"]
---
> 原址：[https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)
>

JavaScript 有一个基于“事件循环”的并发模型。这个模型和其他语言中（C 和 JAVA）中的近似模型有着很大区别。

## 运行时的概念

下面的内容描述了一个理论模型。现在 JavaScript 引擎大量的声明和优化了上述语义。

### 视觉展示

![Stack, heap, queue](https://developer.mozilla.org/files/4617/default.svg)

### 栈

函数的调用形成有一系列帧组成的栈。

```javascript
function foo(b) {
  var a = 10;
  return a + b + 11;
}

function bar(x) {
  var y = 3;
  return foo(x * y);
}

console.log(bar(7));
```

当调用 bar 时，第一帧被产生用以保存 bar 函数的引用参数和局部变量。当 bar 调用 foo 时，第二帧被生成用以保存 foo 的引用参数和局部变量并入栈。当 foo 返回时，栈顶部的元素被抛出。当 bar 执行返回时，栈被清空。

### 堆

对象被分配在一个用于表示一个泛广的非结构化内存区域的堆中。

### 队列

JavaScript 在运行时会产生一个由一系列等待被处理的消息组成的消息队列。每个消息都与一个方法相关联。当栈有足够容量时，一条消息会被拿出队列并被处理。这处理的过程包含调用关联方法（因此会生成一个初始化帧栈）。当栈被重新执行清空后，消息处理结束。

## 事件循环

事件循环之所以得到这么个名字是因为它通常的实现如下：

```javascript
while (queue.waitForMessage()) {
  queue.processNextMessage();
}
```

`queue.waitForMessage` 同步的等待消息到来，如果当前还没有的话。

### 从运行到完成

每一条消息都在另一条消息被处理前完成。这提供了一些不错的特性在对你的程序追责时，它记录了函数运行的状态，函数执行时总会在其他代码捷足先登前完全的被执行，（并可以修改函数操作时的数据）。与 C 不同的是，比如（以下为 C 的情况）， 如果一个方法在一个线程上跑的时候，它可以在任何时候被中断，并且在另一个线程上跑一些其它的代码。

这种模式的缺点是，如果一条消息花费了很长的时间去完成，那么web应用将无法处理用户的交互，比如点击和滚动。浏览器通过一个“一个脚本执行了太长时间”的弹窗来缓解问题。   一个好的做法是将消息变短，如果可能的话，将消息切割成一系列短的消息。

### 添加消息

在web浏览器中，任何时候有事件触发，通过绑定的消息监听器都会触发消息的添加。如果没有监听器的话，事件就会丢失。所以在一个元素上点击会触发一个点击事件并添加消息，相似的还有其他的事件。

调用`setTimeout`会在参数所给的时间后，将消息添加至队列内。如果此时队列内没有其他的消息了，那么这条消息将会被马上执行。然而，如果有其他消息，这个定时消息则不得不等到其他消息都处理完后才被处理。由于这个原因，函数的第二个参数所指定的时间是最快时间而不是确定的时间。

### 0延时

0延时并不意味着回调会在0毫秒后被调用。调用`setTimeout`函数并设置 0 毫秒的延时并不是在指定时间后执行。这个执行取决于当前队列内有多少正在等待的任务。在下方的例子中，“this is just a message”会在回调函数中的那条消息前辈打印出来，因为这个延时是最小时间而不会确定时间。

```javascript
(function() {

  console.log('this is the start');

  setTimeout(function cb() {
    console.log('this is a msg from call back');
  });

  console.log('this is just a message');

  setTimeout(function cb1() {
    console.log('this is a msg from call back1');
  }, 0);

  console.log('this is the end');

})();

// "this is the start"
// "this is just a message"
// "this is the end"
// "this is a msg from call back"
// "this is a msg from call back1"
```

### 运行时的协作通信

一个 web worker 或一个跨源的帧含有他自己的栈，堆还有消息队列。两个不同线程在运行时可以通过postMessage 方法来进行通信。如果后者监听消息事件，此方法会向前者（队列内）添加消息。

## 永不阻塞

事件循环模型的一个非常有趣的属性是，JavaScript（的时间循环模型），不像许多其他语言，永远不会阻塞。处理I / O通常通过事件和回调来执行，因此当应用程序正在等待 IndexedDB 查询返回或 XHR 请求返回时，它仍然可以处理其他事情，如用户输入。

一些遗留的异常情况，如使用 alert 或同步 XHR时 ，但它们被认为是避免交互终端的良好做法。注意，异常的异常确实存在（但通常是出现错误，而不是其他任何东西）。