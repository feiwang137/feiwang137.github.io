# 01-平均负载


## 平均负载

``` 
 15:46:17 up 3 days, 21:29,  1 user,  load average: 0.00, 0.00, 0.00
```

上面分别表示：系统开机时间，当前已登录系统用户，系统在1min，5min，15min的`平均负载`。

### 平均负载的解释

平均负载是指单位时间内，系统处于可运行状态和不可中断状态的平均进程数，也就是平均活跃进程数，它和 CPU 使用率并没有直接关系。

**所谓可运行状态的进程**：是指正在使用 CPU 或者正在等待 CPU 的进程，也就是我们常用 ps 命令看到的，处于 R 状态（Running 或 Runnable）的进程。

**不可中断状态的进程**：则是正处于内核态关键流程中的进程，并且这些流程是不可打断的，比如最常见的是等待硬件设备的 I/O 响应，也就是我们在 ps 命令中看到的 D 状态（Uninterruptible Sleep，也称为 Disk Sleep）的进程。

```language
当一个进程向磁盘中写入数据，此时这个进程就是处于不可中断的状态。（如果此时的进程被打断了，就容易出现磁盘数据与进程数据不一致的问题。）
```



> 举个例子：比如当平均负载为 2 时，意味着什么呢？

  - 在只有 2 个 CPU 的系统上，意味着所有的 CPU 都刚好被完全占用。
  - 在 4 个 CPU 的系统上，意味着 CPU 有 50% 的空闲。
  - 而在只有 1 个 CPU 的系统中，则意味着有一半的进程竞争不到 CPU。

平均负载最理想的情况是等于 CPU 个数

> CPU使用率解释

单位时间内 CPU 繁忙情况的统计。

举个例子：
  1. CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时这两者是一致的；
  2. `I/O 密集型进程，等待 I/O 也会导致平均负载升高，但 CPU 使用率不一定很高；`
  3. 大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU 使用率也会比较高。


## 平均负载案例分析

使用到的工具：
  1. stress :  压力测试工具
  2. mpstat : 一个常用的`多核 CPU 性能分析工具`，用来实时查看每个 CPU 的性能指标，以及所有 CPU 的平均指标。 
  3. pidstat : 是一个常用的`进程性能分析工具`，用来实时查看进程的 CPU、内存、I/O 以及上下文切换等性能指标。


### 场景1: CPU 密集型进程

通过使用stress模仿cpu密集型进程
```
# stress --cpu 1 --timeout 600
```

uptime 查看系统的负载情况
```
# watch -d uptime # -d 参数高亮显示变化的区域

 16:31:00 up 3 days, 22:14,  3 users,  load average: 1.01, 0.66, 0.30
```

mpstat 查看cpu的情况
```
# mpstat -P ALL 5

Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all   50.17    0.00    0.28    0.00    0.00    0.00    0.00    0.00    0.00   49.55
Average:       0    0.33    0.00    0.56    0.00    0.00    0.00    0.00    0.00    0.00   99.11
Average:       1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00 

```


pidstat 查看是哪个进程导致的
```
# pidstat -u 1 5 # -u 表示查看cpu选项的参数

Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0       628    0.00    0.20    0.00    0.00    0.20     -  aliyun-service
Average:        0       978    0.20    0.40    0.00    0.00    0.60     -  AliYunDun
Average:        0     12129    0.00    0.20    0.00    0.00    0.20     -  sshd
Average:        0     12236  100.00    0.00    0.00    0.00  100.00     -  stress  <-使用cpu最高的进程
Average:        0     12329    0.00    0.40    0.00    0.00    0.40     -  pidstat

```

### 场景2: I/O 密集型进程

模仿I/O压力

stress
```
# stress  -i 1 -d 1 --timeout 600

```

uptime
```
# watch -d uptime

17:07:16 up 3 days, 22:50,  5 users,  load average: 3.05, 2.02, 1.36    # 平均负载升高了
```

mpstat
```
# mpstat -P ALL 5

05:07:29 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
05:07:30 PM  all    0.00    0.00    1.51   48.74    0.00    0.00    0.00    0.00    0.00   49.75
05:07:30 PM    0    0.99    0.00    1.98   97.03    0.00    0.00    0.00    0.00    0.00    0.00   # cpu的使用率并没有升高，但是iowait升高了
05:07:30 PM    1    0.00    0.00    1.98    0.00    0.00    0.00    0.00    0.00    0.00   98.02  

```

pidstat
```
# pidstat -d 1  # -d 磁盘

Average:      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
Average:        0       192      0.00     18.64      0.00      10  jbd2/vda1-8
Average:        0     14320      0.00      0.00      0.00       1  stress
Average:        0     14321      0.00 151874.88  36102.50      24  stress  <- 有大量临时文件的读写操作
Average:        0     14325      3.99      0.00      0.00      98  kworker/u4:3

```


此时现象为cpu的平均负载升高，但是cpu使用率缺并不高。


### 场景3: 大量进程的场景

模仿大量进程的场景
```
# stress  -c 8 --timeout 600
# watch -d uptime

 17:38:54 up 3 days, 23:22,  5 users,  load average: 7.98, 5.56, 2.94

```

查看cpu的性能情况
```
# mpstat -P ALL 1

05:34:40 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
05:34:41 PM  all  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
05:34:41 PM    0  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
05:34:41 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
```

分析进程的资源使用情况

```
# pidstat -u 1 

05:35:03 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
05:35:04 PM     0       628    0.00    1.00    0.00    0.00    1.00     0  aliyun-service
05:35:04 PM     0       978    1.00    1.00    0.00    1.00    2.00     0  AliYunDun
05:35:04 PM     0     13411    0.00    1.00    0.00    0.00    1.00     1  top
05:35:04 PM     0     16921   24.00    0.00    0.00   76.00   24.00     0  stress
05:35:04 PM     0     16922   25.00    0.00    0.00   75.00   25.00     0  stress
05:35:04 PM     0     16923   25.00    0.00    0.00   76.00   25.00     1  stress
05:35:04 PM     0     16924   26.00    0.00    0.00   75.00   26.00     1  stress
05:35:04 PM     0     16925   25.00    0.00    0.00   76.00   25.00     0  stress
05:35:04 PM     0     16926   25.00    0.00    0.00   74.00   25.00     1  stress
05:35:04 PM     0     16927   24.00    0.00    0.00   76.00   24.00     0  stress
05:35:04 PM     0     16928   24.00    0.00    0.00   74.00   24.00     1  stress
```
