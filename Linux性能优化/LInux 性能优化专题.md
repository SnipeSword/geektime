# LInux 性能优化专题

[TOC]
## 开篇词
- 基础篇
- 案例篇
- 套路篇
- 答疑篇
## 01 如何学习 Linux 性能优化
### 现状
- 不愿学习基础知识，建立性能全局观
- 不懂得分析和抽丝剥茧
### 性能指标
#### 从应用负载的视角

- 高并发和响应快（吞吐和延时）
#### 从系统资源的视角
- 资源使用率
- 饱和度
### 性能问题的本质
系统资源已经达到瓶颈，但请求的处理却还不够快，无法支撑更多的请求。
### 性能分析要做的事
找出应用或系统的瓶颈，并设法去避免或者缓解它们，从而更高效地利用系统资源处理更多的请求。
### 性能优化核心步骤
- 选择指标评估应用程序和系统的性能
- 为应用程序和系统设置性能目标
- 进行性能基准测试
- 性能分析定位瓶颈
- 优化系统和应用程序
- 性能监控和告警
### 学习的重点

核心话题：建立整体系统性能的全局观
- 理解最基本的几个系统知识原理
- 掌握必要的性能工具
- 通过实际的场景演练，贯穿不同的组件
### 高效学习方法
- 虽然系统的原理很重要，但在刚开始一定不要试图抓住所有的实现细节
- 边学边实践，通过大量的案例演戏掌握 Linux 性能的分析和优化
- 勤思考，多反思，善总结，多问为什么
###  Linux性能工具图谱
![](http://www.brendangregg.com/Perf/linux_perf_tools_full.png)
### 学习思维导图
![](https://static001.geekbang.org/resource/image/0f/ba/0faf56cd9521e665f739b03dd04470ba.png)
## 02|原理篇：到底应该怎么理解“平均负载”
### 平均负载概念
平均负载是指单位时间内，处于可运行状态(R)和不可中断状态(D)的进程数。所以它不仅包括了正在使用CPU的进程，还包括等待CPU和等待IO的进程。
- CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时两者是一致的；
- I/O 密集型进程，等待IO 会导致平均负载升高，但 CPU 利用率不一定很高；
- 大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU 利用率也会比较高；
### 平均负载查看方法
```
[root@performance ~]# uptime
 17:47:08 up 22 min,  2 users,  load average: 0.00, 0.01, 0.05
```
### 平均负载合理值
- 一般看来在平均负载高于 CPU 数量的 70% 时，就应该分析排查负载高的问题了。一旦负载高，就可能导致进程响应变慢。
- 根据历史数据，分析负载变化趋势，若有明显升高趋势，则需要排查。

### 分析平均负载高问题的常用工具

- 系统压力测试工具：stress
```
安装 Extra Packages for Enterprise Linux (EPEL)
# yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
安装 stress
# yum install stress -y
```
- 系统性能监控分析工具：sysstat
  - mpstat：多核CPU性能分析工具。实时查看所有CPU平均性能指标，和每个CPU性能指标；
  - pidstat：进程性能分析工具。实时查看进程的 CPU、内存、IO、上下文切换等性能指标；[pidstat 常见用法](https://my.iblogs.top/2019/05/20/pidstat%E5%B8%B8%E8%A7%81%E7%94%A8%E6%B3%95/#more)
```
安装 Extra Packages for Enterprise Linux (EPEL)
# yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
安装 sysstat
# yum install sysstat -y
```
> 补充：epel 中的 sysstat 最新版本是 10.1.5，pidstat输出中没有%wait，建议升级到 11.5.5 以上版本。[sysstat 官网](http://sebastien.godard.pagesperso-orange.fr/download.html)

- 其他系统性能监控分析工具：htop   atop

### 模拟案例分析

#### 场景一：CPU 密集型进程

- stress 加压（终端1）
```
# stress -c 1 -t 600   
```
- 动态观察 uptime 中平均负载值变化（终端2）
```
# watch -n 1 -d 'uptime'
```
> 发现 1 min负载在快速上升，5,15 min负载在缓慢上升。向1 逼近。

- 运行 mpstat 观察 CPU 的利用率、IOwait 情况（终端3）
```
# mpstat -P ALL 3  #-P ALL 表示监控所有 CPU，后面数字 3 表示间隔 3s 输出一组数据
Linux 3.10.0-862.el7.x86_64 (performance) 	05/18/2019 	_x86_64_	(2 CPU)

06:42:08 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:42:11 PM  all   50.00    0.00    0.33    0.00    0.00    0.00    0.00    0.00    0.00   49.67
06:42:11 PM    0   52.00    0.00    0.33    0.00    0.00    0.00    0.00    0.00    0.00   47.67
06:42:11 PM    1   48.00    0.00    0.33    0.00    0.00    0.00    0.00    0.00    0.00   51.67

06:42:11 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:42:14 PM  all   50.08    0.00    0.17    0.00    0.00    0.00    0.17    0.00    0.00   49.58
06:42:14 PM    0   99.67    0.00    0.00    0.00    0.00    0.00    0.33    0.00    0.00    0.00
06:42:14 PM    1    0.33    0.00    0.33    0.00    0.00    0.00    0.00    0.00    0.00   99.33

06:42:14 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:42:17 PM  all   50.08    0.00    0.33    0.00    0.00    0.00    0.00    0.00    0.00   49.58
06:42:17 PM    0   50.33    0.00    0.33    0.00    0.00    0.00    0.00    0.00    0.00   49.33
06:42:17 PM    1   49.83    0.00    0.33    0.00    0.00    0.00    0.00    0.00    0.00   49.83

06:42:17 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:42:20 PM  all   49.92    0.00    0.50    0.00    0.00    0.00    0.17    0.00    0.00   49.42
06:42:20 PM    0    0.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00   99.00
06:42:20 PM    1   99.67    0.00    0.00    0.00    0.00    0.00    0.33    0.00    0.00    0.00

06:42:20 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:42:23 PM  all   50.08    0.00    0.33    0.00    0.00    0.00    0.00    0.00    0.00   49.58
06:42:23 PM    0   14.33    0.00    0.67    0.00    0.00    0.00    0.00    0.00    0.00   85.00
06:42:23 PM    1   85.95    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   14.05

06:42:23 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:42:26 PM  all   50.08    0.00    0.33    0.17    0.00    0.00    0.00    0.00    0.00   49.42
06:42:26 PM    0  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
06:42:26 PM    1    0.33    0.00    0.66    0.33    0.00    0.00    0.00    0.00    0.00   98.67
```
> 发现 CPU 总的利用率 50% 左右，并无 io 等待。
- 运行 pidstat 找出导致 CPU 使用率高的进程（终端4）
```
# pidstat -u 3 1    # 间隔 3s 输出一组数据
Linux 3.10.0-862.el7.x86_64 (performance) 	05/18/2019 	_x86_64_	(2 CPU)

06:47:29 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
06:47:32 PM     0     19112    0.33    0.00    0.00    0.00    0.33     1  watch
06:47:32 PM     0     20642  100.00    0.00    0.00    0.33  100.00     1  stress

Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0     19112    0.33    0.00    0.00    0.00    0.33     -  watch
Average:        0     20642  100.00    0.00    0.00    0.33  100.00     -  stress
```
> 发现是 stress 进程导致的 CPU 利用率较高

总结：从终端2中可以看到，1min的平均负载会慢慢增加到1.00；从终端3和终端4可以看到，正是 stress 进程导致一颗 CPU 利用率 100%，但是并无 IO 等待。这说明，本案例中的平均负载升高是由于 CPU 利用率增高，跟 IO 等待无关。
#### 场景二：IO 密集型进程
- stress 模拟加压方式不同，分析思路同场景一
```
# stress -i 1 -t 600
```
- mpstat 
```
# mpstat -P ALL 3
06:56:55 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:56:58 PM  all    0.17    0.00   47.83    2.34    0.00    0.00    0.17    0.00    0.00   49.50
06:56:58 PM    0    0.00    0.00   94.98    4.68    0.00    0.00    0.33    0.00    0.00    0.00
06:56:58 PM    1    0.33    0.00    0.67    0.00    0.00    0.00    0.00    0.00    0.00   99.00

06:56:58 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:57:01 PM  all    0.34    0.00   47.99    2.18    0.00    0.00    0.00    0.00    0.00   49.50
06:57:01 PM    0    0.33    0.00   95.32    4.35    0.00    0.00    0.00    0.00    0.00    0.00
06:57:01 PM    1    0.34    0.00    0.34    0.00    0.00    0.00    0.00    0.00    0.00   99.33
```
- pidstat
```
# pidstat -u 3 1
Linux 3.10.0-862.el7.x86_64 (performance) 	05/18/2019 	_x86_64_	(2 CPU)

06:58:29 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
06:58:32 PM     0       237    0.00    1.00    0.00    0.00    1.00     1  kworker/1:1H
06:58:32 PM     0       258    0.00    0.33    0.00    0.00    0.33     0  kworker/0:1H
06:58:32 PM     0     19112    0.00    0.33    0.00    0.00    0.33     1  watch
06:58:32 PM     0     22385    0.00   94.67    0.00    1.33   94.67     1  stress
06:58:32 PM     0     22624    0.00    0.33    0.00    0.00    0.33     0  pidstat

Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0       237    0.00    1.00    0.00    0.00    1.00     -  kworker/1:1H
Average:        0       258    0.00    0.33    0.00    0.00    0.33     -  kworker/0:1H
Average:        0     19112    0.00    0.33    0.00    0.00    0.33     -  watch
Average:        0     22385    0.00   94.67    0.00    1.33   94.67     -  stress
Average:        0     22624    0.00    0.33    0.00    0.00    0.33     
```

总结：从上可以看出，本案例中的平均负载升高是由于 IO 等待导致的，iowait 较高，而等待硬件设备的IO响应是内核态操作，所以对应进程的 %system 使用率也偏高。

#### 场景三：大量进程场景
- stress 模拟加压方式不同，分析思路同场景一
```
# stress -c 8 -t 600
```
- mpstat
```
# mpstat -P ALL 3
Linux 3.10.0-862.el7.x86_64 (performance) 	05/18/2019 	_x86_64_	(2 CPU)

07:04:15 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
07:04:18 PM  all  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
07:04:18 PM    0  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
07:04:18 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

07:04:18 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
07:04:21 PM  all   99.50    0.00    0.33    0.00    0.00    0.00    0.17    0.00    0.00    0.00
07:04:21 PM    0   99.34    0.00    0.33    0.00    0.00    0.00    0.33    0.00    0.00    0.00
07:04:21 PM    1   99.67    0.00    0.33    0.00    0.00    0.00    0.00    0.00    0.00    0.00

07:04:21 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
07:04:24 PM  all   99.83    0.00    0.17    0.00    0.00    0.00    0.00    0.00    0.00    0.00
07:04:24 PM    0   99.67    0.00    0.33    0.00    0.00    0.00    0.00    0.00    0.00    0.00
07:04:24 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
^C
Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all   99.78    0.00    0.17    0.00    0.00    0.00    0.06    0.00    0.00    0.00
Average:       0   99.67    0.00    0.22    0.00    0.00    0.00    0.11    0.00    0.00    0.00
Average:       1   99.89    0.00    0.11    0.00    0.00    0.00    0.00    0.00    0.00    0.00
```
- pidstat
```
# pidstat -u 3 1
Linux 3.10.0-862.el7.x86_64 (performance) 	05/18/2019 	_x86_64_	(2 CPU)

07:04:32 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
07:04:35 PM     0     19112    0.00    0.33    0.00    1.32    0.33     1  watch
07:04:35 PM     0     23316   25.17    0.00    0.00   75.50   25.17     1  stress
07:04:35 PM     0     23317   24.50    0.00    0.00   74.50   24.50     0  stress
07:04:35 PM     0     23318   25.17    0.00    0.00   75.50   25.17     0  stress
07:04:35 PM     0     23319   25.17    0.00    0.00   75.83   25.17     0  stress
07:04:35 PM     0     23320   24.83    0.00    0.00   74.17   24.83     1  stress
07:04:35 PM     0     23321   24.83    0.00    0.00   74.50   24.83     1  stress
07:04:35 PM     0     23322   24.83    0.00    0.00   74.83   24.83     0  stress
07:04:35 PM     0     23323   25.17    0.00    0.00   75.83   25.17     1  stress

Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0     19112    0.00    0.33    0.00    1.32    0.33     -  watch
Average:        0     23316   25.17    0.00    0.00   75.50   25.17     -  stress
Average:        0     23317   24.50    0.00    0.00   74.50   24.50     -  stress
Average:        0     23318   25.17    0.00    0.00   75.50   25.17     -  stress
Average:        0     23319   25.17    0.00    0.00   75.83   25.17     -  stress
Average:        0     23320   24.83    0.00    0.00   74.17   24.83     -  stress
Average:        0     23321   24.83    0.00    0.00   74.50   24.83     -  stress
Average:        0     23322   24.83    0.00    0.00   74.83   24.83     -  stress
Average:        0     23323   25.17    0.00    0.00   75.83   25.17     -  stress
```
总结：可以看出 8 个进程在争抢 2 个 CPU，平均负载直逼 8。每个进程的 CPU 等待时间（%wait）高达75%。这些超出 CPU计算能力的进程最终将导致 CPU 过载。

