---
layout: blog
istop: true
title: "内存泄漏分析工具总结"
category: 笔记
background-image: https://s2.loli.net/2021/12/12/DAliMjwErWc89dH.jpg
background: purple
tags:
- 内存泄漏
- valgrind
- massif
- tcmalloc
---

最近排查了一个内存泄漏问题，比较难搞，尝试了各种工具。这里把尝试过的工具做个总结。

首先区分两个概念，memory check和memory profile。这两个概念在相关的文档中经常出现。memory check是指对内存泄漏等进行检查，输出检查结果。memory profile顾名思义，是指对内存使用情况进行快照，输出当前时刻每个函数的内存占用情况（轮廓）。

## valgrind

Linux系统一般都自带了Valgrind。Valgrind是一套开放源代码（GPLV2）的仿真调试工具的集合，由内核（core）以及基于内核的其他调试工具组成。

内核类似于一个框架（framework），它模拟了一个CPU环境，并提供服务给其他工具；而其他工具则类似于插件（plug-in），利用内核提供的服务完成各种特定的内存调试任务。

Valgrind的体系结构如下图所示：
![1.png](https://s2.loli.net/2021/12/12/oRgquMNZ1P7wnze.jpg)

内存有关的插件有massif和memcheck。

### memcheck

memcheck是valgrind默认加载的插件，其可以检查多种内存使用错误，不仅限于内存泄漏。Valgrind常用的选项如下：

- \-\-log-file 报告文件名。如果没有指定，输出到stderr。
- \-\-tool=memcheck 指定Valgrind加载的插件。memcheck是缺省项。
- \-\-leak-check 指定如何报告内存泄漏，可选值有:
    - no 不报告
    - summary 缺省值,显示简要信息（有多少个内存泄漏等）。
    - yes 和 full 显示每个泄漏的内存在哪里分配。
- \-\-show-leak-kinds 指定内存泄漏的类型。可选的类型有definite, indirect, possible, reachable。缺省值是definite,possible。也可以指定all或none。

```shell
valgrind --log-file=valgrind.log --tool=memcheck --leak-check=full --show-leak-kinds=all ./your_app arg1 arg2
```

memcheck可以检查出的错误包括：
- illegal read/illegal write：内存越界读写。
- use of uninitialised values：使用未初始化的值。
- use of uninitialised or unaddressable values in system calls：系统调用使用未初始化的值或不能访问的地址。
- illegal fress：非法释放。
- mismatched free：不匹配的释放。比如使用new[]申请，delete释放。
- source and destination overlap：比如memcpy()中源地址与目标地址重叠。
- silly arg：函数参数不合法。比如给malloc()传了个负值。
- memory leak detection：memcheck输出的报告最后会对内存泄漏的分析，会报告每一处泄漏内存的分配位置，以及有关内存泄漏的统计总结，例如：
    ```shell
        ==3869== HEAP SUMMARY:
        ==3869==     in use at exit: 100 bytes in 1 blocks
        ==3869==   total heap usage: 1 allocs, 0 frees, 100 bytes allocated
        ==3869== 
        ==3869== 100 bytes in 1 blocks are definitely lost in loss record 1 of 1
        ==3869==    at 0x4C2B6CD: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
        ==3869==    by 0x400798: main (main.cpp:10)
        ==3869== 
        ==3869== LEAK SUMMARY:
        ==3869==    definitely lost: 100 bytes in 1 blocks
        ==3869==    indirectly lost: 0 bytes in 0 blocks
        ==3869==    possibly lost: 0 bytes in 0 blocks
        ==3869==    still reachable: 0 bytes in 0 blocks
        ==3869==    suppressed: 0 bytes in 0 blocks
    ```

关于memleak更详细的使用和原理见参考资料1 [Valgrind memcheck 用法](https://www.jianshu.com/p/78adaba826c3)。

### massif

memchek输出的报告可能不足以分析出程序中的内存泄漏问题，比如栈内存增长（当然，这可能不算泄漏）等。可以使用massif进一步分析。massif会在程序运行过程中对内存使用情况进行快照，通过对比不同时刻的快照，就可以找出那些一直增长的内存。

massif常用选项有：
- \-\-stacks: 栈内存的采样开关，默认关闭。打开后，会针对栈上的内存也进行采样，会使 massif 性能变慢。
- \-\-time-unit：massif按一定的间隔生成内存快照，这个选项指定按什么计算间隔。有三个可选值：
    - i：默认值，按执行的指令算，用于大多数情况。
    - ms：按时间间隔，可用于某些特定事务。
    - B：按分配的内存量，用于很少运行的程序，且用于测试目的，因为它最容易在不同机器中重现。
- \-\-detailed-freq: 详细内存快照的频率，默认是10，即每10个快照会有采集一个详细的内存快照。详细内存快照会消耗更多资源，使massif运行程序变慢，因此默认不是每个都是详细快照。
- \-\-massif-out-file：采样结束后，生成的采样文件。
- \-\-depth：详细快照的最大分配深度。massif输出结果以树状形式显示内存的分配关系（即会追踪内存

```shell
valgrind --tool=massif --time-unit=B --detailed-freq=1 --massif-out-file=./massif.out  ./your_proc args
```

valgrind在程序结束时产生输出。如果是长时间运行的服务器程序，那么需要手动停止程序来生成采样文件。

> 注意：不能使用 `kill -9`去杀死程序，否则不会生成采样文件。用`kill`就好。

massif输出的采样文件可以用ms_print命令来分析：

```shell
ms_print ./massif.out > massif.log
```

massif.log长这样：

```shell
--------------------------------------------------------------------------------
Command:            ./your_proc args
Massif arguments:   --time-unit=B --massif-out-file=./massif.out
ms_print arguments: massif.out
--------------------------------------------------------------------------------

    GB
1.279^                                                                       #
     |                                                                       #
     |                                                                   @  @#
     |                                                                   @::@#
     |                                                                 @:@: @#
     |                                                            @::  @:@: @#
     |                                                      : ::::@: ::@:@: @#
     |                                             @ @@@@ :::::: :@: : @:@: @#
     |                                          :  @:@ @ @: :::: :@: : @:@: @#
     |                                     @  :::::@:@ @ @: :::: :@: : @:@: @#
     |                               @@:::@@::: :: @:@ @ @: :::: :@: : @:@: @#
     |                            :::@ : :@@: : :: @:@ @ @: :::: :@: : @:@: @#
     |                    :: @@::::: @ : :@@: : :: @:@ @ @: :::: :@: : @:@: @#
     |                 :::: :@ :: :: @ : :@@: : :: @:@ @ @: :::: :@: : @:@: @#
     |          @  :::::::: :@ :: :: @ : :@@: : :: @:@ @ @: :::: :@: : @:@: @#
     |        ::@::: : :::: :@ :: :: @ : :@@: : :: @:@ @ @: :::: :@: : @:@: @#
     |      ::::@: : : :::: :@ :: :: @ : :@@: : :: @:@ @ @: :::: :@: : @:@: @#
     |     :: ::@: : : :::: :@ :: :: @ : :@@: : :: @:@ @ @: :::: :@: : @:@: @#
     |   @@:: ::@: : : :::: :@ :: :: @ : :@@: : :: @:@ @ @: :::: :@: : @:@: @#
     | ::@ :: ::@: : : :::: :@ :: :: @ : :@@: : :: @:@ @ @: :::: :@: : @:@: @#
   0 +----------------------------------------------------------------------->GB
     0                                                                   813.9

Number of snapshots: 68
 Detailed snapshots: [2, 7, 16, 21, 24, 25, 30, 32, 33, 34, 41, 44, 46, 48, 51, 52, 58, 59, 61, 64, 65, 66, 67 (peak)]
```

这个图显示程序累计申请了813.9G内存，当前占用内存一直在上升，因此可能存在内存泄漏。图下面的信息显示共生成了68次快照，detailed snapshots显示了详细快照的序号。

继续往下看的化，是各个内存快照的分配栈，即每个函数占用多少内存：
```shell
  0              0                0                0             0            0
  1 20,021,463,688      133,278,776      124,687,612     8,591,164            0
  2 45,201,848,936      204,228,232      191,089,596    13,138,636            0
93.57% (191,089,596B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
->41.07% (83,886,080B) 0xF088E6: rocksdb::Arena::AllocateNewBlock(unsigned long) (in /chain/xtopchain)
| ->41.07% (83,886,080B) 0xF08500: rocksdb::Arena::AllocateFallback(unsigned long, bool) (in /chain/xtopchain)
|   ->41.07% (83,886,080B) 0xF0886C: rocksdb::Arena::AllocateAligned(unsigned long, unsigned long, rocksdb::Logger*) (in /chain/xtopchain)
|     ->41.07% (83,886,080B) 0xDE62BC: rocksdb::ConcurrentArena::AllocateAligned(unsigned long, unsigned long, rocksdb::Logger*)::{lambda()
|     | ->41.07% (83,886,080B) 0xDE7D9A: char* rocksdb::ConcurrentArena::AllocateImpl<rocksdb::ConcurrentArena::AllocateAligned(unsigned long, unsigned long, rocksdb::Logger*)::{lambda()
|     |   ->41.07% (83,886,080B) 0xDE6371: rocksdb::ConcurrentArena::AllocateAligned(unsigned long, unsigned long, rocksdb::Logger*) (in /chain/xtopchain)
|     |     ->41.07% (83,886,080B) 0xE6FAB0: rocksdb::InlineSkipList<rocksdb::MemTableRep::KeyComparator const&>::AllocateNode(unsigned long, int) (in /chain/xtopchain)
|     |       ->41.07% (83,886,080B) 0xE6F472: rocksdb::InlineSkipList<rocksdb::MemTableRep::KeyComparator const&>::AllocateKey(unsigned long) (in /chain/xtopchain)
|     |         ->41.07% (83,886,080B) 0xE6E40A: rocksdb::(anonymous namespace)::SkipListRep::Allocate(unsigned long, char**) (in /chain/xtopchain)
|     |           ->41.07% (83,886,080B) 0xDE32E3: rocksdb::MemTable::Add(unsigned long, rocksdb::ValueType, rocksdb::Slice const&, rocksdb::Slice const&, bool, rocksdb::MemTablePostProcessInfo*) (in /chain/xtopchain)
|     |             ->41.07% (83,886,080B) 0xE5C218: rocksdb::MemTableInserter::PutCFImpl(unsigned int, rocksdb::Slice const&, rocksdb::Slice const&, rocksdb::ValueType) (in /chain/xtopchain)
|     |               ->41.07% (83,886,080B) 0xE5C92C: rocksdb::MemTableInserter::PutCF(unsigned int, rocksdb::Slice const&, rocksdb::Slice const&) (in /chain/xtopchain)
|     |                 ->41.07% (83,886,080B) 0xE570E4: rocksdb::WriteBatch::Iterate(rocksdb::WriteBatch::Handler*) const (in /chain/xtopchain)
|     |                   ->41.07% (83,886,080B) 0xE598D5: rocksdb::WriteBatchInternal::InsertInto(rocksdb::WriteThread::WriteGroup&, unsigned long, rocksdb::ColumnFamilyMemTables*, rocksdb::FlushScheduler*, bool, unsigned long, rocksdb::DB*, bool, bool, bool) (in /chain/xtopchain)
|     |                     ->41.07% (83,886,080B) 0xD45AD7: rocksdb::DBImpl::WriteImpl(rocksdb::WriteOptions const&, rocksdb::WriteBatch*, rocksdb::WriteCallback*, unsigned long*, unsigned long, bool, unsigned long*, unsigned long, rocksdb::PreReleaseCallback*) (in /chain/xtopchain)
|     |                       ->28.75% (58,720,256B) 0x1013B9C: rocksdb::WriteCommittedTxn::CommitWithoutPrepareInternal() (in /chain/xtopchain)
|     |                       | ->28.75% (58,720,256B) 0x1013653: rocksdb::PessimisticTransaction::Commit() (in /chain/xtopchain)
|     |                       |   ->28.75% (58,720,256B) 0xF40E17: rocksdb::PessimisticTransactionDB::Put(rocksdb::WriteOptions const&, rocksdb::ColumnFamilyHandle*, rocksdb
```

使用compare等软件对比不同时刻快照的分配栈，找出哪些函数占用的内存一直在增长，就可以定位出内存泄漏的可能位置。这不仅限于堆内存，一直增长的栈内存也可以找出来。

除了ms_print这样的命令行工具，massif-visualizer可以对massif输出的采样文件做可视化分析。但该工具目前只有Linux版，且需要GUI环境。

关于valgrind更详细的使用可以参考其[官方文档](https://valgrind.org/)。

## tcmalloc(gperftools)

gperftools实现的tcmalloc是一个高性能的多线程内存分配器，可以替换默认的glibc中内存分配相关函数（malloc、free，new，new[]等）。同时gperftools提供了一些性能分析工具，可以对内存、cpu使用等进行分析。这里我们只关注如何用tcmalloc来分析内存泄漏。

### tcmalloc 生成内存快照

一般系统默认未安装tcmalloc，使用前需要先安装。只要下载源码后将其编译成`so`文件即可。

tcmalloc的使用有两种方式：

- 侵入式：
    1. 将tcmalloc链接进程序（-ltcmalloc）。
    2. 设置环境变量HEAPPROFILE，这个环境变量指定tcmalloc的输出文件路径：`export HEAPPROFILE=/tmp/heapprof`。
    3. 运行程序：`<path/to/binary> [binary args]`。
    4. 使用pprf分析输出文件。
- 非侵入式：tcmalloc也可以不直接链接到程序中，即上述侵入式方式中的第一步可以省略，但第二步需要多设置一个环境变量：`export LD_PRELOAD=<path/to/tcmalloc/so>`。

> 注意：
> 如果要将tcmalloc链接到程序中，那么tcmalloc需要最后链接。
> HEAPPROFILE指定的路径最后一个是生成快照的前缀，比如上述`HEAPPROFILE=/tmp/heapprof`的设置，会在tmp目录下生成heapprof.0001.heap，heapprof.0002.heap。。。这样的内存快照文件。

另外，还可设置以下环境变量来控制tcmalloc的更多行为：

- HEAPCHECK：打开内存泄漏检查。可选值有normal、strict、draconian。
- HEAP_PROFILE_ALLOCATION_INTERVAL：生成内存快照的间隔。默认每申请1GB内存，生成一次内存快照。单位为字节。
- HEAP_PROFILE_INUSE_INTERVAL：默认值为100MB。“最高内存占用值”每上升100MB，生成一次内存快照。单位为字节。
- HEAP_PROFILE_TIME_INTERVAL：默认值为0。设置该值后，按固定时间间隔生成内存快照，而不是按内存申请量。单位为秒。（实际上，这个选项设置后，时间和内存申请量达到设置值后，都会生成快照）。
- HEAPPROFILESIGNAL：当指定的信号发送给进程后，生成内存快照。（kill命令可以给进程发送信号）。默认关闭。（注意这个环境变量单词间没有空格）。
- HEAP_PROFILE_MMAP：除了malloc, calloc, realloc, 和new，还会对mmap, mremap 和sbrk进行分析。默认值为false。
- HEAP_PROFILE_ONLY_MMAP：只对mmap, mremap 和sbrk进行分析。默认值为false。

### 使用pprof分析内存快照

不同于ms_print只能输出文本，tcmalloc输出的文件可以用pprof转换成可视化的函数调用关系图，支持svg，pdf等格式。

```shell
pprof --svg <path/to/binary> /tmp/heapprof.0001.heap > tc.log
```

生成的结果类似下图：
![heap.png](https://s2.loli.net/2021/12/12/G6FRrS7lwWtZT9i.png)
其中 176.2（17%）of 729.9（70%）表示当前节点及其子节点共占用内存729.9M，当前节点占用176.2M。

pprof常用的选项有：

- \-\-base：指定另一个内存快照，这样输出的函数调用图是两个快照中数据相减之后的结果。不需要手动去compare。
- \-\-lines：在输出中显示具体占用内存的代码行号。
- \-\-edgefraction：会给当前内存占用总量乘上这个系数，低于乘积的边不显示。可以使用`1e-10`这样的格式。
- \-\-nodefraction：与edgefraction类似，用于过滤较小的节点。当需要显示所有节点和边时，可以将这两个值设为0。

## mtrace

待续

## pmap

待续








## 参考资料
1. [Valgrind memcheck 用法](https://www.jianshu.com/p/78adaba826c3)
2. [valgrind massif 分析内存问题](https://rebootcat.com/2020/06/16/valgrind_massif_memory_analysing)
3. [使用 mtrace 分析内存泄露](https://zhuanlan.zhihu.com/p/83547768)

