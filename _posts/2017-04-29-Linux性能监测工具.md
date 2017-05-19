---
layout:     post
title:      "Linux性能监测工具"
subtitle:   "CPU, memory, network I/O, disk I/O and progress"
author:     "Chris"
header-img: "img/post-bg-6.jpg"
tags:
    - Linux
---

## 监测要点

- CPU - CPU的占有率
- 内存 - 内存的占用率, 换页数
- I/O - 读写请求数, 读写量等
- 带宽 - 进站出站带宽占有率

## CPU与内存监控

### top

```shell
top - 
	14:04:19(当前时间) 
	up 145 days,  3:45(已运行时间),  
	4 users(当前在线用户数),  
load average: 
	0.09(1分钟内的平均负载), 
	0.19(5分钟内的平均负载), 
	0.14(15分钟内的平均负载)
Tasks: 
	304 total(总进程数),   
	1 running(当前进程运行数), 
	303 sleeping(睡眠进程数),   
	0 stopped(停止的进程数),   
	0 zombie(僵尸进程数)
Cpu(s):  
	2.7%us(用户占用百分比),  
	0.3%sy(内核占用百分比),  
	0.0%ni(nice进程的占用百分比), 
	96.2%id(空闲百分比),  
	0.7%wa(IO等待百分比),  
	0.0%hi(硬中断百分比),  
	0.0%si(软中断百分比),  
	0.0%st
Mem:   
	8012288k total(物理内存总量),  
	7459432k used(已使用的物理内存),   
	552856k free(剩余的内存),   
	341600k buffers(用作系统缓存的内存)
Swap:  
	8159224k total(虚拟内存总量),  
	3798692k used(已使用的虚拟内存),  
	4360532k free(剩余的虚拟内存),   
	613344k cached(已被缓存的交换区总量)

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
 1725 root      20   0 10888  360  236 S  2.0  0.0  21:09.29 irqbalance
```

load average 平均负载要根据实际的CPU物理核心数来进行评估, 若15分钟内的平均负载依然是大于其物理核心数那就需要注意

| 列名      | 含义                                       |
| :------ | :--------------------------------------- |
| PID     | 进程id                                     |
| PPID    | 父进程id                                    |
| RUSER   | Real user name                           |
| UID     | 进程所有者的用户id                               |
| USER    | 进程所有者的用户名                                |
| GROUP   | 进程所有者的组名                                 |
| TTY     | 启动进程的终端名。不是从终端启动的进程则显示为 ?                |
| PR      | 优先级                                      |
| NI      | nice值。负值表示高优先级，正值表示低优先级                  |
| P       | 最后使用的CPU，仅在多CPU环境下有意义                    |
| %CPU    | 上次更新到现在的CPU时间占用百分比                       |
| TIME    | 进程使用的CPU时间总计，单位秒                         |
| TIME+   | 进程使用的CPU时间总计，单位1/100秒                    |
| %MEM    | 进程使用的物理内存百分比                             |
| VIRT    | 进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES           |
| SWAP    | 进程使用的虚拟内存中，被换出的大小，单位kb。                  |
| RES     | 进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA     |
| CODE    | 可执行代码占用的物理内存大小，单位kb                      |
| DATA    | 可执行代码以外的部分(数据段+栈)占用的物理内存大小，单位kb          |
| SHR     | 共享内存大小，单位kb                              |
| nFLT    | 页面错误次数                                   |
| nDRT    | 最后一次写入到现在，被修改过的页面数。                      |
| S       | 进程状态。            D=不可中断的睡眠状态            R=运行            S=睡眠            T=跟踪/停止            Z=僵尸进程 |
| COMMAND | 命令名/命令行                                  |
| WCHAN   | 若该进程在睡眠，则显示睡眠中的系统函数名                     |
| Flags   | 任务标志，参考 sched.h                          |

#### 交互命令

`<space>` 即时刷新

`1` 查看各个逻辑核的负载

`d` 更改刷新的时间间隔

`G` 翻页

`u` 监控指定的用户

`k` 结束进程

`r` renice进程

`c` 在COMMAND一栏显示详细的命令

`o` 更改列的显示顺序

`f` 增加或移除列的显示

`H` 显示线程

`i` 显示空闲的进程的开关

`n` 指定只显示的前n个进程

`M` 按照内存使用的倒序排序

`P` 按照CPU使用率的倒序排序

`T` 按照Time的倒序排序

`F`或`O` 选择排序的列

`R` 逆向排序

#### 命令行参数

`-a` 按照内存使用率排序

`-b` 对文件或管道输出友好的Batch Mode

`-c` 显示详细执行的COMMAND命令

`-H` 显示线程

`-i` 不显示空闲的进程

`-n` 执行的次数, 如top -b -n 1 > top.txt

`-u` 显示指定用户相关的进程

`-p` 显示指定PID的进程, 多个用逗号分隔

`-M` 显示内存单位

### mpstat

每2秒采样一次, 监控所有CPU的状况, 采样5次后结束

```shell
mpstat -P ALL 2 5
```

采样一次, 显示CPU的平均情况

```shell
[chris@localhost ~]$ mpstat
Linux 2.6.32-358.el6.x86_64 (localhost) 	04/05/2017 	_x86_64_	(4 CPU)

05:12:24 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest   %idle
05:12:24 PM  all    2.74    0.00    0.34    0.69    0.00    0.02    0.00    0.00   96.22
```

| 参数      | 描述                                       |
| :------ | :--------------------------------------- |
| CPU     | 处理器编号. 关键字all表示统计信息是以所有处理器的平均值计算而得       |
| %user   | 应用程序(user level)执行时CPU的利用率               |
| %nice   | 优先级高的应用程序(user level)执行时的CPU利用率          |
| %sys    | 系统级(system level)执行时的CPU利用率. 注意, 这不包括用于维护硬中断和软中断的时间 |
| %iowait | 在系统处理磁盘I/O请求时, CPU或CPUs的空闲时间百分比          |
| %irq    | 显示CPU或CPUs用于花费在**硬**中断的时间百分比             |
| %soft   | 显示CPU或CPUs用于花费在**软**中断的时间百分比             |
| %steal  | 显示管理程序(hypervisor)在维护另一个虚拟处理器时虚拟CPU或CPUs在非自愿等待中花费的时间百分比 |
| %idle   | 在系统没有磁盘I/O时, CPU或CPUs的空闲时间百分比            |

考究地址

https://linux.die.net/man/1/mpstat

### vmstat

每2秒采样一次, 采样5次后结束

```shell
vmstat 2 5
```

采样一次

```shell
[chris@localhost ~]$ vmstat
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0 3798788 476060 343644 620288    1    1     3    26    1    1  3  0 96  1  0
```

| 参数   | 解释                                       |
| ---- | ---------------------------------------- |
| r    | 等待获得run time的进程数, 对应running和runnable状态   |
| b    | 处于uninterruptible sleep状态(也就是D状态)的进程数, 对应blocked状态, 通常是在等待磁盘I/O或者网络I/O |
| swpd | 已使用的虚拟内存的大小, 如果大于0则说明可能要升级机器(记得要排除内存溢出的状态) |
| free | 剩余的物理内存大小                                |
| si   | 每秒从SWAP虚拟内存中<u>换入</u>至RAM内存的数据大小         |
| so   | 每秒从RAM内存<u>换出</u>至虚拟内存SWAP的数据大小          |
| bi   | 从块设备读入的数量                                |
| bo   | 写至块设备的数量                                 |
| in   | 每秒发生中断的次数                                |
| cs   | 每秒上下文切换的次数，例如线程切换过多该数值便会上升               |
| us   | 运行非内核代码的时间(user time, 包括nice time)       |
| sy   | 运行内核代码的时间(system time)                   |
| id   | CPU空闲时间, 包含IO-wait的时间                    |
| wa   | 等待IO的时间, 包含在CPU空闲时间中                     |
| st   | 在文档中描述为Time stolen from a virtual machine. 但我认为与mpstat中的%steal的意思是一样的 |

考究地址

https://linux.die.net/man/8/vmstat

### sar

#### 命令行参数

`-f` 从filename中提取记录

#### 查看任务队列的繁忙程度

```shell
[chris@localhost sa]$ sar -q -f sa08
Linux 2.6.32-358.el6.x86_64 (localhost) 	03/08/2017 	_x86_64_	(4 CPU)

12:00:01 AM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15
12:10:01 AM         2      1430      0.04      0.06      0.07
12:20:01 AM         0      1420      0.04      0.05      0.05
12:30:01 AM        16      1460      0.01      0.06      0.06
12:40:01 AM         0      1426      0.36      0.24      0.14
12:50:01 AM         0      1420      0.06      0.13      0.13
01:00:01 AM        13      1450      0.02      0.10      0.10
```

| 参数       | 解释                  |
| -------- | ------------------- |
| runq-sz  | 运行队列的长度（等待运行的任务数）   |
| plist-sz | 队列中的任务总数            |
| ldavg-x  | 与top中load average一样 |

#### 查看CPU的占用率

```shell
[chris@localhost sa]$ sar -p -f sa08
Linux 2.6.32-358.el6.x86_64 (localhost) 	03/08/2017 	_x86_64_	(4 CPU)

12:00:01 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
12:10:01 AM     all      3.14      0.00      0.43      0.79      0.00     95.64
12:20:01 AM     all      2.61      0.00      0.38      0.78      0.00     96.23
12:30:01 AM     all      2.40      0.00      0.38      0.73      0.00     96.49
```

里面的参数说明与`mpstat`一致

#### 内存利用率统计信息

```shell
[chris@localhost sa]$ sar -r -f sa08
Linux 2.6.32-358.el6.x86_64 (localhost) 	03/08/2017 	_x86_64_	(4 CPU)

12:00:01 AM kbmemfree kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit
12:10:01 AM    452360   7559928     94.35    138508    314660  16071940     99.38
12:20:01 AM    458024   7554264     94.28    139124    314652  16059996     99.31
12:30:01 AM    444256   7568032     94.46    139140    314608  16105608     99.59
```

cache是对文件的缓存, buffer是对磁盘块的缓存(比文件缓存更底层)

| 参数        | 解释                                       |
| --------- | ---------------------------------------- |
| kbmemfree | 与free命令一致, 代表空余的物理内存, 不包括buffer与cache    |
| kbmemused | 与free命令一致, 代表已用的物理内存, 包含buffer与cache     |
| %memused  | kbmemused与物理内存总量的比值                      |
| kbbuffers | 文件的缓存                                    |
| kbcached  | 磁盘块的缓存                                   |
| kbcommit  | 当前工作负载所需的内存量（以kB计）。这是估计需要多少RAM /交换来保证永远不会有内存不足。 |
| %commit   | 相对于内存总量（RAM +SWAP），当前工作负载所需的内存所占百分比。 这个数字可能会大于100％，因为内核通常会占用内存。 |

%memused + %commit > 100时, 说明物理内存不足, 这样会导致频繁的换页

#### 内存的分页统计(换页的频繁程度)

有换入换出就证明有磁盘I/O, 性能就会收到冲击. 我们主要关注pgpgin/s与pgpgout/s

```shell
[chris@localhost sa]$ sar -B -f sa08
Linux 2.6.32-358.el6.x86_64 (localhost) 	03/08/2017 	_x86_64_	(4 CPU)

12:00:01 AM  pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
12:10:01 AM      7.49     90.25    908.74      0.08    835.69      0.00      0.32      0.23     70.83
12:20:01 AM      1.22     78.41    755.53      0.02    637.72      0.00      0.54      0.43     80.00
12:30:01 AM      0.05     75.95    703.74      0.00    602.02      0.00      0.43      0.41     96.88
```

| 参数        | 解释                        |
| --------- | ------------------------- |
| pgpgin/s  | 表示每秒从磁盘或SWAP置换到内存的字节数(KB) |
| pgpgout/s | 表示每秒从内存置换到磁盘或SWAP的字节数(KB) |

## 磁盘I/O监控

### iostat

`-d` 只显示磁盘使用信息

`-k` 以kilobytes/second显示统计信息

`-m` 以megabytes/second显示统计信息

`-x` 显示扩展信息

```shell
[root@d02645f01e79 /]# iostat -dkx 2
Linux 4.9.12-moby (d02645f01e79) 	04/05/17 	_x86_64_	(4 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     0.20    0.25    0.25     3.62     8.25    46.99     0.00    4.70    1.28    8.17   2.76   0.14
```

| 参数       | 描述                                       |
| :------- | :--------------------------------------- |
| rrqm/s   | 每秒进行合并的**读**请求数                          |
| wrqm/s   | 每秒进行合并的**写**请求数                          |
| r/s      | 每秒向设备发出的读取请求数                            |
| w/s      | 每秒向设备发出的写入请求数                            |
| rkB/s    | 从设备每秒读取的kilobytes数                       |
| wkB/s    | 从每秒写入设备的kilobytes数                       |
| avgrq-sz | 发给设备的请求的平均大小（以扇区为单位）                     |
| avgqu-sz | 发送到设备的请求的平均队列长度                          |
| await    | 发给设备的I/O请求的平均时间（以毫秒为单位）。这包括请求在队列中花费的时间以及为维护请求所花费的时间。 |
| svctm    | 发送到设备的I/O请求的平均服务时间（以毫秒为单位）。**警告！不要再相信这个字段了。此字段将在未来的sysstat版本中被删除。** |
| %util    | 向设备发出I/O请求的CPU时间百分比（设备的带宽利用率）。当该值接近100％时说明已达到饱和状态。 |

考究地址

https://linux.die.net/man/1/iostat

## 网络监控

### sar

#### 命令行参数

`-f` 从filename中提取记录

#### 网络统计报告

DEV显示网络接口信息

```shell
[chris@localhost sa]$ sar -n DEV -f sa08
Linux 2.6.32-358.el6.x86_64 (localhost) 	03/08/2017 	_x86_64_	(4 CPU)

12:00:01 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
12:10:01 AM        lo    112.86    112.86     32.09     32.09      0.00      0.00      0.00
12:10:01 AM      eth0      6.03      9.77      1.12      1.53      0.00      0.00      0.25
12:10:01 AM      eth1      0.00      0.00      0.00      0.00      0.00      0.00      0.00
12:20:01 AM        lo    113.16    113.16     32.11     32.11      0.00      0.00      0.00
```

| 参数       | 解释         |
| -------- | ---------- |
| IFACE    | 网卡         |
| rxpck/s  | 每秒接收的数据包   |
| txpck/s  | 每秒发送的数据包   |
| rxkB/s   | 每秒接收的字节数   |
| txkB/s   | 每秒发送的字节数   |
| rxcmp/s  | 每秒接收的压缩数据包 |
| txcmp/s  | 每秒发送的压缩数据包 |
| rxmcst/s | 每秒接收的多播数据包 |

EDEV关键字会报告来自网络设备故障的统计信息

```shell
Linux 2.6.32-358.el6.x86_64 (WebSrv)    03/31/2017      _x86_64_        (4 CPU)

12:00:01 AM     IFACE   rxerr/s   txerr/s    coll/s  rxdrop/s  txdrop/s  txcarr/s  rxfram/s  rxfifo/s  txfifo/s
12:10:01 AM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
12:10:01 AM      eth0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
12:10:01 AM      eth1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

| 参数          | 解释                            |
| ----------- | ----------------------------- |
| **rxerr/s** | 每秒接收的坏数据包                     |
| **txerr/s** | 每秒发送的坏数据包                     |
| coll/s      | 每秒冲突数                         |
| rxdrop/s    | 由于linux缓冲区中缺少空间，每秒丢弃的已接收数据包数。 |
| txdrop/s    | 由于linux缓冲区中缺少空间，每秒丢弃的已发送数据包数。 |



