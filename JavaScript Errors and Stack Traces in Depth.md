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

### 调用堆栈式如何工作的



