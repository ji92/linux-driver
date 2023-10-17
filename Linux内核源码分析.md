# Linux内核源码分析

# 内核编译

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231016210615.png)

# Linux源码结构

+ arch：不同平台体系结构的相关代码
+ block：设备驱动
+ certs：与认证和签名相关代码
+ crypto：内核常用压缩算法、常用加密算法等等源代码
+ documentation：描述模块功能和协议规范代码
+ drivers：驱动程序（USB 总线、PCI 总线、显示、网卡等等）
+ fs：虚拟文件系统 VFS）代码
+ include：内核源码依赖的绝大部分头文件
+ init：内核初始化代码，直接关联到内存各个组件入口
+ ipc:进程间通信实现，比如信号量、共享内存等等
+ kernel：内核核心代码，包括进程管理、IRQ等等
+ lib：C 标准库的子集
+ licenses：Linux 内核根据 Licenses/preferred/GPL-2.0中提供 GNU 通用公共许可证版本2
+ mm：内存管理的相关实现操作
+ net：网络协议代码，比如TCP、Wifi、IPv6 等
+ samples：内核实例代码
+ scripts：编译和配置内核所需要脚本
+ security：内核安全模型相关的代码
+ sound：声卡驱动源码
+ tools：与内核交互
+ usr：用户打包和压缩内核的实现的源码
+ virt：/kvm 虚拟化目录相关支持实现
  ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231016205223.png)

# 进程管理

## 基本概念

### 进程vs线程

+ 进程是程序的执行过程。
+ 线程是进程内的并发机制的实现

> 进程是操作系统分配和管理系统资源的基本单位
> 线程是程序执行的基本单位
> 同一个进程所有线程共享相同的资源

> 线程相比进程能减少开销体现在：
>
> + **线程的创建时间比进程快**，因为进程在创建的过程中，还需要资源管理信息，比如内存管理信息、文件管理信息，而线程在创建的过程中，不会涉及这些资源管理信息，而是共享它们；
> + **线程的终止时间比进程快**，因为线程释放的资源相比进程少很多；
> + **同一个进程内的线程切换比进程切换快**，因为线程具有相同的地址空间（虚拟内存共享），这意味着同一个进程的线程都具有同一个页表，那么在切换的时候不需要切换页表。而对于进程之间的切换，切换的时候要把页表给切换掉，而页表的切换过程开销是比较大的；
> + 由于同一进程的各线程间共享内存和文件资源，那么在线程之间数据传递的时候，就不需要经过内核了，这就使得线程之间的数据交互效率更高了；

+ 线程又可以分内核线程和用户线程
  + 内核线程没有用户虚拟地址空间
    ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231016212338.png)
  + 用户线程共享用户虚拟地址空间

## 进程的生命周期

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231016213531.png)

| 进程状态 | 定义                                    |
| -------- | --------------------------------------- |
| 新建     | TASK_NEW                                |
| 就绪     | TASK_RUNNING                            |
| 执行     | TASK_RUNNING                            |
| 阻塞     | TASK_INTERRUPTIBLE/TASK_UNINTERRUPTIBLE |
| 死亡     | EXIT_ZOMBIE  / __TASK_STOPPED          |

## task_struct结构体

| 结构体      | 成员     | 内容                                                                   |
| ----------- | -------- | ---------------------------------------------------------------------- |
| task_struct | 线程相关 | 一般是内嵌数据结构，如进程调度相关（优先级、程序状态、调度统计信息等） |
|             | 进程相关 | 一般是指针，如内存空间、文件系统                                       |

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231016214141.png)

## 进程调度
### 调度类
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231017211125.png)

+ 禁令调度：内核启动时创建，执行特别紧急的任务。一个cpu一个进程，名字叫做[migration/CPU_ID]；调度均衡要迁移线程的时候会用到，所以它的名字叫做migration。
+ 限时调度：硬实时，按照配置的运行周期、运行时间和截止时间进行调度。
+ 实时调度：软实时，为每个优先级维护一个队列，实时优先级大的进程先运行；当两个实时进程优先级相同，SCHED_RR会交替调度，SCHED_FIFO则不会
+ 分时调度：适用于广大的普通进程，由CFS算法实现
  + SCHED_BATCH进程希望减少调度次数，每次调度能执行的时间长一点
  + SCHED_IDLE是优先级特别低的进程，其分到的CPU时间的比例非常低，但是也总是能保证分到
+ 闲时调度：内核启动时创建，没有其他进程时运行
> 内核防止实时进程饿死普通进程。提供了一个配置参数，默认值是实时进程如果已经占用了95%的CPU时间，就会把剩余5%的CPU时间分给普通进程
> 
### 进程优先级
+ 只有实时调度类和分时调度类会用到进程优先级
+ 进程的优先级由0-139的整数表示
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231017212539.png)

### 调度策略
+ SCHED_NORMAL:普通进程调度策略，使task 选择CFS调度器来调度运行；
+ SCHED_FIFO:实时进程调度策略，先进先出调度没有时间片，没有更高优先级的状态下，只有等待主动让出CPU
+ SCHED_RR：实时进程调度策略，时间片轮转，进程使用完时间片之后加入优先级对应运行队列当中的尾部，把CPU让给同等优先级的其它进程
+ SCHED_BATCH：普通进程调度策略，批量处理，使task选择CFS调度器来调度运行
+ SCHED_IDLE:普通进程调度策略，使task以最低优先级选择CFS调度器来调度运行
+ SCHED_DEADLINE:限期进程调度策略，使task选择Deadline 调度器来调度运行
> 其中 Stop调度器和IDLE-task调度器，仅使用于内核，用户没有办法进行选择。
### CFS（complete fair schedule）调度器
+ CFS调度器引入权重，使用权重代表进程的优先级nice值，各个进程按照权重比例分配CPU时间
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231017221317.png)

+ vruntime = 实际运行时间 * (NICE_0_LOAD/weight)
+ 在一个调度周期里面，所有进程的虚拟运行时间是相同的，所以在进程调度时，只需要找到虚拟运行时间最小的进程调度运行即可。
+ cfs_rq：跟踪就绪队列信息以及管理就绪态调度实体，并维护一个按照虚拟时间排序的红黑树。tasks_timeline->rb_root是红黑树的根，tasks_timeline->rb_leftmost指向红黑树中最左边的调度实体，即虚拟赶时间最小的调度实体。
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231017221954.png)

### 调度子系统各个组件模块
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231017214823.png)
+ 主调度器：通过调用schedule()函数来完成进程的选择和切换。
+ 周期性调度器：根据频率自动调用scheduler_tick 函数，作用根据进程运行时间触发调度
+ 上下文切换：主要做两个事情（切换地址空间、切换寄存器和栈空间）。

# 参考资料

+ [零声教育](https://gitlab.0voice.com/linux)
+ [深入理解Linux进程调度](https://mp.weixin.qq.com/s/3rV6d04QjO9_8Nkq9SrWYg)
+ [小林codeing](https://xiaolincoding.com/)
+ [深入理解Linux线程同步](https://mp.weixin.qq.com/s?__biz=Mzg2OTc0ODAzMw==&mid=2247508887&idx=1&sn=a47f27306807f0fca638b6de4f762451&chksm=ce9abfb9f9ed36afb1c782b4346e8f01f24897df55e6ea85b234f68218b6acddf3ef2069c68c&scene=178&cur_album_id=2519398872503353344#rd)