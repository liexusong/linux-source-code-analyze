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
从上面的代码可知，要创建一个 `Unix Domain Socket`，首先在调用 `socket()` 函数时第一个参数传入 `AF_UNIX`，这样就可以创建一个 `Unix Domain Socket`。然后调用 `bind()` 函数时需要传入一个 `struct sockaddr_un` 的结构体，这个结构体用来指定这个 `socket` 绑定的名字，因为每个 `Unix Domain Socket` 都需要绑定一个唯一的名字，接着调用 `listen()` 函数开始监听请求，最后在主循环中调用 `accpet()` 函数来接收请求的连接。

接下来我们介绍 `Unix Domain Socket` 客户端的编写：
```cpp
#define UNIX_SOCKET_PATH "/path/to/unixsocket"

int main(int argc, char *argv[])
{
    ...
    sock = socket(AF_UNIX, SOCK_STREAM, 0);
    strcpy(addr.sun_path, UNIX_SOCKET_PATH);
    ...
    if (connect(sock, (struct sockaddr *)&addr, sizeof(struct sockaddr_un)) < 0) {
        close(sock);
        exit(1);
    }

    write(sock, DATA, sizeof(DATA));

    close(sock);
}
```
