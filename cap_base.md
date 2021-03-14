### CAP

Partition tolerance: Continues to function even if there is a “partition”.在分布式系统中， 分区容忍性是指即使分区现象发生，系统仍然可以工作的性质。

Availability: Reads and writes always succeed. 在分布式系统中，可用性是指每次请求都能获取到非错的响应的性质。Every request received by a non-failing node in the system must result in a response. That is, any algorithm used by the service must eventually terminate.

网络分区发生可能持续任意长时间， 如果被请求的节点在一定时间内无法响应， 或者直接简单地响应一个超时或者拒绝服务的错误，都是不具备可用性的。

Consistency: two reads return the same value. 分布式系统中多个节点的数据返回始终一致的性质。 
Any read operation that begins after a write operation completes must return that value, or the result of a later write operation.

All nodes see the same data at the same time.

一致性的代价 - 响应时长 

回头看下 CAP 定理关于可用性的定义， 其中并没有对响应时限做出约束。 如果一个客户端请求的响应非常非常慢，但是最终得到了响应，在 CAP 的范畴下，这仍然是具有可用性的。

更广义的可用性， 应该是在约定时限的要求下做出响应。 现实中，人们往往是非常关心响应时间的。 我们将看到， 强一致性的代价就是相对高的响应时长。

[stackoverflow: how to understand partition tolerance](https://stackoverflow.com/questions/12346326/cap-theorem-availability-and-partition-tolerance)

[春水煎茶：分布式的 CAP 定理和一致性模型](https://writings.sh/post/cap-and-consistency-models)