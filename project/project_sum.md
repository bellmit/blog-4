### 数据治理
开发一个定时worker，清理数据库中失效数据(物理删除)。

原方案：

数据中间件中开发两个接口：
接口1：通过n（remainDay,保留多少天的数据），找到n天以前数据的最大id。
接口2： 提供一个批量删除的接口。

开发一个定时worker。

1. 通过读取远端配置信息，解析到：开关、库表信息、休眠时长等。
2. 针对具体的某库某表，调用`接口1`找到n天以前的max id (id自增)，e.g. 10,000。
3. 批量调用`接口2`，删除数据（每次删除limitNum条，e.g. 1000），当返回值affectRow < limitNum, 退出循环。

note：
1. 接口调用前，需要判断开关是否开启、要避开整点时间。
2. 每轮删除后，需要休眠1s。

问题：
1. 遇到的问题：发现第一个获取最大失效id接口的性能很差，最长可达10s。sql语句：select id from operatecode_pop_0 where created < 给定时间 order by id desc limit 1,并且给定时间久远，sql性能越差。“2020-10-10” 查询只需要400ms，“2017-10-10”需要8s。初步定位原因是created没有索引导致。虽然可以通过配置rpc和数据库的timeout来实现，但作为一个通用的worker，需要考虑其性能问题。
2. 解决方案：设置一个步长step，每轮查询找到< step的最大id及其时间(sql: select id,created from table where id <= step order by id desc limit 1), worker判断created < remainday,批量删除小于该id的数据，步长迭代为step*round。直到查询接口返回created>=remainday时，跳出循环。 

知识点：mysql的覆盖索引和回表相关的。

例如，我们想找pop防重表中的最小id（字段有：id（主键），rf_operate_code (二级索引), created），可能会写 “select id from operatecode_pop_0 limit 1”，但发现返回的id并不是表中最小的。造成此结果的原因是，mysql并没有使用主键索引进行查找，explain发现使用的key是rf_operate_code。因为mysql的主键是聚簇索引，二级索引是非聚簇索引，rf_operate_code叶子节点中存放的是rf_operate_code本身和主键id。而我们只select id，id在二级索引树中有（覆盖索引），不需要在遍历主键的索引树（不需回表）。所以，这时我们拿到的id，是rf_operate_code索引树中的第一叶子节点的id，并不一定是最小的。“select id， rf_operate_code from operatecode_pop_0 limit 1”同理，用到覆盖索引，不需要回表。当select * || select id, created时，没有覆盖索引，需要回表，limit 1找到的id就是最小的。最后开发中使用的是 “select id from operatecode_pop_0 order by id limit 1”。


### 延迟预占

**什么是延迟预占**： 

B端商家下单时，只定位仓，不扣减库存。若剩余产能数量足够并且设置的有效期未到，则下单成功。下单24小时后（或自定义时间点后），再尝试预占扣减库存。每十分钟一次，扫大客无货预占表，直到预占成功为止。

一句话： 下单是不扣减库存，一定时间点后再扣减。

**为什么要开发延迟预占这个功能**
主要出于两方面

其实整个电商，对库存组的要求，除了能准确扣减、记录库存信息外，开发的新功能都是围绕 **提高库存周转率，降低库存落库时间，终极目标：just in time。**所以这次的延迟预占，一方面也是提高库存周转率的，类似于预售。采销侧可以设置产能数量，以及有效期，来拟给定一个计划。同时，采销侧也可以看到已售的产能数量和剩余的产能数量，好根据此来采购入库等，方便其做采销管理。

另一方面，通过延迟预占来区分B端商户和C端用户，防止B端商户（e.g. 品牌专卖店从京东买货）恶意占单，导致C端用户无法下单，影响销售。举个栗子，当大促期间，京东的一些家电的优惠力度特别大，（甚至低于专卖店的采购价），这时，如果是有恶意的B端商家，为了不让C端用户买到这么便宜的商品（好之后从他们那里买），他会立即下大数量的订单，比如10000台，将库存全部占领，导致C端客户无法下单，享受不到优惠。当大促结束之后，在B端商家再取消订单，这样就影响了京东自营商品的销售。


#### 项目开发讲些什么

1. 整体流程，涉及几个系统（inner、stockchange、预占系统、状态机、数据中间件），整体流程是什么：
2. 数据中间件： update时的DataModifyTemplate, queryCo时的DataQueryTemplate+DBTokenAccessRetryTemplate.
3. 预占开发，PaaS,以及符合开闭原则。





### 扣减策略 以及发现的问题









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
4. 如果count==null,`Object lock = getLock(skuId, limitDbType);`,确保同一限流库类型下的同一skuId获取相同的lock。拿到锁之后，double lock check，初始化count：`new AtomicInteger(1)`。如果不为null，原子操作: `int countNum = count.incrementAndGet()`
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

```java

    /**
     * 动态初始化Guava缓存对象
     */
    private void dynamicInitGuavaCache(){
        GuavaConfig config = resourceContainer.getGuavaCacheConfig().getGuavaConfigByAlias(businessAlias);
        // 如果当前对象的刷新标记和远端配置不一致,则重新初始化GuavaCache对象（即远端配置版本发生变化时才重新初始化Guava对象）
        if (StockUtils.isNull(cache) || refreshFlag != config.getRefreshVersion()){
            // 重置刷新标志
            refreshFlag = config.getRefreshVersion();
            //设置并发级别为8，并发级别是指可以同时写缓存的线程数
            int concurrencyLevel = config.getConcurrencyLevel();
            //设置缓存容器的初始容量为10
            int initialCapacity = config.getInitialCapacity();
            //设置缓存最大容量为100，超过100之后就会按照LRU最近虽少使用算法来移除缓存项
            int maximumSize = config.getMaximumSize();
            //缓存项在给定时间内没有被读/写访问，则回收。请注意这种缓存的回收顺序和基于大小回收一样。
            int expireAfterAccess = config.getExpireAfterAccess();
            //缓存项在给定时间内没有被写访问（创建或覆盖），则回收。如果认为缓存数据总是在固定时候后变得陈旧不可用，这种回收方式是可取的。
            int expireAfterWrite = config.getExpireAfterWrite();

            CallerInfo callerInfo = MonitorUtils.registerInfo("JdStock.NewSDK.occupy.GuavaCache.dynamicInitGuavaCache", false, true);
            try {
                //建造者模式
                CacheBuilder cacheBuilder = CacheBuilder.newBuilder();
                cacheBuilder.concurrencyLevel(concurrencyLevel);
                cacheBuilder.initialCapacity(initialCapacity);
                cacheBuilder.maximumSize(maximumSize);
                if (expireFormat == 1) {
                    cacheBuilder.expireAfterAccess(expireAfterAccess, TimeUnit.MILLISECONDS);
                } else if (expireFormat == 2){
                    cacheBuilder.expireAfterWrite(expireAfterWrite, TimeUnit.MILLISECONDS);
                }
                cache = cacheBuilder.build();
            } catch (Exception e) {
                Profiler.functionError(callerInfo);
                LogUtils.error(LogUtils.FILE_EXCEPTION, "Occupy.GuavaCache.dynamicInitGuavaCache.Exception::动态初始化Guava缓存对象出现异常，businessAlias=" + businessAlias, e);
            } finally {
                Profiler.registerInfoEnd(callerInfo);
            }
        }
    }
```

### 最小包裹数
### pipeline 和 dump
### 大keyredis