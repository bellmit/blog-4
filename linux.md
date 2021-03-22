[知乎：39条常见的Linux系统简单面试题](https://zhuanlan.zhihu.com/p/32250942)

[菜鸟教程](https://www.runoob.com/w3cnote/linux-view-disk-space.html)
#### vmstat r, b, si, so, bi, bo 这几列表示什么含义呢？

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
bo表示有多少数据从内存写入磁盘

简记：

i --input，进入内存
o --output，从内存出去
s --swap，交换分区
b --block，块设备，磁盘

#### 查看磁盘空间

df


#### 有一天你突然发现公司网站访问速度变的很慢很慢，你该怎么办呢？

（服务器可以登陆，提示：你可以从系统负载和网卡流量入手）

答：可以从两个方面入手分析：分析系统负载，使用w命令或者uptime命令查看系统负载，如果负载很高，则使用`top`命令查看CPU，MEM等占用情况，要么是CPU繁忙，要么是内存不够，如果这二者都正常，再去使用sar命令分析网卡流量，分析是不是遭到了攻击。一旦分析出问题的原因，采取对应的措施解决，如决定要不要杀死一些进程，或者禁止一些访问等。
