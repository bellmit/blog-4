
[中文翻译《Netty 4.x 用户指南》](https://waylau.gitbooks.io/netty-4-user-guide/content/)


### 聊明白bio、nio、aio。
在这里，
阻塞和非阻塞，说的是但一个事件/状态，未发生或就绪时，是否当前线程是否阻塞在那里，不进行之后的操作。





同步IO模型 vs 异步IO模型 （针对os层，底层）
而在这里，
同步和异步指的是真正进行IO操作时，当前线程是否阻塞（其实本没有阻塞/非阻塞和同步/异步的区别。）
目前，之后windows下的iocp(io完成端口)是纯的aio。linux下都是基于bio、nio做的（nio：select、poll、epoll）。

无论是nio还是bio，都需要程序自己去和内核读数据，这叫做同步IO模型。
异步io模型意味着，当一个线程从kernel中读数据时，该线程不需要阻塞，当io操作完成后，当前线程直接get()就好，向是调用一个future对象。换句话说，线程调用函数，传入buffer(存放读取到数据)，在之后利用类似buffer.get()得到。

此过程，区别于，使用byte[] bytes 读取回来，同步/异步执行计算和逻辑。

socket是什么？ 套接字? 插座?
可以理解socket是一个四元组。
<ip1,port1,ip2,port2>,四个元素确定了唯一的一个socket。

多路复用器selector只是告诉我们一个状态是否就绪，真正的读写操作，还需要程序自己去执行（用户自己去触发）


简易版nio，无selector
```java
List<SocketChannel> clients = new LinkedList<>();
// 返回值 java: ServerSocketChannel对象，底层： 文件描述符，e.g. 3
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.bind(new InetSocketAddress(8888));
ssc.configureBlocking(false);

while(true){
    //接受客户端连接
    SocketChannel client = ss.accept();
    // 不会阻塞？ 无客户端连接请求时，返回null， 底层c的方法是返回-1 
    // accept调用了内核：1、没有客户连接进来，返回值？ 在bio时，会一直卡着，但在nio中，不会卡着，返回-1 （kernel mode）
    //如果来客户端的连接了，accept返回的这个客户端的file descriptor（fd）, e.g. 5 (kernel mode) object （user mode）
    //non-blocking就是代码能往下继续走了，只不过有不同的情况（null、对象）。
    if(client == null){
        //测试时，会不停的打印出这句话
        System.out.println("null.....");
    }else{
        //重点，socket（服务端的listen socket<连接请求三次握手后，往）
        //客户端也不能阻塞
        client.configureBlocking(false);
        int port = client.socket().getPort();
        System.out.println("client...port: "+port);
        clients.add(client);
    }
    //可以在堆内  堆外
    ByteBuffer buffer = BtyeBuffer.allocateDirect(4096);


    //遍历已经连接进来的客户能不能读写数据
    for(SocketChannel c: clients){ // 串行化 ！！！！ 多线程！！！！
        int num = c.read(buffer) //client.read()方法不会阻塞。
        // -1 出错 0 无数据 >0 有数据
        if（num>0） {
            buffer.flip();
            byte[] aaa = new byte[buffer.limit()];
            buffer.get(aaa);

            String b = new String(aaa);
            System.out.println(c.socket.getPort()+":"+b);
            buffer.clear;
        }
    }
}

```
这么写的弊端：
1.全部的逻辑处理都在这一个线程中
2.不会阻塞，会一直轮询，cpu消耗巨大
3.如果1w个client连接，遍历client list，时间复杂度O(1w)。很多都是无用的遍历，可能一次遍历，只有几个client有数据要读


linux的kernel对nio的支持：select poll epoll

select allows a program to monitor multiple file descriptors, waiting until one or more of the file descriptors become "ready" for some class of I/O operation (e.g. input possible) A file descriptor is considered ready if it is **possible** to perform the coresspoding I/O operation (e.g. read(2)) without blocking。 