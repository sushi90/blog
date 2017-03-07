## **setTimeout** vs **setInterval** vs **setImmediate** vs **process.nextTick**

如果你对这四个函数的用法和使用场景还心存疑惑，那么希望本文能帮你熟练的掌握他们。

###event loop
上来就是高大上的event loop，大家可能吃不消，但是如果你不理解event loop，那么你也不能够更好的理解timer之间的关系。可以参看nodejs官网对于[event loop][]每个阶段的讲解。这里会简单的讲一下event loop，帮助大家更好的理解上述的四个函数的触发场景。如图

![event loop flow image](http://docs.libuv.org/en/v1.x/_images/loop_iteration.png)

当一次event loop开始时，首先会去判断当前的loop是否是alive状态(当前是否有活跃的[handles和request][]，这里不做过多的描述)，如果是，则获取当前的系统时间，防止之后的timer重复获取系统时间。接下来就是`Run due timers`阶段，这个阶段在nodejs里会执行[`Class:Timeout`][]相关的方法。`Class:Timeout`包括2个方法，`setTimeout`和`setInterval`。这里让我们先跳到第五个阶段`poll for I/O`阶段，因为这个阶段是一个相对比较重要的阶段，他的前后阶段，都和他有关，所以我们先来看这个阶段，有助于我们理解其他几个阶段。这个阶段的主要任务就是执行I/O操作的回调，而执行回调的过程是一个阻塞操作，所以在执行这个阶段之前，会先计算出最大的`timeout`时间，防止`poll for I/O`阶段长时间阻塞event loop，具体的计算方式可以参考[libuv官方文档][]。回到第二阶段`Call pending callbacks`，大部分的I/O操作的回调会在`poll for I/O`阶段被执行，但是在一些情况下，会将这些回调放到下一次loop中执行，而这些回调，会被在下一次的`Call pending callbacks`阶段执行。第三阶段是`Run idle handles`，这个阶段会

参考：

[The I/O loop][]

[event loop]: https://nodejs.org/it/docs/guides/event-loop-timers-and-nexttick/#event-loop-explained
[The I/O loop]: [http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop]
[handles和request]: [http://docs.libuv.org/en/v1.x/design.html#handles-and-requests]
[`Class:Timeout`]: https://nodejs.org/dist/latest-v6.x/docs/api/timers.html#timers_class_timeout
[libuv官方文档]: http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop