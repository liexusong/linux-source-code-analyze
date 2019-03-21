# 虚拟文件系统
在Linux系统中，可以使用多种文件系统来挂载不同的设备，如 ext2、ext3、nfs等等。但提供给用户的文件处理接口是一致的，也就是说不管使用 ext2 文件系统还是使用 ext3 文件系统，处理文件的接口都是一样的。这样的好处是，用户不用关心使用了什么文件系统，只需要使用统一的方式去处理文件即可。那么Linux是如何做到的呢？这就得益于 `虚拟文件系统(Virtual File System，简称VFS)`。

`虚拟文件系统` 为不同的文件系统定义了一套规范，各个文件系统必须按照 `虚拟文件系统的规范` 编写才能接入到 `虚拟文件系统` 中。这有点像面向对象语言里面的 `接口`，当一个类实现了某个接口的所有方法时，便可以把这个类当做成此接口。下面我们用Java接口的方式模仿一下 `虚拟文件系统` 的实现：
```java
public interface VFS {
    public Buffer read(File fd, int size);
    public bool write(File fd, Buffer buf);
    public bool lseek(File fd, int offset);
}
```
