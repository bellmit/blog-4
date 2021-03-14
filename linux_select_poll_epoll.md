[Linux IO模式及 select、poll、epoll详解](https://www.cnblogs.com/duanxz/p/5155926.html)
[五种IO模型](https://www.yuque.com/gotaoey/vaeroo/iqdfgh)

select 198x年的一个阻塞式的函数，当然还可以配置超时时间来停止阻塞。
问题：
1. 维护一个1024的bitmap，不可重用，每次得重置
2. 用户态内核态切换、拷贝
3. 遍历真整个rset(bitmap),挑出ready的socket。时间复杂度 O(n)。

poll和select的完成思路类似。但不使用bitmap了，使用一个结构体的数组。结构体为pollfd。
解决了上述的1.问题，但2.3.没有解决

epoll会将ready的事件重排到头部，不需要遍历整个数组。时间复杂度为O(m)，m为ready事件的数量。
并且实现内核态和用户态的buffer内存共享，不需要拷贝了。

redis、ngix、java nio底层都是用的epoll。
