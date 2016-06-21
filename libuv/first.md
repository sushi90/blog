#关于libuv的分享
首先大致介绍一下我的理解，大家都知道，libuv的工作原理就是利用不同平台的系统时间通知机制来进行非阻塞io的。libuv会运行一个叫event loop的循环。这个循环的作用就是处理各种已经完成的事件的。线面我就介绍一下，这个循环的完成过程。官方给出的文档中对每一步都做出了一定的说明，这里我将结合nodejs，进行更详细的说明，有助于大家理解。
1.第一步就是就是获取当前的系统时间，主要用于之后的timer判断。这里的timer对于nodejs，就是我们所写的setInterval和setTimeout。通过这个系统时间，我们可判断是不是该执行我们的定时器的回调了。
2.接下来就正式进入到循环了。首先我们要判断，这个循环是否是alive的，也就是这个循环是否到了他应该停止的时候了，如果该停了，就不继续进行，直接退出，如果是alive的，则继续执行。但怎么算这个循环是alive的呢？？这里面有3个判断的条件。
## 第一有active，并且已经ref的handle。handle指的是long alive object，active指的是这个这个handle被激活了。ref值得是这个handle被加入到了event loop的ref里了。那么那些object是handle的呢？这个你可以去看libuv里的文档。举个例子，在nodejs中的tcp,udp,http都是handle，因为他们被active之后，都是会长期存在于内存中接受各种请求的对象。而你在nodejs的文档中经常会看到有的对象，有一个方法是ref,unref，其实他的作用就在于将这个handle总event loop的ref列表中，删除，如果一个handle是这个event loop中唯一个handle，那么将他unref后，这个event loop就会推出，从而整个nodejs的进程也就结束了。
## 第二有active的request。一些handle的具体操作。比如创建了一个一个tcp的handle。之后进行listen这个操作，就是一个request。
## 第三就是closing handles。顾名思义就是正在close handle。

3.判断如果event loop是alive的。那么接下了就是进行具体的操作了。首先是run timers。这里会将所有的timers和之前获取的系统时间进行对比。如果到达需要执行的时间，则会执行。
4.pending cbs。大部分情况下I/O操作的callback会在polling for I/O的时候被调用。但是有一种情况下会在此时调用，那就是在上一次event loop的时候，deferred这个I/O 的cb到下一的 loop iteration。这个被defered的cb就会在下一次的event loop的pending cbs期间被执行到。
5.run idle handles。如果evnet loop里面有idle handle，并且这个handle是active的，那么每次的loop 的iteration都会调用这个handle。
6.run prepare handles。有些handle就是prepare的handle，这些handle会在此时调用，也就是在polling for I/O前被调用。作用就是，如果你想在polling for I/O前调用某些handle，那么请在此时进行操作。
7.polling for I/O。这部分会调用所有被监控的，进行读读写操作的fd的cb。在调用之前需要先计算一下，这次的polling的timeout时间。有一些case会导致timeout为0。这个时候就不进行polling for I/O的操作。如果timeout不是0。timeout就是最接近的一个timer的time时间。如果没有最接近的timer的时间，那么timeout的时间就是无限期的。
8.run check handles。这个操作实际上是和prepare的操作相对的。如果你有什么操作是想在polling for I/O之后做的，就在这个时候做的。
9.call close cb。如果一个handle通过uv_close来关闭。则在这个时候调用的它的close cb。