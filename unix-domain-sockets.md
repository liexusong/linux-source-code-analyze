## Unix Domain Sockets使用
上一章介绍了Socket接口层的实现，接下来我们将会介绍具体的协议层实现，这一章将会介绍用于进程间通信的 `Unix Doamin Sockets` 的实现。要使用 `Unix Domain Sockets` 需要在创建socket时为 `family` 参数传入 `AF_UNIX`，如下代码：
```cpp
fd = socket(AF_UNIX, SOCK_STREAM, 0);
```
这样就可以创建一个类型为 `Unix Domain Sockets` 的socket描述符，如果我们编写的是服务端程序，那就需要在调用 `bind()` 函数时为其指定一个唯一的文件路径，客户端就可以通过这个文件路径来连接服务端程序。唯一路径是通过结构体 `struct sockaddr_un` 来设置的，如下代码：

服务端：
```cpp
...
int fd;
struct sockaddr_un addr;

fd = socket(AF_UNIX, SOCK_STREAM, 0);

memset(&addr, 0, sizeof(addr));

addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/server.sock");

bind(fd, (struct sockaddr *)&addr, sizeof(addr));
...
```
客户端：
```cpp
...
int fd;
struct sockaddr_un addr;

fd = socket(AF_UNIX, SOCK_STREAM, 0);

memset(&addr, 0, sizeof(addr));

addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/server.sock");

connect(fd, (struct sockaddr *)&addr, sizeof(addr));
...
```

## Unix Domain Sockets实现
