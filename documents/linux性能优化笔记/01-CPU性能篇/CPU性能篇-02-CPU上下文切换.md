# CPU上下文切换

## 原理解释

抛出问题： 进程在竞争 CPU 的时候并没有真正运行，为什么还会导致系统的负载升高呢？

Linux 是一个多任务操作系统，它支持远大于 CPU 数量的任务同时运行。
当然，这些任务实际上并不是真的在同时运行，而是因为系统在很短的时间内，将 CPU 轮流分配给它们，造成多任务同时运行的错觉。

### 什么是寄存器和计数器？

CPU 在每个任务运行前，CPU 都需要知道任务从哪里加载、又从哪里开始运行，也就是说，需要系统事先帮它设置好 CPU 寄存器和程序计数器（Program Counter，PC）。

CPU 寄存器： 是 CPU 内置的容量小、但速度极快的内存。

而程序计数器： 则是用来存储 CPU 正在执行的指令位置、或者即将执行的下一条指令位置。

它们都是 CPU 在运行任何任务前，必须的依赖环境，因此也被叫做 CPU 上下文。

![](img/cpu_switch.jpg)


> 什么是上下文切换？

CPU 上下文切换，就是先把前一个任务的 CPU 上下文（也就是 CPU 寄存器和程序计数器）保存起来，然后加载新任务的上下文到这些寄存器和程序计数器，最后再跳转到程序计数器所指的新位置，运行新任务。
而这些保存下来的上下文，会`存储在系统内核中`，并在任务重新调度执行时再次加载进来。这样就能保证任务原来的状态不受影响，让任务看起来还是连续运行。

### 上下文切换的类型

#### 进程上下文切换

Linux 按照特权等级，把进程的运行空间分为内核空间和用户空间，分别对应着下图中， CPU 特权等级的 Ring 0 和 Ring 3。

内核空间（Ring 0）：具有最高权限，可以直接访问所有资源；

用户空间（Ring 3）：只能访问受限资源，不能直接访问内存等硬件设备，必须通过系统调用陷入到内核中，才能访问这些特权资源。

![](img/namespace.jpg)

进程在用户空间运行时，被称为进程的**用户态**，而陷入内核空间的时候，被称为进程的**内核态**。

从用户态到内核态的转变，需要通过系统调用来完成。

**例子**：当我们查看文件内容时，就需要多次系统调用来完成：首先调用 open() 打开文件，然后调用 read() 读取文件内容，并调用 write() 将内容写到标准输出，最后再调用 close() 关闭文件。

系统调用时，将发生2次cpu的上下文切换。（用户态切换到内核态，内核态再切换到用户态。）系统调用不涉及进程用户态的资源变动。

> 进程上下文切换和系统调用的区别

进程的上下文切换就比系统调用时多了一步：在保存当前进程的内核状态和 CPU 寄存器之前，需要先把该进程的**虚拟内存、内核栈**等保存下来；而加载了下一进程的内核态后，还需要刷新进程的虚拟内存和用户栈。

特别注意的是进程的上下文切换时需要耗费cpu时间来更新**cpu寄存器、虚拟内存、内核栈**等资源的保存恢复上，因此也会导致系统平均负载的上升。

另外虚拟内存的更新，也还导致TLB需要更新，导致数据访问变慢。

> 什么情况下会发生进程上下文切换的情况？

进程切换时才需要切换上下文，换句话说，`只有在进程调度`的时候，才需要切换上下文。
Linux 为每个 CPU 都维护了一个就绪队列，将活跃进程（即正在运行和正在等待 CPU 的进程）按照优先级和等待 CPU 的时间排序，然后选择最需要 CPU 的进程，也就是优先级最高和等待 CPU 时间最长的进程来运行。

**场景**

1. 进程执行完了，可以执行下一个进程了。从就绪队列里面获取新的进程运行。
2. CPU的时间片耗尽，系统自动挂起切换其他进程。
3. 系统资源不足，导致的挂起。
4. sleep类型的函数，主动挂起。
5. 有更高的优先级别的进程运行。
6. 发生硬件中断时。

#### 线程上下文切换

线程和进程的区别：线程是调度的基本单位，进程则是资源拥有的基本单位。

说人话，所谓内核中的任务调度，实际上的调度对象是线程；而进程只是给线程提供了虚拟内存、全局变量等资源。所以，对于线程和进程，我们可以这么理解：

- 当进程只有一个线程时，可以认为进程就等于线程。
- 当进程拥有多个线程时，这些线程会共享相同的虚拟内存和全局变量等资源。这些资源在上下文切换是不需要修改的。
- 另外，线程也有自己的私有数据，比如栈和寄存器等，这些在上下文切换时也是需要保存的。同进程的上下文切换是一样的。

因此线程比进程上下文切换带来的优势是，当同一个进程里面的线程需要进行上下文切换时是比多进程进行上下文切换少了一个步骤，少消耗资源，有性能上的优势。

#### 中断上下文切换

为了快速响应硬件的事件，中断处理会打断进程的正常调度和执行，转而调用中断处理程序，响应设备事件。

中断上下文切换`并不涉及到进程的用户态`，只包括内核态中断服务程序执行所必需的状态，包括 CPU 寄存器、内核堆栈、硬件中断参数等。

中断处理比进程拥有更高的优先级，所以中断上下文切换并不会与进程上下文切换同时发生。`由于中断会打断正常进程的调度和执行，所以大部分中断处理程序都短小精悍，以便尽可能快的执行结束。`

跟进程上下文切换一样，中断上下文切换也需要消耗 CPU，切换次数过多也会耗费大量的 CPU，甚至严重降低系统的整体性能。

## 如何分析上下文切换

### 怎么查看上下文切换

使用 vmstat 查看 cpu 的上下文切换情况。

```
root@test01:~# vmstat  1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 2711100 167204 776656    0    0     1   102    3   10  0  1 99  0  0

```

cpu 列的参数解释：

- cs（context switch）是每秒上下文切换的次数。
- in（interrupt）则是每秒中断的次数。
- r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待 CPU 的进程数。
- b（Blocked）则是处于不可中断睡眠状态的进程数。


查看每个进程的详细上下文切换信息：

```
root@test01:~# pidstat -w 1 1
Linux 4.15.0-143-generic (test01) 	06/29/2021 	_x86_64_	(2 CPU)

01:39:32 PM   UID       PID   cswch/s nvcswch/s  Command
01:39:33 PM     0         8     37.00      0.00  rcu_sched
01:39:33 PM     0       978     10.00      0.00  AliYunDun
01:39:33 PM     0     10298      1.00      0.00  kworker/0:0
01:39:33 PM     0     11830      1.00      0.00  kworker/1:0
01:39:33 PM     0     29597      7.00      0.00  kworker/u4:0
01:39:33 PM     0     29602      8.00      0.00  kworker/u4:1
01:39:33 PM     0     29798      1.00      0.00  pidstat

```

> cswch : 表示每秒自愿上下文切换（voluntary context switches）的次数

所谓自愿上下文切换，是指进程无法获取所需资源，导致的上下文切换。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。

> nvcswch : 表示每秒非自愿上下文切换（non voluntary context switches）的次数

而非自愿上下文切换，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。


### 案例分析


使用到的工具：
- sysbench
- pidstat
- vmstat
- /proc 虚拟文件系统


sysbench 模拟压测
```
root@test01:~# sysbench --threads=10 --max-time=300 threads run

```


vmstat 查看上下文的切换次数
```
root@test01:~# vmstat  1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 5  0      0 2704372 167352 778140    0    0     1   102    5   16  0  1 99  0  0
 9  0      0 2704396 167352 778140    0    0     0     0 16311 1619500 18 82  1  0  0 
 7  0      0 2704396 167352 778140    0    0     0     0 15573 1561679 18 79  3  0  0


in 中断次数 和 cs 切换次数一样的暴涨很多, 还有r cpu的就绪队列里有任务等待被调度执行。

```

查看进程的上下文切换情况
```
root@test01:~# pidstat -w  -u 1  

Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0       978    0.33    0.00    0.00    0.00    0.33     -  AliYunDun
Average:        0     29881    0.00    0.33    0.00    0.00    0.33     -  kworker/u4:0
Average:        0     29888   40.86  100.00    0.00    0.00  100.00     -  sysbench <- 异常进程
Average:        0     29901    0.00    0.33    0.00    1.00    0.33     -  pidstat

Average:      UID       PID   cswch/s nvcswch/s  Command
Average:        0         8     31.56      0.00  rcu_sched
Average:        0        11      0.33      0.00  watchdog/0
Average:        0        14      0.33      0.00  watchdog/1
Average:        0        16      1.00      0.00  ksoftirqd/1
Average:        0        30      0.33      0.00  khugepaged
Average:        0       222      0.33      0.00  kworker/1:1H
Average:        0       978      9.97      0.00  AliYunDun
Average:        0     10298      1.00      0.00  kworker/0:0
Average:        0     11830      1.00      0.00  kworker/1:0
Average:        0     29802     84.39      0.00  kworker/u4:2
Average:        0     29817     70.76      1.99  sshd
Average:        0     29881     80.73      0.00  kworker/u4:0
Average:        0     29901      1.00    151.50  pidstat
.....
```

查看线程的上下文切换情况
```
root@test01:~# pidstat -wt   1  

Average:      UID      TGID       TID   cswch/s nvcswch/s  Command
.....
Average:        0         -     29910  20224.51 132958.82  |__sysbench
Average:        0         -     29911  19161.76 150462.75  |__sysbench
Average:        0         -     29912  23149.02 137368.63  |__sysbench
Average:        0         -     29913  25295.10 136438.24  |__sysbench
Average:        0         -     29914  24739.22 136786.27  |__sysbench
Average:        0         -     29915  30319.61 136362.75  |__sysbench
Average:        0         -     29916  32198.04 127918.63  |__sysbench
Average:        0         -     29917  24967.65 147376.47  |__sysbench
Average:        0         -     29918  24191.18 136058.82  |__sysbench
Average:        0         -     29919  37285.29 103967.65  |__sysbench
.....

```

此时可以确定是由sysbench导致的大量的上下文切换。

-----

**开始排查中断次数上升的原因**

/proc 实际上是 Linux 的一个虚拟文件系统，用于内核空间与用户空间之间的通信。
可以从 /proc/interrupts 这个只读文件中读取中断的信息

从 interrupts 查看详细的中断信息，watch -d 参数可以显示变化幅度比较大的信息
```
root@test01:~# watch -d cat /proc/interrupts 
           CPU0       CPU1       
  1:          9          0   IO-APIC   1-edge      i8042
  4:        773          0   IO-APIC   4-edge      ttyS0
  6:          3          0   IO-APIC   6-edge      floppy
  8:          0          0   IO-APIC   8-edge      rtc0
  9:          0          0   IO-APIC   9-fasteoi   acpi
 10:          0          0   IO-APIC  10-fasteoi   virtio3
 11:         22          0   IO-APIC  11-fasteoi   uhci_hcd:usb1
 12:          0         15   IO-APIC  12-edge      i8042
 14:          0          0   IO-APIC  14-edge      ata_piix
 15:          0          0   IO-APIC  15-edge      ata_piix
 24:          0          0   PCI-MSI 49152-edge      virtio0-config
 25:          0         19   PCI-MSI 49153-edge      virtio0-virtqueues
 26:          0          1   PCI-MSI 81920-edge      virtio2-config
 27:     392904          0   PCI-MSI 81921-edge      virtio2-input.0
 28:          0     286819   PCI-MSI 81922-edge      virtio2-output.0
 29:          0          0   PCI-MSI 65536-edge      virtio1-config
 30:   14723308          0   PCI-MSI 65537-edge      virtio1-req.0
NMI:          0          0   Non-maskable interrupts
LOC:  266285308  294330260   Local timer interrupts
SPU:          0          0   Spurious interrupts
PMI:          0          0   Performance monitoring interrupts
IWI:          0          0   IRQ work interrupts
RTR:          0          0   APIC ICR read retries
RES:   16131822   28103864   Rescheduling interrupts  <------------------- here
CAL:      17333      17724   Function call interrupts
TLB:      14997      15274   TLB shootdowns
TRM:          0          0   Thermal event interrupts
THR:          0          0   Threshold APIC interrupts
DFR:          0          0   Deferred Error APIC interrupts
MCE:          0          0   Machine check exceptions
MCP:       1339       1339   Machine check polls
ERR:          0
MIS:          0
PIN:          0          0   Posted-interrupt notification event
NPI:          0          0   Nested posted-interrupt event
PIW:          0          0   Posted-interrupt wakeup event

```

观察一段时间，可以发现，变化速度最快的是重调度中断（RES），这个中断类型表示，唤醒空闲状态的 CPU 来调度新的任务运行。
这是多处理器系统（SMP）中，调度器用来分散任务到不同 CPU 的机制，通常也被称为处理器间中断（Inter-Processor Interrupts，IPI）。

> 上下文切换多少数值算正常呢？

这个数值其实取决于系统本身的 CPU 性能，如果系统的上下文切换次数比较稳定，那么从数百到一万以内，都应该算是正常的。
但当上下文切换次数超过一万次，或者切换次数出现数量级的增长时，就很可能已经出现了性能问题。

这时，还需要根据上下文切换的类型，再做具体分析。比方说：
- 自愿上下文切换变多了，说明进程都在等待资源，有可能发生了 I/O 等其他问题；
- 非自愿上下文切换变多了，说明进程都在被强制调度，也就是都在争抢 CPU，说明 CPU 的确成了瓶颈；
- 中断次数变多了，说明 CPU 被中断处理程序占用，还需要通过查看 /proc/interrupts 文件来分析具体的中断类型。










