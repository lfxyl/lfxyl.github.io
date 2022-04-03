---
layout: blog
istop: false
title: "Linux系统监测整理"
category: 知识归纳
background-image: https://b2.kuibu.net/file/imgdisk/imgs/2022/04/5409a8af8839d3f5.png
background: purple
tags:
- Linux c++
- 系统监测
---

最近要做个系统监测模块，对CPU、内存等状态进行监测。由于是C++程序，不希望调用shell命令。查了一些资料，这里做个总结。

## CPU 监测
### CPU 占用率

```bash
$ cat /proc/stat
cpu  776476 1721 220476 2278603 4439 0 3668 0 0 0
cpu0 97389 244 27921 284716 553 0 114 0 0 0
cpu1 95843 135 27343 286773 525 0 91 0 0 0
cpu2 95566 58 27562 285293 602 0 2110 0 0 0
...
```

输出的所有值都是**从系统启动到当前的累计值**。不同内核版本中，输出字段可能有差异。第一列为cpu名，后续各列含义如下：


| 参数         | 解析（单位：jiffies）                                        |
| ------------ | ------------------------------------------------------------ |
| user(638)    | 用户态的运行时间，不包含 nice值为负进程 |
| nice(0)      | nice值为负的进程所占用的CPU时间 |
| system(1677) | 处于核心态的运行时间           |
| idle(868357) | 除IO等待时间以外的其它等待时间 |
| iowait(2397) | IO等待时间(since 2.5.41)       |
| irq(7)       | 硬中断时间(since 2.6.0-test4)  |
| softirq(510) | 软中断时间(since 2.6.0-test4)  |
| steal(0)     | 在虚拟环境中运行时在其他操作系统上花费的时间。(Since Linux 2.6.11) |
| guest(0)     | 在内核的控制下为客户操作系统运行虚拟CPU的时间。(Since Linux 2.6.24) |

> jiffies是内核中的一个全局变量，记录自系统启动以来产生的节拍数。一个节拍大致可理解为操作系统进程调度的最小时间片。不同linux内核可能值有不同，通常在1ms到10ms之间

各列求和可以得到总的CPU时间total。由于以上值为系统启动以后的累计值，要获得瞬时CPU占用率，可以连着读取两次。那么瞬时CPU占用率即为：
$$
    cpuRate = 1- (idle2-idle1)/(total2-total1)
$$

> top 命令也是通过读取/proc/stat来计算CPU状态信息的。

### 系统负载
首先区分下**系统负载**和**CPU占用率**。**CPU占用率**如上一小节所述，是CPU非空闲时间与总时间的比值。而**系统负载**是指特定时间内运行队列中的平均进程数。

```bash
$ cat /proc/loadavg 
1.69 1.52 1.74 2/1964 57889
```

输出前三列分别是1、5、15分钟内的平均进程数。第四列分子为正在运行的进程数，分母是进程总数。最后一列是最近运行的进程ID号。

> 一般来说每个CPU的当前活动进程数不大于3那么系统的性能就是良好的。如果每个CPU的任务数大于5，那么就表明机器的性能有严重问题。
> 对于上面的例子来说，假设系统有8个CPU，那么其每个CPU在1分钟内的进程数为：1.69/8=0.21。

### CPU 温度
```bash
cat /sys/class/thermal/thermal_zone0/temp
```
输出值除以1000就是摄氏度。关于Linux Thermal的更多信息见参考资料3 [Linux Thermal 学习笔记](https://www.cnblogs.com/hellokitty2/p/15600099.html)

### CPU 频率

读取以下三个文件，都可以获得CPU实时频率：
```bash
$ cat /proc/cpuinfo | grep MHz
$ cat /sys/devices/system/cpu/cpuX/cpufreq/cpuinfo_cur_freq
$ cat /sys/devices/system/cpu/cpuX/cpufreq/scaling_cur_freq
```
> cpuX 中的 X 指CPU序号0，1，2，3。。。

> `/sys/devices/system/cpu/cpuX/cpufreq/`目录下，以`cpuinfo`为前缀的文件代表的是CPU硬件上支持的频率，以`scaling`为前缀的文件代表的是可以通过CPUFreq系统用软件进行调节时所支持的频率。

第一个文件内容太多，c++程序里不调用grep，过滤比较麻烦。第二个和第三个理论上应该是一致的，只会有细微差异。但我实际执行时发现我的机子上没有第二个文件。应该新内核废弃了。查[资料](https://bugzilla.kernel.org/show_bug.cgi?id=197009)发现更推荐用第三个。

## 内存监测

```bash
$ cat /proc/meminfo
```
输出内容很多，主要字段含义如下：
- MemTotal：内存总数。系统从加电开始到引导完成，BIOS等要保留一些内存，内核要保留一些内存，最后剩下可供系统支配的内存就是MemTotal。这个值在系统运行期间一般是固定不变的。
- MemFree：空闲内存数。表示系统尚未使用的内存。MemUsed=MemTotal-MemFree就是已被用掉的内存。
- MemAvailable：可用内存数。系统中有些内存虽然已被使用，但这部分内存是可以回收的，比如cache/buffer、slab都有一部分可以回收，所以MemFree不能代表全部可用的内存。这部分可回收的内存加上MemFree才是系统可用的内存，即：MemAvailable≈MemFree+Buffers+Cached。它是内核使用特定的算法计算出来的，是一个估计值。
- Buffer：缓冲区内存数
- Cache：缓存区内存数

buffer与cache的区别：
- Cache解决的是低速存储的问题，就是说数据存储的地方访问速度太慢了，我们将数据从低速存储设备上缓存到高速存储设备上，让数据的读写更快点。这里的速度更多强调的是单次访问时间。
- Buffer解决的是点操作的慢速问题，就是说同样一件事，我们如果一个一个做，因为物理设备或者软件系统的特性可能很慢，但是如果我们一批一批做，可能就很快。本质上就是将点操作转换成批操作。

Cache的存在原因是对资源调用的空间局部性，因此可以把最常用的东西先放到手边，以提高访问速度。Buffer的存在原因是生产者和消费者对资源的生产/速率不一致，就像你家的垃圾桶，你平时的垃圾先扔在垃圾桶里，等垃圾桶满了再扔垃圾。更详细的分析参见知乎问题：[Cache 和 Buffer 都是缓存，主要区别是什么？](https://www.zhihu.com/question/26190832)

## 磁盘监测
可以使用`statfs()`函数获得磁盘使用情况，具体使用方法见参考资料7 [linux下利用statfs函数查看磁盘使用情况](https://www.cnblogs.com/taoye1997/p/11815088.html)

## 网络监测
```bash
$ cat /proc/net/dev
Inter-|   Receive                                                |  Transmit
 face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
    lo: 28113128    7894    0    0    0     0          0         0 28113128    7894    0    0    0     0       0          0
wlp0s20f3: 3987327501 2971563    0    0    0     0          0         0 168990144 1148220    0    0    0     0       0          0
uengine0:    3801      59    0    0    0     0          0        17   204121    2843    0    0    0     0       0          0
vethSQQT4P:    4627      59    0    0    0     0          0         0   205871    2870    0    0    0     0       0          0
```
输出从第三行开始，分别是各网卡收发数据的累计值。那么间隔时间t读取两次，取差值再除以时间t便可得到当前网速。



## 参考资料

1. [linux系统/proc/stat信息与top的cup信息的联系及区别](https://www.cnblogs.com/harrymore/p/9094898.html#_label1_0)
2. [/proc/loadavg](https://blog.csdn.net/b2222505/article/details/54135805)
3. [Linux Thermal 学习笔记](https://www.cnblogs.com/hellokitty2/p/15600099.html)
4. [Linux动态频率调节系统CPUFreq之一：概述](https://blog.csdn.net/droidphone/article/details/9346981)
5. [Linux之网络实时检测功能](https://www.cnblogs.com/johnnyzen/p/8009053.html)
6. [Linux MemFree与MemAvailable的区别](https://blog.51cto.com/xujpxm/1961072)
7. [linux下利用statfs函数查看磁盘使用情况](https://www.cnblogs.com/taoye1997/p/11815088.html)

