---
title: 二、深入分析Java_IO的工作机制
date: 2017-08-07 22:08:00
updated: 2018-12-22 17:34:00
tags:
  - io
  - javaweb
categories: 
  - [读书总结, 深入分析Java_Web技术]
comments: true
permalink: deepknowjavaweb/2_java_io.html  
---

![][0]

<!-- more -->

# 1 JAVA的I/O类库的基本架构

1. 基于字节操作的I/O接口：InputStream和OutputStream。
2. 基于字符操作的I/O操作：Writer和Reader。
3. 基于磁盘操作的I/O操作：File。
4. 基于网络操作的I/O操作：Socket。

# 2 字节字符的转换

低级的字节转字符，有InputStreamReader，以及OutputStreamWriter。 而字符转字节一般直接用new String(byte[])。
>注意：字符字节的转换在开发中一定要显示指明编码。 在OutputStreamWriter的官方注释中，错误的理解为从字符到字节，其实应该理解>成字符与字节之间的桥梁。

# 3 访问文件的几种方式

前言：读取和写入都是调用操作系统的提供的接口，而操作系统调用就会存在内核空间地址和用户空间地址切换的问题，一般的IO都是数据从 磁盘复制到内核空间，然后再从内核空间复制到用户空间，操作系统为了加速IO访问，在内核空间使用了缓存，即如果是第二次访问同一段的磁盘地址，直接从内核缓存中取出。

## 3.1 标准访问文件的方式

读取：调用操作系统的Read接口，操作系统先检查内核的高速缓存，如果有缓存则直接返回，如果没有则从磁盘中读取，缓存，返回。  
写入：调用操作系统的Writer接口，写入到高速缓存中，则通知应用程序完成，什么时候写入磁盘由操作系统决定。当然你可以使用sync强制刷新。  
![][1]

## 3.2 直接I/O的方式

即应用程序直接访问磁盘数据，减少一次从内核缓冲区到用户空间的数据复制，例如数据库管理系统，数据库明确的知道哪些数据需要缓存哪些不需要，以及哪些数据需要先放到内存中预热，但是不好的地方在于，你接管了数据缓存，如果你没有命中，则每次都是IO磁盘，比较耗时，通常结合直接IO与异步IO。

## 3.3 同步访问文件的方式

与标准访问文件不同点在于，写入了磁盘，操作系统才会应用程序返回成功的标志，用于安全性高的场景。

## 3.4 异步访问文件的方式

访问文件的请求线程发出后，不会阻塞等待，继续做别的事，完成文件访问后回调某个方法，提高应用程序的效率而不是访问文件的效率。

## 3.5 内存映射的方式

操作系统将内存中的某一块区域与磁盘中的文件关联，理解为快捷方式。这样中间加了一层地址映射，空间换时间，在实际开发中，多台业务 服务器对一个统一的路径下进行共享，方便数据的存储。例如A服务器下的data和B服务器下的data进行共享，便于文件的统一上传下载路径管理。  
![][2]

# 4 访问磁盘文件

前面介绍了操作数据，接着这里介绍数据写向何处，例如持久化到物理磁盘。  
FileInputStream 对象是操作一个文件的接口，创建的同时会创建该文件的描述对象 FileDescriptor。操作文件对象的时候可以通过getFD() 方法获取真正与底层操作系统相关联的文件描述。例如调用FileDescriptor.sync()方法将操作系统缓存中的数据强制刷新到物理磁盘中。  
byte->char是解码过程，因此读取文件都是需要StreamDecoder类帮助。  
![][3]

## 4.1 Java序列化技术

将对象转化成一串二进制表示的字符数组，反序列化时需要原始类作为模板，原因在于序列化之后的文件不保存类的完整结构信息。  
建议保存为通用的json/xml格式，比较耗的序列化工具：protobuf。序列化以及反序列需要注意一些常见的问题，例如serialVersionUID被修改，序列化对象中有属性为对象但是该属性对象没有实现Serializable等。

# 5 网络I/O工作机制

## 5.1 TCP状态

![][4]  
1. 三次握手 客户端CLOSED、SYN-SEND、ESTABLISHED。  
服务端LISTEN、SYN-RCVD、ESTABLISHED。
2. 四次挥手
客户端ESTABLISHED、FIN_WAIT_1、FIN_WAIT_2、TIME_WAIT。  
服务端ESTABLISHED、CLOSE_WAIT、LAST_ACK、CLOSE。

## 5.2 影响网络传输的因素

1. 网络带宽：物理链路在1s内传输的最大比特值，一般都是1.7Mb/s。
2. 传输距离。
3. TCP拥塞控制：TCP传输是一个“停等停等”的过程，要步调一致则需要通过拥塞控制来调节。TCP在传输时会设定一个“窗口”，窗口大小由带宽和数据 在两端的来回时间，即响应时间决定的。

## 5.3 Java Socket的工作机制

![][5]  
客户端开始建立一个Socket实例时，操作系统将为这个Socket实例分配一个没有被使用的本地端口号，并创建一个包含本地地址、 远程地址和端口号的套接字数据结构，这个数据结构一直保存在系统中直到这个连接关闭。在创建Socket实例的构造函数正确返回之前，将进行TCP的三次握手协议， 三次握手，完成之后，Socket实例创建完成。  
服务端将创建一个ServerSocket实例，只要指定的端口号没有被占用，一般实例都会创建成功，操作系统底层也会为ServerSocket实例创建一个底层 数据结构，这个数据结构中包含指定的端口号和包含监听地址的通配符，通常都是“*”，即监听所有地址。之后调用accept()方法，进入阻塞状态，等待 客户端的请求。当一个新的请求到达时，为这个连接创建一个新的套接字数据结构，该套接字数据的信息包含的地址和端口信息正是请求源地址 和端口，同时这个新创建的数据结构将会关联到ServerSocket实例的一个未完成的连接数据结构列表中。注意，此时服务端的与之对应的Socket实例 并没有完成创建，而是要等待与客户端的3次握手完成后，这个服务端的Socket实例才会返回，并从未连接数据结构列表移到已完成列表。所以与ServerSocket 所关联的列表中每个数据结构都代表与一个客户端建立的TCP连接。

## 5.4 数据传输

服务端和客户端都会拥有一个Socket实例，每个Socket实例都有一个InputStream和OutputStream，通过这两个对象来交换数据，同时操作系统会为 这两个对象分配一定大小的缓存区。

>写入：数据->OutputStream对应的SendQ队列，队列填满时，数据将会转移到另一端的InputStream的RecvQ队列中，如果RecvQ已经满了，那么 OuptStream的write方法将会阻塞，直到RecvQ队列可以容纳SendQ队列的数据。因此网络IO还需要一个协调的过程，如果两边同时传输数据则会产生死锁。

# 6 NIO的工作方式(建议先阅读课外学习：关于NIO)

## 6.1 BIO的缺点

阻塞IO，即BIO，在读取和写入时（InputStream、OutputStream）都有可能堵塞，一旦有堵塞，线程将会失去CPU的使用权，一些方法，例如：一个客户端 一个处理线程、线程池用来减少线程创建和回收的成本。但是，当需要大量的HTTP长连接，例如Web旺旺，虽然并不是每个连接都一直在传输数据，但是如果要 对某个客户端（VIP）提供更高的服务优先，很难通过线程本省的优先级完成，同时访问一些竞争资源时，也会有问题，因此需要同步。因此NIO应运而生。

## 6.2 NIO的工作机制

通过等待读以及等待写的轮询，在真正进行IO的时候才是使用CPU阻塞，但是由于是memory copy，在带宽足够大的1GB/s基本可以忽略。

## 6.3 Buffer的工作方式

可以简单理解为操作一组基本数据类型的元素列表：capacity、position、limit、mark。
注意，通过Channel获取的IO数据首先经过操作系统的Socket缓冲区，再将数据复制到Buffer中，这个操作系统缓冲区就是底层的TCP所关联的RecvQ或者 SendQ队列。  
Buffer提供了另一种直接操作操作系统缓冲区的方式，即ByteBuffer.allocateDirector()，这个方法直接返回底层存储空间关联的缓冲区，它通过 Native代码操作非JVM堆的内存空间，每次创建或者释放都要手动调用一次System.gc()。
>使用该方法直接操作非JVM堆空间会引起JVM内存泄漏问题。适用于数据量比较大，生命周期比较长的情况下，而普通的allocate()方法 适用并发连接少于1000。

## 6.4 FileChannel的数据访问

### 6.4.1 FileChannel.transferXXX

传统的数据访问方式：  
![][6]
  
FileChannel.transferXXX方式：  
![][7]


### 6.4.2 FileChannel.map

将文件按照一定大小块映射为内存区域，当程序访问这个内存区域时将直接操作这个文件数据，省去了数据从内核空间向用户空间复制的损耗。 适用于对大文件的只读性操作，如大文件的MD5校验。

# 7 IO调优

## 7.1 磁盘I/O优化

### 7.1.1 性能检测

在Linux下的iostat命令，查看I/O wait指标是否正常，即CPU等待I/O指标，如果是4核CPU，那么I、O wait参数不应该超过25%。

### 7.1.2 提升I/O性能

1. 增加缓存，减少磁盘访问次数。
2. 优化磁盘的管理系统
3. 设计合理的磁盘存储数据块。

## 7.2 TCP网络参数调优

操作系统的端口号：2^16 = 65535个。 通过查看 `cat /proc/sys/net/ipv4/ip_local_port_range` 查看端口范围。  
![][8]  
大量并发，端口号的数量就变成瓶颈，还有 `TIME_WAIT` 的数量，如果过多，需要将参数设小，提前释放。

## 7.3 网络I/O优化

1. 减少网络交互的次数  
SQL在客户端和数据库端设置缓存，请求css、js等可以合并为一个http链接，每个文件通过逗号隔开，服务端一次请求全部返回。
2. 减少网络传输数据量的大小  
通常Web服务器将请求的Web页面gzip压缩后再传输给浏览器。以及通过简单的协议，读取协议头来获取有用的价值信息。尽量避免读取整个通信数据，例如4层代理和7层代理，都是精良避免要读取整个通信数据。
![][9]
3. 尽量减少编码
4. 尽量以字节形式发送

### 7.3.1 同步与异步

1. 同步  
一个任务的完成需要依赖另外一个任务时，只有等待被依赖的任务完成后，依赖的任务才能完成，这是一种可靠的任务序列，同生同死。 同步能保证程序的可靠性。
2. 异步  
不需要等待被依赖的任务完成只是通知被依赖的任务要完成什么工作，依赖的任务也立即执行。 异步可以提高程序的性能，需要在同步与异步中保持平衡

### 7.3.2 阻塞和非阻塞

阻塞和非阻塞主要从CPU的消耗上来说。

1. 阻塞  
CPU停下等待一个慢的操作完成之后，CPU才接着完成其他的工作。  
2. 非阻塞  
这个慢操作执行时，CPU去做其他工作，这个慢操作完成时，CPU收到通知继续完成这个慢操作之后的事。
3. 两种方式的组合  
**同步阻塞**  
常用，简单，但是IO性能差，CPU大部分处于空闲状态。  
**同步非阻塞**  
常用于网络IO是长连接同时传输数据不多的情况。提升IO性能的常用手段，会增加CPU消耗，要考虑增加的IO性能能不能补偿CPU的消耗，也就是系统的瓶颈是在IO还是CPU上。  
**异步阻塞**  
常用于分布式数据库中。例如一个分布式数据库中写一条记录，通常会有一份是同步阻塞的记录，还有2~3份备份记录会写到 其他机器上，这些备份记录通常都采用异步阻塞的方式写IO，异步阻塞对网络IO能够提升效率，尤其像上面这种同时写多份 相同数据的情况。  
**异步非阻塞**  
比较复杂，只有在非常负载的分布式情况下使用，集群之间的消息同步机制一般使用这种IO组合方式。如Cassandra的Gossip通信机制就采用 异步非阻塞的方式。适用于同时要传多份相同的数据到集群中不同的机器，同时数据的传输量虽然不大但非常频繁的情况。  
**虽然异步和非阻塞能够提高IO整体性能，但是会增加性能成本，以及程序设计复杂的上升，需要经验丰富的人去设计，如果设计的不合理反而会导致性能下降。**[怎样理解阻塞非阻塞与同步异步的区别？](https://www.zhihu.com/question/19732473)

# 8 适配器模式

博主做一个Integer转化为String的例子，仿造InputStream转化Reader的简单例子。

```java
public class InputInteger_ implements Integer_ {
    private Integer a;
    public InputInteger_(Integer a){
        this.a = a;
    }
    public Integer getInteger(){
        return a;
    }
}
public interface Integer_ {
    public Integer getInteger();
}
public class String_ implements InputString_ {
    public void readString() {
    }
}
public interface InputString_ {
    public void readString();
}
public class InputInteger2String implements InputString_ {
    Integer_ s;
    public InputInteger2String(Integer_ s) {
        this.s = s;
    }
    public void readString() {
        // StreamDecoder
        Integer r = Integer.valueOf(s.getInteger());
        System.out.println(r);
    }
}
public class Demo {
    public static void main(String[] args) {
        InputInteger2String s = new InputInteger2String(new InputInteger_(4));
        s.readString();
    }
}
```

# 9 装饰器模式

赋予被装饰的类更多的功能，就像IO中的BufferedInputStream有缓冲的功能，LineNumberInputStream有提高按行读取数据的功能。

```java
public abstract class InputStream_ {
    public abstract void read();
}
public class FileInputStream_ extends InputStream_ {
    public void read() {
        
    }
}
public class FilterInputStream_ extends InputStream_ {
    
    protected InputStream_ inputStream_;
    
    public FilterInputStream_(InputStream_ inputStream_){
        this.inputStream_ = inputStream_;
    }
    
    public void read() {
        inputStream_.read();
    }
}
public class BufferInputStream_ extends FilterInputStream_ {
    
    public BufferInputStream_(InputStream_ inputStream_) {
        super(inputStream_);
    }
    
    private void bufferFirst(){
        
    }
    
    private void bufferEnd(){
        
    }
    
    public void read(){
        bufferFirst();
        super.read();
        bufferEnd();
    }
}
public class Demo {
    
    public static void main(String[] args){
        InputStream_ inputStream_ = new FileInputStream_();
        BufferInputStream_ bufferInputStream_ = new BufferInputStream_(inputStream_);
        bufferInputStream_.read();
    }
}
```

# 10 适配器模式与装饰器模式区别

它们有个别名，叫包装模式，都起到了包装一个类或对象的作用，但是作用不同。适配器通过改变接口来达到重复使用的目的（如果系统在设计初期，就尽量不要用 适配器模式），而装饰器模式保持原有的接口，增强原有对象的功能。

# 11 总结

Java中IO的基本库结构，磁盘IO和网络IO的工作方式。

[0]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/background/2015-04-29%E9%A3%9E%E6%9C%BA%E9%93%BE.jpg
[1]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-2-1-1.png
[2]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-2-1-2.png
[3]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-2-1-3.png
[4]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-2-1-4.png
[5]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-2-1-5.png
[6]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-2-1-6.png
[7]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-2-1-7.png
[8]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-2-1-8.png
[9]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-2-1-9.png