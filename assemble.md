#### 记录些零敲碎打的知识点


### 适配器


[Carson_Ho:适配器模式](https://blog.csdn.net/carson_ho/article/details/54910430)


### uml图
各种关系的强弱顺序：
泛化（箭头带三角形的实线，箭头指向父类） 
= 实现（箭头带三角形的虚线，箭头指向接口） 
\> 组合（带实心菱形的实线，箭头指向整体） 
\> 聚合 （带空心菱形的实线，箭头指向整体）
\> 关联（双向：实线, 单向：带普通箭头的实线，指向拥有者） 
\> 依赖（普通带箭头的虚线，由**使用者指向被使用者**）
组合和聚合是关联性更强的一种关联
[刘涤生: UMl类图的几种关系](https://www.jianshu.com/p/4fbfb1bd4cfa)

### CQRS(命令查询责任隔离)
[Java的CQRS](https://www.jdon.com/54180)


### Quartz
[Quartz应用与集群原理分析](https://blog.csdn.net/yuanlaishini2010/article/details/54427234)
[cron公式](https://www.cnblogs.com/lcchuguo/p/4063371.html)
	
名称|是否必须|同意值|特殊字符
-------|:-------:|------:|--------:
秒|是|0-59|, - * /
分|是|0-59|	, - * /
时|是|0-23|	, - * /
日|是|1-31|	, - * ? / L W C
月|是|1-12 或 JAN-DEC|, - * /
周|是|1-7 或 SUN-SAT|, - * ? / L C #
年|否| 空 或 1970-2099|, - * /

使用**星号(*)** 指示着你想在这个域上包括全部合法的值。

**?号** 仅仅能用在日和周域上，可是不能在这两个域上同一时候使用。你能够觉得? 字符是 "我并不关心在该域上是什么值。" 这不同于星号，星号是指示着该域上的每个值。**? 是说不为该域指定值。**

逗号 (,) 是用来在给某个域上指定一个值列表的。比如，使用值 0,15,30,45 在秒域上意味着每15秒触发一个 trigger。

**斜杠 (/) 是用于时间表的递增的。** 我们刚刚用了逗号来表示每15分钟的递增，可是我们也能写成这样0/15。

中划线 (-) 用于指定一个范围。比如，在小时域上的 3-8 意味着 "3,4,5,6,7 和 8 点。" 域的值不同意回卷，所以像 50-10 这种值是不同意的。

### 设计原则: 依赖倒置原则DIP
[知乎：依赖倒置原则DIP](https://zhuanlan.zhihu.com/p/24175489)

原始定义：High level modules should not depend upon low level modules. Both should depend upon abstractions. Abstractions should not depend upon details. Details should depend upon abstractions。

官方翻译：高层模块不应该依赖低层模块，两者都应该依赖其抽象；抽象不应该依赖细节，细节应该依赖抽象。


### 动态代理
[博客园:java动态代理 好！](https://www.cnblogs.com/xiaoluo501395377/p/3383130.html)
[小旋锋：java动态代理详解](https://juejin.cn/post/6844903744954433544)
![proxy](other_source/proxy.png)
代理模式：给某一个对象提供一个代理，并由代理对象来控制对真实对象的访问。代理模式是一种结构型设计模式。
+ 静态代理：所谓静态也就是在程序运行前就已经存在代理类的字节码文件，代理类和真实主题角色的关系在运行前就确定了。
+ 动态代理：而动态代理的源码是在程序运行期间由JVM根据反射等机制动态的生成，所以在运行前并不存在代理类的字节码文件。
  + 通过实现接口的方式 -> JDK动态代理
  + 通过继承类的方式 -> CGLIB动态代理

#### jdk动态代理
在java的动态代理机制中，有两个重要的类或接口，一个是 InvocationHandler(Interface)、另一个则是 Proxy(Class)，这一个类和接口是实现我们动态代理所必须用到的。首先我们先来看看java的API帮助文档是怎么样对这两个类进行描述的：
```java
InvocationHandler is the interface implemented by the invocation handler of a proxy instance. 

Each proxy instance has an associated invocation handler. When a method is invoked on a proxy instance, the method invocation is encoded and dispatched to the invoke method of its invocation handler.
```
每一个动态代理类都必须要实现InvocationHandler这个接口，并且每个代理类的实例都关联到了一个handler，当我们通过代理对象调用一个方法的时候，这个方法的调用就会被转发为由InvocationHandler这个接口的 invoke 方法来进行调用。我们来看看InvocationHandler这个接口的唯一一个方法 invoke 方法：
```java
Object invoke(Object proxy, Method method, Object[] args) throws Throwable

proxy: 指代我们所代理的那个真实对象
method: 指代的是我们所要调用真实对象的某个方法的Method对象
args: 指代的是调用真实对象某个方法时接受的参数
```
如果不是很明白，等下通过一个实例会对这几个参数进行更深的讲解。

接下来我们来看看Proxy这个类：
```java
Proxy provides static methods for creating dynamic proxy classes and instances, and it is also the superclass of all dynamic proxy classes created by those methods. 
```
Proxy这个类的作用就是用来动态创建一个代理对象的类，它提供了许多的方法，但是我们用的最多的就是 newProxyInstance 这个方法：
```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,  InvocationHandler h)  throws IllegalArgumentException

Returns an instance of a proxy class for the specified interfaces that dispatches method invocations to the specified invocation handler.
```
这个方法的作用就是得到一个动态的代理对象，其接收三个参数，我们来看看这三个参数所代表的含义:
+ **loader**:　　一个ClassLoader对象，定义了由哪个ClassLoader对象来对生成的代理对象进行加载
+ **interfaces**:　　一个Interface对象的数组，表示的是我将要给我需要代理的对象提供一组什么接口，如果我提供了一组接口给它，那么这个代理对象就宣称实现了该接口(多态)，这样我就能调用这组接口中的方法了
+ **h**:　　一个InvocationHandler对象，表示的是当我这个动态代理对象在调用方法的时候，会关联到哪一个InvocationHandler对象上

Proxy.newProxyInstance返回值类型：$Proxy0
```java
Subject subject = (Subject)Proxy.newProxyInstance(handler.getClass().getClassLoader(), realSubject.getClass().getInterfaces(), handler);
```
可能我以为返回的这个代理对象会是Subject类型的对象，或者是InvocationHandler的对象，结果却不是，首先我们解释一下为什么我们这里可以将其转化为Subject类型的对象？原因就是**在newProxyInstance这个方法的第二个参数上，我们给这个代理对象提供了一组什么接口，那么我这个代理对象就会实现了这组接口，这个时候我们当然可以将这个代理对象强制类型转化为这组接口中的任意一个，因为这里的接口是Subject类型，所以就可以将其转化为Subject类型了。**

**同时我们一定要记住，通过 Proxy.newProxyInstance 创建的代理对象是在jvm运行时动态生成的一个对象，它并不是我们的InvocationHandler类型，也不是我们定义的那组接口的类型，而是在运行是动态生成的一个对象，并且命名方式都是这样的形式，以$开头，proxy为中，最后一个数字表示对象的标号。**

java动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理。

而cglib动态代理是利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。

Spring AOP会在两种方式下切换:
1. 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP 
2. 如果目标对象实现了接口，可以强制使用CGLIB实现AOP 
3. 如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换

### 什么是分布式系统
[xbbaby: 什么是分布式系统](https://www.cnblogs.com/xybaby/p/7787034.html)

### google 三驾马车 gfs mapreduce bigtable


### hadoop
[haddop设计原理](https://www.jianshu.com/p/ad9b9e2ecd25)

### mysql死锁问题
[博客园：Mysql并发时经典常见的死锁原因及解决方法](https://www.cnblogs.com/zejin2008/p/5262751.html)

[MySQL的死锁系列-常见加锁场景分析](https://cloud.tencent.com/developer/article/1635491)
当对存在的行进行锁的时候(主键)，mysql就只有行锁。
当对未存在的行进行锁的时候(即使条件为主键)，mysql是会锁住一段范围（有gap锁）

锁住的范围为：
(无穷小或小于表中锁住id的最大值，无穷大或大于表中锁住id的最小值)
如：如果表中目前有已有的id为（11 ， 12）
那么就锁住（12，无穷大）
如果表中目前已有的id为（11 ， 30）
那么就锁住（11，30）

对于这种死锁的解决办法是：
insert into t3(xx,xx) on duplicate key update `xx`='XX';
用mysql特有的语法来解决此问题。因为insert语句对于主键来说，插入的行不管有没有存在，都会只有行锁。


库存db死锁的两个场景
1. insertOrUpdate数据时，用的套路是：先update，若返回0，则认为没有该数据，再insert。在并发场景下，由于mysql MVCC的机制，这种套路会有死锁
2. 当多个连接并发对一个表中执行事务时，若没有对skuId排序，可能造成死锁，因为出现首尾相接的循环等待。注：事务执行结束前，不会释放已获取的锁

### 数据库限流实现
Guava:
[Guava的布隆过滤器](https://juejin.cn/post/6844903832577654797)
[Guava的限流策略](https://mp.weixin.qq.com/s?__biz=Mzg2NjE5NDQyOA==&mid=2247483768&idx=1&sn=1df06849222072ac87d1410aef969125&source=41#wechat_redirect)

保护高并发系统的三把利器：
+ 缓存
+ 限流
+ 降级

限流方式与场景：
+ 限制总并发数（比如数据库连接池、线程池）
+ 限制瞬时并发数（如nginx的limitconn模块，用来限制瞬时并发连接数，Java的Semaphore也可以实现）
+ 限制时间窗口内的平均速率（如Guava的RateLimiter、nginx的limitreq模块，限制每秒的平均速率）；
+ 其他还有如限制远程接口调用速率、限制MQ的消费速率。另外还可以根据网络连接数、网络流量、CPU或内存负载等来限流。

令牌桶和漏桶对比：
+ 令牌桶是按照固定速率往桶中添加令牌，请求是否被处理需要看桶中令牌是否足够，当令牌数减为零时则拒绝新的请求；漏桶则是按照常量固定速率流出请求，流入请求速率任意，当流入的请求数累积到漏桶容量时，则新流入的请求被拒绝；
+ 令牌桶限制的是平均流入速率，允许突发请求，只要有令牌就可以处理，支持一次拿3个令牌，4个令牌；漏桶限制的是常量流出速率，即流出速率是一个固定常量值，比如都是1的速率流出，而不能一次是1，下次又是2，从而平滑突发流入速率；
+ **令牌桶允许一定程度的突发，而漏桶主要目的是平滑流出速率**；


限流策略：
1. 如果 “是否开始统计限流数量” 的开关开启，进入记录模式，否则return false
2. 定义 `private Map<String, GuavaCache<AtomicInteger>> dbLimitGuavaCacheMap_dbType`
   1. key：限流的库类型，包括stockNumDbLimit，popNumDbLimit，partitionNumDbLimit
   2. value：用于限流的guava实例（不同的库限流用不同的guava，防止不同库的skuId重复）, GuavaCache<AtomicInteger>是我们对Cache<T,E>的一层封装。其实它的key是String skuId。
3. 拿到某个Guava实例中的count。
4. 如果count==null,`Object lock = getLock(skuId, limitDbType);`,确保同一限流库类型下的同一skuId获取相同的lock。拿到锁之后，初始化count：`new AtomicInteger(1)`,如果不为null，原子操作: `int countNum = count.incrementAndGet()`
6. 如果"限流开关"开启&&countNum>=limitNum, heartMonitor+return true;
7. **两个设计巧妙的点是：** 
   1. 通过guava的失效时间后自动清理对象的机制（通过expireAfterWrite/Access）, 保证缓存中的原子整型对象不会一直存在且不会有增无减。失效时间和limitNum配合起来让其有类似窗口的概念，e.g. 80ms2次，失效后重新计数。
   2. 每一个限流类型下都定义了一个size=1024的lockMap（通过skuId的散列找到对应的lock对象），减少了不同skuId的并发冲突问题（一定程度减少了无必要的锁等待），提高了并发能力。如果把size做成远端可配置的，那么就可以控制该限流策略获取锁的并发能力。

```java
package com.jd.stock.sdk.service.hotlimit.impl;

import com.jd.stock.sdk.common.config.ResourceContainer;
import com.jd.stock.sdk.common.config.DbLimitSwitch;
import com.jd.stock.sdk.common.guava.GuavaCache;
import com.jd.stock.sdk.common.monitor.MonitorUtils;
import com.jd.stock.sdk.common.utils.LogUtils;
import com.jd.stock.sdk.common.utils.StockUtils;
import com.jd.stock.sdk.service.hotlimit.DbHotLimitService;

import java.util.HashMap;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * @description: 数据库热点限流服务
 * @author: lixiangfeng
 * @create: 2018-04-09 15:27
 */
public class DbHotLimitServiceImpl implements DbHotLimitService {

    private static final String HOT_SKU_LIMIT_KYE = "JDStock.NewSDK.db.hotSku.limit.";

    // key：限流的库类型，value：用于限流的guava实例（不同的库限流用不同的guava，防止不同库的skuId重复）
    private Map<String, GuavaCache<AtomicInteger>> dbLimitGuavaCacheMap_dbType;

    private ResourceContainer resourceContainer;

    private Map<String, Map<Long, Object>> lockMap_dbType = new HashMap<String, Map<Long, Object>>();

    public void init() {
        // 自营库
        Map<Long, Object> lockMap_stock = new HashMap<Long, Object>();
        for (long i = 0; i < 1024; i++) {
            Object lock = new Object();
            lockMap_stock.put(i, lock);
        }
        lockMap_dbType.put("stockNumDbLimit", lockMap_stock);
        // POP库
        Map<Long, Object> lockMap_pop = new HashMap<Long, Object>();
        for (long i = 0; i < 1024; i++) {
            Object lock = new Object();
            lockMap_pop.put(i, lock);
        }
        lockMap_dbType.put("popNumDbLimit", lockMap_pop);
        // POP分区库
        Map<Long, Object> lockMap_partition = new HashMap<Long, Object>();
        for (long i = 0; i < 1024; i++) {
            Object lock = new Object();
            lockMap_partition.put(i, lock);
        }
        lockMap_dbType.put("partitionNumDbLimit", lockMap_partition);
    }

    /**
     * 根据db类型、skuId获取对应的锁对象
     * @param skuId
     * @param limitDbType
     * @return
     */
    private Object getLock(Long skuId, String limitDbType) {
        Map<Long, Object> lockMap = lockMap_dbType.get(limitDbType);
        long lockNum = skuId & 1023;
        return lockMap.get(lockNum);
    }

    public boolean isDbLimit(Set<Long> skuIdSet, String limitDbType) {
        try {
            // 获取库相关配置
            DbLimitSwitch dbLimitSwitch = resourceContainer.getStockSdkConfig().getDbLimitSwitch(limitDbType);
            if (!dbLimitSwitch.getReckonLimit()) {
                return false;
            }
            // skuId 维度判断是否限流
            for (long skuId : skuIdSet) {
                GuavaCache<AtomicInteger> dbLimitGuavaCache = dbLimitGuavaCacheMap_dbType.get(limitDbType);
                AtomicInteger count = dbLimitGuavaCache.get(String.valueOf(skuId));
                // 第一次不限流
                if (null == count) {
                    MonitorUtils monitorUtils = MonitorUtils.umpRegister("JdStock.NewSDK.DbHotLimitServiceImpl.lock." + limitDbType);
                    try {
                        Object lock = getLock(skuId, limitDbType);
                        synchronized (lock) {
                            count = dbLimitGuavaCache.get(String.valueOf(skuId));
                            if (count == null) {
                                dbLimitGuavaCache.put(String.valueOf(skuId), new AtomicInteger(1));
                                continue;
                            }
                        }
                    } finally {
                        monitorUtils.umpRegisterEnd();
                    }
                    MonitorUtils.heartMonitor("JdStock.NewSDK.DbHotLimitServiceImpl.count.retrieve");
                }
                int countNum = count.incrementAndGet();
                // ump 监控
                this.umpMonitor(limitDbType, countNum);

                // 限流开关打开，并且访问量超过阈值（结合guava的失效时间），进行限流
                if (dbLimitSwitch.getLimitSwitch() && countNum > dbLimitSwitch.getLimitNum()) {
                    MonitorUtils.heartMonitor(HOT_SKU_LIMIT_KYE + limitDbType + ".hit");
                    // 限流日志开关打开时，打印限流商品的相关信息
                    if (dbLimitSwitch.getPrintLimitLog()) {
                        LogUtils.error(LogUtils.FILE_LIMIT, limitDbType + " db hotLimitHit, skuId = " + skuId + ", count = " + countNum);
                    }
                    return true;
                }
            }
        } catch (Exception e) {
            MonitorUtils.heartMonitor("JdStock.NewSDK.DbHotLimitServiceImpl.isDbLimit.exception");
            LogUtils.error(LogUtils.FILE_EXCEPTION, limitDbType + " db hot sku limit error!", e);
        }
        return false;
    }
}
```


### mysql、hbase、es的区别和使用场景
[简书：聊聊MySQL、HBase、ES的特点和区别](https://www.jianshu.com/p/4e412f48e820)

### mysql索引

[淘宝：Mysql二级索引定义](http://mysql.taobao.org/monthly/2020/01/01/)
[掘金：联合索引数据结构](https://juejin.cn/post/6844904073955639304)
[公众号：mysql索引那些事，好！](https://mp.weixin.qq.com/s?__biz=MzUxNTQyOTIxNA==&mid=2247484041&idx=1&sn=76d3bf1772f9e3c796ad3d8a089220fa&chksm=f9b784b8cec00dae3d52318f6cb2bdee39ad975bf79469b72a499ceca1c5d57db5cbbef914ea&token=2025456560&lang=zh_CN#rd)
[简书：MySQL中char、varchar和text的区别](https://www.jianshu.com/p/cc2d99559532)
二叉树、红黑树做索引的问题：
1. 树的高度过高导致查询效率变慢。二叉太少了，一个节点大小很小，但load次数太多了。
Hash做数据库索引的问题：
1. 不支持范围查找。
B-树做索引的问题：
1. 中间节点的有数据，导致一个节点的size变大，IO费劲，不能用有限的空间分更多的叉

mysql把索引做成b+树的原因：
1. 中间节点不需要放实际数据,只存索引（造成冗余），那么就意味着可以有更多分叉。
   1. 从磁盘中load一个节点，是一次IO操作，IO操作很费时间，且根据计算机组成原理，一次IO交互，一般只能传输4kb的数据（可能有时候有一些局部读取的原理可能会取几十K（4的整数倍），取个16K，24K也是可以的）。但不可能是mb或gb级别的。mysql对这个节点大小设置的是16K
   2. 假设索引字段类型是Bigint，8bit，每两个元素之间存的是下一个节点的地址，mysql分配的是6bit，也就是说一个索引后面配对一个节点地址，成对出现，可以算一下**16K的节点**可以存多少对也就是多少个索引，8b+6b=14b，16K /14b=**1170**个索引，叶子节点有索引有data元素，假设占1K，那一个节点就放16K/1K=**16个元素**，假设树高是**3**，所有节点都放满，能放多少数据？可以算一下，1170 * 1170 * 16=21902400，**2千多万**，mysql设置16K的大小，数据就可以存2千多万就已经足够了吧，既能保证一次磁盘IO不要Load太多的数据 又能保证一次load的性能，即便表的数据在几千万的数量也能保证树的高度在一个可控的范围。
2. 叶子节点之间还有指针，方便范围查找。

### Java8的Stream

[博客园：深入理解Java8中Stream的实现原理](https://www.cnblogs.com/sunshinekevin/p/11576893.html)

Stream上的所有操作分为两类：中间操作和结束操作，中间操作只是一种标记，只有结束操作才会触发实际计算。中间操作又可以分为无状态的(Stateless)和有状态的(Stateful)，无状态中间操作是指元素的处理不受前面元素的影响，而有状态的中间操作必须等到所有元素处理之后才知道最终结果，比如排序是有状态操作，在读取所有元素之前并不能确定排序结果；结束操作又可以分为短路操作和非短路操作，短路操作是指不用处理全部元素就可以返回结果，比如找到第一个满足条件的元素。之所以要进行如此精细的划分，是因为底层对每一种情况的处理方式不同。 为了更好的理解流的中间操作和终端操作，可以通过下面的两段代码来看他们的执行过程。

```java

     Stream.of("one", "two", "three", "four")
        .filter(e -> e.length() > 3)
        .peek(e -> System.out.println("Filtered value: " + e))
        .map(String::toUpperCase)
        .peek(e -> System.out.println("Mapped value: " + e))
        .collect(Collectors.toList());

    //result:
    /**
    *
    *Filtered value: three
    *Mapped value: THREE
    *Filtered value: four
    *Mapped value: FOUR
    */
```



### RESTful 架构
[Oracle: What Are RESTful Web Services? 重要！](https://docs.oracle.com/javaee/6/tutorial/doc/gijqy.html)
[知乎：webservice、RESTful、REST通俗理解 ](https://www.zhihu.com/question/30547012)
[腾讯云：什么是restful web服务](https://cloud.tencent.com/developer/article/1620023)

REST是完全不同的思路，它充分利用了HTTP协议的4个主要verb把RPC操作分成4类：
+ GET：进行幂等的资源获取操作
+ POST：创建资源
+ PATCH：修改资源
+ DELETE：删除资源

仔细想一下这其实就是数据库的CRUD操作
+ POST=create
+ GET=read
+ PATCH=update
+ DELETE=delete

RESTful是个形容词，形容根据这个思路设计出来的REST风格的API
真正的REST API很少见，绝大多数只是像REST而已，只用GET/POST两个verb。

REST提出设计概念和准则为：　　
1. 网络上的所有事物都可以被抽象为资源(resource)　　
2. 每一个资源都有唯一的资源标识(resource identifier)，对资源的操作不会改变这些标识　　
3. 所有的操作都是无状态的。

REST简化开发，其架构遵循CRUD原则，该原则告诉我们对于资源(包括网络资源)只需要四种行为：创建，获取，更新和删除就可以完成相关的操作和处理。我们可以通过统一资源标识符（Universal  Resource Identifier，URI）来识别和定位资源，并且针对这些资源而执行的操作是通过 HTTP  规范定义的。其核心操作只有GET,PUT,POST,DELETE。由于REST强制所有的操作都必须是stateless的，这就没有上下文的约束，如果做分布式，集群都不需要考虑上下文和会话保持的问题。极大的提高系统的可伸缩性。


[SOAP VS REST 重要！](https://smartbear.com/blog/test-and-monitor/soap-vs-rest-whats-the-difference/)
A Quick Overview of REST

REST provides a lighter-weight alternative. Many developers found SOAP cumbersome and hard to use. For example, working with SOAP in JavaScript means writing a ton of code to perform simple tasks because you must create the required XML structure every time.

**Instead of using XML to make a request, REST (usually) relies on a simple URL.** In some situations you must provide additional information, but most web services using REST rely exclusively on using the URL approach. **REST can use four different HTTP 1.1 verbs (GET, POST, PUT, and DELETE) to perform tasks.**

Unlike SOAP,**REST doesn’t have to use XML to provide the response.** You can find REST-based web services that output the data in Command Separated Value (CSV), JavaScript Object Notation (JSON) and Really Simple Syndication (RSS). The point is you can obtain the output you need, in a form that’s easy to parse within the language you’re using for your application.


### redis pipeline指令

[CSDN: Pipeline详解](https://blog.csdn.net/w1lgy/article/details/84455579)
[CSDN: Pipeline详解2](https://blog.csdn.net/diweikang/article/details/90674296?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control)

Pipeline的初衷： 通过用户端一次打包多条指令到服务端，来减少网络传输，和IO。
并不是越大越好：太大消耗内存严重、客户端等待时间过长(串行、阻塞)可能会导致超时。

三、原生批命令(mset, mget)与Pipeline对比
1、原生批命令是原子性，pipeline是非原子性
(原子性概念:一个事务是一个不可分割的最小工作单位,要么都成功要么都失败。原子操作是指你的一个业务逻辑必须是不可拆分的. 处理一件事情要么都成功，要么都失败，原子不可拆分)
2、原生批命令一命令多个key, 但pipeline支持多命令（存在事务），非原子性
3、原生批命令是服务端实现，而pipeline需要服务端与客户端共同完成。




### mysql执行计划expalin

#### extra
[MySQL 执行计划中Extra](https://www.cnblogs.com/kerrycode/p/9909093.html)
Using index (JSON property: using_index)

The column information is retrieved from the table using only information in the index tree without having to do an additional seek to read the actual row. This strategy can be used when the query uses only columns that are part of a single index.

Using where (JSON property: attached_condition)


A WHERE clause is used to restrict which rows to match against the next table or send to the client. Unless you specifically intend to fetch or examine all rows from the table, you may have something wrong in your query if the Extra value is not Using where and the table join type is ALL or index.


Using where has no direct counterpart in JSON-formatted output; the attached_condition property contains any WHERE condition used.

where 子句用于限制与下一个表匹配的行记录或发送到客户端的行记录。除非您特意打算从表中提取或检查所有行,否则如果Extra值不是Using where并且表连接类型为ALL或index，则查询可能会出错。


### MySQL 坑
###### select id from operatecode_pop_2 limit 1; 并没有用id索引，而是用的二级索引 rf_operate_code
###### 需要改为 select id from operatecode_pop_2 order by id limit 1
[MySQL优化：如何避免回表查询？什么是索引覆盖？](https://www.cnblogs.com/myseries/p/11265849.html)

### java Generics 泛型
[掘金：聊一聊-JAVA 泛型](https://juejin.cn/post/6844903917835419661)


### 理解异步函数、callback function
[知乎：回调函数？回调？事件循环？](https://zhuanlan.zhihu.com/p/41824005)
[segmentfault:JavaScript：彻底理解同步、异步和事件循环(Event Loop)](https://segmentfault.com/a/1190000004322358)

一个异步过程通常是这样的：

主线程发起一个异步请求，相应的工作线程接收请求并告知主线程已收到(异步函数返回)；主线程可以继续执行后面的代码，同时工作线程执行异步任务；工作线程完成工作后，通知主线程；主线程收到通知后，执行一定的动作(调用回调函数)。

异步函数通常具有以下的形式：
`A(args..., callbackFn)`
它可以叫做异步过程的发起函数，或者叫做异步任务注册函数。args是这个函数需要的参数。callbackFn也是这个函数的参数，但是它比较特殊所以单独列出来。
所以，从主线程的角度看，一个异步过程包括下面两个要素：
+ 发起函数(或叫注册函数)A
+ 回调函数callbackFn

它们都是在主线程上调用的，其中注册函数用来发起异步过程，回调函数用来处理结果。

注意：前面说的形式A(args..., callbackFn)只是一种抽象的表示，并不代表回调函数一定要作为发起函数的参数，例如：
```javascript
var xhr = new XMLHttpRequest();
xhr.onreadystatechange = xxx; // 添加回调函数
xhr.open('GET', url);
xhr.send(); // 发起函数
```