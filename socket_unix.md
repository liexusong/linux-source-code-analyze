# Unix Domain Socket 实现原理
上一节介绍了 `Socket族接口层` 的实现，接下来我们将会介绍 `Unix Domain Socket` 的实现。

`Unix Domain Socket` 是一种进程间通信方式(IPC)，用于同一台主机的不同进程间通信，不需要基于网络协议，主要是基于文件系统。而且 `Unix Domain Socket` 是一种全双工的通信方式，就是可以通过一个 `socket` 句柄来进行读写操作。
