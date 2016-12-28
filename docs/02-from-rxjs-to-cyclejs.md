# 从RxJS到Cycle.js

在上一篇文章（[Observable与RxJS](01-observable-and-rxjs.md)）中提到`RxJS`的 slogan 是 *Think of RxJS as Lodash for events*。利用`RxJS`我们可以方便的处理用户交互、网络请求等异步操作（就像使用`Lodash`一样）。

但是，`RxJS`仅仅是`Observable`和常用“运算符”的实现，它距离一个完整的前端框架还有些距离。在这篇文章中，我们将以`RxJS`为基础，一步步实现一个完整的函数响应式前端框架——`Cycle.js`。

>![Cycle.js](https://cloud.githubusercontent.com/assets/1812118/21467832/c22b0eee-ca35-11e6-994b-91b6dd181f0b.png)

> A functional and reactive JavaScript framework for cleaner code.

> Cycle.js 官网：[https://cycle.js.org/](https://cycle.js.org/)

> Cycle.js Github 地址：[https://github.com/cyclejs/cyclejs](https://github.com/cyclejs/cyclejs)

# 背景

`Cycle.js`是一个基于`Observable`的“函数响应式前端框架”。

作为一个现代的前端框架，它具有如下特性：

* Virtual DOM
* JSX
* 服务端渲染
* 组件化
* MV*框架（Model-View-Intent）
* 路由
* 数据流
* 时间旅行（撤销、回放）
* 热加载
* Native
* 插件化

`Cycle.js`比较类似于 [React](https://facebook.github.io/react/) + [Redux](https://github.com/reactjs/redux) 或者 [Elm](http://elm-lang.org/)：

`React`是`MV*`中的`View`部分，通过`JSX`实现了一套性能不错的`Virtual DOM`机制和组件机制，但它通常需要搭配合适的数据流（如`Flux`、`Redux`）才能成为完整的前端框架；`Redux`很好的通过“单向数据流”实现对`state`的控制，但对于用户交互、数据模型等缺乏足够的抽象，同时大量`switch`的存在也不是非常好的编程范式；而且由于`React`和`Redux`两者割裂，虽然有 [react-redux](https://github.com/reactjs/react-redux) 这样的封装，但依然需要具有四个参数的胶水函数——`connect`，引入了不必要的学习成本。

`Elm`是一个函数响应式框架的实现，`Elm`使用自己的`Elm-Lang`（它是`Haskell`语言的子集），`Elm`实现了多层级的`Model-View-Action-Update`的数据流。但这种数据流比较复杂，加之其函数式的语法，导致其学习成本极高，少有工业界的应用。但`Elm`的数据流思想非常有价值，直接启发了`Cycle.js`和`Redux`诞生。

`Cycle.js`集百家之长，克服了上述技术的大部分问题，保持了简单、纯粹的编程范式。由于`Cycle.js`采用了插件机制（类似于`Koa`），其核心代码简单、高效，值得我们深入学习。

因此在这篇文章中我们将通过将一段`Observable`代码（使用`RxJS`实现）进行不断的改造，最终实现我们自己的`Cycle.js`。当然在最后我们将使用`Cycle.js`官方模块替代自己的代码，以证明它们是等价的。

# main

首先我们从一个简单的例子开始。这个`RxJS`程序的功能很简单，它在DOM中显示了当前时间流逝了多少秒：[Demo](http://jsbin.com/xehoqop/edit?js,output)。

```javascript
Rx.Observable.timer(0, 1000)
  .map(i => `Secondes elapsed ${i}`)
  .subscribe({
    next: text => {
      const container = document.querySelector('.app');
      container.innerHTML = text;
    }
  });
```

接下来我们将逻辑（logic）与副作用（effcet）进行分离，并将数据流中的数据打印到控制台：[Demo](http://jsbin.com/tilora/edit?js,console,output)。

```javascript
function main() {
  return Rx.Observable.timer(0, 1000)
    .map(i => `Secondes elapsed ${i}`);
}

function DOMEffect(text$) {
  // render DOM with text$
}

function consoleLogEffect(text$) {
  // log text
}

const sink = main();
DOMEffect(sink);
consoleLogEffect(sink);
```

`main`函数负责实现计算逻辑，在这里我们使用了`sink`，它是`main`函数的返回值，用于传递给不同的带有副作用的函数。

接下来，我们将以“分离逻辑和副作用”为目的不断修改代码，最终获得一个只包含逻辑、没有副作用的纯函数（pure function）框架。

# run

现在，我们希望控制台中与DOM显示不一样的数据，这意味着我们需要在`main`函数中返回两条数据流：[Demo](http://jsbin.com/xidula/edit?js,console,output)。

```javascript
function main() {
  return {
    DOM: Rx.Observable.timer(0, 1000)
      .map(i => `Secondes elapsed ${i}`),
    Log: Rx.Observable.timer(0, 2000)
      .map(i => i * 2)
  }
}
// ...
const sinks = main();
DOMEffect(sinks.DOM);
consoleLogEffect(sinks.Log);
```

在`sinks`对象中，不同的字段对应着不同的数据流，只要将它们合理的分配给不同`effect`函数即可。接下来我们对这一步进行封装：[Demo](http://jsbin.com/temeset/edit?js,console,output)。

```javascript
// ...
function run(mainFn, effects) {
  const sinks = mainFn();
  Object.keys(effects).forEach(key => {
    effects[key](sinks[key]);
  });
}

const effectsFns = {
  DOM: DOMEffect,
  Log: consoleLogEffect
}

run(main, effectsFns);
```

`run`是一个函数，它接受两个参数：`main`和`effectsFns`。为了便于处理我们规定`effectsFns`的字段需要与`main`中返回的字段一致，这样在`run`函数中就不需要额外的代码进行映射了。

# drivers

在`Cycle.js`中，`run`的第二个参数被称为`drivers`。

在操作系统中，“驱动程序”是程序与硬件之间的接口；而在`Cycle.js`中，`drivers`则是“逻辑”与“副作用”之间的接口。

修改后的代码如下：[Demo](http://jsbin.com/dowiqiw/edit?js,console,output)。

```javascript
function run(mainFn, drivers) {
  const sinks = mainFn();
  Object.keys(drivers).forEach(key => {
    drivers[key](sinks[key]);
  });
}

const drivers = {
  DOM: DOMDriver,
//   Log: consoleLogDriver
}

run(main, drivers);
```

如果在2016款Mac Book上安装Windows系统，由于缺少驱动程序Touch Bar可能无法正常工作；同样的如果注释掉`drivers`中的部分字段，对应的副作用也就无法发生了。

因此上面这段代码中，因为注释掉了`drivers.Log`，即使`main`函数返回了`sinks.Log`，但控制台将不会进行任何输出。

# cycle

现在我们的程序实现了DOM渲染、控制台输出，并且分离了逻辑和副作用，但它还没有接收任何输入信息。

接下来我们需要实现这样一个功能，每次点击网页时重置页面的计时器，完整的效果见：[Demo](http://jsbin.com/polecu/edit?js,output)。

首先修改`DOMDriver`函数的返回值，它将返回页面点击的数据流`DOMSource`。利用`Observable.fromEvent`可以获取点击事件：

```javascript
function DOMDriver(text$) {
  text$
    .subscribe({
      next: text => {
        const container = document.querySelector('.app');
        container.innerHTML = text;
      }
    });
  return Rx.Observable.fromEvent(document, 'click');
}
```

然后修改`main`函数，它接受一个参数`DOMSource`，也就是`DOMDriver`的返回值。使用`switchMapTo`运算符可以实现点击重置的效果：

```javascript
function main(DOMSource) {
  const click$ = DOMSource;
  return {
    DOM: click$
      .startWith(null)
      .switchMapTo(Rx.Observable.timer(0, 1000))
      .map(i => `Secondes elapsed ${i}`)
  }
}
```

但仅仅修改`DOMDriver`和`main`是不够的，我们还需要修改`run`函数实现`DOMSource`的传递。但显然下面这段代码是行不通的：

```javascript
function run(mainFn, drivers) {
  const sinks = mainFn(DOMSource);
  const DOMSource = drivers.DOM(sinks.DOM);
}
```

在这里发生了“循环依赖”，简化一下代码可以更明显的看到问题所在：

```javascript
a = f(b);
b = g(a);
```

为了解决循环依赖，需要使用上篇文章中介绍过的`Subject`（[链接](./01-observable-and-rxjs.md#subject)）。通过`Subject`引入一个代理数据流实现循环依赖：

```javascript
bProxy = new Rx.Subject();
a = f(bProxy);
b = g(a);
b.subscribe(bProxy);
```

最后修改`run`函数如下，代码即可成功运行：[Demo](http://jsbin.com/polecu/16/edit?js,output)。

```javascript
function run(mainFn, drivers) {
  const proxyDOMSource = new Rx.Subject();
  const sinks = mainFn(proxyDOMSource);
  const DOMSource = drivers.DOM(sinks.DOM);
  DOMSource.subscribe(proxyDOMSource);
}
```

下面这张图很好的解释了这种循环：

![cycle](https://cloud.githubusercontent.com/assets/1812118/21468308/141848e8-ca45-11e6-9afe-7b6e55672c87.png)

循环的两侧是`main`和`driver`，在App端（`main`）编写逻辑，在框架端（`driver`）实现副作用；两个方向分别对应了输入和输出，共同实现了代码与现实世界的交互。

**循环是`Cycle.js`的核心，也是其名字的由来。**

# rxjs-run

接下来，就如同`sinks`一样，我们将单一的`DOMSource`改写为一组数据流`sources`。而且`sinks`与`sources`的字段一一对应。完整的效果见：[Demo](http://jsbin.com/loxezar/edit?js,output)。

首先是`main`函数：

```javascript
function main(sources) {
  const click$ = sources.DOM;
  // ...
}
```

然后是`run`函数。为了解决“循环”问题，需要创建一组`Subject`，它们保存在`proxySources`中，并分别订阅各个`driver`返回的`source`：

```javascript
function run(mainFn, drivers) {
  const proxySources = {};
  Object.keys(drivers).forEach(key => {
    proxySources[key] = new Rx.Subject();
  });
  const sinks = mainFn(proxySources);
  Object.keys(drivers).forEach(key => {
    const source = drivers[key](sinks[key]);
    source.subscribe(proxySources[key]);
  });
}
```

我们获得了一个非常通用的`run`函数，它可以用于不同的`main`和`drivers`，实现了`sinks`和`sources`的循环。如果我们将它单独抽象出来，它就是`Cycle.js`的`@cycle/rxjs-run`包（[链接](https://github.com/cyclejs/cyclejs/tree/master/rxjs-run)）。

最后我们使用`@cycle/rxjs-run`替代我们自己写的`run`函数，代码依然运行良好：[Demo](http://jsbin.com/duhifud/edit?js,output)。

```javascript
Cycle.run(main, drivers);
```

`Cycle.js`的核心非常简单，简单到只包含一个`run`函数，但它却支撑起整个框架的运行。也许你会质疑它太过简单，那么不妨让我们继续实现一个`Virtual DOM`。

# DOM Object

我们成功的抽象出了`run`函数，现在还有`main`函数和`drivers`对象（包含一个`DOMDriver`），接下来我们将抽象`drivers`。

现在我们的`main`函数看起来是这个样子：

```javascript
function main(sources) {
  const click$ = sources.DOM;
  return {
    DOM: click$
      .startWith(null)
      .switchMapTo(Rx.Observable.timer(0, 1000))
      .map(i => `Secondes elapsed ${i}`)
  }
}
```

我们需要拓展`main`函数，使其能返回更加复杂的数据渲染到`DOM`结点上。我们将使用一个对象来描述`DOM`：

```javascript
click$
  // ...
  .map(i => ({
    tagName: 'H1',
    children: [`Seconds elapsed ${i}`]
  }))
```

其中`tagName`表示`DOM`的标签名；`children`是该`DOM`的内容，现在它的值是包含一个字符串的数组。同样的，我们需要编写一个`createElement`函数，接受`DOM`描述对象，并产生真正的`DOM`对象：

```javascript
function createElement(obj) {
  const element = document.createElement(obj.tagName);
  obj.children.forEach(child => {
    element.innerHTML += child;
  });
  return element;
}
```

然后修改`DOMDriver`，将接受到的`DOM`描述对象通过`createElement`获取真实`DOM`对象，并通过`container.append`渲染到页面上（由于使用了`.append`，所以需要额外清空已有`DOM`）：[Demo](http://jsbin.com/towowo/edit?js,output)。

```javascript
function DOMDriver(obj$) {
  obj$
    .subscribe({
      next: obj => {
        const container = document.querySelector('.app');
        container.innerHTML = '';
        container.append(createElement(obj));
      }
    });
  return Rx.Observable.fromEvent(document, 'click');
}
```

代码运行良好，成功的在页面上渲染了一个`h1`结点。

接下来我们需要完善下代码，使其能渲染带嵌套的`DOM`结构，比如`h1`标签中嵌套`span`标签：

```javascript
click$
  // ...
  .map(i => ({
    tagName: 'H1',
    children: [{
      tagName: 'SPAN',
      children: [`Seconds elapsed ${i}`]
    }]
  }))
```

需要修改`createElement`支持递归，以实现嵌套：

```javascript
  function createElement(obj) {
    const element = document.createElement(obj.tagName);
    obj.children.forEach(child => {
      if (typeof child === 'string') {
        element.innerHTML += child;
      } else {
        element.append(createElement(child));
      }
    });
    return element;
  }
```

完整的代码见[Demo](http://jsbin.com/lazoba/edit?js,output)。

# Virtual DOM

我们回顾整个`DOMDriver`，它已经足够通用，除了三个细节。

第一，`DOMDriver`只能返回点击事件：

```javascript
function DOMDriver(obj$) {
  // ...
  return Rx.Observable.fromEvent(document, 'click');
}
```

可以通过简单的修改使`DOMDriver`支持任意事件绑定，这样就可以在`main`函数中通过链式调用实现事件绑定：

```javascript
function main(sources) {
  const click$ = sources.DOM.select('span').events('mouseover');
  // ...
}

function DOMDriver(obj$) {
  // ...
  return {
    select: tagName => ({
      events: eventType => {
        return Rx.Observable.fromEvent(document, eventType)
          .filter(ev => ev.target.tagName === tagName.toUpperCase())
      }
    })
  };
}
```

在这里我们使用了事件代理，这样就不比每次更新`DOM`结点时都重新绑定事件。需要注意的是在`HTML`中，所有的标签名都是大写的（[REC-DOM-Level-1-19981001](https://www.w3.org/TR/REC-DOM-Level-1/level-one-core.html#ID-5DFED1F0)）。

第二，`DOMDriver`现在只能渲染到`className`为`app`元素上，我们可以通过高阶函数进行简单的封装：[Demo](http://jsbin.com/vekegi/edit?js,output)。

```javascript
function makeDOMDriver(mountSelector) {
  return function DOMDriver(obj$) {
    // ...
    const container = document.querySelector(mountSelector);
    // ...
  }
}

const drivers = {
  DOM: makeDOMDriver('.app')
}
```

第三，`DOM`描述对象看起来还是有些简单，我们将使用函数嵌套来实现`DOM`嵌套：

```javascript
const h = (tagName, children) => ({ tagName, children });
const h1 = children => h('H1', children);
const span = children => h('SPAN', children);

function main(sources) {
  click$
    // ..
    .map(i =>
      h1([
        span([`Seconds elapsed ${i}`])
      ])
    )
}
```

`h`函数是一个通用的标签生成函数，`h1`和`span`可以生成特定标签的`DOM`描述对象。这样的`DOM`描述对象看起来更类似于我们熟悉的`HTML`标签了（事实上借助 [babel-plugin-transform-react-jsx](http://babeljs.io/docs/plugins/transform-react-jsx/) 和 [snabbdom-jsx](https://www.npmjs.com/package/snabbdom-jsx) 我们可以使用`jsx`编写它）。

最后，我们得到了一个较为通用的`makeDOMDriver`和一些`DOM`相关的函数（`h`、`h1`、`span`），使用这些函数我们可以实现各种`DOM`结构的渲染。

事实上在`Cycle.js`中这些函数都被抽象在`@cycle/dom`包中（[链接](https://github.com/cyclejs/cyclejs/tree/master/dom)），因此是时候使用`@cycle/dom`替代我们自己的函数了：[Demo](http://jsbin.com/xasahih/edit?js,output)。

```javascript
const { makeDOMDriver, h1, span } = CycleDOM;
const { startWith, switchMapTo, map } = Rx.Observable.prototype;

function main(sources) {
  const click$ = sources.DOM.select('span').events('mouseover');
  return {
    DOM: click$
      ::startWith(null)
      ::switchMapTo(Rx.Observable.timer(0, 1000))
      ::map(i =>
        h1([
          span([`Seconds elapsed ${i}`])
        ])
      )
  }
}

const drivers = {
  DOM: makeDOMDriver('.app')
}

Cycle.run(main, drivers);
```

> 由于最新版`Cycle.js`的默认数据流从`RxJS`切换到`XStream`，因此即使使用`cycle/rxjs-run`也需要引入`xstream`。
> 在`npm`中不需要显式依赖`xstream`，因为`@cycle/dom`会自动依赖`xstream`。

# 总结

终于，通过一步步的修改，我们从一段简单的`RxJS`改造出了`Cycle.js`框架，并证明了它们的等价性。

正如本文开头所说，实现逻辑与副作用的分离是我们的目的。显然在`Cycle.js`中，我们只需要编写`main`函数，它一般是纯函数，只包含计算逻辑；而副作用都隐藏在了`drivers`中，在实际开发中通常我们不需要自己实现`drivers`，因为`Cycle.js`提供了大量的`drivers`实现，所以可以认为副作用都隐藏在了框架之中。

在本文我们实现了与`@cycle/dom`类似的`Virtual DOM`，通过这个例子可以发现`Cycle.js`插件机制的强大，只要通过引入不同的`drivers`即可实现各种副作用：`DOM`操作、网络请求、访问存储，甚至可以在`Native`上渲染（[@cycle/react-native](https://github.com/cyclejs/react-native)）。

正如本文开头所说，`Cycle.js`作为完整的前端框架还拥有很多的特性，篇幅所限不能在此一一列举。在下一篇文章中我们将对`Cycle.js`的设计和使用做更详细的介绍，并探讨`Cycle.js`的工程化问题
