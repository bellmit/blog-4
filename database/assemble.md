### 细说浏览器输入URL后发生了什么
[细说浏览器输入URL后发生了什么](https://segmentfault.com/a/1190000012092552)

### http 和 https
[http 和 https](https://www.runoob.com/w3cnote/http-vs-https.html)

### get请求和post请求的区别
[get请求和post请求的区别](https://www.oschina.net/news/77354/http-get-post-different)

1. `GET把参数包含在URL中，POST通过request body传递参数`。
2. GET在浏览器回退时是无害的，而POST会再次提交请求。
3. GET比POST更不安全，因为参数直接暴露在URL上，所以不能用来传递敏感信息。
4. GET请求只能进行url编码，而POST支持多种编码方式。
5. `语义不同。get语义：获取资源，post语义：处理资源`。

GET和POST是什么？HTTP协议中的两种发送请求的方法。

HTTP是什么？HTTP是基于TCP/IP的关于数据如何在万维网中如何通信的协议。

HTTP的底层是TCP/IP。所以GET和POST的底层也是TCP/IP，也就是说，GET/POST都是TCP链接。GET和POST能做的事情是一样一样的。你要给GET加上request body，给POST带上url参数，技术上是完全行的通的。

在我大万维网世界中，TCP就像汽车，我们用TCP来运输数据，它很可靠，从来不会发生丢件少件的现象。但是如果路上跑的全是看起来一模一样的汽车，那这个世界看起来是一团混乱，送急件的汽车可能被前面满载货物的汽车拦堵在路上，整个交通系统一定会瘫痪。为了避免这种情况发生，交通规则HTTP诞生了。HTTP给汽车运输设定了好几个服务类别，有GET, POST, PUT, DELETE等等，HTTP规定，当执行GET请求的时候，要给汽车贴上GET的标签（设置method为GET），而且要求把传送的数据放在车顶上（url中）以方便记录。如果是POST请求，就要在车上贴上POST的标签，并把货物放在车厢里。当然，你也可以在GET的时候往车厢内偷偷藏点货物，但是这是很不光彩；也可以在POST的时候在车顶上也放一些数据，让人觉得傻乎乎的。HTTP只是个行为准则，而TCP才是GET和POST怎么实现的基本。

#### GET和POST还有一个重大区别

简单的说：

`GET产生一个TCP数据包；POST产生两个TCP数据包`。

长的说：

对于GET方式的请求，浏览器会把http header和data一并发送出去，服务器响应200（返回数据）；

而对于POST，浏览器先发送header，服务器响应100 continue，浏览器再发送data，服务器响应200 ok（返回数据）。

也就是说，GET只需要汽车跑一趟就把货送到了，而POST得跑两趟，第一趟，先去和服务器打个招呼“嗨，我等下要送一批货来，你们打开门迎接我”，然后再回头把货送过去。

因为POST需要两步，时间上消耗的要多一点，看起来GET比POST更有效。因此Yahoo团队有推荐用GET替换POST来优化网站性能。但这是一个坑！跳入需谨慎。为什么？

1. GET与POST都有自己的语义，不能随便混用。

2. 据研究，在网络环境好的情况下，发一次包的时间和发两次包的时间差别基本可以无视。而在网络环境差的情况下，两次包的TCP在验证数据包完整性上，有非常大的优点。

3. 并不是所有浏览器都会在POST中发送两次包，Firefox就只发送一次。


#### tcp实现可靠传输的原因

1. 面向连接：3次握手、四次挥手
2. 校验和：tcp将保持它header和data的校验和。如果校验和有差错，这些报文段会被丢弃并不被确认。
3. tcp丢弃重复的数据
4. 流量控制（tcp中维护一个receiver windows）+拥塞控制（congestion window）===》防止处理不过来导致的丢包
5. ARQ（Automatic Repeat-reQuest,自动重传请求）：每发完一个分组，停止发送，等待对方的ack。如果过了超时时间后，仍然没收到ack，重新发送这个分组，知道收到ack为止。

#### http1.0 http1.1 http2.0

1.0 默认短连接
1.1 默认长连接，不需要每一次request重新建立连接
2.0 ： 多路复用，头部压缩，服务器推送。