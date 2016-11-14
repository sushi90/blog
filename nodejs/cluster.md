# 关于cluster负载均衡整理

最近认真研究了一下cluster相关的内容，再此做个总结，供大家参考。

## cluster模块作用
充分利用多核系统的性能。

## 如何实现的
通过NodeJs官方文档：[How It Works](https://nodejs.org/dist/latest-v6.x/docs/api/cluster.html#cluster_how_it_works) 首先，用cluster模块创建一个父进程master。然后，master通过cluster模块的fork方法来创建出一定数量的子进程worker。master和woker之间可以通过IPC进行通讯和传递server handles。cluster提供了2种方法分发连接。
第一种是round-robin，也是默认的分发方式（windows除外）。master直接接受连接。利用轮训的方式进行分发。这种分发方式的好处是负载比较均衡，但是缺点是master进程的压力会比较的大。
第二种是master创建socket并共享给所有的worker，由worker自己来接受连接。这种方式理论上应该有更好的性能，并且负载也更为均衡。但是[实际上并不是这样的](https://strongloop.com/strongblog/whats-new-in-node-js-v0-12-cluster-round-robin-load-balancing/)。因为这种方式将哪个worker接受连接的调度交给了操作系统，而操作系统的调度收到上下文切换等影响，最终会导致，大部分的连接会被负载到2-3个进程上，达不到真正的负载均衡。

## 实际应用
一篇[关于NodeJs三种负载均衡的性能测试](https://medium.com/@fermads/node-js-process-load-balancing-comparing-cluster-iptables-and-nginx-6746aaf38272#.z20x6nv1s)
大致结论是：
使用Nginx：缺点是会消耗双倍的socket文件描述符，自身运行也消耗一定的内存。
使用cluster模块：简单易用，但是cluster的压力比较大，QPS也比另外两种来的弱些。
使用iptables：配置起来稍微繁琐写，灵活性差些。
