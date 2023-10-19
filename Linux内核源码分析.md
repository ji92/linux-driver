# Linux内核源码分析

# 内核编译

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231016210615.png)

# 体系架构
## SMP和NUMA架构

## CPU管理
+ bitmap方式管理CPU
> cat /proc/cpuinfo查询cpu信息

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
+ 闲时调度：内核启动时创建，没有其他进程时运行

> 内核防止实时进程饿死普通进程。提供了一个配置参数，默认值是实时进程如果已经占用了95%的CPU时间，就会把剩余5%的CPU时间分给普通进程

### 进程优先级

+ 只有实时调度类和分时调度类会用到进程优先级
+ 进程的优先级由0-139的整数表示
  ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231017212539.png)

### 调度策略

+ SCHED_FIFO:实时进程调度策略，先进先出调度没有时间片，没有更高优先级的状态下，只有等待主动让出CPU
+ SCHED_RR：实时进程调度策略，时间片轮转，进程使用完时间片之后加入优先级对应运行队列当中的尾部，把CPU让给同等优先级的其它进程
+ SCHED_OTHER:
  + SCHED_NORMAL:普通进程调度策略，使task选择CFS调度器来调度运行；
  + SCHED_BATCH：普通进程调度策略，批量处理，使task选择CFS调度器来调度运行
  + SCHED_IDLE:普通进程调度策略，使task以最低优先级选择CFS调度器来调度运行
+ SCHED_DEADLINE:限期进程调度策略，使task选择Deadline调度器来调度运行

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

### 调度策略案例
#### 案例1

```c
#include <stdio.h>
#include <pthread.h>
#include <sched.h>
#include <assert.h>

static int GetThreadPolicyFunc(pthread_attr_t *pAttr)
{
    int iPlicy;
    int igp=pthread_attr_getschedpolicy(pAttr,&iPlicy);

    assert(igp==0);

    switch (iPlicy)
    {
    case SCHED_FIFO:
        printf("Policy is --> SCHED_FIFO.\n");
        break;

    case SCHED_RR:
        printf("Policy is --> SCHED_RR.\n");
        break;

    case SCHED_OTHER:
        printf("Policy is --> SCHED_OTHER.\n");
        break;
  
    default:
    printf("Policy is --> Unknown.\n");
        break;
    }

    return iPlicy;
}

static void PrintThreadPriorityFunc(pthread_attr_t *pAttr,int iPolicy)
{
    int iPriority=sched_get_priority_max(iPolicy); //对普通进程，优先级返回均为0
    assert(iPriority!=-1);
    printf("Max_priority is : %d\n",iPriority);

    iPriority=sched_get_priority_min(iPolicy); //对普通进程，优先级返回均为0
    assert(iPriority!=-1);
    printf("Min_priority is : %d\n",iPriority);
}

static int GetThreadPriorityFunc(pthread_attr_t *pAttr)
{
    struct sched_param sParam;
    int irs=pthread_attr_getschedparam(pAttr,&sParam);

    assert(irs==0);

    printf("Priority=%d\n",sParam.__sched_priority);

    return sParam.__sched_priority;
}

static void SetThreadPolicyFunc(pthread_attr_t *pAttr,int iPolicy)
{
    int irs=pthread_attr_setschedpolicy(pAttr,iPolicy);

    assert(irs==0);

    GetThreadPolicyFunc(pAttr);
}

int main(int argc,char *argv[])
{
    pthread_attr_t pAttr;
    struct sched_param sched;

    int irs=pthread_attr_init(&pAttr);
    assert(irs==0);

    int iPlicy=GetThreadPolicyFunc(&pAttr);

    printf("\nExport current Configuration of priority.\n");
    PrintThreadPriorityFunc(&pAttr,iPlicy);

    printf("\nExport SCHED_FIFO of prioirty.\n");
    PrintThreadPriorityFunc(&pAttr,SCHED_FIFO);

    printf("\nExport SCHED_RR of prioirty.\n");
    PrintThreadPriorityFunc(&pAttr,SCHED_RR);


    printf("\nExport priority of current thread.\n");
    int iPriority=GetThreadPriorityFunc(&pAttr);
    printf("Set thread policy.\n");

    printf("\nSet SCHED_FIFO policy.\n");
    SetThreadPolicyFunc(&pAttr,SCHED_FIFO);

    printf("\nSet SCHED_RR policy.\n");
    SetThreadPolicyFunc(&pAttr,SCHED_RR);

    printf("\nRestore current policy.\n");
    SetThreadPolicyFunc(&pAttr,iPlicy);

    irs=pthread_attr_destroy(&pAttr);
    assert(irs==0);

    return 0;
}

```

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019195125.png)

#### 案例2

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <pthread.h>

void TestThread1Func()
{
    sleep(1);
    int i,j;
    int iPolicy;
    struct sched_param sParam;

    pthread_getschedparam(pthread_self(),&iPolicy,&sParam);

    if(iPolicy==SCHED_OTHER)
        printf("SCHED_OTHER.\n");

    if(iPolicy==SCHED_FIFO)
        printf("SCHED_FIFO.\n");
  
    if(iPolicy==SCHED_RR)
        printf("SCHED_RR TEST001.\n");

    for(i=1;i<=5;i++)
    {
        for(j=1;j<=5000000;j++){}
        printf("Execute thread function 1.\n");
    }
    printf("Pthread 1 Exit.\n\n");
}

void TestThread2Func()
{
    sleep(2);
    int i,j;
    int iPolicy;
    struct sched_param sParam;

    pthread_getschedparam(pthread_self(),&iPolicy,&sParam);

    if(iPolicy==SCHED_OTHER)
        printf("SCHED_OTHER.\n");

    if(iPolicy==SCHED_FIFO)
        printf("SCHED_FIFO.\n");
  
    if(iPolicy==SCHED_RR)
        printf("SCHED_RR TEST002.\n");

    for(i=1;i<=6;i++)
    {
        for(j=1;j<=6000000;j++){}
        printf("Execute thread function 2.\n");
    }
    printf("Pthread 2 Exit.\n\n");
}

void TestThread3Func()
{
    sleep(3);
    int i,j;
    int iPolicy;
    struct sched_param sParam;

    pthread_getschedparam(pthread_self(),&iPolicy,&sParam);

    if(iPolicy==SCHED_OTHER)
        printf("SCHED_OTHER.\n");

    if(iPolicy==SCHED_FIFO)
        printf("SCHED_FIFO.\n");
  
    if(iPolicy==SCHED_RR)
        printf("SCHED_RR TEST003.\n");

    for(i=1;i<=7;i++)
    {
        for(j=1;j<=7000000;j++){}
        printf("Execute thread function 3.\n");
    }
    printf("Pthread 3 Exit.\n\n");
}

int main(int argc,char* argv[])
{
    int i=0;
    i=getuid();

    if(0==i)
        printf("The current user is root.\n\n");
    else
        printf("The current user is not root.\n\n");

    pthread_t ppid1,ppid2,ppid3; 
    struct sched_param sParam;
    pthread_attr_t pAttr1,pAttr2,pAttr3;

    pthread_attr_init(&pAttr1);
    pthread_attr_init(&pAttr2);
    pthread_attr_init(&pAttr3);

    sParam.sched_priority=31;
    pthread_attr_setschedpolicy(&pAttr2,SCHED_RR);
    pthread_attr_setschedparam(&pAttr2,&sParam);
    pthread_attr_setinheritsched(&pAttr2,PTHREAD_EXPLICIT_SCHED);

    sParam.sched_priority=11;
    pthread_attr_setschedpolicy(&pAttr1,SCHED_FIFO);
    pthread_attr_setschedparam(&pAttr1,&sParam);
    pthread_attr_setinheritsched(&pAttr1,PTHREAD_EXPLICIT_SCHED);

    pthread_create(&ppid3,&pAttr3,(void*)TestThread3Func,NULL);
    pthread_create(&ppid2,&pAttr2,(void*)TestThread2Func,NULL);
    pthread_create(&ppid1,&pAttr1,(void*)TestThread1Func,NULL);

    pthread_join(ppid3,NULL);
    pthread_join(ppid2,NULL);
    pthread_join(ppid1,NULL);

    pthread_attr_destroy(&pAttr3);
    pthread_attr_destroy(&pAttr2);
    pthread_attr_destroy(&pAttr1);

    return 0;
}
```

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019195151.png)

### 多核调度分析

## 进程间同步

### per-CPU

### RCU(Read-Copy-Update)机制
+ RCU 重要的应用场景是链表，有效地提高遍历读取数据的效率，读取链表成员数据时候通常只需要rcu_read_lock()，允许多个线程同时读取链表，并且允许一个线程同时修改链表
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019205015.png)
  
+ 优势： 
  + 读者没有任何同步开销

+ 劣势
  + 写者同步开销大，需要互斥操作。
> 接口函数：list_add_rcu()/list_del_rcu()/list_replace_rcu()

### 内存屏障
+ 由于编译器乱序和超标量乱序处理器的存在，使内存访问顺序不符合预期
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019205946.png)

+ 为了阻止编译器错误的重排指令，在禁止内核抢占和开启内核抢占的里面添加编译器优化屏障
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019210121.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019210237.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019210325.png)


## 内存管理
> cat /proc/meminfo 查询内存信息
```


```
### 虚拟内存
.init模块初始化结束可以释放
#### 堆管理

### 物理内存



# 参考资料

+ [零声教育](https://gitlab.0voice.com/linux)
+ [深入理解Linux进程调度](https://mp.weixin.qq.com/s/3rV6d04QjO9_8Nkq9SrWYg)
+ [小林codeing](https://xiaolincoding.com/)
+ [深入理解Linux线程同步](https://mp.weixin.qq.com/s?__biz=Mzg2OTc0ODAzMw==&mid=2247508887&idx=1&sn=a47f27306807f0fca638b6de4f762451&chksm=ce9abfb9f9ed36afb1c782b4346e8f01f24897df55e6ea85b234f68218b6acddf3ef2069c68c&scene=178&cur_album_id=2519398872503353344#rd)
