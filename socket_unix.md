# Unix Domain Socket 实现原理
上一节介绍了 `Socket族接口层` 的实现，接下来我们将会介绍 `Unix Domain Socket` 的实现。

`Unix Domain Socket` 是一种进程间通信方式(IPC)，用于同一台主机的不同进程间通信，不需要基于网络协议，主要是基于文件系统。而且 `Unix Domain Socket` 是一种全双工的通信方式，就是可以通过一个 `socket` 句柄来进行读写操作。

## 创建 Unix Domain Socket
现在我们先来介绍一下在用户态怎么创建和使用 `Unix Domain Socket` 吧，先来介绍一下服务端的创建：
```cpp
#define UNIX_SOCKET_PATH "/path/to/unixsocket"

int main(int argc, char *argv[])
{
    ...
    struct sockaddr_un addr;
    ...
    int sock = socket(AF_UNIX, SOCK_STREAM, 0);
    strcpy (addr.sun_path, UNIX_SOCKET_PATH);
    ...
    bind(sock, (structsockaddr *)&addr, sizeof(struct sockaddr_un));   
    ...
    listen(sock, 100);
    ...
    while (1) {
        ...
        newsock = accept(sock, 0, 0);
        ...
        rval = read(newsock, buf, 1024));
        ...
    }
}
```
