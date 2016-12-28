# 从RxJS到Cycle.js

在上一篇文章（[Observable与RxJS](01-observable-and-rxjs.md)）中提到`RxJS`的 slogan 是 *Think of RxJS as Lodash for events*。利用`RxJS`我们可以方便的处理用户交互、网络请求等异步操作（就像使用`Lodash`一样）。

但是，`RxJS`仅仅是`Observable`和常用“运算符”的实现，它距离一个完整的前端框架还有些距离。在这篇文章中，我们将以`RxJS`为基础，一步步实现一个完整的函数响应式前端框架——`Cycle.js`。

>![Cycle.js](https://cloud.githubusercontent.com/assets/1812118/21467832/c22b0eee-ca35-11e6-994b-91b6dd181f0b.png)

> A functional and reactive JavaScript framework for cleaner code.

> Cycle.js 官网：[https://cycle.js.org/](https://cycle.js.org/)

> Cycle.js Github 地址：[https://github.com/cyclejs/cyclejs](https://github.com/cyclejs/cyclejs)


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

最后我们使用`@cycle/rxjs-run`替代我们自己写的`run`函数：[Demo](http://jsbin.com/duhifud/edit?js,output)。

```javascript
Cycle.run(main, drivers);
```

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

# Cycle.js

终于，通过一步步的修改，我们从一段简单的`RxJS`实现了`Cycle.js`框架（虽然我们的`run`函数、`makeDOMDriver`函数很简单，但其原理与`Cycle.js`是一致的）。

正如本文开头所说，实现逻辑与副作用的分离是我们的目前。显然在`Cycle.js`中，我们只需要编写`main`函数，它一般是纯函数，只包含计算逻辑；而副作用都隐藏在了`Cycle.js`内部，准确讲是`drivers`中。

## 函数式

`Cycle.js`是函数式的，它借助`JavaScript`这门“披着C外衣的Lisp”的语言（出自[JavaScript: 世界上最被误解的语言](http://javascript.crockford.com/zh/javascript.html)）和`RxJS`的`Observable`实现了一个干净的、纯的、函数式的前端框架。

`Cycle.js`受启发于`Elm`（[Elm](http://elm-lang.org/)），后者是一门纯面向函数语言、是`Haskell`语言的子集、同时也启发了`Redux`。随着对`Cycle.js`的深入了解你会发现它与`React`框架和`Redux`单向数据流是如此的相像。

对于函数式编程，其实很多人会有误解：认为函数式编程就一定是没有副作用的。事实上，副作用是普遍存在的，任何程序都不可避免的需要与标准输入输出、鼠标键盘、屏幕输出等打交道，因此任何具有交互功能的程序都不可能不存在副作用。而函数式编程是让我们尽量编写没有副作用的纯函数，并将副作用进行剥离。`Observable`——甚至还有`Promise`——就是函数式编程中的`Monad`（[Monad](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)），就像铅桶可以隔离辐射源一样，`Monad`能够把副作用封装起来，避免副作用污染到函数的“纯度”。

## 响应式

`Cycle.js`是响应式的，它“总是能够及时响应用户请求，并且保持很低的延迟”。（出自[响应式编程的基本概念](http://www.infoq.com/cn/news/2016/01/reactive-basics)），是基于“响应式流”——Observable实现的。

`Cycle.js`与`React` + `Redux`相比很大的不同在于：在`Redux`中只有`action`和`state`组成的数据流；而在`Cycle.js`中一切“可变的”都是流——用户交互（鼠标、键盘、触摸屏等）是流、网络请求是流、`model`（数据模型）是流、`state`是流、`DOM`和`RN`（使用`Cycle.js`编写`Native`代码使用的底层驱动）也是流。

这种“面向数据流编程”除了可以利用`Observable`的`push`模型编写更加响应式的代码、更容易的历史和回放、更方便的组件状态传递等，更使我们习惯于“源源不断的数据流”而不是“可变的变量”，变量可能偷偷的改变了但用户无法感知，但数据流的数据总会被订阅和消费。

因此，`Cycle.js`是“函数响应式编程”的一种合适选型。
