---
layout: post
title: 性能测试和调优
category: thinking
tags: [pyinotify]
---

性能测试和调优是一个比较复杂的问题，涉及到很多方面，这里总结一下之前做性能测试，查找系统瓶颈的经验。业界已经有人写了很多关于性能优化方法的文章，如：coolshell的博主陈皓的[《性能调优攻略》][perf]

#系统性能的定义
对系统性能的度量，是我们后续定位和优化的第一步，开展性能测试，一般都是先测试出单机单模块的性能，根据这个基础性能，对各个模块的配比有个基本的认识。在组合出最平衡的组合后，对其进行性能测试，线上也按照这个比例上线，基本可以确定整个集群的服务能力。

一般我们无法直接对整个集群的服务能力进行测试，比如线上1000台云存储服务的集群，可能没有条件对这1000台进行性能测试，因为你需要大量的client机器，当然这种计算可能会不太准，会有一定比例的折损，因为大的系统中可能涉及网络通讯，试错等逻辑，导致实际结果偏低。

系统性能的定义一般有三个指标：

1. **QPS**(query per second) 每秒请求数，即服务器每秒能够服务的请求数目，单位：次/s；
2. **Latency** 时延，即每次请求所花费的时间，单位：ms；
3. **Throughput** 吞吐量 即网路卡上的数据流量，单位：KB/s, MB/s, etc；

三者之间有一定的制约关系：

1. 一般QPS越高，latency会越长，Throughput会越大，因为系统会更繁忙，可能会存在额外的调度等待导致latency变长，Throughput也会响应增加；
2. Latency越低，代表请求很快就结束了，这样单位时间内能够容纳的请求就越多，QPS就可以越高；
3. Throughput一般只要不是到达网卡瓶颈，基本上都不会有影响，但是需要注意，在目前多核心的机器上，如果网卡的中断能够均匀分布到多个核心上，能提高不少机器的性能；

系统的这个三个指标随着压力的增大，而没有提升，均是系统的某种资源达到瓶颈。下面介绍如何查看系统的稀缺资源

# 系统资源介绍

对于计算机系统，一般的瓶颈在于三个方面：CPU、内存、IO，IO又分磁盘IO和网卡IO，下面分别对这四个资源的度量进行介绍

## CPU

CPU在linux系统中主要完成三个任务：

1. 用户态进程的执行
2. 系统内核的执行
3. idle状态

###cpu的度量

CPU的上述三个状态进一步细分可以分为：

* user CPU用于执行用户态进程的比例
* nice CPU用于切换用户态进程的nice值的比例
* sys CPU用于执行内核态的的比例，不包括软中断和硬中断的时间
* iowait CPU用于IO操作的比例
* irq CPU用户硬中断的比例
* soft CPU用于软中断的比例
* steal hypervisor服务另一个虚拟处理器的时候，虚拟CPU等待实际CPU的时间的百分比[英文](http://rackerhacker.com/2008/11/04/what-is-steal-time-in-my-sysstat-output/).[^1]
* guest CPU执行虚拟CPU作业的时间
* idle CPU闲置的时间比例

满足关系：

	CPU总的工作时间 = total_cur = user + system + nice + idle + iowait + \
		irq + softirq

	total_pre = pre_user + pre_system + pre_nice + pre_idle + \
		pre_iowait + pre_irq + pre_softirq

	user = user_cur - user_pre

	total = total_cur - total_pre
	
	total = user + system + nice + idle + iowait + irq + softirq

###cpu时间的计算

####tick
Linux核心每隔固定周期会发出timer interrupt (IRQ 0)，这个时间就是tick，它是核心度量时间的基本单位[^2]。
tick是HZ的倒数，HZ的值可在编译核心时设定，可设定100、250、300或1000，查看本机的HZ的值可以通过如下命令来查看：

	cat /boot/config-`uname -r` | grep "^CONFIG_HZ"

####CPU时间
CPU的数据信息被记录在/proc/stat文件中，单位时tick，内核中有个变量，jiffies，来记录自系统启动以来经历的tick数量。内核在时钟中断程序中，即每个tick更新一次CPU信息。CPU的时间消耗，记录在/proc/stat文件中，CPU的时间消耗就是根据该文件的内容计算而来，文件内容大致如下:

	cpu  111262 29371 158809 4489873 24328 2420 1609 0 0
	cpu0 53992 14114 68808 2228130 20065 1123 876 0 0
	cpu1 57269 15256 90000 2261743 4263 1296 733 0 0
	intr 7848496 4165587 4555 0 0 0 0 0 0 1 29100 0 0 184931 0 0 0 2878 739856 0 82597 0 0 0 19 0 0 0 65126 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
	ctxt 15480484
	btime 1354326511
	processes 4721
	procs_running 1
	procs_blocked 0
	softirq 8334431 513266 2992461 1 27910 82629 0 3492 1637012 1203 3076457

CPU信息中，cpu代表CPU总的信息，cpu0,...,cpun为各个具体CPU的信息。

* cpu信息分别代表的意义：user, nice, sys, idle, iowait time, Hard irq time, Soft irq time, steal time, guest time
* intr 这行给出中断的信息，第一个为自系统启动以来，发生的所有的中断的次数；然后每个数对应一个特定的中断自系统启动以来所发生的次数。 </font></p>
* ctxt 给出了自系统启动以来CPU发生的上下文交换的次数
* btime 给出了从系统启动到现在为止的时间，单位为秒[^3].
* processes (total_forks) 自系统启动以来所创建的任务的个数目。
* procs_running 当前运行队列的任务的数目。
* procs_blocked 当前被阻塞的任务的数目。
* 后一个softirq意义不明确

CPU时间的计算，就是通过intval时间内tick数目的对比，得到CPU的统计信息

###进程在CPU中的状态
我们进行性能测试，除了需要关注上面的CPU Utilization信息，也需要查看进程在操作系统中的状态，进程在CPU中的状态迁移如下图：

![cpu-process](/assets/images/2012-11-24-cpu-proccess.jpg "进程的状态迁移图")

process有两种状态：

1. runnable
2. blocked waiting for an event to complete

一个blocked状态的process可能在等待一个I/O操作获取的数据，或者是一个系统调用的结果。

一个process在runnable状态，这就意味着它将同其他runnable状态的process等待CPU时间，而不是立即获得CPU时间，一个runnable状态的process不需要消耗CPU时间。所有process处在runnable状态且等待CPU时间时，形成的等待队列称作Run Queue。Run Queue越大，表示CPU的压力越大。

性能工具通常显示runnable processes的数目和blocked processes的数目。  
还有一个很常见的系统状态是load average，系统的load是指running和runnable process的总和。  
例如：如果有2个processes在running和有3个在等待运行(runnable)，那么系统的load为5。load average是指在指定时间内load的平均值。一般load average显示的三个数字的时间分别为1分钟，五分钟和十五分钟。一般loadavg的个数，不超过CPU核心的个数，代表CPU的资源不是瓶颈。

###cpu状态的查看和相关命令

####vmstat

vmstat只能查看所有CPU的平均信息，包括CPU，内存，虚拟内存，IO，SYS和CPU等，同时也可以查看cpu队列信息；。

	guowei@guowei-laptop:~$ vmstat -n 1
	procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----
	r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
	3  0      0 559184 136856 1028320    0    0    25    34  221  489 17  7 76  1
	0  0      0 560112 136856 1028680    0    0     0     0  686 2182 46 48  6  0
	3  0      0 559324 136864 1029244    0    0     0    52  841 3286 42 43 15  0
	2  0      0 558960 136864 1029708    0    0     0     0  891 2905 41 41 19  0
	1  0      0 558228 136864 1030052    0    0     0     0  678 1778 46 43 10  0
	1  0      0 556800 136864 1030632    0    0     0     0  664 1966 44 50  6  0

1. PROC(ESSES)
	* --r:如果在processes中运行的序列(process r)是连续的大于在系统中的CPU的个数,表示系统现在运行比较慢,有多数的进程等待CPU。如果r的输出数大于系统中可用CPU个数的4倍的话,则系统面临着CPU短缺的问题,或者是CPU的速率过低,系统中有多数的进程在等待CPU,造成系统中进程运行过慢.
	* --b:在internal时间段里，被资源阻塞的任务数（I/0，页面调度，等等.），通常情况下是接近0的 procs_blocked

2. SYSTEM
	* --in:每秒产生的中断次数
	* --cs:每秒产生的上下文切换次数
	上面2个值越大，会看到由内核消耗的CPU时间会越大
 
3. CPU
	* --us:用户进程消耗的CPU时间百分。us的值比较高时，说明用户进程消耗的CPU时间多，但是如果长期超50%的使用，那么我们就该考虑优化程序算法或者进行加速（比如PHP/PERL）
	* --sy:内核进程消耗的CPU时间百分比。（sy的值高时，说明系统内核消耗的CPU资源多，这并不是良性表现，我们应该检查原因）
	* --wa:IO等待消耗的CPU时间百分比。wa的值高时，说明IO等待比较严重，这可能由于磁盘大量作随机访问造成，也有可能磁盘出现瓶颈（块操作）。
	* --id:CPU处于空闲状态时间百分比。如果空闲时间(cpu id)持续为0并且系统时间(cpu sy)是用户时间的两倍(cpu us) 系统则面临着CPU资源的短缺.

4. 解决办法

* 分析程序的业务场景，看看使用的模型是否适合该种业务；
* 使用多种诊断工具，具体定位程序消耗了更多的何种资源；
* prof代码，查看代码的瓶颈；
* 考虑增加更多更好的CPU. 
更多的分析方法，参见[《性能调优攻略》][perf]

####mpstat：

mpstat 是Multiprocessor Statistics的缩写，是实时系统监控工具。其报告与CPU相关的一些统计信息，这些信息存放在/proc/stat文件中。在多CPUs系统里，其不但能查看所有CPU的平均状况信息，而且能够查看特定CPU的信息。下面只介绍 mpstat与CPU相关的参数，mpstat的语法如下：

	guowei@guowei-laptop:~$ mpstat -P ALL 2 2
	Linux 2.6.32-21-generic (guowei-laptop) 	12/01/2012 	_i686_	(2 CPU)

	08:46:16 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest   %idle
	08:46:18 PM  all   39.60    0.00   29.82    0.00    0.00   11.53    0.00    0.00   19.05
	08:46:18 PM    0   33.33    0.00   28.86    0.00    0.00   11.44    0.00    0.00   26.37
	08:46:18 PM    1   45.73    0.00   30.65    0.00    0.00   12.06    0.00    0.00   11.56

	08:46:18 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest   %idle
	08:46:20 PM  all   36.88    0.00   29.70    0.00    0.00    9.90    0.00    0.00   23.51
	08:46:20 PM    0   35.50    0.00   29.00    0.00    0.00   11.00    0.00    0.00   24.50
	08:46:20 PM    1   38.42    0.00   30.05    0.00    0.00    8.87    0.00    0.00   22.66

	Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest   %idle
	Average:     all   38.23    0.00   29.76    0.00    0.00   10.71    0.00    0.00   21.30
	Average:       0   34.41    0.00   28.93    0.00    0.00   11.22    0.00    0.00   25.44
	Average:       1   42.04    0.00   30.35    0.00    0.00   10.45    0.00    0.00   17.16

参数的含义如下：

* -P{\|ALL} 表示监控哪个CPU，cpu在[0,cpu个数-1]中取值
* internal 相邻的两次采样的间隔时间
* count 采样的次数，count只能和delay一起使用

当没有参数时，mpstat则显示系统启动以后所有信息的平均值。有interval时，第一行的信息自系统启动以来的平均信息。从第二行开始，输出为前一个interval时间段的平均信息。与CPU有关的输出的参数的意义如上在CPU的度量中的解释

####iostat: 
只能查看所有CPU的平均信息。

	guowei@guowei-laptop:~$ iostat -c 2 2
	Linux 2.6.32-21-generic (guowei-laptop) 	12/01/2012 	_i686_	(2 CPU)

	avg-cpu:  %user   %nice %system %iowait  %steal   %idle
	          26.79    0.22   22.35    0.42    0.00   50.21

	avg-cpu:  %user   %nice %system %iowait  %steal   %idle
	          42.89    0.00   47.88    0.00    0.00    9.23

注：这里的system包括了irq和softirq的时间	          

####sar： 
与mpstat 一样，不但能查看CPU的平均信息，还能查看指定CPU的信息。  
sar \[options\] \[-A\] \[-o file\] t \[n\]

在命令行中，n 和t 两个参数组合起来定义采样间隔和次数，t为采样间隔，是必须有的参数，n为采样次数，是可选的，默认值是1，-o file表示将命令结果以二进制格式存放在文件中，file 在此处不是关键字，是文件名。options 为命令行选项，sar命令的选项很多，下面只列出常用选项：

	-A：所有报告的总和。
	-u：CPU利用率
	-v：进程、I节点、文件和锁表状态。
	-d：硬盘使用报告。
	-r：内存和交换空间的使用统计。
	-g：串口I/O的情况。
	-b：缓冲区使用情况。
	-a：文件读写情况。
	-c：系统调用情况。
	-q：报告队列长度和系统平均负载
	-R：进程的活动情况。
	-y：终端设备活动情况。
	-w：系统交换活动。
	-x { pid | SELF | ALL }：报告指定进程ID的统计信息，SELF关键字是sar进程本身的统计，ALL关键字是所有系统进程的统计。

* 用sar进行CPU利用率的分析  

例如:

	guowei@guowei-laptop:home$ sar -u 1 5
	Linux 2.6.32-21-generic (guowei-laptop) 	12/01/2012 	_i686_	(2 CPU)

	10:56:49 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
	10:56:50 PM     all     39.80      0.00     50.75      0.00      0.00      9.45
	10:56:51 PM     all     43.22      0.00     46.23      0.50      0.00     10.05
	10:56:52 PM     all     47.50      0.00     40.00      0.00      0.00     12.50
	10:56:53 PM     all     40.30      0.00     46.77      0.00      0.00     12.94
	10:56:54 PM     all     38.31      0.00     43.28      0.00      0.00     18.41
	Average:        all     41.82      0.00     45.41      0.10      0.00     12.67
 
在显示内容包括：

* %user：CPU处在用户模式下的时间百分比。
* %nice：CPU处在带NICE值的用户模式下的时间百分比；
* %system：CPU处在系统模式下的时间百分比，包括irq和softirq的值；
* %iowait：CPU等待输入输出完成时间的百分比。
* %steal：管理程序维护另一个虚拟处理器时，虚拟CPU的等待时间百分比。
* %idle：CPU空闲时间百分比。

在所有的显示中，我们应主要注意%iowait和%idle，%iowait的值过高，表示硬盘存在I/O瓶颈，%idle值高，表示CPU较空闲，如果%idle值高但系统响应慢时，有可能是CPU等待分配内存，此时应加大内存容量。%idle值如果持续低于10，那么系统的CPU处理能力相对较低，表明系统中最需要解决的资源是CPU。
 
* 用sar进行运行进程队列长度分析：

例如：

	guowei@guowei-laptop:home$ sar -q 2 3
	Linux 2.6.32-21-generic (guowei-laptop) 	12/01/2012 	_i686_	(2 CPU)
	11:02:03 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15
	11:02:05 PM         1       280      2.51      2.28      2.24
	11:02:07 PM         2       280      2.51      2.28      2.24
	11:02:09 PM         3       280      2.63      2.31      2.25
	Average:            2       280      2.55      2.29      2.24

其中:  

* runq-sz 准备运行的进程运行队列。
* plist-sz  进程队列里的进程和线程的数量
* ldavg-1  前一分钟的系统平均负载(load average)
* ldavg-5  前五分钟的系统平均负载(load average)
* ldavg-15  前15分钟的系统平均负载(load average)
 
load average可以理解为每秒钟CPU等待运行的进程个数. 在Linux系统中，sar -q、uptime、top等命令都会有系统平均负载load average的输出, 系统平均负载被定义为在特定时间间隔内运行队列中的平均任务数。如果一个进程满足以下条件则其就会位于运行队列中：  

* 它没有在等待I/O操作的结果
* 它没有主动进入等待状态(也就是没有调用'wait')
* 没有被停止(例如：等待终止)

####top：
显示的信息同ps接近，但是top可以了解到CPU消耗，可以根据用户指定的时间来更新显示。

	Tasks: 172 total,   4 running, 166 sleeping,   1 stopped,   1 zombie
	Cpu0  : 38.6%us, 34.3%sy,  0.0%ni, 14.4%id,  0.0%wa,  0.0%hi, 12.7%si,  0.0%st
	Cpu1  : 40.3%us, 31.5%sy,  0.0%ni, 14.8%id,  0.0%wa,  0.0%hi, 13.4%si,  0.0%st
	Mem:   2050816k total,  1995024k used,    55792k free,     8468k buffers
	Swap:  2999992k total,      464k used,  2999528k free,  1690900k cached

	PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND              
	2635 guowei    20   0 13668  10m  964 R   74  0.5 131:26.01 curl-loader               
	2417 guowei    20   0  5364 3704  620 R   47  0.2  79:23.80 lighttpd     

在显示内容包括：

* Tasks：表示总共运行的task，running的个数，sleeping的个数，stop的个数，zobie的个数
* Cpun：第n个CPU的使用信息，sys包括softirq和irq
* Mem的状态：CPU处在系统模式下的时间百分比，包括irq和softirq的值；
* Swap的状态：CPU等待输入输出完成时间的百分比。
* 进程对应的信息：进程ID，所属的用户，优先级，Nice值，这些信息都涉及到过;
* IRT：virtual memory usage 
	1. 进程“需要的”虚拟内存大小，包括进程使用的库、代码、数据等 
	2. 假如进程申请100m的内存，但实际只使用了10m，那么它会增长100m，而不是实际的使用量 

* RES：resident memory usage 常驻内存 
	1. 进程当前使用的内存大小，但不包括swap out 
	2. 包含其他进程的共享 
	3. 如果申请100m的内存，实际使用10m，它只增长10m，与VIRT相反 
	4. 关于库占用内存的情况，它只统计加载的库文件所占内存大小 

* SHR：shared memory 
	1. 除了自身进程的共享内存，也包括其他进程的共享内存 
	2. 虽然进程只使用了几个共享库的函数，但它包含了整个共享库的大小 
	3. 计算某个进程所占的物理内存大小公式：RES – SHR 
	4. swap out后，它将会降下来 

* DATA 
    1. 数据占用的内存。如果top没有显示，按f键可以显示出来。 
    2. 真正的该程序要求的数据空间，是真正在运行中要使用的。

top命令下按f键可以看到详细说明

	* A: PID        = Process Id
	* E: USER       = User Name
	* H: PR         = Priority
	* I: NI         = Nice value
	* O: VIRT       = Virtual Image (kb)
	* Q: RES        = Resident size (kb)
	* T: SHR        = Shared Mem size (kb)
	* W: S          = Process Status
	* K: %CPU       = CPU usage
	* N: %MEM       = Memory usage (RES)
	* M: TIME+      = CPU Time, hundredths
	b: PPID       = Parent Process Pid
	c: RUSER      = Real user name
	d: UID        = User Id
	f: GROUP      = Group Name
	g: TTY        = Controlling Tty
	j: P          = Last used cpu (SMP)
	p: SWAP       = Swapped size (kb)
	l: TIME       = CPU Time
	r: CODE       = Code size (kb)
	s: DATA       = Data+Stack size (kb)
	u: nFLT       = Page Fault count
	v: nDRT       = Dirty Pages count
	y: WCHAN      = Sleeping in Function
	z: Flags      = Task Flags <sched.h>
	* X: COMMAND    = Command name/line

top命令下要查看某个用户启动的进程：先输入u，然后输入用户名，再回车

###cpu的调优
CPU的各种调优方法，具体可以参看[《性能调优攻略》][perf]及其所列的参考资料，这里给出几种可能的调优方法
####改多线程/进程模型为使用epool

####多核心cpu绑定特定进程到指定cpu
多核心的CPU系统有一个叫CPU亲和力的概念，能够将进指定程绑定到特定的CPU上。该方法能够优化性能的主要原因是由于CPU cache的优化。  
由于将进程绑定至特定的CPU上，避免了进程在各个CPU之间的调度切换，当进程在CPU间进行一次调度切换时，新CPU上的cache必须被flushed，如果进程在多个核心间来回切换，会导致多次cache的flushed，从而消耗更多CPU资源，这样导致每个的进程会花费更多的CPU时间。将进程绑定到特定的CPU上，能更好的局部化CPU的cache，同时提高cache命中率

* 将mysql的进程绑定到指定的CPU

	1. 如果程序还没有启动，使用taskset将其启动:
	taskset -cp 1,2,3 /etc/init.d/mysql start
	2. 如果程序已经启动，并且不宜重启, 获取到其pid
	taskset -cp 1,2,3 pid

* 将nginx的进程绑定到特定的CPU

	nginx 也可以使用上述方式，但nginx提供了更精确的控制。在conf/nginx.conf中，有如下一行：  

		worker_processes  1;  
	这是用来配置nginx启动几个工作进程的，默认为1。  
	nginx还支持一个名为worker_cpu_affinity的配置项，也就是说，nginx可以为每个工作进程绑定CPU。如下配置：  

		worker_processes  3;  
		worker_cpu_affinity 0010 0100 1000;  
	这里0010 0100 1000是掩码，分别代表第2、3、4颗cpu核心。
	
	重启nginx后，3个工作进程就可以各自用各自的CPU了。

* 自己编写程序，绑定到指定的CPU
	1. 如果自己写代码，要把进程绑定到CPU，该怎么做？可以用sched_setaffinity函数。在Linux上，这会触发一次系统调用。
	2. 如果父进程设置了affinity，之后其创建的子进程是否会有同样的属性？我发现子进程确实继承了父进程的affinity属性。

####将单核心运行的程序，修改为多核心运行

##内存
内存完成的任务：

1. 进程的执行空间
2. 进程内分配的空间
3. 缓存
4. 空闲

###内存状态的查看和相关命令

### malloc和延时分配

###虚拟内存

##磁盘IO

###磁盘状态的查看和相关命令

###磁盘，内存和CPU之间的影响

##网卡IO
###网卡的基本状态

###网卡的状态查看和相关工具

###网卡的极限性能


#常用的性能测试工具

##HTTP：

###ab
###siege
###curl-loader

##XMPP
###TSung

#如何分析定位分析系统的瓶颈
1. 首先查看cpu，因为其最容易成为瓶颈

2. 查看网卡

3. 查看磁盘和内存

#典型的瓶颈demo


[^1]: <small>steal 值比较高的话，你需要向主机供应商申请扩容虚拟机。服务器上的另一个虚拟机可能拥有更大更多的 CPU 时间片，你可能需要申请升级以与之竞争。另外，高 steal 值可能意味着主机供应商在服务器上过量地出售虚拟机。如果升级了虚拟机，steal值还是不降的话，你应该寻找另一家服务供应商。低steal值意味着你的应用程序在目前的虚拟机上运作良好。因为你的虚拟机不会经常地为了CPU时间与其它虚拟机激烈竞争，你的虚拟机会更快地响应。这一点也暗示了，你的主机供应商没有过量地出售虚拟服务，绝对是一件好事情。</small>
[^2]: <small>你可能发现，如果tick是linux操作系统度量时间的基本单位，那我们常用的timeval变量中的tv_usec这个变量准确么？因为如果tick=250，则最小的时间间隔为4ms，实际上这是两个概念，tick时操作系统进行系统调度等的时间间隔，而我们一般获取操作系统的时间，是从cmos电路中获取的时间，一般是从某一历史时刻开始到现在的时间，也就是为了取得我们操作系统上显示的日期。这个就是所谓的“实时时钟”，它的精确度是微秒。另外，系统还有一个与时间有关的时钟：实时时钟（RTC），这是一个硬件时钟，用来持久存放系统时间，系统关闭后靠主板上的微型电池保持计时。系统启动时，内核通过读取RTC来初始化Wall Time,并存放在xtime变量中，这是RTC最主要的作用。一般是在BIOS中，这也是我们的机器在关机重启后，仍然能够获取正确的取得时间的原因</small>
[^3]: <small>从网上找的含义是这个，但是实际上，我查看的意义为该机器上次启动的时间戳。</small>

[perf]: http://coolshell.cn/articles/7490.html
