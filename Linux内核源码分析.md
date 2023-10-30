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

### 进程的亲缘关系
+ 0号进程在BSP中通过硬编码实现，负责系统初始化，初始化完成后退化成IDLE进程
+ 0号进程产生init(1号进程)和kthread(2号进程)
+ 所有的进程都是通过fork创建，2号进程是第一个内核线程，也是所有内核线程的父进程。由于所有内核线程共享同一个内核空间，可以看做一个进程下的多个线程。
> 会话组（session group）：一个用户运行的所有程序构成一个会话组，一个会话组的所有进程都是其会话组组长的子孙进程
> 进程组（process group）：一行命令的执行作为一个进程组，一个进程组的所有进程都是其进程组组长的子孙进程
> ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231024202550.png)

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
> **直接调度：直接按线程维度进行调度**
> 间接调度：先调度进程，再在进程内部调度线程
> 评价指标：响应性、吞吐量、公平性、适应性、节能性 

### 调度类

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231017211125.png)

+ 禁令调度：内核启动时创建，执行特别紧急的任务。一个cpu一个进程，名字叫做[migration/CPU_ID]；调度均衡要迁移线程的时候会用到，所以它的名字叫做migration。
+ 限时调度：硬实时，按照配置的运行周期、运行时间和截止时间进行调度。
+ 实时调度：软实时，为每个优先级维护一个队列，实时优先级大的进程先运行；当两个实时进程优先级相同，SCHED_RR会交替调度，SCHED_FIFO则不会
+ 分时调度：适用于广大的普通进程，由CFS算法实现
+ 闲时调度：内核启动时创建，没有其他进程时运行

> 内核防止实时进程饿死普通进程。提供了一个配置参数，默认值是实时进程如果已经占用了95%的CPU时间，就会把剩余5%的CPU时间分给普通进程

### 进程优先级
+ nice值表示进程的优先级，`nice=static_prio - 120`
+ 只有实时调度类和分时调度类会用到进程优先级
+ 进程的优先级由0-139的整数表示

  ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231017212539.png)
#### 进程nice值获取
```c
#include <linux/sched.h>
#include <linux/pid.h>
#include <linux/module.h>
#include <linux/kthread.h>


int MyFuncTest(void *args)
{
    printk("Prompt:kernel thread function.\n");
    // 打印输出当前进程的静态优先级
    printk("Prompt:Current static_prio : %d\n",current->static_prio);
    // 打印输出当前进程的PID值
    printk("Prompt:Current Process PID : %d\n",current->pid);
    // 打印输出当前进程的nice值
    printk("Prompt:Current Process nice : %d\n",task_nice(current));
    return 0;
}

static int __init Tasknice_InitFunc(void)
{
    struct task_struct *pointer_task;
    int priority;

    printk("Prompt:Tasknice_InitFunc function.\n");
    // 创建一个新的进程
    pointer_task=kthread_create_on_node(MyFuncTest,NULL,-1,"mytasknicedemo");
    // 主要作用唤醒一个处于睡眠状态的进程
    wake_up_process(pointer_task);
    // 打印输出创建新进程的nice值
    priority=task_nice(pointer_task);
    // 打印输出新进程的静态优先级
    printk("Prompt:static_prio of the child thread : %d\n",pointer_task->static_prio);
    // 打印输出新进程的nice值
    printk("Prompt:New process nice : %d\n",priority);
    // 打印输出当前进程的PID值
    printk("Prompt:Current PID : %d\n",current->pid);
    printk("Prompt:Current nice : %d\n",task_nice(current));

    return 0;
}

static void __exit Tasknice_ExitFunc(void)
{
    printk("Prompt:Normal exit of kernel module.\n");
}

module_init(Tasknice_InitFunc); // 内核模块入口函数
module_exit(Tasknice_ExitFunc); // 内核模块退出函数
MODULE_LICENSE("GPL"); // 模块的许可证声明
MODULE_AUTHOR("0voice 2023/07/08"); // 声明由那一位作者或机构单位所编写的模块
```
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231024204426.png)

> 从打印日志看，是主进程执行完后，子进程再继续执行

#### 进程nice值设置
```c
#include <linux/module.h>
#include <linux/pid.h>
#include <linux/kthread.h>
#include <linux/sched.h>

int MyFunc_Test(void *arg)
{
    printk("Prompt:********* MyFunc_Test Start *********");
    // 打印输出当前进程的PID
    printk("Prompt:Current PID : %d\n",current->pid);
    // 打印输出当前进程的静态优先级
    printk("Prompt:Current static_prio : %d\n",current->static_prio);
    // 打印输出当前进程的nice值
    printk("Prompt:Current nice :%d\n",task_nice(current));
    printk("Prompt:********* MyFunc_Test End *********");
    return 0;
}

static int __init SetUserNice_InitFunc(void)
{
    struct task_struct *pointer_task=NULL;

    printk("Prompt:********* SetUserNice_InitFunc Start *********");
    // 创建新进程
    pointer_task=kthread_create_on_node(MyFunc_Test,NULL,-1,"SetUserNice_TestDemo");
    // 唤醒新进程（唤醒处于睡眠状态的进程，意思让睡眠状态进程转换为RUNNING状态，它就能够被CPU重新调整执行）
    wake_up_process(pointer_task);
    // 打印输出新进程的静态优先级
    printk("Prompt:New static_prio child thread : %d\n",pointer_task->static_prio);
    // 打印输出新进程的nice的值
    printk("Prompt:New nice child thread : %d\n",task_nice(pointer_task));
    // 打印输出新进程的动态优先级
    printk("Prompt:New prio child thread : %d\n",pointer_task->prio);
    // 设置新进程nice的值
    set_user_nice(pointer_task,16);
    printk("Prompt:New value static_prio child thread : %d\n",pointer_task->static_prio);
    printk("Prompt:New value nice child thread ： %d\n",task_nice(pointer_task));
    printk("Prompt:New thread PID : %d\n",pointer_task->pid);
    printk("Prompt:********* SetUserNice_InitFunc End *********");

    return 0;
}

static void __exit SetUserNice_ExitFunc(void)
{
    printk("Prompt:Normal exit of kernel module.\n");
}

module_init(SetUserNice_InitFunc); // 内核模块入口函数
module_exit(SetUserNice_ExitFunc); // 内核模块退出函数
MODULE_LICENSE("GPL"); // 模块的许可证声明
MODULE_AUTHOR("0voice 2023/07/08"); // 声明由那一位作者或机构单位所编写的模
```
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231024204659.png)

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
+ cfs运行队列在内核中由[红黑树](https://github.com/ji92/linux-driver/blob/main/redblack_tree.pptx)（查找、插入、删除的时间复杂度都是log(n)）维护

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
#### SMP调度
+ 负载均衡
  + 限期调度类的处理器负载均衡(static void pull_dl_task(struct rq *this_rq))
  + 实时调度类的处理器负载均衡(static void pull_rt_task(struct rq *this_rq))
  + 公平调度类的处理器负载均衡
+ 亲和性
  + 编程接口
    + 用户线程提供sched_setaffinity()和sched_getaffinity()
    + 内核线程提供kthread_bind()和set_cpus_allowed_ptr() 
  + int nr_cpus_allowed
  + cpumask_t cpus_allowed
+ 进程在处理器之间迁移

#### NUMA调度
  + 每个NUMA节点内部按照SMP进行调度

#### 调度组和调度域的概念
+ 逻辑核/物理核/芯片组/NUMA节点等维度
[ARM调度域/调度组之概念理解](https://blog.csdn.net/wukongmingjing/article/details/81664820)  

## 进程间同步

### per-CPU

### RCU(Read-Copy-Update)机制
+ RCU重要的应用场景是链表，允许多个线程同时读取链表，并且允许一个线程同时修改链表，待临界区读完毕后，将指针指向新的节点
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019205015.png)
  
+ 优势：读者没有任何同步开销
+ 劣势：写者同步开销大，需要互斥操作。
```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/slab.h>
#include <linux/spinlock.h>
#include <linux/kthread.h>
#include <linux/delay.h>

struct RCUStruct{
    int a;
    struct rcu_head rcu;
};

static struct RCUStruct *Global_pointer;

// 创建2个读内核线程和1个写内核线程
static struct task_struct *RCURD_Thr1,*RCURD_Thr2,*RCUWriter_Thread;

static int MyRCU_ReaderThreadFunc1(void *data) // 读者线程1
{
    struct RCUStruct *pointer1=NULL;
    while(1)
    {
        msleep(5);      
        // 读多写少并发场景：高效、低开销  
        rcu_read_lock();

        mdelay(10);
        pointer1=rcu_dereference(Global_pointer);
        if(pointer1)
        printk("%s:read a=%d\n",__func__,pointer1->a);

        rcu_read_unlock();
    }

    return 0;
}

static int MyRCU_ReaderThreadFunc2(void *data) // 读者线程2
{
    struct RCUStruct *pointer2=NULL;
    while(1)
    {
        msleep(5);      
        // 读多写少并发场景：高效、低开销  
        rcu_read_lock();

        mdelay(10);
        pointer2=rcu_dereference(Global_pointer);
        if(pointer2)
        printk("%s:read a=%d\n",__func__,pointer2->a);

        rcu_read_unlock();
    }

    return 0;
}

static void myrcu_del(struct rcu_head *rcuh)
{
    struct RCUStruct *p=container_of(rcuh,struct RCUStruct,rcu);
    printk("%s:a=%d\n",__func__,p->a);
    kfree(p);
}

static int MyRCU_WriterThread_Func(void *pointer) // 写者线程
{
    struct RCUStruct *old;
    struct RCUStruct *new_ptr;
    int value=(unsigned long)pointer;

    while(1)
    {
        msleep(10);

        new_ptr=kmalloc(sizeof(struct RCUStruct),GFP_KERNEL);

        old=Global_pointer;
        *new_ptr=*old;
        new_ptr->a=value;

        rcu_assign_pointer(Global_pointer,new_ptr);

        call_rcu(&old->rcu,myrcu_del);

        printk("%s:write to new %d\n",__func__,value);

        value++;
    }

    return 0;
}

static int __init MyRCU_TestFunc_Init(void)
{
    int value=2;
    // 提示：初始化内核模块成功
    printk("Prompt:Successfully initialized the kernel module.\n");

    Global_pointer=kzalloc(sizeof(struct RCUStruct),GFP_KERNEL);

    RCURD_Thr1=kthread_run(MyRCU_ReaderThreadFunc1,NULL,"RCU_RDer1");
    RCURD_Thr2=kthread_run(MyRCU_ReaderThreadFunc2,NULL,"RCU_RDer2");
    RCUWriter_Thread=kthread_run(MyRCU_WriterThread_Func,(void*)(unsigned long)value,"RCU_Writer");

    return 0;
}

static void __exit MyRCU_TestFunc_Exit(void)
{
    // 卸载内核模块成功
    printk("Prompt:Successfully uninstalled kernel module!\n");
    kthread_stop(RCURD_Thr1);
    kthread_stop(RCURD_Thr2);
    kthread_stop(RCUWriter_Thread);

    if(Global_pointer)
    kfree(Global_pointer); 
}

module_init(MyRCU_TestFunc_Init); // 内核模块入口函数
module_exit(MyRCU_TestFunc_Exit); // 内核模块退出函数
MODULE_LICENSE("GPL"); // 模块的许可证声明
MODULE_AUTHOR("0voice 2023/07/02"); // 声明由那一位作者或机构单位所编写的模块

```

## 进程通信

## 





### 内存屏障
+ 由于编译器乱序和超标量乱序处理器的存在，使内存访问顺序不符合预期
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019205946.png)

+ 为了阻止编译器错误的重排指令，在禁止内核抢占和开启内核抢占的里面添加编译器优化屏障
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019210121.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019210237.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231019210325.png)


# 内存管理
> cat /proc/meminfo 查询内存信息
```


```
## 虚拟内存
.init模块初始化结束可以释放
### 堆管理

## 物理内存

# 中断管理

# 文件系统

# 网络协议栈
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231024210450.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231024212612.png)




## 传输层
### TCP
### UDP
### SCTP
### DCCP

### socket编程
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231024213527.png) 

## 网络层
+ [小林-coding：IP基础知识全家桶](https://xiaolincoding.com/network/4_ip/ip_base.html)

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231029180550.png)

+ 接收分组
  + 在分组转发到ip_rcv之后，必须检查接收到的信息确保它是正确的。主要检查计算的校验和与首部中存储的校验和是否一致。其他的检查包括分组是否达到了IP首部的最小长度，分组的协议是否确实是IPv4。 
+ 分组转发
  ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231029180749.png)
  
+ 分组发送
  + ip_queue_xmit()完成面向连接套接字的包输出，当套接字处于连接状态时，所有从套接字发出的包都具有确定的路由，无需为每个包进行查询, 可以将套接字绑定到路由路口上，这由套接字的目的缓冲指针dst_cache完成
  ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231029181147.png)
  ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231029181209.png)

### ICMP
+ [小林-coding：ICMP]
### route subsystem

### neighbor subsystem
+ Linux邻接子系统负责发现当前链路上的节点，并且将L3(网络层)地址转换为L2(数据链路层)地址。在IPv4当中，实现这种转换的协议为**地址解析协议（Address Resolution Protocol,ARP）**, 而在IPv6则为**邻居发现协议（Neighbour Discovery Protocol，NDISC或ND）**，邻接子系统为执行L3到L2映射提供了独立于协议的基础设施。
+ 在IPv4中，使用邻接协议为ARP，相应的请求和应答分别被称为ARP请求和ARP应答，在IPv6中，使用的邻接协议为NDISC，相应的请求和应答分别称为邻居请求和邻居通告
+ 为避免在每次传输数据包时都发送请求，内核将L3地址和L2地址之间的映射存储在邻接表的数据结构中。在IPv4中，这个表就是ARP表，有时也称为ARP缓存。在IPv6中，邻接表是NDISC表（也叫NDISC缓存）
> 用户空间可以通过iproute2包中的命令ip neigh，也可以使用net_tools包中的命令arp和邻接子系统进行交互
> ![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231029172056.png)

### netfilter subsystem
+ 主要功能：
  + 数据包选择（iptables）
  + 数据包过滤
  + 网络地址转换（NAT）
  + 数据包操纵
  + 连接跟踪（为NAT的基础）
  + 网络统计信息

+ netfilter挂接点

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231029195516.png)
+ 注册hook回调函数

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231029195636.png)
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231029200117.png)


#### Network Address Translation(NAT)
+ [小林-coding：NAT](https://xiaolincoding.com/network/4_ip/ip_base.html#nat)
+ [网络地址转换协议NAT功能详解及NAT基础知识介绍](https://zhuanlan.zhihu.com/p/26992935)
+ [Linux iptables&&Firewalld](https://firststory.feishu.cn/docs/doccnr9KMt9Ynbmv4oJIRWrh4vh#v3dkHe)

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20231029200353.png)

### IPsec
+ 见[IPsec](https://github.com/ji92/linux-driver/blob/main/IPsec.md)

## net_device子系统

## NIC收发包流程




# 参考资料

+ [零声教育](https://gitlab.0voice.com/linux)
+ [深入理解Linux进程调度](https://mp.weixin.qq.com/s/3rV6d04QjO9_8Nkq9SrWYg)
+ [小林coding](https://xiaolincoding.com/)
+ [深入理解Linux线程同步](https://mp.weixin.qq.com/s?__biz=Mzg2OTc0ODAzMw==&mid=2247508887&idx=1&sn=a47f27306807f0fca638b6de4f762451&chksm=ce9abfb9f9ed36afb1c782b4346e8f01f24897df55e6ea85b234f68218b6acddf3ef2069c68c&scene=178&cur_album_id=2519398872503353344#rd)
