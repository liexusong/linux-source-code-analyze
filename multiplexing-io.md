## 什么是多路复用IO
`多路复用IO (IO multiplexing)` 是指内核一旦发现进程指定的一个或者多个IO条件准备读取，它就通知该进程。常用的 `多路复用IO` 手段有 `select`、`poll` 和 `epoll`。

`多路复用IO` 主要用于处理网络请求，例如可以把多个请求句柄添加到 `select` 中进行监听，当有请求可进行IO的时候就会告知进程，并且把就绪的请求句柄保存下来，进程只需要对这些就绪的请求进行IO操作即可。下面通过一幅图来展示 `select` 的使用方式（图片来源于网络）：

![select-model](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/select-model.png)

