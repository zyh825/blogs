---
title: 【翻译】深入 React Pref 调试 (PART2)
date: 2016-11-34 12:21:33
tags: ["react", "debug"]
---

这是 React 的性能优化的第二部分。在第一部分中，我们简单的了解了怎么去使用 React 的 Perf 调试工具，常规的 React 的渲染瓶颈，和一些调试的小技巧。如果您还没准备好（继续深入）的话，那最好先去把它（Part 1） 看了

在第二部分中，我们会继续深入react调试的工作流 - 既然已经给了这些个想法，那又会如何在练习中体现呢？（自问自答）我们将会通过一些切合时机的例子，并且使用chrome的开发工具来分析和解决性能问题。（如果您在看完后有任何建议和新的想法，请让我们知道！）

我们将会参考下面这段简单的代码 - 你可以看出它通过 React 渲染了一个简单的 todo list。在后面的 JS fiddle 代码片段中，你可以点击 “Result” 来看一个带交互的实际例子，complete with performance repros（不知道怎么翻译）。我们将会这过程中提交并更新 JS fiddles。

## CASE STUDY #1: TODOLIST

让我们开始上面的 `TodoList` 。通过尝试在代码未优化的例子中，快速的输入，你会发现它有多慢。

让我们开始用*Chrome* 开发工具中的 *Timeline profiler* ，通过一些细节的切面的来看浏览器在做了些什么： 处理用户触发的事件，运行JS，渲染和绘制。在输入框内输入一个字符，然后停止 *Timeline profiler* 。这个过程中你不会感到明显的缓慢，因为我们只输入了一个字符，这是产生用以分析的少量信息的最快的手段。

![img](http://benchling.engineering/content/images/2016/02/0.png)

注意图中 `Event(textInput)` 进度条在 `Scripting(Children)` 中总计花费了121.10 ms 。这个时段切面中表明了这个缓慢问题是一个脚本的问题，而不是样式或重复计算导致的性能问题。

所以让我们深入脚本。切换到 *Profiles tab* - *Timeline* 不仅给了我们浏览器（和 *JS Profile*）的一个概况，通过这些个代码运行切面则让我们得以继续深入 JS 的内部实现，而且给了我们各种各样可视化工具。根据另一份Profile， 它中指出运行缓慢的问题并不在于我们的应用代码：

![img](http://benchling.engineering/content/images/2016/02/IipbV.png)

将 Profile 的 `Heavy（Bottom up）` 按 `Total` 字段降序排列，结果指出消耗时间最多的部分是React `batchUpdates`方法的调用，这很明确的提示了问题出在 React 上。相反，通过 `Self` 来测量函数中除去子函数的花费的时间 - 根据  `Self` 排序来查看是否有明显的昂贵（花费时间多）的函数时，这并没有明显的性能瓶颈在应用层面的函数中，所以让我们换 React 的 Pref 来看看。

为了声称一个针对缓慢操作的的评估切面，在console中，我们调用 `React.addons.Perf.start()`， 然后通过输入一个字符来重现缓慢的操作，然后输入`React.addons.Perf.stop()`结束监控。(最后) 通过输入 `React.addons.Perf.printWasted()`我们可以看到应用在不必要的渲染中花费的时间：

![img](http://benchling.engineering/content/images/2016/02/2.png)

这第一项表明`TodoItem`是由`Todos`渲染的，而（输入）`Perf.printWated()`可以看出，如果我们避免render树的反复构建，我们就可以省下100ms。 这看起来像是我们优化中最主要的部分。

为了分析为什么TodoItem会浪费这么多时间，我们创建一个自定义的输入函数，以很明确的 `WhyDidYouUpdateMinxin`来命名它，它会嵌入组件并且打印一些信息，如触发了哪些更新和为什么更新。下面是代码；按你的需求随意使用它

```javascript
/* eslint-disable no-console */
import _ from 'underscore';

/*
Drop this mixin into a component that wastes time according to Perf.getWastedTime() to find
out what state/props should be preserved. Once it says "Update avoidable!" for {state, props},
you should be able to drop in React.addons.PureRenderMixin
React.createClass {
  mixins: [WhyDidYouUpdateMixin]
}
*/
function isRequiredUpdateObject(o) {
  return Array.isArray(o) || (o && o.constructor === Object.prototype.constructor);
}

function deepDiff(o1, o2, p) {
  const notify = (status) => {
    console.warn('Update %s', status);
    console.log('%cbefore', 'font-weight: bold', o1);
    console.log('%cafter ', 'font-weight: bold', o2);
  };
  if (!_.isEqual(o1, o2)) {
    console.group(p);
    if ([o1, o2].every(_.isFunction)) {
      notify('avoidable?');
    } else if (![o1, o2].every(isRequiredUpdateObject)) {
      notify('required.');
    } else {
      const keys = _.union(_.keys(o1), _.keys(o2));
      for (const key of keys) {
        deepDiff(o1[key], o2[key], key);
      }
    }
    console.groupEnd();
  } else if (o1 !== o2) {
    console.group(p);
    notify('avoidable!');
    if (_.isObject(o1) && _.isObject(o2)) {
      const keys = _.union(_.keys(o1), _.keys(o2));
      for (const key of keys) {
        deepDiff(o1[key], o2[key], key);
      }
    }
    console.groupEnd();
  }
}

const WhyDidYouUpdateMixin = {
  componentDidUpdate(prevProps, prevState) {
    deepDiff({props: prevProps, state: prevState},
             {props: this.props, state: this.state},
             this.constructor.displayName);
  },
};

export default WhyDidYouUpdateMixin;
```

一旦我们在TodoItem中注入了这个函数，我们可以看到发生了一些什么：

![img](http://benchling.engineering/content/images/2016/02/3.png)

Aha! 我们看到tags这个变量在操作前后近似 - 这个注入函数告诉我们当两个状态对象深度相等而不是严格相等时，这时（渲染）是可以避免的。换个角度说，这个问题的难点在于如何判断两个方法是相等的，因为 `Fucntion.bind`声明了一个新的方法，尽管绑定的参数相同。这是一些有用的线索，既然如此 - 我们回过头来看看我们是如何传入`tags`和`deleteItem`的，看起来没错我们构建一个`TodoItem`时我们都传递了一些新的值。

假如我们用传递未绑定的进入TodoItem, 同时我们将tags储存成常量，我们将可以避免这些个问题：

```javascript
diff --git i/example.js w/example.js
index ba2427a..2edc85e 100644
--- i/example.js
+++ w/example.js
@@ -11,10 +11,13 @@ const TodoItem = React.createClass({
       id: React.PropTypes.number.isRequired,
     }).isRequired,
   },
+  deleteItem() {
+    this.props.deleteItem(this.props.item.id);
+  },
   render() {
     return (
       <div>
-        <button style={{width: 30}} onClick={this.props.deleteItem}>x</button>
+        <button style={{width: 30}} onClick={this.deleteItem}>x</button>
         <span>{this.props.item.text}</span>
         {this.props.tags.map((tag) => {
           return <span key={tag} className="tag"> {tag}</span>;
@@ -26,6 +29,9 @@ const TodoItem = React.createClass({
 
 const Todos = React.createClass({
   mixins: [React.addons.LinkedStateMixin],
+  statics: {
+    tags: ['important', 'starred'],
+  },
   propTypes: {
     initialItems: React.PropTypes.arrayOf(React.PropTypes.shape({
       text: React.PropTypes.string.isRequired,
@@ -60,8 +66,8 @@ const Todos = React.createClass({
         </form>
         {this.state.items.map((item) => {
           return (
-            <TodoItem key={item.id} item={item} tags={['important', 'starred']}
-             deleteItem={this.deleteItem.bind(null, item.id)} />
+            <TodoItem key={item.id} item={item} tags={Todos.tags}
+             deleteItem={this.deleteItem} />
            );
         })}
       </div>
```

`WhyDidYouUpdateMixin` 现在表明了 prevProps 和 newProps 浅相等。我们可以使用 PureRenderMixin，来避免当props（和state）浅相等时的组件更新。

![img](http://benchling.engineering/content/images/2016/02/4.png)

当我们重新运行Profiler，我们可以看到只花费了35ms（比原来的快了4倍）

![img](http://benchling.engineering/content/images/2016/02/hSPOL.png)

这比原来的好了，但是仍然不够理想。在输入框内打字不应该导致这么慢。这表明我们任然没有做到0（list的item数量级别）的工作。我们仅仅定义了常量，我们依然需要对每个item进行浅比较。

此时，你可以能会认为1000个items在todolist中已经是极端的情况了，而且30ms的延迟，对你的应用而言并不是问题。如果你希望能够支持几千个子元素，那么，这任然没有达到理想的60fps

（16ms 每帧 - 慢一点点你都会感受的到）。

将组件拆分成多个组件作为下一步工作是有道理的（同样将之视为第一步也是合理的）。我们观察到Todos这个组件由两个没有交集的子组建构成，一个`AddTaskForm`组件包含了输入框和按钮，一个`TodoItems`组件包含了Items的列表。

```javascript
let ID = 0; // incrementing counter for todo item ids

const AddTaskForm = React.createClass({
  mixins: [React.addons.LinkedStateMixin, React.addons.PureRenderMixin],
  getInitialState() {
    return {
      text: '',
    };
  },
  addTask(e) {
    e.preventDefault();
    this.props.addTask(this.state.text);
    this.setState({text: ''});
  },
  render() {
    return (
      <form onSubmit={this.addTask}>
        <input valueLink={this.linkState('text')} />
        <button>Add Task</button>
      </form>
    );
  }
});

const TodoItems = React.createClass({
  mixins: [React.addons.PureRenderMixin],
  render() {
    return (
      <div>
        {this.props.items.map((item) => {
          return (
            <TodoItem key={item.id} item={item} tags={Todos.tags}
             deleteItem={this.props.deleteItem} />
           );
        })}
      </div>
    );
  }
});

const TodoItem = React.createClass({
  mixins: [React.addons.PureRenderMixin],
  propTypes: {
    deleteItem: React.PropTypes.func.isRequired,
    tags: React.PropTypes.arrayOf(React.PropTypes.string.isRequired).isRequired,
    item: React.PropTypes.shape({
      text: React.PropTypes.string.isRequired,
      id: React.PropTypes.number.isRequired,
    }).isRequired,
  },
  deleteItem() {
    this.props.deleteItem(this.props.item.id);
  },
  render() {
    return (
      <div>
        <button style={{width: 30}} onClick={this.deleteItem}>x</button>
        <span>{this.props.item.text}</span>
        {this.props.tags.map((tag) => {
          return <span key={tag} className="tag"> {tag}</span>;
        })}
      </div>
    );
  },
});

const Todos = React.createClass({
  statics: {
    tags: ['important', 'starred'],
  },
  propTypes: {
    initialItems: React.PropTypes.arrayOf(React.PropTypes.shape({
      text: React.PropTypes.string.isRequired,
      id: React.PropTypes.number.isRequired,
    }).isRequired).isRequired,
  },
  getInitialState() {
    return {
      items: this.props.initialItems,
    };
  },
  addTask(text) {
    this.setState({
      items: [{id: ID++, text}].concat(this.state.items),
    });
  },
  deleteItem(itemId) {
    this.setState({
      items: this.state.items.filter((item) => item.id !== itemId),
    });
  },
  render: function() {
    return (
      <div>
        <h1>My TODOs</h1>
        <AddTaskForm addTask={this.addTask} />
        <TodoItems items={this.state.items} deleteItem={this.deleteItem} />
      </div>
   );
  },
});

// Create a Todos component, initialized with 1000 items.
const items = [];
for (let i = 0; i < 1000; i++) {
  items.push({id: ID++, text: 'Todo Item #' + i});
}
React.render(<Todos initialItems={items} />, document.body);
```

任何一项重构都能提供实质的性能增益：

- 如果我们通过PureRenderMixin来创建一个TodoItems，因为`prevProps.items === this.props.items`，我们将能通避免重新渲染每一个item来减少O(n)的时间消耗。
- 如果我们创建一个AddTaskForm组件时，将文本的state至存在于组件内使，当文本改变时，将不会，Todos组件（列表部分）将不会重新渲染（避免了O（n）的渲染消耗）。

（将以上的工作）合起来，我们每次键盘的按键操作只会消耗10ms

## CASE STUDY #2:

场景： 我们想在用户由太多的任务（>3000）时，渲染一个警告，同时我们想给这些todo项添加一个背景样式。

实施：

- 我们由一个近似的todo列表的案例，用（之前的）`TodoItems`来实现 - 在这个李子中，我们将会把input的文本内容储存在最顶层的组件状态中。
- 我们创建了一个`TaskWarning`的组件，来根据任务的数量来决定消息的渲染。为了封装这层逻辑，我们将会返回null，假如它不该被渲染。
- 我们用样式给`div:nth-child(even)`所匹配到的元素添加一个灰色的背景。

```javascript
let ID = 0; // incrementing counter for todo item ids

const TodoWarning = React.createClass({
  propTypes: {
  	itemCount: React.PropTypes.number.isRequired
  },
  render() {
    if (this.props.itemCount > 3000) {
      return <div>'YOU HAVE TOO MANY TASKS. SLOW DOWN.'</div>;
    }
    return null;
  }
});

const TodoItems = React.createClass({
  mixins: [React.addons.PureRenderMixin],
  render() {
    return (
      <div className="todoItems">
        {this.props.items.map((item) => {
          return (
            <TodoItem key={item.id} item={item} tags={Todos.tags}
             deleteItem={this.props.deleteItem} />
           );
        })}
      </div>
    );
  }
});

const TodoItem = React.createClass({
  mixins: [React.addons.PureRenderMixin],
  propTypes: {
    deleteItem: React.PropTypes.func.isRequired,
    tags: React.PropTypes.arrayOf(React.PropTypes.string.isRequired).isRequired,
    item: React.PropTypes.shape({
      text: React.PropTypes.string.isRequired,
      id: React.PropTypes.number.isRequired,
    }).isRequired,
  },
  deleteItem() {
    this.props.deleteItem(this.props.item.id);
  },
  render() {
    return (
      <div>
        <button style={{width: 30}} onClick={this.deleteItem}>x</button>
        <span>{this.props.item.text}</span>
        {this.props.tags.map((tag) => {
          return <span key={tag} className="tag"> {tag}</span>;
        })}
      </div>
    );
  },
});

const Todos = React.createClass({
  mixins: [React.addons.LinkedStateMixin],
  statics: {
    tags: ['important', 'starred'],
  },
  propTypes: {
    initialItems: React.PropTypes.arrayOf(React.PropTypes.shape({
      text: React.PropTypes.string.isRequired,
      id: React.PropTypes.number.isRequired,
    }).isRequired).isRequired,
  },
  getInitialState() {
    return {
      items: this.props.initialItems,
      text: '',
    };
  },
  addTask(e) {
    e.preventDefault();
    this.setState({
      items: [{id: ID++, text: this.state.text}].concat(this.state.items),
      text: '',
    });
  },
  deleteItem(itemId) {
    this.setState({
      items: this.state.items.filter((item) => item.id !== itemId),
    });
  },
  render: function() {
    return (
      <div>
        <TodoWarning itemCount={this.state.items.length} />
        <h1>My TODOs</h1>
        <form onSubmit={this.addTask}>
          <input valueLink={this.linkState('text')} />
          <button>Add Task</button>
        </form>
        <TodoItems items={this.state.items} deleteItem={this.deleteItem} />
      </div>
   );
  },
});

// Create a Todos component, initialized with 1000 items.
const items = [];
```

观察诊断：在输入框内快速的输入，页面由很明显的延迟（不超过3000个任务）。如果我们继续添加一个任务（> 3000 个任务），延迟随着按钮消失了。令人惊讶的地方，添加更多的任务似乎解决了这个任务!

调试：Timeline Profile 展现了一些很有意思的东西：

![img](http://benchling.engineering/content/images/2016/02/6.png)

因为某些原因，输入单个字符时触发了大量的`Recalculate Style`，超过了30ms（这就是为什么当我们打字的速度大于30ms/每个字符时，我们会观察到Jank【注：应用刷新的速率没有达到设备的刷新速率而产生的卡顿现象】）。

看看图片底部的`First invalidated`一节内容。它指出`Danger.dangerouslyReplaceNodeWithMarkup` 导致了页面的布局失效，从而导致了样式的重新计算。`react-with-addons.js:2301`处：

```javascript
oldChild.parentNode.replaceChild(newChild, oldChild);
```

因为某些原因，React 用一个完全新的DOM节点替换了原先的DOM节点！ 回想起来，生成DOM的操作时非常昂贵的。使用`Perf.printDOM()`，我们可以看到React在DOM操作时的性能：

![img](http://benchling.engineering/content/images/2016/02/7.png)

更新的属性反映了当在输入框输入`abc`时，TaskWarning是不可见的。然而，replace 项（图中的type）又指出了，此时React正在为TaskWarning组件创建DOM，尽管它看起来不应该有明确可见的实体DOM。

正如上面所展示的，React（<= v0.13）使用了一个`noscript`标签来渲染“no component”，但是却（在做diff时）错误的将两个noscript的标签视为了不相等：`noscript`最后被另一个noscript替换。此外，回想起来我们给其他的每个元素添加的一个灰色背景的样式。因为CSS的缘故，这3000个节点中任何一个的单独渲染都依赖与它之前的兄弟节点。每次noscript标签被替换，它随后的DOM节点的样式都会被重新计算。

为了修复这个问题，我们可以：

- 让TaskWarning 返回一个空的div
- 将TaskWarning组件用一个div包裹起来，这样它就不会去影响css选择器对随后节点的选择。【意味不明】
- 升级React :-)

除此以外。这里最关键的一点时，我们能够自己分析出这些，仅仅通过Timeline Profiler!

# CONCLUSION

我希望展示react的性能问题在各种开发工具中的表现是有用的（能够帮到大家）- 通过*Timeline *， *profiles*，和React的Perf工具的配合使用还有很长的道路要走。

在todolisst中包含上千个项，和随意的着色，似乎有些刻意（简单将就是说我上面举得例子看起来有点不切实际），不过它所变面出的问题却和我们在做electronic lab notebook的项目中实际中遇到的渲染大量文档和表格时的问题很近似。

（后面时招聘广告和客套话，省略...）





## 读后总结：

​	很老的一篇文章了（最然是今年年初的），原本是奔着文章的题目去的，不过发觉内容里貌似也没怎么深入，Perf的介绍和使用主要还是 [Part1](http://benchling.engineering/performance-engineering-with-react/) 中，不过在文中 timeline 和 profile的配合使用中还是学到了一些东西（思考问题的角度），之后看样子还是要好好的学习如何使用chrome 的devtool。

总结几点吧：

- 要使用PureComponentMixin，使用es6的class来创建组件时则应该继承PureComponent。
- 合理的拆分组件（组件解耦）。
- 常量不要在render时声明，应该抽到外面去。
- 文中所提到的noscript的问题，在react的https://github.com/facebook/react/issues/2770 中已经解决了，所以返回null也没什么问题了。具体的[commit](https://github.com/facebook/react/pull/4825/commits/db589a717510d1a2d235d934a0eec4b5609b4e0b)
- 交互的相应理想的状态下要做到30ms内（难点）。文中提到 Jank 的概念。http://jankfree.org/
- 看源码！看源码！看源码！重要的事情说三遍。
- 文中的图片请挂代理...