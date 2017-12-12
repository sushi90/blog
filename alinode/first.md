# 如何用alinode来优化你的node程序
使用nodejs最令人头疼的一个问题就是内存泄露的问题。我们常常会发出感叹，测试环境是好的啊，没有为释放的资源和闭包啊。为什么一到生产环境，内存就和疯了似的，涨涨涨涨涨！
相信很多人和我一样，会使用各种npm来查问题，例如node-heapdump，node-memwatch...。这些包使用起来并不难，但是需要手动的去profile堆快照。有的会说，so stupid！你可以监控内存啊，到达一定的内存量，你就可以让它自动profile啊。但是你怎么知道，是不是因为当时的连接数就是很大，就应该是消耗那么多的内存。即使你profile到了堆快照，通过chrome打开了堆快照，你会发现，这仅仅是开始。因为你需要明白各种参数的意义,如图
![v8 profile](http://7qnbf7.com1.z0.glb.clouddn.com/v8profile.png)。不可否认你可以通过[google开发者文档](https://developers.google.com/web/tools/chrome-devtools/profile/?hl=zh-cn)了解到并且可以慢慢定位，最后或许可以解决问题。但是此时可能已经花费了好几天的时间，需要做的业务需求也已经堆了一大堆了。而你仅仅只想优化你的nodejs程序，提高它的效率。但你却降低了自己的开发效率效率。

那么问题和重点来了，如何能高效准确的定位到nodejs的cpu负载过高，内存持续泄露这些问题呢，[alinode](https://alinode.aliyun.com/)这款工具应运而生。下面我将详细介绍我是如何利用alinode来优化自己的node程序的。

## install
可以使用参考[官方文档](https://alinode.aliyun.com/doc/deploy)里的交互式部署，一键安装 alinode 服务。然后根据提示一步一步即可完成 alinode 运行，方便快捷。注意选择自己适合的alinode版本，这里有alinode和官方node的[版本对照表](https://alinode.aliyun.com/doc/alinode_versions)。因为安装alinode，它会给你安装一个他们改进过的node版本，里面暴露出了官方node没有暴露出来的一些v8和libuv等底层的信息，用来实时采集node进程的信息，并上传的alinode服务器进行分析。当然这些修改并不影响node的性能和各项功能。

## run
先`which node`来看一下是否已经将默认的node设置成alinode的node。如果显示如`/home/work/.tnvm/versions/alinode/v0.3.4/bin/node`，则设置成功。
接着启动你的程序，然后运行`nohup agentx <yourpath>/yourconfig.json &`。那么一切就都ok啦。

## 实战
### cpu profile
接下来可以登录alinode的平台来监控你的node程序了。关于界面和功能我就不详细介绍啦，可以看[文档](https://alinode.aliyun.com/blog/27)。界面还是很漂亮的！接下来我来详细介绍一下我的优化实战过程。

我部署alinode的机器，配置是16核32G。启动了16个基于tcp的node服务进程。主要功能是提供聊天功能。包括私聊，群聊，聊天室。发现cpu的使用率在晚高峰时间会到80%，90%，非常的恐怖。于是在晚高峰的时候我截取了一个CPU Profile。在操作记录里的CPU Profile类型中，找到刚才截图的记录。点击`分析文件`。操作界面如图![cpu profile control](http://7qnbf7.com1.z0.glb.clouddn.com/cpuprofile-control_v.png)
之后就可以获取到分析后的结果页面了，如图![cpu profile no optmize](http://7qnbf7.com1.z0.glb.clouddn.com/cpuprofile-no-optmize_v.png)
从图中的可以看出来pack这个函数占了cpu的11.33%，耗时相当严重，这个操作是我用来打包数据的。而调用这个操作的有3个地方，其中画红框的的sendChatroomMsg这而是用时最长的。大致解释一下，我这个函数的功能。主要功能是当从服务器接受到一个需要广播的数据，我通过这个方法把要广播的数据进行pack打包，然后将pack过的数据进行广播。于是我找到这个段代码，进行了一下review发现了一个严重的代码逻辑问题。在实现时，我是先广播了从服务器接收到的data，然后监听这个广播事件的listener分别进行了一次打包。也就是说如果这个聊天室里有10000个人正在聊天，那么将会对同一句话打包10000....当然性能要大打折扣了。果断改之。改后再profile如图。![cpu profile optmize](http://7qnbf7.com1.z0.glb.clouddn.com/cpuprofile-optimze-1.png)pack的cpu 占用率从11.33直降到0.69%！！！再在高峰期看cpu的使用率也从80%~90%降到了30%~40%。可谓非常成功。

### 堆快照
说完cpu接下来说一说另一个令人头疼的问题，就是内存泄露的问题。同样alinode提供了方便查找内存泄露的问题。再控制台界面，我们点击`堆快照`按钮，则一个此时的堆快照就轻松的dump出来了。同样的，我们到操作记录里的堆快照类型中找到我们dump出来的堆快照，点击`分析文件`。alinode会直接将分析后的结果发送给你。如图![heapdump no optmize](http://7qnbf7.com1.z0.glb.clouddn.com/heapdump-no-optimize.png)是不是一目了然！不用去分析各种复杂的数据，不用去找对象之间的关系，一键式定位到内存溢出的位置。那么详细解释一下如果具体解决这个问题。从说明可以知道，是Manager @65893(这是一个编号，来区别相同类的不同实例)这个实例占用了过多的内存。而内存主要积累在@72201这个对象上。点击@72201，看看这到底是个什么对象。如图![heapdump no optmize-2](http://7qnbf7.com1.z0.glb.clouddn.com/heapdump-no-optmize-2.jpg)可以清楚的在最底部看到，一共创建了31356个Room对象，没有释放。这就是根源所在了，找到创建Room的地方，发现果然在一种情况下没有正确的清理掉Room。果断改之，再heapdump一下。如图![heapdump profile](http://7qnbf7.com1.z0.glb.clouddn.com/heapdump-optmize.png)，Room的数量从原来的31356降到了2308，也就是可以正常的释放了。

此外还定位到了一个tcp连接在半开状态上，没有正常释放的问题。可谓功能非常的强大。

### others
除了这些功能，alinode平台还提供了cpu使用率，内存使用率，磁盘使用率，gc，libuv句柄等一系列的监控，并用图表进行了展示，可以很好的分析node进程的状态。此外，还提供了各种监控报警功能。

## end
alinode平台在定位node问题和监控node进程方面，可谓是必备利器。可以极大的提高node程序优化和维护的效率。
这里也感谢alinode开发的各位大牛，对于我在使用过程当中遇到的问题的耐心解答和帮助。

That's all.

