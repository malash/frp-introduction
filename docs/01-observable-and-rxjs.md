# Observable与RxJS

在ECMAScript（JavaScript）中，利用事件循环可以非常容易的在单个线程中实现非阻塞的、异步的操作，如文件IO、网络IO、交互事件等。

实现异步的方式也有很多，常用的有`Callback`、`EventEmitter`、`Promise`、`Stream`，在这里我将简绍一种新的实现异步的方式：`Observable`。

# 背景
## Callback

`Callback`（回调）是实现异步操作最简单的实现方法，也是JavaScript长久以来一直提供的。

这是一种典型的Node风格回调实现：

```javascript
const fs = require('fs');
fs.readFile('./input.json', (err, data) => {
  if (err)  {
    console.error(err);
    return;
  }
  console.log(data);
});
```

众所周知，回调非常容易造成“回调大坑（Callback Hell）”。为了解决这个问题诞生了很多轮子，比如：

## EventEmitter

```javascript
const EventEmitter = require('events');
const myEmitter1 = new EventEmitter();
const myEmitter2 = new EventEmitter();
myEmitter1.on('event', (data) => {
  console.log(`myEmitter1 recieved ${data}`);
  myEmitter2.emit('event', data * data);
});
myEmitter2.on('event', (data) => {
  console.log(`myEmitter2 recieved ${data}`);
});

myEmitter1.emit('event', 1);
```
除了用于交互事件绑定外，通过在不同`EventEmitter`间传递事件，可以实现复杂的计算逻辑。
但是随着`EventEmitter`实例的增多，管理这些`EventEmitter`会越来越困难。复杂计算逻辑用`Promise`会更适合：[Demo](http://jsbin.com/civoqol/edit?js,console)

## Promise
```javascript
const myPromise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('malash');
  }, 100);
});
myPromise
  .then(user => fetch(`https://api.github.com/users/${user}`))
  .then(res => res.json())
  .then(data => console.log(data))
  .catch(e => console.error(e));
```
通过对`.then`方法的链式调用，代码更加整洁，逻辑也更加清晰。

`Promise`是一种非常好的处理单次异步操作的方式，特别是与`async/await`相结合，可以以同步的形式写出异步的代码。因此早在其成为规范之前就已经有不少`Promise`的实现，如`Q`、`Bluebird`、`es6-promise`；也可结合`co`和`Generator`模拟`async/await`。

>`Promise`已经进入 [ECMAScript 2015 (ES6)](http://www.ecma-international.org/ecma-262/6.0/#sec-promise-objects)。

>`async/await`目前进入 [ECMAScript Stage 3 Draft](https://tc39.github.io/ecmascript-asyncawait/#async-function-definitions)。

然而，Promise并不是万能的。

其一，总结异步的常见使用场景：

* AJAX
* FILE IO (Async or Stream)
* User Events (clicks, mousemoves, keyups, etc)
* Animations
* Sockets
* Workers

可以发现，`Promise`是单值的，因此只适合处理`AJAX`场景下的数据和非流式的文件IO，事件监听、流式计算往往会异步返回多个值，这些场景`Promise`实现会有些困难。

在ECMAScript 2015中，还提供了一种可以返回多值的对象：`Iterator`，以及它的生成器`Generator`。`Iterator`本身并不支持异步，一种实现异步的思路是将`Iterator`与`Promise`相结合——每次调用`.next`返回一个`Promise`。这是一种可行的实现方式，稍晚我们将详细讨论它。

其二，`Promise`保证了它将在一段时间之后通过`.then`返回数据，但没有办法简单的办法“取消”对该值的处理。

试想这样一种场景：

![不可取消造成的问题](https://cloud.githubusercontent.com/assets/1812118/20671603/ef197f62-b5b8-11e6-9fe0-7f251b41ef74.png)

页面上有两个按钮（`foo`和`bar`），点击后会异步获取数据，并将数据渲染到结果区域（斜体区域）。因为获取数据的过程一般是异步的，如`AJAX`、`fetch`，因此结果返回的顺序与请求的顺序并不能保证一致。

这个例子（[Demo](http://jsbin.com/daboxap/71/edit?html,js,console,output)）展示了当没有正确处理异步请求的时候会带来显示的错乱。

```
// foo
-------req-----------------------------------------res--------------
    第一次点击                                 渲染第一次点击的数据
// bar
-----------------req-----------------res----------------------------
               第二次点击        渲染第二次点击的数据
```

事实上，在第二次点击后，应当有办法能“取消”对第一次点击产生的异步操作。

由此，引入了本文的主题：`Observable`。

# Observable

*Observables are lazy Push collections of multiple values.*

*Observable模型让你可以像使用集合数据一样操作异步事件流，对异步事件流使用各种简单、可组合的操作。*

*They fill the missing spot in the following table:*

*它填补了这个表格的空白：*

|         |Single    |Multiple   |
|---------|----------|-----------|
|Pull     |Function  |Iterator   |
|Push     |Promise   |Observable |

这是一个简单的（[Demo](http://jsbin.com/qagukij/edit?js,console)）：

```javascript
Rx.Observable.interval(1000)
  .take(5)
  .map(data => [data, data * data])
  .subscribe({
    next: data => console.log(data),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

也许你注意到了，我们使用了`Rx.Observable`，这是`RxJS`的一部分。

因为`Observable`目前还未成为ECMAScript标准，因此有各种它的实现。如[RxJS](https://github.com/ReactiveX/rxjs)、[Rx](https://github.com/Reactive-Extensions/RxJS)、[XStream](https://github.com/staltz/xstream)、[Most.js](https://github.com/cujojs/most)。`RxJS`之于`Observable`，就如同`Bluebird`之于`Promise`。

在这之中，`RxJS 5`完全重构了`Rx 4`的代码和接口，并且兼容了`Observable`规范，因此被独立出来。

**本文为与`Observable`规范兼容，所有代码将基于`RxJS 5`编写。**

>![RxJS](https://cloud.githubusercontent.com/assets/1812118/20669679/69a646d8-b5b0-11e6-9dc7-0d5c606e22aa.png)

> RxJS 5目前处于Release Candidate版本，接口已基本稳定。

>`Observable`目前进入 [ECMAScript Stage 1 Draft](https://tc39.github.io/proposal-observable/)。

## 创建数据流

*落其实者思其树，饮其流者怀其源。——庾信《徵调曲》*

`Observable`提供了多种产生数据流的方法。

### Observable.from
`Observable.from`可以从数组产生数据流：[Demo](http://jsbin.com/refomiv/edit?js,console)；`Observable.of`类似，只是参数不同：[Demo](http://jsbin.com/jelamo/edit?js,console)。

```javascript
Rx.Observable.from([1, 2, 3])
  .subscribe({
    next: data => console.log(data),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

### Observable.fromEvent

`Observable.fromEvent`可以从事件产生数据流：[Demo](http://jsbin.com/dasiwo/edit?html,js,console,output)。

```javascript
const button = document.querySelector('button');
Rx.Observable.fromEvent(button, 'click')
  .subscribe({
    next: event => console.log(event.target.innerHTML),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

### Observable.fromPromise

`Observable.fromPromise`可以从标准的`Promise`对象产生数据流：[Demo](http://jsbin.com/tehoxo/edit?js,console,output)。

```javascript
const promise = fetch('https://api.github.com/users/malash');
Rx.Observable.fromPromise(promise)
  .subscribe({
    next: data => console.log(data.status),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

### new Observable
除了使用现有的`.from*`方法，通过`new`实例化Observable对象，可以产生自定义的数据流：[Demo](http://jsbin.com/vikeqof/edit?js,console,output)。

```javascript
new Rx.Observable((observer) => {
  observer.next(1);
  observer.next(2);
  observer.next(3);
  setTimeout(() => observer.next(4), 1000);
  setTimeout(() => observer.next(5), 2000);
  setTimeout(() => observer.complete(), 5000);
})
  .subscribe({
    next: data => console.log(data),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

`Observer`是一个接口（interface），它的定义如下：
```javascript
interface Observer<T> {
  closed?: boolean;
  next: (value: T) => void;
  error: (err: any) => void;
  complete: () => void;
}
```

通过同步或异步的调用`Observer`的方法即可产生数据流。

## 订阅数据流

### Observable.prototype.subscribe

也许你注意到了，对于Observable创建的每一条数据流，我们都使用了`.subscribe`方法进行订阅。`.subscribe`方法接受一个参数：`observer`，它也是`Observer`接口的实现，因此也有`.next`、`.error`、`.complete`方法。

```
someObservable
  .subscribe({
    next: data => console.log(data),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

> `.subscribe`也接受三个参数调用方式，在一些文档中你会看到这种用法：
>`Observable.prototype.subscribe([nextCallback], [errorCallback], [completeCallback])`

### Subscription.prototype.unsubscribe

数据流可以订阅，也可以取消订阅。

调用`Observable`的`.subscribe`方法后后，将返回一个`Subscription`实例。调用它的`.unsubscribe`方法可以实现取消订阅：[Demo](http://jsbin.com/codelo/edit?js,console)。

```
const subscription = Rx.Observable.interval(1000)
  .subscribe({
    next: data => console.log(data),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });

setTimeout(() => {
  subscription.unsubscribe();
}, 3000);
```

### 惰性求值

`Promise`不是[惰性求值](https://zh.wikipedia.org/wiki/%E6%83%B0%E6%80%A7%E6%B1%82%E5%80%BC)的，当一个`Promise`对象被创建时就开始运行了，即使它的`.then`方法没有被调用。

在这个例子中，只要`fetch`被运行，HTTP请求就已经被发送了：[Demo](http://jsbin.com/xuxaku/edit?js,output)。

```javascript
fetch('https://api.github.com/users/malash');
```

`Observable`是惰性求值的，只有一个`Observable`实例被调用`.subscribe`，才会真的运行其构建方法中的代码：[Demo](http://jsbin.com/xitoye/edit?js,console)。

```javascript
const observable = new Rx.Observable((observer) => {
  console.log('emit data');
  observer.next('hello');
})

console.log('observable created');

setTimeout(() => {
  observable
    .subscribe({
      next: data => console.log(`get data: ${data}`),
      error: e => console.error(e),
      complete: () => console.log('complete')
    });
}, 3000);
```

> 准确来讲，

>在 [Promises/A+](https://promisesaplus.com/) 中并没有规定`Promise`的运行时机，

>在 [ECMAScript 2015 (ES6)](http://www.ecma-international.org/ecma-262/6.0/#sec-promise-objects) 中规定了`Promise`创建后会被立刻执行；

>在 [ECMAScript Stage 1 Draft](https://tc39.github.io/proposal-observable/) 中规定了`Observable`只有在`subscribe`后才会执行。

##  变换数据流

也许你注意到了，在本文的第一个（[Demo](http://jsbin.com/qagukij/edit?js,console)）中，我们使用了`.take`和`.map`这两个方法实现了对数据流的变换，`.take`截取数据流中的前N个数据，`.map`对数据流的每个元素进行映射。

这些方法被称为运算符（operators），它是`Observable`千变万化的基础。`operators`的设计与`Array.prototype`类似，使用运算符可以像操作数组一样操作数据流。

`RxJS`提供了[近百个operators](http://reactivex.io/rxjs/class/es6/operator/groupBy.js~GroupedObservable.html)，甚至还有[工具帮你选择合适的operators](http://reactivex.io/rxjs/manual/overview.html#choose-an-operator)。鉴于`operators`数量过多不能一一展示，我将选择几个典型的应用场景对`operators`进行演示。

### 累加器

实现一个简单的累加器，每次点击按钮时将结果加`1`，并在前端展示：[Demo](http://jsbin.com/geqiyac/edit?html,js,output)。

```javascript
const button = document.querySelector('button');
const span = document.querySelector('span');

Rx.Observable.fromEvent(button, 'click')
  .map(event => 1)
  .scan((count, data) => count + data)
  .subscribe({
    next: data => {
      span.innerHTML = data;
    },
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

与数组不同，因为流数据常常是没有终止（complete）的，因此对`Observable`可以使用`.scan`替代`.reduce`。

`.reduce`需要触发`.complete`时才停止计算返回结果：

```
// source
------------1--------------2--------------3-----------|
// result
------------------------------------------------------6|
```

`.scan`将返回每一次运算的结果：

```
// source
------------1--------------2--------------3-----------
// result
------------1--------------3--------------6-----------
```

因此使用`.scan`可以实时将结果渲染到前端。

### 带节流的累加器

*永远不要相信用户的输入。*

一个不加限制的按钮是很危险的，特别是在它能触发网络请求的时候，连续高频的点击最终会导致后端服务的雪崩。因此我们需要限制这个按钮的点击频率：[Demo](http://jsbin.com/sedinis/edit?html,js,output)。

```javascript
Rx.Observable.fromEvent(button, 'click')
  .throttleTime(1000)
  ...
```

throttle与debounce都是前端开发常用的节流方法，`RxJS`中有`.throttleTime`与`.debounceTime`进行了实现。

### 多加数累加器

现在我们希望拓展一下累加器，它提供三个按钮，点击后可以累加不同的值。

也许一个最简单的办法是将代码复制三份：[Demo](http://jsbin.com/xasuze/edit?html,js,output)。

但简单试一试会发现存在着Bug：当点击不同按钮时计算结果并不正确。

这是因为每一个`.scan`运算符内部都保留该数据流的累加状态，三条数据流的状态彼此是隔离的，只是最后渲染到了同一个DOM结点。

我们需要有办法能合并三条数据流：[Demo](http://jsbin.com/nenozuj/edit?html,js,output)。

```javascript
const $button1 = Rx.Observable.fromEvent(button1, 'click');
const $button10 = Rx.Observable.fromEvent(button10, 'click');
const $button100 = Rx.Observable.fromEvent(button100, 'click');
Rx.Observable.merge($button1, $button10, $button100)
  .map(event => parseInt(event.target.dataset.value))
  ...
```

`Observable.merge`提供了合并数据流的功能，各个数据流中的元素将按照时间顺序组成新的数据流。

```
// A
----------A---------A---------A---------------------------------------
// B
---------------------------B---------B--------B-----------------------
// C
------------------------------------------C--------C-------C----------
// result
----------A---------A------B--A------B----C---B----C-------C----------
```

此外，由于数据流合并后无法区分被点击的按钮是哪一个，因此可以通过DOM的`data`属性绑定数值，并使用`.map(event => event.target.dataset.value)`获取数值。

### 带自动补全的搜索框

实现这样一个搜索框：随着用户的不断输入，自动将建议搜索结果展现出来。


![美团网的搜索框](https://cloud.githubusercontent.com/assets/1812118/20669723/b6454624-b5b0-11e6-8993-f9206a875622.png)

这是一个常见的应用场景，虽然看起来简单，但其中有很多细节需要注意。比如：实现AJAX（或fetch）请求、限制请求频率、并且避免异步造成错误渲染——如本文开篇所遇到的那样。

这里以Github的搜索为例，给出了一个[Demo](http://jsbin.com/didewu/edit?html,js,output)。

在这个Demo中，使用了`.switchMap`运算符，但为了便于理解，我们先看一下`.mergeMap`。

`.mergeMap`实现了两件事情：

1. 将数据流中的每一个元素映射到一条数据流

2. 将这些数据流合并

```
// source
-------------A----------------B--------------C----------------------------
// A => A'
             A---------A---------A-----|
// B => B'
                              B---------B--------B-----|
// C => C'
                                             C--------C-------C------|
// result
-------------A---------A------B---A-----B----C---B----C-------C----------
```

`.switchMap`与`.mergeMap`类似，但所做的不是“合并”，而是“覆盖”：

新的数据流一旦产生，老的数据流就不会再进入结果中了。

```
// source
-------------A----------------B--------------C----------------------------
// A => A'
             A---------A---------A-----|
// B => B'
                              B---------B--------B-----|
// C => C'
                                             C--------C-------C------|
// result
-------------A---------A------B---------B----C--------C-------C----------
```

依靠`.switchMap`可以避免因异步请求响应时间不一致导致的错误渲染问题。

### 自定义运算符

通过这几个示例，我们可以了解到了`RxJS`运算符的丰富与强大。对于常见数据流操作这些运算符已经足够，但我们仍需了解下如何实现自定义运算符，这有助于理解运算符的原理。

我们需要实现一个`multiplyByTen`运算符，它将数据流中的每一个元素乘以10。如果用`map`运算符它将非常简单：`.map(data => data * 10)`，但在这里我们需要手动实现它。

首先我们实现了一个`multiplyByTen`函数，它接收一个`Observable`实例，并返回一个新的`observable`实例：[Demo](http://jsbin.com/gonito/edit?js,console)。

```javascript
function multiplyByTen($input) {
  return Rx.Observable.create((observer) => {
    $input.subscribe({
      next: (data) => observer.next(data * 10),
      error: (err) => observer.error(err),
      complete: () => observer.complete()
    });
  });
}

multiplyByTen(Rx.Observable.interval(1000))
  .subscribe({
    next: data => console.log(data),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

`multiplyByTen`函数订阅了`$input`数据流，在`.subscribe`中实现了乘以10的计算，并将结果传递给`observer.next`，实现了数据流的转换。

>`$input`的前缀`$`代表这是一个`Observable`实例，这种写法参考了[Cycle.js](https://github.com/cyclejs/cyclejs)。

`multiplyByTen`函数现在和运算符还有一点小的区别——调用方式，如果我们将`multiplyByTen`函数添加到`Observable.prototype`中，并将`this`赋值给`$input`，就得到了真正的运算符：[Demo](http://jsbin.com/fudehec/edit?js,console)

```javascript
Rx.Observable.prototype.multiplyByTen = function() {
  const $input = this;
  return Rx.Observable.create((observer) => {
    $input.subscribe({
      next: (data) => observer.next(data * 10),
      error: (err) => observer.error(err),
      complete: () => observer.complete()
    });
  });
}

Rx.Observable.interval(1000)
  .multiplyByTen()
  .subscribe({
    next: data => console.log(data),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

当然，修改`prototype`并不是一个很好的选择，使用`::`（[bind operator](https://github.com/tc39/proposal-bind-operator)）绑定`this`运行更合适：[Demo](http://jsbin.com/loniboj/edit?js,console)。

```javascript
const multiplyByTen = function() {
  const $input = this;
  return Rx.Observable.create((observer) => {
    $input.subscribe({
      next: (data) => observer.next(data * 10),
      error: (err) => observer.error(err),
      complete: () => observer.complete()
    });
  });
}

Rx.Observable.interval(1000)
  ::multiplyByTen()
  .subscribe({
    next: data => console.log(data),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

在`RxJS 5`进行重构后，所有的运算符都可以使用`::`调用：

```javascript
import { Observable } from 'rxjs/Observable';
import { of } from 'rxjs/observable/of';
import { map } from 'rxjs/operator/map';

Observable::of(1,2,3)::map(x => x + '!!!'); // etc
```

## Subject

### 单播与广播

前文讲到，只有当调用`.subscribe`时才运行初始化函数。因此当多次`.subscribe`同一个`Observable`实例时，初始化函数将被调用多次：[Demo](http://jsbin.com/baciqem/edit?js,console,output)。

```javascript
const observable = new Rx.Observable((observer) => {
  console.log('start');
  observer.next(1);
});

observable  
  .subscribe({
    next: data => console.log(data),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });

observable  
  .subscribe({
    next: data => console.log(data),
    error: e => console.error(e),
    complete: () => console.log('complete')
  });
```

可以看出，`Observable`是“单播”的，每个`.subscribe`拥有自己的初始化函数运行状态，彼此之间互相独立。

在`RxJS`中，还提供了用于实现“广播”的类：`Subject`。

### Subject

首先，`Subject`也是一种`Observable`。因此可以调用`.subscribe`订阅数据流，特殊的是`Subject`的`subscribe`会维护一组`subscribion`——如同`EventEmitter`的`addListener`一样——每当有新数据时将依次调用这些`subscribion`，实现了广播。

这是一个简单的例子：[Demo](http://jsbin.com/bepuwuh/edit?js,console,output)。

```javascript
const subject = new Rx.Subject();

subject.subscribe({
  next: data => console.log('observerA', data),
  error: e => console.error(e),
  complete: () => console.log('complete')
});
subject.subscribe({
  next: data => console.log('observerB', data),
  error: e => console.error(e),
  complete: () => console.log('complete')
});

subject.next(1);
subject.next(2);
```

其次，`Subject`也是一种`Observer`。通过调用`.next`、`.error`、`complete`方法，可以向数据流中写入新数据。这意味着`Subject`可以作为另一个`Observable`的`.subscribe`方法的参数：[Demo](http://jsbin.com/figehip/edit?js,console,output)。

```javascript
const subject = new Rx.Subject();

// subject.subscribe(...)

const observable = Rx.Observable.interval(1000);

observable.subscribe(subject);
```

这可以很方便的将一个单播的`Observable`转化为广播的`Subject`。

### 冷与热

*逝者如斯夫，不舍昼夜。——孔子《论语》*

`Subject`除了将“单播”变化为“广播”，还将数据流从“冷”（`cold`）变“热”（`hot`）。

调用`.subscribe`与`.next`的顺序很重要，否则会漏掉对数据的订阅。在这个例子中，`observerA`能输出`1`和`2`，而`observerB`只能输出`2`：[Demo](http://jsbin.com/cokexo/edit?js,console,output)。

```javascript
const subject = new Rx.Subject();

subject.subscribe({
  next: data => console.log('observerA', data),
  error: e => console.error(e),
  complete: () => console.log('complete')
});

subject.next(1);

subject.subscribe({
  next: data => console.log('observerB', data),
  error: e => console.error(e),
  complete: () => console.log('complete')
});


subject.next(2);
```

“热”的数据流无论是否被订阅，它都将初始化，并开始产生数据；而“冷”的数据流是惰性求值的，被订阅后才产生数据。

“冷”、“热”数据流在有着不同的适用场景，例如这里有一个简单的`WebSocket`通信例子：[Demo](http://jsbin.com/goqabi/edit?js,console,output)。

```javascript
const observable = new Rx.Observable((observer) => {
  const socket = new WebSocket('wss://echo.websocket.org');
  socket.addEventListener('message', (e) => observer.next(e));
  socket.addEventListener('open', () => socket.send('hello'));
  return () => socket.close();
});

observable.subscribe({
  next: event => console.log('observerA', event.data),
  error: e => console.error(e),
  complete: () => console.log('complete')
});

observable.subscribe({
  next: event => console.log('observerB', event.data),
  error: e => console.error(e),
  complete: () => console.log('complete')
});
```

由于使用了“冷”数据流，每次调用`.subscribe`时都会重新创建一条`WebSocket`连接，很显然这样做浪费了不必要的网络消耗，因为我们只是希望同样的数据能够多次订阅。

使用“热”数据流将更加合适：[Demo](http://jsbin.com/zuwunu/edit?js,console,output)。

这里使用了`.share`运算符，它实现的功能与`Subject`相同。通过使用“热”数据流，仅需要创建一条`WebSocket`连接即可实现多次订阅。

### ReplaySubject

需要注意的是，如果在“热”数据流开始产生数据后才进行`.subscribe`调用，将可能错过数据，通过这个例子可以很清楚的看到这个问题：[Demo](http://jsbin.com/luvawi/edit?js,console,output)。

因此`RxJS`提供了带“回放”功能的`Subject`——`ReplaySubject`。

`ReplaySubject`支持两种回放策略——数量或时间，这是一个根据数量回放的例子，它将提供回放最后3个数据的功能：[Demo](http://jsbin.com/zuwunu/edit?js,console,output)。

```javascript
const subject = new Rx.ReplaySubject(3);

subject.subscribe({
  next: data => console.log('observerA', data),
  error: e => console.error(e),
  complete: () => console.log('complete')
});

subject.next(1);
subject.next(2);
subject.next(3);
subject.next(4);

subject.subscribe({
  next: data => console.log('observerB', data),
  error: e => console.error(e),
  complete: () => console.log('complete')
});

subject.next(5);
```

而这是一个根据时间窗口回放的例子：[Demo](http://jsbin.com/yakayi/edit?js,console,output)。

**注意：不论使用哪种回放策略，`ReplaySubject`的第一个参数——缓存大小——非常重要，它的默认值为`Infinity`。一个缓存长度为`Infinity`的`ReplaySubject`很容易因忘记调用`.unsubscribe`而产生内存泄露。**

## 总结

本文以`Observable`为核心介绍了`RxJS`的简单用法，了解了如何创建、变换、订阅一个数据流，以及`RxJS`的各种使用样例。

正如`RxJS`的slogan所说：*Think of RxJS as Lodash for events*，使用数据流可以让我们对常见异步操作——特别是事件处理的得心应手。因此不止是`JavaScript`，`Rx`在很多语言都有对应的实现：[Rx.NET](https://github.com/Reactive-Extensions/Rx.NET)、 [RxJava](https://github.com/ReactiveX/RxJava)、[RxSwift](https://github.com/ReactiveX/RxSwift)、[RxScala](https://github.com/ReactiveX/RxScala)、[RxPHP](https://github.com/ReactiveX/RxPHP)及其他语言实现，还有类似`Rx`的[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)。

但是，数据流应用价值显然不止本文所述。利用数据流我们可以使用“函数响应式编程”构建应用，如 [Cycle.js](https://cycle.js.org/)、[Elm](http://elm-lang.org/)，在接下来的文章里将会做更详细的介绍。

一个`Cycle.js`的示例：

```javascript
import Cycle from '@cycle/rxjs-run';
import {makeDOMDriver} from '@cycle/dom';
import {html} from 'snabbdom-jsx';

function main(sources) {
  return {
    DOM: sources.DOM.select('.myinput').events('input')
      .map(ev => ev.target.value)
      .startWith('')
      .map(name =>
        <div>
          <label>Name:</label>
          <input classNames="myinput" type="text" />
          <hr />
          <h1>{`Hello ${name}`}</h1>
        </div>
      )
  };
}

Cycle.run(main, {
  DOM: makeDOMDriver('#main-container')
});
```
