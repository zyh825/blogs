---
title: 【翻译】展示组件和容器组件
date: 2016-10-03 12:21:33
tags: ["JavaScript", "translation"]
---

作者：Dan Abramov
链接：[https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)



当我在写react应用时有一种很简单却非常使用的模式。如果你已经写过一段实践的react，那么可能你已经发现了它。[这篇文章](https://medium.com/@learnreact/container-components-c0e67432e005#.hr23xqjul) 将这种模式解释的很好，不过我希望加入一些额外的要点。

你会发现当你把你的组件拆分成两类时，它们会变得更加易于复用和理解。我称呼他们为容器组件和展示组件，不过我同样也听到如下几种说法，胖组件(Fat)和瘦(Shinny)组件，聪明组件(Smart)和呆（Dumb）组件，多状态（Statefull）组件和纯(Pure)组件，放映和组件（Screens and Components）。它们不尽相同，但核心的思想却非常接近。

我所谓的展示组件：
- 考虑组件长啥样。
- 可能会同时包含展示组件和容器组件在其中，而且通常会含有写DOM标签和私有样式。
- 通常用this.props.children来包其他组件。
- 不依赖于应用内的其他部分，比如Flux的动作和存储。
- 不会定义数据是如何被加载或改变的。
- 仅通过props来获取数据和回调函数。
- 很少包含私有的状态(如果有的话，也只会是UI的状态而不是应用的数据)
- 一般用[方法组件](https://facebook.github.io/react/blog/2015/10/07/react-v0.14.html#stateless-functional-components) 的写法来写，除非它们需要状态，或者控制生命周期，或者优化性能。
- 举例： *Page, Siderbar, Strory, UserInfo, List*

我所谓的容器组件：

- 考虑组件是如何运作的。
- 可能会同时包含展示组件和容器组件在其中，出了部分包裹用的div不会拥有任何DOM标签，更绝不会有任何样式。
- 给展示组件或其他容器组件提供数据和动作。
- 调用flux动作，并为展示组件提供回调函数
- 通常会有很多状态，而且通常是应用服务的数据。
- 通常通过[高阶组件](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750#.tghakfcnl) 来生成，比如React Redux的connect()组件， Relay的createContainer()，Flux Utils的Container.create()，而不是手工的去写。
- 举例：*UserPage, FollowersSidebar, StoryContainer, FollowedUserList.*

我将它们放在不同的文件夹下，从而使它们的作用更加明确。

### 这种做法的好处

- 将专注的点分离，通过这种方式，你可以更好的理解你的应用和你的UI。
- 更好的复用。你可以使用相同的展示组件通过不同的状态源，也可以封装成容器组件，在未来复用它。
- 展示组件本质上是你的应用的“调色板”。你可以将他们放到一个单独的页面内，并且让设计师随意的来调整它们的样式，而不会碰触来应用的逻辑部分。你可以在哪个页面内进行screenshot regression测试。
- 这会强迫你去解析“布局组件”，比如Sidebar,Page,ContextMeanu 并且使用this.props.children来传递，而不是粘贴复制那块的jsx。

记住，组件不一定要生成DOM，它们只需要提供UI上的组合关系和界限。

### 什么时候去进去容器？

我建议你在构建你的应用时，先写展示组件的部分。最后，你会发现，你在中间组件中传递了太多的props。当你注意到部分组件并不需要它所接受的prop，而只是传递给它们的子组件，而当子组件需要更多数据，你不得不如重写所有这些中间组件，这时就是引入容器的最佳时机。这样做，你可以将数据和一些动作的props给余下的组件，而不必去在组件树中负担一些无关的组件。

这是一种循序渐进的不断重构的过程，所以没必要在一开始就做到位。当你不断地实现这种模式时，记得只要像你知道何时去增加新的方法一样来增加新的组件就可以了。我的免费的redux教程系列也许会对你有帮助。

### 其他的二分性

有很重要的一点你必须知道，容器组件和展示组件的区别并不是技术上的差别，而是两者在目的上的区别。为了比较，我再列举了一些有联系的（但是是不同的！）二分性。

- 多状体和无状态。一部分组件使用了React.setState()方法，而另一部分则没有。容器组件有很多状态，展示组件状态很少，没错，但这并不是一套铁规则。展示组件也可以有很多状态，同理，容器组件的状态也可以很少。
- 类和函数。从React0.14版本开始，组件可以用类和函数两种方式来生命了。函数组件更容易声明，不过它们缺少了一些只属于类的特性。这些限制或许再未来会消失，但至少现在是存在的。因为函数组件更易于理解，所以我建议你使用它们，除非你需要状态，再生命周期上添加处理，或是需要优化性能，这种时候只能使用类声明的组件了。
- 纯的和不纯的。人们说一个组件是纯的，给它相同的props和state时，返回的结果一定时相同的。纯组件可以被定义为类和函数，同时既可以时充满状态的也可以使无状态的。纯组件的另一个重要的部分是，春组件并不会因为所谓的props和state产生一些深层的改变，所以可以通过在shouldComponentUpdate函数内浅比较state和props来优化性能。目前只有类组件可以声明shouldComponentUpdate方法，不过在未来这可能会改变。

展示组件和容器组件都有上述的特性。在我的经验来看，展示组件倾向于无状态的纯函数，而容器组件情况于多状态的纯类。当然了，这只是我的经验之谈，而不是铁则，我也见过一些完全相反的情况。不要将展示组件和容器组件当作教条。有些时候，划清这条线并不是必要，而且有时很难分清。如果你感到不确定一个组件是展示组件还是容器组件时，不要太纠结，可能时候未到。

### 例子

Michael Chan的这篇梗概中讨论了这个。

### 衍生阅读

- [Getting Started with Redux](https://egghead.io/series/getting-started-with-redux)


- [Mixins are Dead, Long Live Composition](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750)


- [Container Components](https://medium.com/@learnreact/container-components-c0e67432e005)


- [Atomic Web Design](http://bradfrost.com/blog/post/atomic-web-design/)


- [Building the Facebook News Feed with Relay](http://facebook.github.io/react/blog/2015/03/19/building-the-facebook-news-feed-with-relay.html)

### 脚注

在这篇文章的早些版本中我们称之为聪明组件和呆组件，不过这对展示组件来说有些讲的太过分了，而且这并不能很好的解释两者的意图。我很喜欢这对新的叫法，希望你也是。

在这篇文章的早些版本中我声称展示组件只能包含其他的展示组件。如今我不太认为这时对的。一个组件时展示组件还是容器组件应该由他的内部细节决定。你应该能够用通过容器组件来替换掉一个展示组件同时不修改任何调用的地方。因此，展示组件和容器组件都可以包含其他展示组件和容器组件，这是没问题的。