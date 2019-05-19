# Unix Domain Sockets实现
上一章介绍了Socket接口层的实现，接下来我们将会介绍具体的协议层实现，这一章将会介绍用于进程间通信的 `Unix Doamin Sockets` 的实现。要使用 `Unix Domain Sockets` 需要在创建socket时为 `family` 参数传入 `AF_UNIX`，如下代码：
```cpp
socket(AF_UNIX, SOCK_STREAM, 0);
```
