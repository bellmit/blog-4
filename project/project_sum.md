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


### 扣减策略 以及发现的问题

### 限流策略


### 最小包裹数