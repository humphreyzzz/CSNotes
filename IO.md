# Unix的IO模型
一个输入操作通常包括两个阶段：
* 等待数据准备好
* 从内核向进程复制数据   

对于一个套接字上的输入操作，第一步通常涉及等待数据从网络中到达。当所等待数据到达时，它被复制到内核中的某个缓冲区。第二步就是把数据从内核缓冲区复制到应用进程缓冲区。   
Unix 有五种 I/O 模型：
* 阻塞式 I/O
* 非阻塞式 I/O
* I/O 复用（select 和 poll）
* 信号驱动式 I/O（SIGIO）
* 异步 I/O（AIO）

五大 I/O 模型比较
* 同步I/O：将数据从内核缓冲区复制到应用进程缓冲区的阶段（第二阶段），应用进程会阻塞。
* 异步I/O：第二阶段应用进程不会阻塞。
同步I/O包括阻塞式I/O、非阻塞式I/O、I/O复用和信号驱动I/O，它们的主要区别在第一个阶段。   
非阻塞式I/O、信号驱动I/O和异步I/O在第一阶段不会阻塞。

## 阻塞式I/O
应用进程被阻塞，直到数据从内核缓冲区复制到应用进程缓冲区中才返回。   
应该注意到，在阻塞的过程中，其它应用进程还可以执行，因此阻塞不意味着整个操作系统都被阻塞。因为其它应用进程还可以执行，所以不消耗 CPU 时间，这种模型的 CPU 利用率会比较高。

## 非阻塞式 I/O
应用进程执行系统调用之后，内核返回一个错误码。应用进程可以继续执行，但是需要不断的执行系统调用来获知 I/O 是否完成，这种方式称为轮询（polling）。   
由于 CPU 要处理更多的系统调用，因此这种模型的 CPU 利用率比较低。

## I/O复用
使用 select 或者 poll 等待数据，并且可以等待多个套接字中的任何一个变为可读。这一过程会被阻塞，当某一个套接字可读时返回，之后再使用 recvfrom 把数据从内核复制到进程中。   
它可以让单个进程具有处理多个 I/O 事件的能力。又被称为 Event Driven I/O，即事件驱动 I/O。   
如果一个 Web 服务器没有 I/O 复用，那么每一个 Socket 连接都需要创建一个线程去处理。如果同时有几万个连接，那么就需要创建相同数量的线程。相比于多进程和多线程技术，I/O 复用不需要进程线程创建和切换的开销，系统开销更小。

## 信号驱动I/O
应用进程使用 sigaction 系统调用，内核立即返回，应用进程可以继续执行，也就是说等待数据阶段应用进程是非阻塞的。内核在数据到达时向应用进程发送 SIGIO 信号，应用进程收到之后在信号处理程序中调用 recvfrom 将数据从内核复制到应用进程中。   
相比于非阻塞式 I/O 的轮询方式，信号驱动 I/O 的 CPU 利用率更高。

## 异步I/O
应用进程执行 aio_read 系统调用会立即返回，应用进程可以继续执行，不会被阻塞，内核会在所有操作完成之后向应用进程发送信号。   
异步 I/O 与信号驱动 I/O 的区别在于，异步 I/O 的信号是通知应用进程 I/O 完成，而信号驱动 I/O 的信号是通知应用进程可以开始 I/O。

# I/O复用
select/poll/epoll 都是 I/O 多路复用的具体实现，select 出现的最早，之后是 poll，再是 epoll。
## select
```
int select(int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```
select 允许应用程序监视一组文件描述符，等待一个或者多个描述符成为就绪状态，从而完成 I/O 操作。（轮询）
* fd_set 使用数组实现，数组大小使用 FD_SETSIZE 定义，所以只能监听少于 FD_SETSIZE 数量的描述符。有三种类型的描述符类型：readset、writeset、exceptset，分别对应读、写、异常条件的描述符集合。
* timeout 为超时参数，调用 select 会一直阻塞直到有描述符的事件到达或者等待的时间超过timeout。
* 成功调用返回结果大于 0，出错返回结果为 -1，超时返回结果为0。   

**时间复杂度O(n)**（轮询）
## poll
```
int poll(struct pollfd *fds, unsigned int nfds, int timeout);
```
poll的功能与select类似，也是等待一组描述符中的一个成为就绪状态。    
**时间复杂度O(n)**（轮询）   
poll中的描述符是pollfd类型的数组，pollfd的定义如下：
```c
struct pollfd {
               int   fd;         /* file descriptor */
               short events;     /* requested events */
               short revents;    /* returned events */
           };
```

```c
// The structure for two events
struct pollfd fds[2];

// Monitor sock1 for input
fds[0].fd = sock1;
fds[0].events = POLLIN;

// Monitor sock2 for output
fds[1].fd = sock2;
fds[1].events = POLLOUT;

// Wait 10 seconds
int ret = poll( &fds, 2, 10000 );
// Check if poll actually succeed
if ( ret == -1 )
    // report error and abort
else if ( ret == 0 )
    // timeout; no event detected
else
{
    // If we detect the event, zero it out so we can reuse the structure
    if ( fds[0].revents & POLLIN )
        fds[0].revents = 0;
        // input event on sock1

    if ( fds[1].revents & POLLOUT )
        fds[1].revents = 0;
        // output event on sock2
}
```

## epoll
```c
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```
epoll可以理解为event poll，不同于忙轮询和无差别轮询，epoll会把哪个流发生了怎样的I/O事件通知我们。所以我们说epoll实际上是事件驱动（每个事件关联上fd）的，此时我们对这些流的操作都是有意义的。   
**复杂度降低到了O(1)**

epoll_ctl() 用于向内核注册新的描述符或者是改变某个文件描述符的状态。已注册的描述符在内核中会被维护在一棵红黑树上，通过回调函数内核会将 I/O 准备好的描述符加入到一个链表中管理，进程调用 epoll_wait() 便可以得到事件完成的描述符。

从上面的描述可以看出，epoll 只需要将描述符从进程缓冲区向内核缓冲区拷贝一次，并且进程不需要通过轮询来获得事件完成的描述符。

epoll 仅适用于 Linux OS。

epoll 比 select 和 poll 更加灵活而且没有描述符数量限制。

epoll 对多线程编程更有友好，一个线程调用了 epoll_wait() 另一个线程关闭了同一个描述符也不会产生像 select 和 poll 的不确定情况。

### 工作模式
epoll 的描述符事件有两种触发模式：LT（level trigger）和 ET（edge trigger）。
#### LT 模式
当 epoll_wait() 检测到描述符事件到达时，将此事件通知进程，进程可以不立即处理该事件，下次调用 epoll_wait() 会再次通知进程。是默认的一种模式，并且同时支持 Blocking 和 No-Blocking。
#### ET 模式
和 LT 模式不同的是，通知之后进程必须立即处理事件，下次再调用 epoll_wait() 时不会再得到事件到达的通知。

很大程度上减少了 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。只支持 No-Blocking，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

## 比较
### 1. 功能
select 和 poll 的功能基本相同，不过在一些实现细节上有所不同。
* select 会修改描述符，而 poll 不会；
* select 的描述符类型使用数组实现，FD_SETSIZE 大小默认为 1024，因此默认只能监听少于 1024 个描述符。如果要监听更多描述符的话，需要修改 FD_SETSIZE 之后重新编译；而 poll 没有描述符数量的限制，原因是它是基于链表存储的；
* poll 提供了更多的事件类型，并且对描述符的重复利用上比 select 高。
* 如果一个线程对某个描述符调用了 select 或者 poll，另一个线程关闭了该描述符，会导致调用结果不确定。
### 2. 速度
select 和 poll 速度都比较慢，每次调用都需要将全部描述符从应用进程缓冲区复制到内核缓冲区。
### 3. 可移植性
几乎所有的系统都支持 select，但是只有比较新的系统支持 poll。

## 应用场景
很容易产生一种错觉认为只要用 epoll 就可以了，select 和 poll 都已经过时了，其实它们都有各自的使用场景。

### select 应用场景
select 的 timeout 参数精度为微秒，而 poll 和 epoll 为毫秒，因此 select 更加适用于实时性要求比较高的场景，比如核反应堆的控制。

select 可移植性更好，几乎被所有主流平台所支持。

### poll 应用场景
poll 没有最大描述符数量的限制，如果平台支持并且对实时性要求不高，应该使用 poll 而不是 select。

### epoll 应用场景
只需要运行在 Linux 平台上，有大量的描述符需要同时轮询，并且这些连接最好是长连接。

需要同时监控小于 1000 个描述符，就没有必要使用 epoll，因为这个应用场景下并不能体现 epoll 的优势。

需要监控的描述符状态变化多，而且都是非常短暂的，也没有必要使用 epoll。因为 epoll 中的所有描述符都存储在内核中，造成每次需要对描述符的状态改变都需要通过 epoll_ctl() 进行系统调用，频繁系统调用降低效率。并且 epoll 的描述符存储在内核，不容易调试。

综上，在选择select，poll，epoll时要根据具体的使用场合以及这三种方式的自身特点。

1、表面上看epoll的性能最好，但是在连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。

2、select低效是因为每次它都需要轮询。但低效也是相对的，视情况而定，也可通过良好的设计改善。


# JAVA I/O
![IO流](http://p1.pstatp.com/large/pgc-image/88e8dc8113e64cf7850d1d4950518383 "IO流")
Java 中将数据的输入输出抽象为流，流是一组有顺序的，单向的，有起点和终点的数据集合，就像水流。按照流中的最小数据单元又分为字节流和字符流。

**主要区别：字节流在处理输入输出时不会用到缓存，而字符流使用了缓存。**

1. 字节流：以 8 位（即 1 byte，8 bit）作为一个数据单元，数据流中最小的数据单元是字节。包含两个抽象类：InputStream和OutputStream。

2. 字符流：以 16 位（即 1 char，2 byte，16 bit）作为一个数据单元，数据流中最小的数据单元是字符，Java中的字符是Unicode编码，一个字符占用两个字节。

## 装饰者模式
JAVA IO类在设计时使用了Decorator设计模式，以上图的InputStream为例，ByteArrayInputStream、StringBufferInputStream、FileInputStream和PipedInputStream是JAVA提供的最基本的对流进行处理的类，FilterInputStream为一个封装类的基类，可以对基本的IO类进行封装，通过调用这些类提供的基本的流操作方法可以实现更复杂的流操作。   
使用这种设计模式的好处是可以在运行时动态地给对象添加一些额外的职责，与使用继承的设计方式相比，该方法更灵活。   
如果要设计一个输入流的类，可以通过继承抽象装饰者类（FileInputStream）来实现一个装饰类。

## 字节流和字符流的使用场景
1. 大多数情况下使用字节流会更好，因为字节流是字符流的包装，而大多数时候 IO 操作都是直接操作磁盘文件，所以这些流在传输时都是以字节的方式进行的（图片等都是按字节存储的）
2. 如果对于操作需要通过 IO 在内存中频繁处理字符串的情况使用字符流会好些，因为字符流具备缓冲区，提高了性能

## 什么是缓冲区
1. 缓冲区就是一段特殊的内存区域，很多情况下当程序需要频繁地操作一个资源（如文件或数据库）则性能会很低，所以为了提升性能就可以将一部分数据暂时读写到缓存区，以后直接从此区域中读写数据即可，这样就显著提升了性。
2. 对于 Java 字符流的操作都是在缓冲区操作的，所以如果我们想在字符流操作中主动将缓冲区刷新到文件则可以使用 flush() 方法操作。

## JAVA序列化
JAVA提供了两种对象持久化的方式，分别为序列化和外部序列化。
## 序列化
序列化就是一种用来处理对象流的机制，将对象的内容进行流化。可以对流化后的对象进行读写操作，可以将流化后的对象传输于网络之间。

所有要实现序列化的类都必须实现Serializable接口，这个接口里没有包含任何方法，使用一个输出流（如FileOutPutStream）来构造一个ObjectOutputSteam（对象流）对象，再使用该对象的wirteObject(Object obj)方法就可以将对象写出。

### 特点
1. 如果一个类可以序列化，它的子类也可以序列化。
2. 被声明为static和transient类型的数据成员无法序列化。

### 使用序列化的两个场景
1. 需要通过网络发送对象，或者对象的状态需要被持久化到文件或数据库中
2. 深复制

### 反序列化
反序列化将流转为对象。序列化和反序列化的过程中，最好设置serialVersionUID。每个类都有特定的serialVersionUID，在反序列化的过程中，使用serialVersionUID来判定兼容性，如果待序列化对象与目标对象的serialVersionUID不同，会抛出InvalidClassException异常。自定义serialVersionUID是一个非常好的编程习惯，有以下好处：
1. 显式声明serialVersionUID省去了程序生成serialVersionUID的计算时间。
2. 不同平台的编译器计算serialVersionUID可能采取不同的计算方式，显示声明serialVersionUID可以避免这个问题。
3. 增强程序各个版本的兼容性。如果对类进行属性的修改，会导致重新计算serialVersionUID，显示声明serialVersionUID可以避免这个问题。

## 外部序列化
Java提供了Externalizable接口来实现对象持久化，即外部序列化。

### 区别
序列化是内置API，只需要实现Serializable接口即可，而外部序列化需要程序员来实现接口中的读写方法，相当于把控制权交给了开发人员，有更多的灵活性，可以对需要持久化的属性进行控制。


## 网络操作
Java 中的网络支持：
* InetAddress：用于表示网络上的硬件资源，即 IP 地址；
* URL：统一资源定位符；
* Sockets：使用 TCP 协议实现网络通信；
* Datagram：使用 UDP 协议实现网络通信。
### InetAddress
没有公有的构造函数，只能通过静态方法来创建实例。
```java
InetAddress.getByName(String host);
InetAddress.getByAddress(byte[] address);
```
### URL
可以直接从 URL 中读取字节流数据。
```java
public static void main(String[] args) throws IOException {

    URL url = new URL("http://www.baidu.com");

    /* 字节流 */
    InputStream is = url.openStream();

    /* 字符流 */
    InputStreamReader isr = new InputStreamReader(is, "utf-8");

    /* 提供缓存功能 */
    BufferedReader br = new BufferedReader(isr);

    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }

    br.close();
}
```
### Socket
网络上两个程序通过一个双向的通信连接实现数据的交换，这个双向链路的一端称为一个SOCKET（套接字）。


![Socket](https://camo.githubusercontent.com/3371f62d744525df3ede0684ab932fe6aa52a188/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f31653661666663342d313865352d343539362d393665662d6662383463363362663838612e706e67 "Socket")
* ServerSocket：服务器端类
* Socket：客户端类
* 服务器和客户端通过 InputStream 和 OutputStream 进行输入输出。

### Datagram
* DatagramSocket：通信类
* DatagramPacket：数据包类

## NIO
新的输入/输出 (NIO) 库是在 JDK 1.4 中引入的，弥补了原来的 I/O 的不足，提供了高速的、面向块的 I/O。
### 流与块
I/O 与 NIO 最重要的区别是数据打包和传输的方式，I/O 以流的方式处理数据，而 NIO 以块的方式处理数据。

面向流的 I/O 一次处理一个字节数据：一个输入流产生一个字节数据，一个输出流消费一个字节数据。为流式数据创建过滤器非常容易，链接几个过滤器，以便每个过滤器只负责复杂处理机制的一部分。不利的一面是，面向流的 I/O 通常相当慢。

面向块的 I/O 一次处理一个数据块，按块处理数据比按流处理数据要快得多。但是面向块的 I/O 缺少一些面向流的 I/O 所具有的优雅性和简单性。

I/O 包和 NIO 已经很好地集成了，java.io.* 已经以 NIO 为基础重新实现了，所以现在它可以利用 NIO 的一些特性。例如，java.io.* 包中的一些类包含以块的形式读写数据的方法，这使得即使在面向流的系统中，处理速度也会更快。

### 通道与缓冲区
1. 通道
通道 Channel 是对原 I/O 包中的流的模拟，可以通过它读取和写入数据。

通道与流的不同之处在于，流只能在一个方向上移动(一个流必须是 InputStream 或者 OutputStream 的子类)，而通道是双向的，可以用于读、写或者同时用于读写。

通道包括以下类型：

* FileChannel：从文件中读写数据；
* DatagramChannel：通过 UDP 读写网络中数据；
* SocketChannel：通过 TCP 读写网络中数据；
* ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel。
2. 缓冲区
发送给一个通道的所有数据都必须首先放到缓冲区中，同样地，从通道中读取的任何数据都要先读到缓冲区中。也就是说，不会直接对通道进行读写数据，而是要先经过缓冲区。

缓冲区实质上是一个数组，但它不仅仅是一个数组。缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。

缓冲区包括以下类型：

* ByteBuffer
* CharBuffer
* ShortBuffer
* IntBuffer
* LongBuffer
* FloatBuffer
* DoubleBuffer

### 缓冲区状态变量
capacity：最大容量；
position：当前已经读写的字节数；
limit：还可以读写的字节数。
状态变量的改变过程举例：

① 新建一个大小为 8 个字节的缓冲区，此时 position 为 0，而 limit = capacity = 8。capacity 变量不会改变，下面的讨论会忽略它。
![1](https://camo.githubusercontent.com/2a7124242c8d67203a0644255fa0c266ab047afa/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f31626561333938662d313761372d346636372d613930622d3965326432343365616139612e706e67)

② 从输入通道中读取 5 个字节数据写入缓冲区中，此时 position 为 5，limit 保持不变。
![2](https://camo.githubusercontent.com/cbbc8451d44890665c19f2a38df033cb9e871ab0/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38303830346635322d383831352d343039362d623530362d3438656566336565643563362e706e67)

③ 在将缓冲区的数据写到输出通道之前，需要先调用 flip() 方法，这个方法将 limit 设置为当前 position，并将 position 设置为 0。
![3](https://camo.githubusercontent.com/8bbcea5699a8eea9c58fc91686ba864aa3c23ff9/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f39353265303662642d356136352d346361622d383265342d6464313533363436326633382e706e67)

④ 从缓冲区中取 4 个字节到输出缓冲中，此时 position 设为 4。
![4](https://camo.githubusercontent.com/f075744aafa9559fffb84311e34036d6d66b9b7f/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f62356264636265322d623935382d346165662d393135312d3661643936336362323862342e706e67)

⑤ 最后需要调用 clear() 方法来清空缓冲区，此时 position 和 limit 都被设置为最初位置。
![5](https://camo.githubusercontent.com/249f208058a7acab94a75516a404eb2dde74a816/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f36376266353438372d633435642d343962362d623963302d6130353864386336383930322e706e67 )

### 文件NIO实例
```java
public static void fastCopy(String src, String dist) throws IOException {

    /* 获得源文件的输入字节流 */
    FileInputStream fin = new FileInputStream(src);

    /* 获取输入字节流的文件通道 */
    FileChannel fcin = fin.getChannel();

    /* 获取目标文件的输出字节流 */
    FileOutputStream fout = new FileOutputStream(dist);

    /* 获取输出字节流的文件通道 */
    FileChannel fcout = fout.getChannel();

    /* 为缓冲区分配 1024 个字节 */
    ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    while (true) {

        /* 从输入通道中读取数据到缓冲区中 */
        int r = fcin.read(buffer);

        /* read() 返回 -1 表示 EOF */
        if (r == -1) {
            break;
        }

        /* 切换读写 */
        buffer.flip();

        /* 把缓冲区的内容写入输出文件中 */
        fcout.write(buffer);

        /* 清空缓冲区 */
        buffer.clear();
    }
}
```

### 选择器
 NIO 常常被叫做非阻塞 IO，主要是因为 NIO 在网络通信中的非阻塞特性被广泛使用。

NIO 实现了 IO 多路复用中的 Reactor 模型，一个线程 Thread 使用一个选择器 Selector 通过轮询的方式去监听多个通道 Channel 上的事件，从而让一个线程就可以处理多个事件。

通过配置监听的通道 Channel 为非阻塞，那么当 Channel 上的 IO 事件还未到达时，就不会进入阻塞状态一直等待，而是继续轮询其它 Channel，找到 IO 事件已经到达的 Channel 执行。

因为创建和切换线程的开销很大，因此使用一个线程来处理多个事件而不是一个线程处理一个事件，对于 IO 密集型的应用具有很好地性能。

应该注意的是，只有套接字 Channel 才能配置为非阻塞，而 FileChannel 不能，为 FileChannel 配置非阻塞也没有意义。
![5](https://camo.githubusercontent.com/249f208058a7acab94a75516a404eb2dde74a816/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f36376266353438372d633435642d343962362d623963302d6130353864386336383930322e706e67 )

1. 创建选择器
```java
Selector selector = Selector.open();
```
2. 将通道注册到选择器上
```java
ServerSocketChannel ssChannel = ServerSocketChannel.open();
ssChannel.configureBlocking(false);
ssChannel.register(selector, SelectionKey.OP_ACCEPT);
```
通道必须配置为非阻塞模式，否则使用选择器就没有任何意义了，因为如果通道在某个事件上被阻塞，那么服务器就不能响应其它事件，必须等待这个事件处理完毕才能去处理其它事件，显然这和选择器的作用背道而驰。

在将通道注册到选择器上时，还需要指定要注册的具体事件，主要有以下几类：
```java
SelectionKey.OP_CONNECT
SelectionKey.OP_ACCEPT
SelectionKey.OP_READ
SelectionKey.OP_WRITE
它们在 SelectionKey 的定义如下：

public static final int OP_READ = 1 << 0;
public static final int OP_WRITE = 1 << 2;
public static final int OP_CONNECT = 1 << 3;
public static final int OP_ACCEPT = 1 << 4;
```
可以看出每个事件可以被当成一个位域，从而组成事件集整数。例如：
```java
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```
3. 监听事件
```java
int num = selector.select();
```
使用 select() 来监听到达的事件，它会一直阻塞直到有至少一个事件到达。

4. 获取到达的事件
```java
Set<SelectionKey> keys = selector.selectedKeys();
Iterator<SelectionKey> keyIterator = keys.iterator();
while (keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if (key.isAcceptable()) {
        // ...
    } else if (key.isReadable()) {
        // ...
    }
    keyIterator.remove();
}
```
5. 事件循环
因为一次 select() 调用不能处理完所有的事件，并且服务器端有可能需要一直监听事件，因此服务器端处理事件的代码一般会放在一个死循环内。
```java
while (true) {
    int num = selector.select();
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> keyIterator = keys.iterator();
    while (keyIterator.hasNext()) {
        SelectionKey key = keyIterator.next();
        if (key.isAcceptable()) {
            // ...
        } else if (key.isReadable()) {
            // ...
        }
        keyIterator.remove();
    }
}
```
### 套接字 NIO 实例
```java
public class NIOServer {

    public static void main(String[] args) throws IOException {

        Selector selector = Selector.open();

        ServerSocketChannel ssChannel = ServerSocketChannel.open();
        ssChannel.configureBlocking(false);
        ssChannel.register(selector, SelectionKey.OP_ACCEPT);

        ServerSocket serverSocket = ssChannel.socket();
        InetSocketAddress address = new InetSocketAddress("127.0.0.1", 8888);
        serverSocket.bind(address);

        while (true) {

            selector.select();
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = keys.iterator();

            while (keyIterator.hasNext()) {

                SelectionKey key = keyIterator.next();

                if (key.isAcceptable()) {

                    ServerSocketChannel ssChannel1 = (ServerSocketChannel) key.channel();

                    // 服务器会为每个新连接创建一个 SocketChannel
                    SocketChannel sChannel = ssChannel1.accept();
                    sChannel.configureBlocking(false);

                    // 这个新连接主要用于从客户端读取数据
                    sChannel.register(selector, SelectionKey.OP_READ);

                } else if (key.isReadable()) {

                    SocketChannel sChannel = (SocketChannel) key.channel();
                    System.out.println(readDataFromSocketChannel(sChannel));
                    sChannel.close();
                }

                keyIterator.remove();
            }
        }
    }

    private static String readDataFromSocketChannel(SocketChannel sChannel) throws IOException {

        ByteBuffer buffer = ByteBuffer.allocate(1024);
        StringBuilder data = new StringBuilder();

        while (true) {

            buffer.clear();
            int n = sChannel.read(buffer);
            if (n == -1) {
                break;
            }
            buffer.flip();
            int limit = buffer.limit();
            char[] dst = new char[limit];
            for (int i = 0; i < limit; i++) {
                dst[i] = (char) buffer.get(i);
            }
            data.append(dst);
            buffer.clear();
        }
        return data.toString();
    }
}
```

```java
public class NIOClient {

    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1", 8888);
        OutputStream out = socket.getOutputStream();
        String s = "hello world";
        out.write(s.getBytes());
        out.close();
    }
}
```