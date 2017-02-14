---
title: 【翻译】React Fiber架构
date: 2017-02-13 12:21:33
tags: ["react", "translation"]
---

原文链接: [https://github.com/acdlite/react-fiber-architecture](https://github.com/acdlite/react-fiber-architecture)

## 引言

React Fiber 是一个正在重新实现的React核心算法，它是过去两年react团队的研究的顶峰。

React Fiber 的目标是为了增强react在动画，布局和手势等领域的适应性，它的头号特性就是增量渲染：一种将渲染任务切割成多个小块并分布到复数个帧中。

其他的关键特性包括，在新的更新来临时暂停，退出和复用任务，为不同类型的更新设置优先级，和新的并发原函数。

### 关于本文档

Fiber 引入了几个新的概念，它们并不能简单的通过代码来理解。这篇文档最初是我在跟进react项目的Fiber实现时整理的一些笔记引入，随着积累，我意识到它对其他人或许会是有用的资源。

我尝试着用简单易懂的语言来讲述，并通过界定术语来避免行话。如果可能的话，我也会引用大量的外部链接。

请注意我并不在react团队，而且并不能待变官方发言。这并不是一片官方文档，不过我请求了react团队的一些人来验证了文章的准确性。

要记住，他是一个正在进行中的工作。Fiber是一个进行中的项目，在它完成之前可能会颠覆性的重构。同样的进行中也是我为这篇文档定义的理念。任何改进或者建议都非常欢迎。

我的目标是在你读完了这篇文档后，你会对Fiber足够认知去理解[它的实现](https://github.com/facebook/react/commits/master/src/renderers/shared/fiber)，并且最终能够反哺给react。

### 必要的知识

我强烈建议你先熟悉下面的资源来继续你的阅读：

- [React Components, Elements, and Instances](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) - “组件”是一个经常提起的术语，对其有一个坚实的认知是至关重要的。
- [Reconciliation](https://facebook.github.io/react/docs/reconciliation.html) - React 协调算法的高阶阐述
- [React Basic Theoretical Concepts](https://github.com/reactjs/react-basic) - 不带实现细节的react概念模型的说明。部分内容或许不会再阅读的初期产生意义。但是没关系，它会随着进展而变得更有价值。
- [React Design Principles](https://facebook.github.io/react/contributing/design-principles.html) - 特别注意文中的“调度“部分，它花了很大的功夫来阐述为什么要有React Fiber。

## 回顾

请在继续前确认你以对上述知识有所了解。

在我们深入到新的内容前，让我们先回顾一些观念。

### 什么是协调算法？

#### 协调算法

​	一个在react中比较一棵树与另一棵树来找出那一部分需要被改变的算法。

#### 更新

​	选人React应用时发生的数据变化。通常是“setState”引发的。最终的结果就是重新渲染。

React API 的核心思想是让更新能够触发整个应用的重新渲染。它允许开发者进行声明式的开发，而不用担心应用在一个状态转移到另一个状态时的效率（A到B， B到C， C到A，等等）。

事实上，重新渲染整个应用只适用于一些小型的应用。在现实中的应用，（重新渲染）代价是高昂的。React在这基础上做了一系列优化，使其在对整个页面重新渲染时依然能够保证很好的性能。这些优化都是 **协调算法** 的一部分。

协调选法是一种隐藏在一个广泛认知的概念“虚拟DOM”之下的算法。一个高阶的阐述如下：当你渲染一个react应用时，一棵用于描述整个应用的树被生成并保存在内存中。然后这棵树被刷新到正式的渲染环境中 - 举个例子，在浏览器应用中，它被转成一系列的DOM操作。当应用更新时（通常是通过 setState），一棵新的树被生成。这棵新的树与之前的树的差异决定了使用哪些更新操作来重新渲染应用。

尽管Fiber是一个协调算法的推倒重写，但是它与[React官方文档中描述的高阶算法](https://facebook.github.io/react/docs/reconciliation.html)大致相同。其关键点在于：

- 不同的组件类型被用以生成不同的树。比起比较两者的区别，react采用了替换掉整个旧树。
- 通过key来进行diff比较。key必须“稳定，可预测且独一无二”。

### 协调算法vs渲染

DOM只是React可以采用的一种渲染环境，其他的主要环境还有原生的IOS 和 android 视图，通过React Native.（这既是为什么“虚拟DOM”其实有一点用词不当）。

React支持那么多渲染环境的原因是React在设计时就将协调算法和渲染拆分成了不同的部分。协调算法用以计算一棵树的哪些部分被改变了。渲染器使用这些信息来实际更新应用。

这个分离意味着 React DOM 和 React Native 能使用它们各自的渲染器的前提下共享React核心提供的协调算法，

Fiber重新实现了协调器。渲染并不是Fiber需要考虑的，但是渲染器需要调整以适应新的架构。

### 调度

#### 调度

​	决定任务什么时候被执行的过程

#### 任务

​	任何计算必须被执行。"任务"通常是更新的结果（如 `setState`）

React 的[设计原则](https://facebook.github.io/react/contributing/design-principles.html#scheduling)中对于这部分讲的十分不错，我贴在这里了：

> 在当前的实现中，react在一个tick中历遍并调用了整棵树的渲染函数。但在未来，它有可能会延迟部分更新来避免丢帧。
>
> 这是react设计的一个常见的主题。一些流行的库实现了一种”push“的方法，当新的数据准备好时触发执行运算。然而，React依然使用了"pull"的方式，计算可以被延迟到必须执行的时候。
>
> React不是一个通用的数据处理库，而是一个用户构建用户界面应用的库。我们认为（在用户界面应用中）知道那些东西该立即关联，哪些则不必是有着独一无二的地位的。
>
> 如果有些东西超出了屏幕，我们可以延迟相关逻辑的执行。如果数据来的比帧绘制快，我们可以合并数据并批量更新。我们可以优先处理来自用户的交互（比如按钮点击出发的动画），而那些不是非常重要的后台任务（比如渲染来自网络的新加载的内容）来避免掉帧。

关键的点如下：

- 在一个UI界面中，不是每一次更新都有必要立即执行。事实上，这么做很浪费资源，而且会导致丢帧和降低用户体现。
- 不同类型的更新有着不同的优先级- 一个动画的更新需要在数据源的更新前完成。
- 一个基于push的方案需要应用（你，敲代码的）来决定如何调度任务。但是一个基于pull的方案允许框架（react）更为智能的来为你做这些决定。

当前React 并没有明显的利用调度： 一个更新会导致整棵树被立即重绘。利用调度来重写React核心算法是Fiber背后的驱动理念。

---

现在我们已经准备好深入Fiber的实现了。下一章节将比我们讨论到现在所讲的内容更加具有技术性。在继续前，请确认你已经理解上面的材料。

## 什么是Fiber？

我们将讨论React Fiber架构的核心。纤维（Fibers）是一种比应用开发者想象中还要低阶的抽象。如果你在尝试去理解时出现了困惑，不要灰心。继续下去，最终你会明白的。（当你最终明白了，请提一些意见来优化这个章节）。

现在让我们开始！

---

利用调度是React Fiber的一个即认目标。具体来说，我们需要有能力做到：

- 暂停任务，并在之后恢复。
- 为不同类型的任务指派优先级。
- 复用之前完成了的任务。
- 在任务不再需要时放弃任务。

为了做到当中的任意一点，我们首先需要一个方法将任务拆分成单元。从某种第一种，这就是Fiber。一个纤维待变了任务的一个单元。

为了跟进一部，让我们回到[React组件是一个包含数据方法](https://github.com/reactjs/react-basic#transformation)的概念，通常表示为

```javascript
v = f(d)
```

它遵循以下规则，渲染一个React应用类似于调用一个主体包含其它函数的函数。这个比喻在我们思考fiber时很有用。

计算机通常跟踪程序执行的方式是使用[调用堆栈](https://en.wikipedia.org/wiki/Call_stack)。当一个方法被执行时，一个新的栈帧被添加到栈顶。这个栈帧表示这个任务由这个函数执行。

当处理UI时，这个问题是如果一次性执行了太多的任务，会导致动画掉帧和页面卡顿。更糟的是，当中的一些工作最终会被跟近的更新所替代，完全不是必须的。这就是UI组件和普通方法的分界线，因为组件比一般方法有更详细的关注点。

较新的浏览器（和React Native）实现接口来解决这个问题。`requestIdleCallback` 调度了一个更低优先级的方法在空闲时调用，而 `requestAnimationFrame` 调度了一个更高优先级的方法在下一个动画帧执行。问题在于，为了使用这些接口，你需要一个方法去把任务切分为增量的任务。如果你依赖于调用栈，它将会继续执行执行栈被清空。

如果我们能自定义调用栈的行为来优化UI渲染会不会更好？如果我们可以随意中断调用栈并且可以手动调控栈帧会不会更好？

这就是React Fiber的设计动机。Fiber是栈的重新实现，特针对于React组件。你可以将一个fiber想象成一个虚拟栈帧。

重新实现堆栈的优点是，你可以[保持堆栈帧在内存中](https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/)，并执行它们（和任何时候）你想要的。这对于实现我们的计划目标至关重要。

除了调度，手动处理堆栈帧解锁了诸如并发和错误边界之类的功能的潜力。我们将在以后的章节中讨论这些主题。

在下一节中，我们将更多地了解fiber的结构。

### fiber的结构

*注意：当我们更具体地了解实现细节时，一些东西可能会已经随着时间被改变了。如果您发现任何错误或过时的信息，请提交公关。*

具体来说，光纤是一个JavaScript对象，它包括组件本身以及其输入及其输出的信息。

Fiber对应一个栈帧，但是同样也对应一个组件的实例。

下面是一些属于fiber的重要字段。（这个列表并不完整）。

#### `type`和`key`

fiber的`type`和`key`的作用和React元素一样。（实际上，一个fiber从组件创建时，这两个字段会直接复制过来）

fiber的 `type`字段描述了它对应的组件。对复合组件这个类型就是函数组件或类组件本身。对于原生组件(div, span,等等)，这个字段就是一个字符串。

概念上，type是在执行时会被堆栈帧跟踪到的函数（如在v = f（d）中）。

除了`type`之外，`key`是在协调算法中用来决定fiber是否可以重用的字段。

#### `child`和`sibling`

这些字段指向别的fiber，描述了fiber的历遍树的结构。

这子fiber指的是组件render方法的返回值。所以，在下面的例子中

```javascript
function Parent() {
  return <Child />
}
```

Parent 对应的子Fiber就是Child组件。

`sibling`字段指代着render方法返回多个子元素的情况（fiber的一个新的特性！）:

```javascript
function Parent() {
  return [<Child1 />, <Child2 />]
}
```

子fiber组成了一个首元素是第一个子元素的单向链表。所以在例子中，Parent的子元素是Child1，child1的兄弟元素是Child2。

回顾我们之前的函数类比，你可以把子fiber当作一个[尾调用函数](https://en.wikipedia.org/wiki/Tail_call)。

#### `return `

返回fiber是程序处理完当前fiber时返回的fiber。概念上来说，它和栈帧返回的地址相同。它同样可以被认为是父fiber。

如果一个fiber包含多个子fiber，每一个子fiber的return fiber 都是它的父fiber。所以在我们先前一节的例子中，Child1和Child2的return fiber 就是Parent。

#### `pendingProps`和`memoizedProps`

概念上来说，props是函数的参数。一个Fiber的 `pendingProps`在他执行前就被设定好，而`memoizedProps`则会在尾部被设置。

当传入的`pendingProps`和`memoizedProps`相同时，这传递了一个信号这个fiber的的输出可以被复用，从而避免了不必要的工作。

#### `pendingWorkPriority`

一个数字代表了fiber的执行优先级。 [React优先级](https://github.com/facebook/react/blob/master/src/renderers/shared/fiber/ReactPriorityLevel.js) 模块列举了不同的优先级和它们代表的意义。

除了例外的`NoWork`的值是0，值越大代表优先级越低。比如，你可以使用下述的方法来确认一个fiber的优先级是不是不低于给定的等级：

```javascript
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```

这个方法仅供说明用，它并不是React Fiber源码的一部分。

调度器使用优先级字段来搜索要执行的下一个工作单元。这个算法将在以后的章节中讨论。

#### `alternate`

##### flush

​	flush 一个fiber意味着将它打印输出到屏幕上。

##### work-in-progress

​	一个fiber如果还没有完成，那么概念上，栈帧就尚未返回。

在任何时候，一个组件的实例，最多有两个fiber关联： 当前的fiber，flush fiber，和work-in-progress fiber。

当前fiber的替代(`alternate`)就是进行中的fiber，进行中的fiber的替代就是当前fiber。

一个fiber的替代是通过`cloneFiber`函数懒创建的。`cloneFiber`会尝试重用fiber的替代（如果存在）来最小化分配空间，而不是总创建新的对象。

你应当把`alternate`当做是一个实现细节，但它在代码里面出现了很多次，所以值得在这里讨论。

#### `output`

##### *host component*

​	React应用的叶子节点。它们是跟特定的渲染环境相关的（比如，在浏览器应用中，宿主组件是指`div`, `span`等）。在JSX中，它们是用小写字母的tag名称表示的。

从概念上讲，fiber的output是一个函数的返回值。

每个fiber最终都会有output，不过output只在宿主组件的叶子节点上创建。这个输出会向上转移到整棵树。

output最终会递交给渲染器让其根据渲染环境来flush。定义输出结果怎么样创建和更新就是渲染器的职责了。

## 未来的章节

目前为止就这些了，不过这篇文档远远没有完成，未来的章节将会描述更新周期时使用到的算法。相关的标题包含如下：

- 调度器如何找到下一个需要被执行的单元
- 如何在整棵fiber树上最终和传递优先级
- 调度器怎么知道神恶魔时候去暂停和恢复任务
- 任务如何被冲洗并标记成完成
- 副作用（如生命周期函数）是如何工作的
- 什么是协同程序，以及如何使用它来实现上下文和布局等功能。

## 相关视频

- [What's Next for React (ReactNext 2016)](https://youtu.be/aV1271hd9ew)