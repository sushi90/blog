## **setTimeout** vs **setInterval** vs **setImmediate** vs **process.nextTick**

如果你对这四个函数的用法和使用场景还心存疑惑，那么希望本文能帮你熟练的掌握他们。

### event loop

上来就是高大上的event loop，大家可能吃不消，但是如果你不理解event loop，那么你也不能够更好的理解timer之间的关系。可以参看nodejs官网对于[event loop][]每个阶段的讲解。这里会简单的讲一下event loop，帮助大家更好的理解上述的四个函数的触发场景。如图

![event loop flow image](http://docs.libuv.org/en/v1.x/_images/loop_iteration.png)

event loop一共分为七个阶段。当一次event loop开始时，首先会去判断当前的loop是否是alive状态(当前是否有活跃的[handles和request][]，这里不做过多的描述)，如果是，则获取当前的系统时间，防止之后的timer重复获取系统时间。

`Run due timers`阶段。这个阶段在nodejs里会执行[`Class:Timeout`][]相关的方法。`Class:Timeout`包括2个方法，`setTimeout`和`setInterval`。

`poll for I/O`阶段。因为这个阶段是一个相对比较重要的阶段，他的前后阶段，都和他有关，所以我们先来看这个阶段，有助于我们理解其他几个阶段。这个阶段的主要任务就是执行I/O操作的回调，而执行回调的过程是一个阻塞操作，所以在执行这个阶段之前，会先计算出最大的`timeout`时间，防止`poll for I/O`阶段长时间阻塞event loop，具体的计算方式可以参考[libuv官方文档][]。

`Call pending callbacks`。大部分的I/O操作的回调会在`poll for I/O`阶段被执行，但是在一些情况下，会将这些回调放到下一次loop中执行，而这些回调，会被在下一次的`Call pending callbacks`阶段执行。

`Run idle handles`。这个阶段会将idle handles执行一遍，只要这些handles处于活跃状态，每次event loop都会执行一遍。

`Run prepare handles`。会执行想在`poll for I/O`阶段之前执行的handles。

`Run check handles`。正好是和`Run prepare handles`相对应，它是执行，想在`poll for I/O`阶段之后执行的handles。

`Call close callbacks`。这个阶段主要是调用由`uv_close()`来关闭的handle的回调。

至此，基本的介绍都已经完毕了，这里的每个阶段，都对应着nodejs在运行时，不同的方法和事件的调用阶段，具体的对应关系，这里不做一一介绍，接下来的介绍，会对应以上的4中函数，在各个阶段的调用，做一个详细的说明。

### 对比说明

1.setTimeout & setInterval
这2个方法都是在一段时间后执行对应的回调，区别在于一个是执行一次， 一个是反复执行。这两个方法的运行阶段都在`Run due timers`阶段执行。但是这里有一点需要注意，就是这两个方法的执行时间并不一定会像用户期望的那样准确，有可能因为系统的调度和其它回调的执行而延迟。

```js
var fs = require('fs');

function someAsyncOperation (callback) {
  // 假设读取文件这个操作需要95ms来完成
  fs.readFile('/path/to/file', callback);
}

var timeoutScheduled = Date.now();

setTimeout(function () {

  var delay = Date.now() - timeoutScheduled;

  console.log(delay + "ms have passed since I was scheduled");
}, 100);


// 执行这个读取文件的异步操作
someAsyncOperation(function () {

  var startCallback = Date.now();

  // 做一个延迟10ms的操作
  while (Date.now() - startCallback < 10) {
    ; // do nothing
  }

});
```
上述代码主要过程是，设定了一个100ms后执行的timer，然后进行文件读取的操作，这个操作耗时95ms，95ms后读取完成，这个读取文件的回调会被放到event loop的task queue中等待执行，在`poll for I/O `阶段，会将这个回调取出，并执行，执行的过程中，整个进程处于一种阻塞的状态，回调会执行10ms，也就是说此时已经过去了105ms，但是却没有执行100ms的那个timer，因为此时进行处于一种阻塞的状态。当这个10ms的回调处理完成后，继续执行接下来的几个阶段，开始下一次的`event loop`的时候，会取出系统时间，在`Run due timer`的那个阶段，回去执行已经到了需要执行的timer，但是此时已经超时5ms了。

2.setTimeout vs setImmediate
这个两个方法都是执行一次，但他们的区别在于它们调用的阶段。`setTimeout`之前说过是在`Run due timer`阶段调用。而`setImmediate`是在`Run check handles`阶段调用的。

```js
// timeout_vs_immediate.js
setTimeout(function timeout () {
  console.log('timeout');
},0);

setImmediate(function immediate () {
  console.log('immediate');
});
```
执行结果如下

```
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```
执行的结果是不确定的。取决于你执行的时候的，进程的性能。

```js
// timeout_vs_immediate.js
var fs = require('fs')

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout')
  }, 0)
  setImmediate(() => {
    console.log('immediate')
  })
})
```
结果如下

```
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```
结果是确定的，为什么呢?因为readFile的回调会在`poll for I/O`阶段执行，会执行`setTimeout`和`setImmediate`这两个函数。当`poll for I/O`阶段完成后，紧接着就是`Run check handles`，这时就会`setImmediate`的回调，所以，`setImmediate`总是先于timer被调用。

3.process.nextTick()
这个函数，特别的特殊，也很容易让人迷惑他的作用。他并不是在上述的某一个阶段被调用，因为它并不属于event loop中定义的东西。有一个`nextTickQueue`队列，在当前所处的event loop阶段的操作完成之后，将会执行这个队列中的回调，无论你现在处于event loop中的哪个阶段。为什么会有他的存在呢？因为这涉及到一个api涉及的哲学问题，看如下的代码片段。

```js
// WARNING!  DO NOT USE!  BAD UNSAFE HAZARD!
function maybeSync(arg, cb) {
  if (arg) {
    cb();
    return;
  }

  fs.stat('file', cb);
}
```
这段代码摘自nodejs官方文档。可以看到这个方法在`arg`这个参数为true的时候，会直接调用`cb()`，他是一个同步的方法。而在`arg`这个参数为false的时候，回去调用一个fs.stat这个方法，这是一个异步的方法。这样就导致了，这个方法没有办法确定是同步的，还是异步的。看如下的调用代码

```js
var a = 1;

maybeSync(true, () => {
  foo();
});
bar();

function foo () {
  console.log(a)
}

function bar () {
  a = 2;
}
```
上述代码的结果是1，而不是2，原因就在于当传入的`arg`参数为true的时候，它其实是一个同步函数，`foo()`这个函数会被立即调用，而如果参数为`false`的时候，`maybeSync`会被当做一个异步函数来调用，`bar()`会先于`foo()`调用，结果则为2。这就导致了一个api函数，却会呈现2中不同的调用状态，这非常的危险。但是如果将代码改为如下方式，那么这个api，将会一直呈现成一种异步的调用状态。

```js
var a = 1;

maybeSync(true, () => {
  process.nextTick(foo)
});
bar();

function foo () {
  console.log(a)
}

function bar () {
  a = 2;
}
```

同样的，如果你想在实例化一个对象之后，调用他实例化之后的方法，那么你依然需要用的process.nextTick()。
```js
function MyThing(options) {
  this.setupOptions(options);

  process.nextTick(() => {
    this.startDoingStuff();
  });
}

var thing = new MyThing();
thing.getReadyForStuff();
```
这样这段代码将会被正确的执行。而如果你在`MyThing()`这个构造函数中，直接调用`this.startDongStuff()`这个方法，则会报错。因为这个实例化化操作并没有完成。

此外，process.nextTick()还有一个点需要注意，递归的设置process.nextTick的callback会引起I/O的阻塞，因为它会先执行不断执行`next tick queue`中的回调，从而阻塞到event loop其它阶段的执行。

4.`setImmediate()` vs `process.nextTick()`
首先它们的名字让人非常的费解。`setImmediate()`是立即执行的意思，但事实上它是在`Run check handles`阶段才会被执行。而`process.nextTick()`其实并没有到所谓的next tick才执行，而是在当前的阶段执行完，立即就会被执行。所以看起来，它俩的名字，应该反过来，会更合适一些。但是官方给出的解释是，因为0.9.1才加入了`setImmediate`这个方法，而`process.nextTick`此时已经被大规模的应用，而也有越来越多的npm包在使用，所以替换名字这个显然是不安全的，索性就这样吧。但是官方建议的是，开发者在所有的场景下都用`setImmediate`来替代`process.nextTick()`。优点在于它可以和浏览器端的JavaScript保持一致，其次它的递归调用，并不不会阻塞I/O，因为`setImmediate`回调里的`setImmediate`会在下一次的event loop中被调用，而不会被立即调用。

最后，推荐一个辅助你理解event loop的[视频][]，演示的很生动形象！

参考：

[The I/O loop][]

[event-loop-timers-and-nexttick][]

[event loop]: https://nodejs.org/it/docs/guides/event-loop-timers-and-nexttick/#event-loop-explained
[The I/O loop]: [http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop]
[event-loop-timers-and-nexttick]: [https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/]
[handles和request]: http://docs.libuv.org/en/v1.x/design.html#handles-and-requests
[`Class:Timeout`]: https://nodejs.org/dist/latest-v6.x/docs/api/timers.html#timers_class_timeout
[libuv官方文档]: http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop
[视频]: https://www.youtube.com/watch?v=8aGhZQkoFbQ