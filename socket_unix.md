# Unix Domain Socket 实现原理
前一节接收了 `Socket族接口层` 的实现，那么接下来我们将会接收 `Unix Domain Socket` 的实现。`Unix Domain Socket` 用于同一台主机的不同进程间通信，不需要基于网络协议，主要是基于文件系统。
