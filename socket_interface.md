# Socket族接口层
Socket的英文原本意思是 `孔` 或 `插座`。但在计算机科学中通常被称作为 `套接字`，主要用于相同机器的不同进程间或者不同机器间的通信。Socket的使用很多网络编程的书籍都有介绍，所以本文不打算介绍Socket的使用，只讨论Socket的具体实现，所以如果对Socket不太了解的同学可以先查阅Socket相关的资料或者书籍。

## Socket族系统调用
`Soket族系统调用` 提供了一系列的接口函数供用户使用，如下：
* socket()
* bind()
* listen()
* accept()
* connect()
* recv()
* send()
* recvfrom()
* sendto()

例如 `socket()` 接口用于创建一个socket句柄，而 `bind()` 函数将一个socket绑定到指定的IP和端口上。
