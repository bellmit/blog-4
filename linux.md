[知乎：39条常见的Linux系统简单面试题](https://zhuanlan.zhihu.com/p/32250942)


vmstat r, b, si, so, bi, bo 这几列表示什么含义呢？

答：

[root@centos6 ~ 10:57 #39]# vmstat
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
0  0      0 1783964  13172 106056    0    0    29     7   15   11  0  0 99  0  0
r即running，表示正在跑的任务数
b即blocked，表示被阻塞的任务数
si表示有多少数据从交换分区读入内存
so表示有多少数据从内存写入交换分区
bi表示有多少数据从磁盘读入内存