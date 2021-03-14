
握手和挥手过程，全部在kernel mode中进行。
![三次握手](source/tcp3connect.png)
可以看到，握手的包中，有seq、ack、win(窗口大小)、length。length=0，代表包中body无数据。

![四次挥手](source/tcp4close.png)



