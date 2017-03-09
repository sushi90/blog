## timer 运行原理

timer.js中有两个对象`refedLists`，`unrefedLists`，都是用来存储`timeout`对象的，但区别在于这些`timeout`对象是否被`refed`(具体refed定义可以看event loop，简单来说，就是你的进程是否运行后会因为这个`timeout`一直运行，还是立即退出)。

`refedLists`存的是什么呢？举个例子：

```js
refedLists = {
  '40': TimesList
}

TimesList = {
  _idleNext: Timeout1,
  _idlePrev: TimesList,
  _timer: TimeWrap,
  msecs: 40
}

Timeout1 = {
  _idleNext: Timout2,
  _idlePrev: TimList,
  _onTimeout: callback,
  _idleStart: TimeWrap.now(),
  _idleTimeout: 40
}

setTimeout(() => {}, 40)
```

当我们调用`setTimeout()`函数生成一个40ms的timer的时候，首先会创建一个`Timeout1`的timer，并将回调函数绑定到`_onTimeout`属性上。接下来就是启动这个`timer`，首先通过`TimeWrap.now()`得到当前的系统时间，并作为这个`timer`的启动时间绑定到`timer`上。然后去`refedLists`里面去寻找，是否已经创建40ms延迟的`TimesList`，如果没有，则创建一个。`TimerList`对象有一个`_timer`属性，这个字段绑定是一个`TimeWrap`的实例，这是从c++类库引入的一个计时器。传入40ms, 启动这个计时器，并且绑定`listOnTimeout`方法到实例的`kOnTimeout`属性，当40ms后，就会调用`listOnTimeout`方法。取出当前时间和链表第一个`timer`对象的`_idleStart`时间，做一个差值，如果小于40ms，则触发这个执行这个`timer`的回调，否则，将差值作为下一次的计时时间，重新启动`TimeWrap`这个计时器。这样做的好处就是，40ms这个链表的第一个`timer`是先添加的，它一定会先于之后的40ms的`timer`，所以，如果第一个节点没有到达时间，后边的timer一定都没有到达时间，这样就减少了遍历次数，使效率大大提高！

