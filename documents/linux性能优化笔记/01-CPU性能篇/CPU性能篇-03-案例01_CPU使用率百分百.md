# CPU使用率百分之百该怎么排查


## CPU使用率

Linux 作为一个多任务操作系统，将CPU的时间划分为时间片，然后通过调度器分配给任务使用。

为了维护 CPU 时间，Linux 通过事先定义的节拍率（内核中表示为 HZ），触发时间中断，并使用全局变量 Jiffies 记录了开机以来的节拍数。每发生一次时间中断，Jiffies 的值就加 1。

```
root@test01:~# grep 'CONFIG_HZ=' /boot/config-$(uname -r)
CONFIG_HZ=250   # 比如在我的系统中，节拍率设置成了 250，也就是每秒钟触发 250 次时间中断。
```

因为节拍率 HZ 是内核选项，所以用户空间程序并不能直接访问。为了方便用户空间程序，内核还提供了一个用户空间节拍率 USER_HZ，它总是固定为 100，也就是 1/100 秒。

可通过 /proc/stat 系统的 CPU 和任务统计信息。

```
root@test01:~# cat /proc/stat | grep ^cpu
cpu  447995 142 517122 83243517 125144 0 308955 0 0 0
cpu0 196222 56 246107 41675042 67783 0 140466 0 0 0
cpu1 251773 86 271014 41568475 57360 0 168488 0 0 0

# 具体每列的信息，可通过man proc去查询, 搜索/proc\/stat
```

查询进程的CPU统计信息, 通过 /proc/[pid]/stat 查看


## CPU 使用率过高怎么办？

可查找占用CPU使用率高的函数的工具：
- GDB PS: 在调试程序过程中，会中断程序的运行。这在线上环境往往是不允许的。
- perf 推荐，它以性能事件采样为基础，不仅可以分析系统的各种事件和内核性能，还可以用来分析指定应用程序的性能问题。


perf top : 类似于 top，它能够实时显示占用 CPU 时钟最多的函数或者指令，因此可以用来查找热点函数。

perf record : 保存数据

perf report : 输出数据报告

## 案例1 找出CPU高的进程


检查流程：
1. top查看系统的资源使用概览
2. 使用perf分析占用cpu高的进程


启动测试程序
```
# docker run --name nginx -p 10000:80 -itd feisky/nginx
# docker run --name phpfpm -itd --network container:nginx feisky/php-fpm
```

在另外一台机器上用ab进行压测
```
# ab -c 10 -n 10000 http://10.0.0.232:10000/

```

top查看系统资源的概念
```
# top

top - 16:34:05 up 4 days, 22:17,  3 users,  load average: 1.51, 0.36, 0.12
Tasks: 103 total,   6 running,  60 sleeping,   0 stopped,   0 zombie
%Cpu(s): 99.3 us(用户态cpu使用率99了),  0.7 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3847416 total,  2536952 free,   231568 used,  1078896 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  3353008 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                       
  856 daemon    20   0  336696  13092   5420 R  40.2  0.3   0:07.54 php-fpm                                       
  858 daemon    20   0  336696  13100   5428 R  39.9  0.3   0:07.37 php-fpm                                       
  854 daemon    20   0  336696  15928   8252 R  38.9  0.4   0:07.32 php-fpm                                       
  857 daemon    20   0  336696  16052   8380 R  38.9  0.4   0:07.43 php-fpm                                       
  853 daemon    20   0  336696  16224   8544 R  38.5  0.4   0:07.54 php-fpm   
.......
```

使用perf 分析php-fpm进程
```

root@test01:~# perf top -g -p 856  # -g开启调用关系分析，-p指定进程号

Samples: 63K of event 'cpu-clock', Event count (approx.): 7210358168
  Children      Self  Shared Object       Symbol                                                                  ◆
-   95.94%     5.39%  php-fpm             [.] execute_ex                                                          ▒
   - 53.10% execute_ex                                                                                            ▒
      - 22.97% 0x8c4a7c                                                                                           ▒
           6.22% sqrt                                                                                             ▒
           1.50% 0x681b9d                                                                                         ▒
      + 14.25% 0x98dea3                                                                                           ▒
   + 5.39% 0x6cb6258d4c544155                                                                                     ▒
+   85.60%     0.00%  php-fpm             [.] php_execute_script                                                  ▒
.....................   

```

发现是调用sqrt函数导致，最后去分析代码，发现确实有这么一段调用sqrt的代码。

## 案例2 CPU使用率高，但不找到对应的应用


启动测试应用，并ab压测
```

$ docker run --name nginx -p 10000:80 -itd feisky/nginx:sp
$ docker run --name phpfpm -itd --network container:nginx feisky/php-fpm:sp
$ ab -c 100 -n 1000 http://10.0.0.232:10000/
```

top 查看资源情况
```
top - 17:38:30 up 4 days, 23:21,  4 users,  load average: 2.60, 0.69, 0.30
Tasks: 122 total,   7 running,  76 sleeping,   0 stopped,   0 zombie
%Cpu(s): 72.4 us, 24.1 sy,  0.0 ni,  2.8 id,  0.0 wa,  0.0 hi,  0.7 si,  0.0 st
KiB Mem :  3847416 total,  2515204 free,   244520 used,  1087692 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  3337732 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                       
 1653 daemon    20   0  336696  16296   8612 S   4.7  0.4   0:01.63 php-fpm                                       
 1649 daemon    20   0  336696  16292   8608 S   4.0  0.4   0:01.64 php-fpm                                       
 1648 daemon    20   0  336696  17404   9716 S   3.7  0.5   0:01.60 php-fpm                                       
 1652 daemon    20   0  336696  16296   8612 S   3.7  0.4   0:01.59 php-fpm                                       
 1654 daemon    20   0  336696  16296   8612 S   3.3  0.4   0:01.59 php-fpm                                       
 1521 systemd+  20   0   33932   4704   2452 S   2.7  0.1   0:01.03 nginx                                         
 4658 root      20   0   33236   6564   4784 S   2.0  0.2   0:00.72 ab                                            
 1374 root      20   0  113392   8004   6872 S   1.7  0.2   0:00.74 containerd-shim                               
  957 root      20   0 1059652 109264  51544 S   1.0  2.8   0:47.08 dockerd                                       
 1522 systemd+  20   0   33104   3876   2452 S   1.0  0.1   0:00.51 nginx                                         
    8 root      20   0       0      0      0 R   0.3  0.0   1:11.26 rcu_sched                                     
   16 root      20   0       0      0      0 S   0.3  0.0   0:53.43 ksoftirqd/1                                   
    1 root      20   0   77536   8500   6520 S   0.0  0.2   0:02.63 systemd                                       
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.04 kthreadd                                      
    4 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/0:0H                                  
    6 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 mm_percpu_
    ........

```


现象：cpu占用很高，但是缺没有发现占用cpu高的应用


使用perf 获取系统进程性能信息
```
# perf record -g
# perf  report

Samples: 193K of event 'cpu-clock', Event count (approx.): 48373000000
  Children      Self  Command          Shared Object             Symbol                                           ◆
+   67.83%     0.00%  stress           stress                    [.] 0x000000000000168d                           ▒ < -------------
+    6.80%     0.00%  php-fpm          [kernel.kallsyms]         [k] entry_SYSCALL_64_after_hwframe               ▒
+    6.79%     0.05%  php-fpm          [kernel.kallsyms]         [k] do_syscall_64                                ▒
+    5.44%     2.07%  stress           stress                    [.] 0x0000000000002f25                           ▒
+    5.15%     0.00%  php-fpm          [unknown]                 [.] 0x6cb6258d4c544155                           ▒
+    5.15%     0.00%  php-fpm          libc-2.24.so              [.] 0x00000000000202e1                           ▒
+    4.97%     0.00%  stress           [kernel.kallsyms]         [k] async_page_fault   
......


```

这里还是很明显的吧？有个叫stress的Commnad占用大量的cpu时钟，于是通过ps 进行查找分析
```
root@test01:~# ps uax|grep stress
daemon   25740  0.0  0.0   4292   744 pts/0    S+   17:50   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
daemon   25749  0.0  0.0   7284   924 pts/0    S+   17:50   0:00 /usr/local/bin/stress -t 1 -d 1
daemon   25750  0.0  0.0   8184  1372 pts/0    R+   17:50   0:00 /usr/local/bin/stress -t 1 -d 1
daemon   25751  0.0  0.0   4292   732 pts/0    S+   17:50   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
daemon   25752  0.0  0.0   4292   748 pts/0    S+   17:50   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
daemon   25753  0.0  0.0   7284   868 pts/0    S+   17:50   0:00 /usr/local/bin/stress -t 1 -d 1
.......
root     25764  0.0  0.0  14436  1076 pts/3    S+   17:50   0:00 grep --color=auto stress
root@test01:~# ps uax|grep stress
daemon   26150  0.0  0.0   4292   760 pts/0    S+   17:50   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
daemon   26152  0.0  0.0   7284   928 pts/0    S+   17:50   0:00 /usr/local/bin/stress -t 1 -d 1
daemon   26154  0.0  0.0   8188  1632 pts/0    R+   17:50   0:00 /usr/local/bin/stress -t 1 -d 1
daemon   26155  0.0  0.0   4292   712 pts/0    S+   17:50   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
daemon   26156  0.0  0.0   4292   732 pts/0    S+   17:50   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
daemon   26157  0.0  0.0   7284   880 pts/0    S+   17:50   0:00 /usr/local/bin/stress -t 1 -d 1
......
```

发现进程的pid在不停的变动, 查找他的父进程。
在这里，其实有2种情况：
- 进程不停的崩溃重启导致的
- 短时进程导致的


```
root@test01:~# pstree|grep stre
        |                 |         |-2*[php-fpm---sh---stress---stress]
        |                    `-pstree

```

发现了吧？是php-fpm的进程启动的。

后面经过分析应用的源码，发现确实是这么回事。


PS: 这里老师分享了一个专门分析短进程的工具 execsnoop https://github.com/brendangregg/perf-tools/blob/master/execsnoop
















