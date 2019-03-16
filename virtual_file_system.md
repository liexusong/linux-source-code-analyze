# 虚拟文件系统
Linux支持多种文件系统，但提供给用户的文件操作接口却是一样的（`open()`、`read()`、`write()` ...），那么Linux是如何实现的呢？这就得益于 `虚拟文件系统(Virtual File System, 简称VFS)` 了。虚拟文件系统是在软件层实现的一个中间层，它屏蔽了底层文件系统的实现细节，对用户提供统一的接口来访问真实的文件系统。譬如说当用户使用 `open()` 系统调用打开一个文件时，如果文件所在的文件系统是 `ext2`，真正调用的就是 `ext2_open_file()`。如果文件所在的文件系统是 `nfs`，那么真正调用的就是 `nfs_open()`。

## 虚拟文件系统实现细节
那么虚拟文件系统究竟用了什么魔法来实现这个神奇的功能呢？其实用过面向对象编程语言（如Java、C++等）的同学都应该知道有个 `接口(interface)` 的概念，`接口` 是一系列方法的声明，是一些方法特征的集合。例如，我们使用Java定义一个 人类(Human) 的接口:
```java
public interface Human {
    public void eat();
    public void walk();
    public void speak(String content);
}
```
上面定义了一个 Human 的接口，接口中有3个方法：`eat()`、`walk()` 和 `speak()`，现在我们定义一个名为 Man 和一个名为 Woman 的类来实现 Human 这个接口：
```java
public class Man implements Human {
    public void eat() {
        System.out.println("Man eating...");
    }
    
    public void walk() {
        System.out.println("Man walking...");
    }
    
    public void speak(String content) {
        System.out.println("Man speaking..." + content);
    }
}

public class Woman implements Human {
    public void eat() {
        System.out.println("Woman eating...");
    }
    
    public void walk() {
        System.out.println("Woman walking...");
    }
    
    public void speak(String content) {
        System.out.println("Woman speaking..." + content);
    }
}
```
因为 Man 和 Woman 类都实现了 Human 这个接口，所以我们可以通过 Human 这个接口来访问这两个类的方法：
```java
public class Main {
    public static void main(String[] args) {
        Human human;
        Man man = new Man();
        Woman woman = new Woman();
        
        human = man;
        human.eat();
        
        human = woman;
        human.eat();
    }
}
```
上面代码输出如下：
```
Man eating...
Woman eating...
```
可以看出，Man 对象和 Woman 对象都可以当成 Human 接口类型来使用。读者可以有些疑惑，为什么讲 `虚拟文件系统` 会扯到Java去了，原因是 `虚拟文件系统` 使用的方式与面向对象的接口非常相似。
