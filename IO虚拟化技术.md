# IO 虚拟化

在虚拟化系统中，I/O外设只有一套，需要被多个guest VMs共享。VMM/hypervisor提供了两种机制来实现对I/O设备的访问，一种是直通（passthrough），一种是模拟（emulation）。
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904221816.png)

# 直通

所谓passthrough，就是指guest VM可以透过VMM，直接访问I/O硬件，这样guest VM的I/O操作路径几乎和无虚拟化环境下的I/O路径相同，性能自然是非常高的

# 直通有哪些技术

vfio(virtual function io)直通和SR-IOV
PCI直通是vfio直通的其中一种。
PCI直通式使用host的物理网卡提供虚拟机使用，是独占式的。SR-IOV虚拟一个物理网卡为多个虚拟网卡，同时提供给多个虚拟机使用
`前置条件:`打开IOMMU功能

# SR-IOV

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911204854.png)

SR-IOV 技术是一种基于硬件的虚拟化解决方案，可提高性能和可伸缩性。
SR-IOV 标准允许在虚拟机之间高效共享 PCIe设备，并且它是在硬件中实现的，可以获得能够与本机性能媲美的 I/O 性能。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911204914.png)

优点：
+ 高性能
+ 减少交换机端口
+ 简化布线
+ 减少适配器数量
+ 节能

## SR-IOV vs MR-IOV
SR-IOV只支持一个Root Complex，而MR-IOV支持多个Root Complex，并且，需要特殊的支持MR-IOV的Switch才能够组网。
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911205248.png)

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911205313.png)

## SR-IOV架构
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911205615.png)

所有的IO访问都要通过软件实现的`Virtualization Intermediary(VI)`统一调度和统一转发,性能损失极大。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911205738.png)

SR-PCIM：Single Root PCI Manager
ATPT: Address Translation and Protection Table
TA: Translation Agent
ATC: Address Translation Cache

## PF vs VF
Physical Function(PF)
+ 基本规范定义的PCI功能（Function）的名称。
+ 具有SR-IOV能力的PF负责用于配置IOV功能以及管理PF功能。
+ PF被SR-PCIM来管理配置虚拟化VF功能。

Virtual Function(VF)
+ 虚拟化后的功能的名称。
+ VF有自己的Bus、device和Function号和自己的BAR地址。
+ SI可以直接访问这些功能上的资源。
+ VF由SR-PCIM创建/管理。
+ 一个VF只与一个PF相关联。
+ VF创建后，可以通过RC（Root Complex）按照传统PCIe方法进行配置和访问。

SR-IOV规范对于支持SR-IOV的Phyesical Function增加了一个新的能力寄存器（SR-IOV Capability）用于管理、配置、使能SR-IOV能力。
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911210731.png)

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911211640.png)

+ `SR-IOV Control`字段的bit0位是SR-IOV的使能位，默认为0，表示关闭，如果需要开启SR-IOV功能，需要配置为1。
+ `TotalVFs`字段表示PCIe Device支持VF的数量。
+ `NumVFs`字段表示开启VF的数量，此值不应超过PCIe Device支持的VF的数量TotalVFs的值。
+ `First VF Offset`字段表示第一个各VF相对PF的Routing ID（即Bus number、Device number、Function number）的偏移量。
+ `VF Stride`字段表示相邻两个VF的Routing ID的偏移量。
其他字段含义详见《Single Root I/O Virtualization and Sharing Specification Revision 1.1》。

 在Linux下，内核是支持SR-IOV特性的。每个支持SR-IOV的设备厂商的驱动往往也是两个，一个是基本PF驱动，一个是虚拟VF驱动。

# VFIO直通

## IOMMU(x86)

### IOMMU功能简介

IOMMU主要功能包括 `DMA Remapping`和 `Interrupt Remapping`，**这里主要讲解 `DMA Remapping`**，Interrupt Remapping会独立讲解。对于DMA Remapping，IOMMU与MMU类似。IOMMU可以将一个设备访问地址转换为存储器地址。

IOMMU 通常是实现在北桥之中，现在北桥通常被集成进SOC中了，所以IOMMU通常都放在SOC内部了。不同芯片厂商的实现大同小异，可是名字却不太一样。Intel的芯片上叫做 `VT-d （Virtualization Technology for Directed I/O ）`，AMD还是叫做 `IOMMU`，ARM中叫做 `SMMU`。下图针对有无IOMMU情况说明IOMMU作用。

### DMA remapping

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904221649.png)

+ 传统DMA方式，由内核对DMA配置完PA后，DMA直接以PA访问内存。
+ **在虚拟化场景下，直通设备进行DMA操作的时候Guest驱动直接使用GPA来访问内存（从虚机视角来看，每个设备都是传统的DMA方式，IOMMU和VMM对它来讲是透明的），如果不加以地址隔离和翻译必然会访问到其他VM的物理内存或者破坏Host内存，因此需要一套地址翻译和隔离机制**

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904221726.png)

+ RC内部实现IOMMU，DMA发出VA翻译成PA后，访问内存。
+ 还有一点，传统DMA一般都是连续的物理内存分配，使用内存翻译可以将离散的物理内存组合成连续的虚拟内存空间，提升内存的使用率。

所以针对没有IOMMU的情况，不能用透传的方式，对于设备的直接访问都会有VMM接管，这样就不会对虚机暴露HPA.

右图是有IOMMU的情况，虚机可以将GPA直接写入到设备，当设备进行DMA传输时，设备请求地址GPA由IOMMU转换为HPA（硬件自动完成），进而DMA操作真实的物理空间。IOMMU的映射关系是由VMM维护的，HPA对虚机不可见，保障了安全问题，利用IOMMU可实现设备的透传。这里先留一个问题，既然IOMMU可以将设备访问地址映射成真实的物理地址，那么对于右图中的Device A和Device B，IOMMU必须保证两个设备映射后的物理空间不能存在交集，否则两个虚机可以相互干扰，这和IOMMU的映射原理有关，后面会详细介绍。

### Interrupt remapping原理（空）

### DMA remapping原理

IOMMU通过引入 `Root Context Entry`和 `IOMMU Domain Page Table`等机制来实现直通设备隔离和DMA地址转换目的。那么具体怎么实现的呢？下面对其进行介绍。

根据DMA Request是否包含地址空间标志(address-space-identifier)我们将DMA Request分为2类：

+ Requests without address-space-identifier: 不含地址空间标志的DMA Request，这种一般是endpoint devices的普通请求，请求内容仅包含请求的类型(read/write/atomics)，DMA请求的address/size以及请求设备的标志符等。
+ Requests with address-space-identifier: 包含地址空间描述标志的DMA Request，此类请求需要包含额外信息以提供目标进程的地址空间标志符(PASID)，以及Execute-Requested (ER) flag和 Privileged-mode-Requested 等细节信息。

为了简单，通常称上面两类DMA请求简称为：`Requests-without-PASID`和 `Requests-with-PASID`。**本节我们只讨论Requests-without-PASID，后面我们会在讨论Shared Virtual Memory的文中单独讨论Requests-with-PASID。**

首先要明确的是DMA Isolation是以Domain为单位进行隔离的，**在虚拟化环境下可以认为每个VM的地址空间为一个Domain，直通给这个VM的设备只能访问这个VM的地址空间这就称之为“隔离”**。根据软件的使用模型不同，直通设备的DMA Address Space可能是某个VM的Guest Physical Address Space或某个进程的虚拟地址空间（由分配给进程的PASID定义）或是由软件定义的一段抽象的IO Virtual Address space (IOVA)。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230905222908.png)

#### 如何实现地址翻译

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904223755.png)

+ 地址翻译借助Request请求中的BDF号完成。IOMMU首先根据 `Bus`搜索 `Root table`，然后利用 `Device`、`Func`在 `Context Table`中找到对应的 `Context Entry`，即页表首地址，然后利用页表即可将设备请求的虚拟地址翻译成物理地址。
+ 这里做以下说明：

  + 图中红线的部分，是两个Context Entry指向了同一个页表。这种情况在虚拟化场景中的典型用法就是这两个Context Entry对应的不同PCIe设备属于同一个虚机，那样IOMMU在将GPA->HPA过程中要遵循同一规则；
  + 每个具有Source Identifier(包含Bus、Device、Func)的设备都会具有一个Context Entry。如果不这样做，所有设备共用同一个页表，隶属于不同虚机的不同GPA就会翻译成相同HPA，会产生问题。

有了页表之后，就可以按照MMU那样进行地址映射工作了，这里也支持不同页大小的映射，包括4KB、2MB、1GB，不同页大小对应的级数也不同，下图以4KB页大小为例说明，映射过程和MMU类似，不再详细阐述。
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230904223908.png)

#### Source Identifier

在讲述IOMMU的工作原理时，讲到了设备利用自己的Source Identifier(包含Bus、Device、Func)来找到页表项来完成地址映射，不过存在下面几个特殊情况需要考虑。

+ **对于由PCIe switch扩展出的PCI桥及桥下设备，在发送DMA请求时，Source Identifier是PCIe switch的，这样的话该PCI桥及桥下所有设备都会使用PCIe switch的Source Identifier去定位Context Entry，找到的页表也是同一个，如果将这个PCI桥下的不同设备分给不同虚机，由于会使用同一份页表，这样会产生问题，针对这种情况，当前PCI桥及桥下的所有设备必须分配给同一个虚机，这就是VFIO中组的概念，下面会再讲到。**
+ 对于SRIO-V，之前介绍过VF的Bus及devfn的计算方法，所以**不同VF会有不同的Source Identifier，映射到不同虚机也是没有问题的。**

## SMMU(ARM)

在虚拟化的环境中，借助 IOMMU 的 GPA->HPA 转换，DMA 控制器可以直接以 guest VM 提供的 GPA 作为 source 或者 destination，减少了 VMM 的参与。

在非虚拟化的环境中，借助 IOMMU 的 HVA->HPA 转换，DMA 控制器也可以直接以 userspace buffer 的 HVA 作为 source/destination，而不需要 OS kernel 将 user process 的 VA 转换后的 PA 填入 DMA 控制器。

两者其实是统一的，不管 HVA 还是 GPA，都不是 real phsical address，有了 IOMMU/SMMU 之后，DMA 控制器就可以使用虚拟地址，也不再要求对应的内存是物理连续的了。

### 原理

在Linux的实现中，一个进程有一个对应的页表，而SMMU是为设备服务的，几个设备可能同属于一个guest VM，因此多个设备可能会共用一个GPA->HPA的转换页表。同一个guest VM的设备可能属于或者不属于某一个特定的进程，因此也可能共用或者不共用GVA->GPA的转换页表。

在SMMU中，一个发起DMA传输（transaction）的设备的信息由一个 `Stream Table Entry(STE)`来描述。所有的STEs共同构成了 `Stream Table`，可由 `StreamID`作为Stream Table数组的索引，查找得到对应的STE，因此**StreamID也就成了设备唯一性的标识**。Stream table可以是1-level的,也可以是2-level的.比如一个10位的StreamID使用[9:8]的2个bits作为第一级Stream table的索引，[7:0]的8个bits作为第二级Stream table的索引，这和使用虚拟地址作为索引查找页表是一样的（参考[这篇文章](https://zhuanlan.zhihu.com/p/65348145)）。

**不同的是，多级页表中每级页表的大小是相同的，而第二级Stream table的大小可以是不同的**，比如下图中A就是占满了的，有255个entries，而B和C分别只有4个entries和1个entry。用户可以根据实际的需要灵活配置，以节约内存空间。
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911193504.png)

SMMUv3只规定了StreamID的大小是0-32个bits，但具体采用多少个bits，以及StreamID是怎样构成的，则都属于implementation defined。但SMMUv3同时也做出了限制：**如果StreamID的数目超过了64（也就是超过了6个bits），那么必须使用2-level的Stream table。**

在同一个VM内，一个设备可能被VM内的多个进程共享，而这些进程会有不同的页表，因此一个设备在和不同进程交互时，也会使用不同的页表。为了区分，需要一个设备对应进程的描述信息，这个描述信息被称为 `CD(Context Descriptor)`。

**如果该设备需要进行的是GVA->GPA的stage 1转换（比如前面提到的用户空间的DMA传输），那么描述该设备的STE将指向当前进程对应的这个CD（下图中的B）**。和同一个进程交互的不同设备将共用这个进程的stage 1页表，因此这些设备的STEs将指向同一个CD

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911193745.png)

每个CD包含了一个标识进程的 `address space ID(ASID)`，由TTB0和TTB1指向进程对应的stage 1页表（上图中的C）。熟悉ARM架构的同学应该知道，**TTBR0和TTBR1是ARM中分别用来存储进程页表和内核页表的基地址的寄存器**，由于这里不再是Register，而是内存中的Descriptor，但用途类似，所以就叫TTB0和TTB1。

同一个设备对应的所有CDs构成了一个 `CD table`。CD table使用的查找索引被称为 `SubstreamID`。SubstreamID最多占20个bits，这个规定是有出处的。熟悉PCIe的同学可能知道，PCIe中有个PASID，而PASID最大就是20个bits。SubstreamID基本就是和PASID对等的概念，虽然SMMU并不限于只支持PCIe的设备，但这里SubstreamID还是和PASID保持了统一的步调。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911193808.png)

同Stream table一样，CD table可以是1-level的（下图中的D），也可以是2-level的（下图中的E和F）。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911193825.png)

如果是熟悉x86的segmentation机制或者是看过[这个系列的文章](https://zhuanlan.zhihu.com/p/67714693)的同学，会不会发现SMMU中的这个Stream table/CD table，其实和x86中的GDT/LDT是很相似的。都是通过索引查找得到对应的entry，每一个entry都是记录的一些描述信息，而且后面还都是跟的page tables。

SMMU支持stage 1的GVA->GPA（按ARM的叫法则是VA->IPA）的转换，也支持stage 2的GPA->HPA（IPA->PA）的转换，你可以选择只进行stage 1的转换，不进行stage 2的转换，也可以绕过stage 1的转换，只用stage 2的转换，还可以stage 1和stage 2的转换都用。stage 1和stage 2的转换都不用……那就没SMMU什么事了。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230911193912.png)

linux内核中关于SMMU的实现形式，也就是上面提到的这四种情况：

```c
enum arm_smmu_domain_stage {
  /* 如果DMA直接访问PA，无论是GPA还是HPA，都不需要stage1; 如果OS没有虚拟化，则不需要stage2 */
	ARM_SMMU_DOMAIN_S1 = 0, /* OS没有虚拟化，DMA访问线程用户/内核地址空间 */
	ARM_SMMU_DOMAIN_S2, /* OS虚拟化，DMA直接访问用户/内核地址空间，即在这个GuestOS中，使用同一个translation table（S2TTB，stage 2 translation table base） */
	ARM_SMMU_DOMAIN_NESTED, /* OS虚拟化，DMA访问线程用户/内核地址空间 */
	ARM_SMMU_DOMAIN_BYPASS, /* 无虚拟化，DMA场景直接访问内存 */
}
```

[疑问] stage1和stage2都生效场景下，tage1的页表保存一份还是两份。
[答] ARM 的 SMMU 查询用的 I/O 页表和普通 MMU 页表可共用，而 x86 的 IOMMU 用的 I/O 页表则不共用。关于 IOMMU/SMMU 硬件上的功能和设计差异，可看下 [再议 IOMMU](https://zhuanlan.zhihu.com/p/76643300) 一文


## 内存虚拟化

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230907234523.png)

### Shadow page table(sPT) （可以忽略）

为了支持GVA->GPA->HPA的两次转换，可以计算出GVA->HPA的映射关系，将其写入一个单独的 `sPT`。在一个运行Linux的guest VM中，每个进程有一个由内核维护的页表，用于GVA->GPA的转换，这里我们把它称作 `gPT`(guest Page Table)。

VMM层的软件会将gPT本身使用的物理页面设为 `write protected`的，那么每当gPT有变动的时候（比如添加或删除了一个页表项），就会产生被VMM截获的 `page fault`异常，之后VMM需要重新计算GVA->HPA的映射，更改sPT中对应的页表项。可见，这种纯软件的方法虽然能够解决问题，但是其存在两个缺点：

+ 实现较为复杂，需要为每个guest VM中的每个进程的gPT都维护一个对应的sPT，增加了内存的开销。
+ VMM使用的截获方法增多了page fault和trap/vm-exit的数量，加重了CPU的负担。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230907234603.png)

### 硬件辅助-EPT/NPT

为此，各大CPU厂商相继推出了硬件辅助的内存虚拟化技术，比如Intel的 `EPT`(Extended Page Table)和AMD的 `NPT`(Nested Page Table），它们都能够从硬件上同时支持GVA->GPA和GPA->HPA的地址转换的技术。
GVA->GPA的转换依然是通过查找gPT页表完成的，而GPA->HPA的转换则通过查找nPT页表来实现，每个guest VM有一个由VMM维护的nPT。其实，EPT/NPT就是一种扩展的MMU（以下称EPT/NPT MMU），它可以交叉地查找gPT和nPT两个页表：
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230907234939.png)

假设gPT和nPT都是4级页表，那么EPT/NPT MMU完成一次地址转换的过程是这样的（不考虑TLB）：
首先它会查找guest VM中CR3寄存器（gCR3）指向的PML4页表，由于gCR3中存储的地址是GPA，因此CPU需要查找nPT来获取gCR3的GPA对应的HPA。nPT的查找和前面文章讲的页表查找方法是一样的，这里我们称一次nPT的查找过程为一次nested walk。
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230907235102.png)

如果在nPT中没有找到，则产生EPT violation异常（可理解为VMM层的page fault）。如果找到了，也就是获得了PML4页表的物理地址后，就可以用GVA中的bit位子集作为PML4页表的索引，得到PDPE页表的GPA。接下来又是通过一次nested walk进行PDPE页表的GPA->HPA转换，然后重复上述过程，依次查找PD和PE页表，最终获得该GVA对应的HPA。
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230907235123.png)

**不同于影子页表是一个进程需要一个sPT，EPT/NPT MMU解耦了GVA->GPA转换和GPA->HPA转换之间的依赖关系，一个VM只需要一个nPT，减少了内存开销。如果guest VM中发生了page fault，可直接由guest OS处理，不会产生vm-exit，减少了CPU的开销。可以说，EPT/NPT MMU这种硬件辅助的内存虚拟化技术解决了纯软件实现存在的两个问题。**

### EPT/NPT MMU优化

EPT/NPT MMU作为传统MMU的扩展，自然也是有TLB的，它在查找gPT和nPT之前，会先去查找自己的TLB（前面为了描述的方便省略了这一步）。但这里的TLB存储的并不是一个GVA->GPA的映射关系，也不是一个GPA->HPA的映射关系，而是最终的转换结果，也就是 `GVA->HPA`的映射。

不同的进程可能会有相同的虚拟地址，为了避免进程切换的时候flush所有的TLB，可通过给TLB entry加上一个标识进程的PCID/ASID的tag来区分（参考[这篇文章](https://zhuanlan.zhihu.com/p/66971714)）。同样地，不同的guest VM也会有相同的GVA，为了flush的时候有所区分，需要再加上一个标识虚拟机的tag，这个tag在ARM体系中被叫做 `VMID`，在Intel体系中则被叫做 `VPID`。

> ASID：Address Space ID   地址空间标识符
> PCID：Process context identifier 进程上下文标识符
> PCID（进程上下文标识符）是在Westmere架构引入的新特性。简单来说，**在此之前，TLB是单纯的VA到PA的转换表，进程1和进程2的VA对应的PA不同，不能放在一起。加上PCID后，转换变成VA + 进程上下文ID到PA的转换表，放在一起完全没有问题了**。这样进程1和进程2的页表可以和谐的在TLB中共处，进程在它们之前切换完全不需要预热了！

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230907235316.png)

在最坏的情况下（也就是TLB完全没有命中），gPT中的每一级转换都需要一次nested walk【1】，而每次nested walk需要4次内存访问，因此5次nested walk总共需要 (4+1)*5-1=24 次内存访问（就像一个5x5的二维矩阵一样）：

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230907235339.png)

虽然这24次内存访问都是由硬件自动完成的，不需要软件的参与，但是内存访问的速度毕竟不能与CPU的运行速度同日而语，而且内存访问还涉及到对总线的争夺，次数自然是越少越好。

要想减少内存访问次数，要么是增大EPT/NPT TLB的容量，增加TLB的命中率，要么是减少gPT和nPT的级数。gPT是为guest VM中的进程服务的，通常采用4KB粒度的页，那么在64位系统下使用4级页表是非常合适的（参考这篇文章）。

而nPT是为guset VM服务的，对于划分给一个VM的内存，粒度不用太小。64位的x86_64支持2MB和1GB的large page，假设创建一个VM的时候申请的是2G物理内存，那么只需要给这个VM分配2个1G的large pages就可以了（这2个large pages不用相邻，但large page内部的物理内存是连续的），这样nPT只需要2级（nPML4和nPDPE）。

如果现在物理内存中确实找不到2个连续的1G内存区域，那么就退而求其次，使用2MB的large page，这样nPT就是3级（nPML4, nPDPE和nPD）。

## VFIO

VFIO就是内核针对IOMMU提供的软件框架，支持 `DMA Remapping`和 `Interrupt Remapping`，这里只讲DMA Remapping。VFIO利用IOMMU这个特性，可以屏蔽物理地址对上层的可见性，可以用来开发用户态驱动，也可以实现设备透传。

### 概念介绍

先介绍VFIO中的几个重要概念，主要包括 `Group`和 `Container`。

+ `Group：`group 是IOMMU能够进行DMA隔离的最小硬件单元，一个group内可能只有一个device，也可能有多个device，这取决于物理平台上硬件的IOMMU拓扑结构。 设备直通的时候一个group里面的设备必须都直通给一个虚拟机。 不能够让一个group里的多个device分别从属于2个不同的VM，也不允许部分device在host上而另一部分被分配到guest里， 因为就这样一个guest中的device可以利用DMA攻击获取另外一个guest里的数据，就无法做到物理上的DMA隔离。
+ `Container：`对于虚机，Container 这里可以简单理解为一个VM Domain的物理内存空间。对于用户态驱动，Container可以是多个Group的集合。
+ 上图中PCIe-PCI桥下的两个设备，在发送DMA请求时，PCIe-PCI桥会为下面两个设备生成Source Identifier，其中Bus域为红色总线号bus，device和func域为0。这样的话，PCIe-PCI桥下的两个设备会找到同一个Context Entry和同一份页表，所以这两个设备不能分别给两个虚机使用，这两个设备就属于一个Group。

### 使用示例

这里先以简单的用户态驱动为例，在设备透传小节中，在分析如何利用vfio实现透传。

```c
int container, group, device, i;
struct vfio_group_status group_status =
{ .argsz = sizeof(group_status) };
struct vfio_iommu_type1_info iommu_info = { .argsz = sizeof(iommu_info) };
struct vfio_iommu_type1_dma_map dma_map = { .argsz = sizeof(dma_map) };
struct vfio_device_info device_info = { .argsz = sizeof(device_info) };

/* Create a new container */
container = open("/dev/vfio/vfio", O_RDWR);

if (ioctl(container, VFIO_GET_API_VERSION) != VFIO_API_VERSION)
/* Unknown API version */

if (!ioctl(container, VFIO_CHECK_EXTENSION, VFIO_TYPE1_IOMMU))
/* Doesn't support the IOMMU driver we want. */

/* Open the group */
group = open("/dev/vfio/26", O_RDWR);

/* Test the group is viable and available */
ioctl(group, VFIO_GROUP_GET_STATUS, &group_status);

if (!(group_status.flags & VFIO_GROUP_FLAGS_VIABLE))
/* Group is not viable (ie, not all devices bound for vfio) */

/* Add the group to the container */
ioctl(group, VFIO_GROUP_SET_CONTAINER, &container);

/* Enable the IOMMU model we want */ // type 1 open | attatch
ioctl(container, VFIO_SET_IOMMU, VFIO_TYPE1_IOMMU);

/* Get addition IOMMU info */
ioctl(container, VFIO_IOMMU_GET_INFO, &iommu_info);

/* Allocate some space and setup a DMA mapping */
dma_map.vaddr = mmap(0, 1024 * 1024, PROT_READ | PROT_WRITE,
MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
dma_map.size = 1024 * 1024;
dma_map.iova = 0; /* 1MB starting at 0x0 from device view */
dma_map.flags = VFIO_DMA_MAP_FLAG_READ | VFIO_DMA_MAP_FLAG_WRITE;

ioctl(container, VFIO_IOMMU_MAP_DMA, &dma_map);

/* Get a file descriptor for the device */
device = ioctl(group, VFIO_GROUP_GET_DEVICE_FD, "0000:06:0d.0");

/* Test and setup the device */
ioctl(device, VFIO_DEVICE_GET_INFO, &device_info);
```

对于dev下Group就是按照上一节介绍的Group划分规则产生的，上述代码描述了如何使用VFIO实现映射，对于Group和Container的相关操作这里不做过多解释，主要关注如何完成映射，下图解释具体工作流程。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230905220655.png)

首先，利用 `mmap`映射出1MB字节的虚拟空间，因为物理地址对于用户态不可见，只能通过虚拟地址访问物理空间。然后执行 `ioctl`的 `VFIO_IOMMU_MAP_DMA`命令，传入参数主要包含 `vaddr`及 `iova`，其中 `iova`代表的是设备发起DMA请求时要访问的地址，也就是 `IOMMU`映射前的地址，`vaddr`就是 `mmap`的地址。`VFIO_IOMMU_MAP_DMA`命令会为虚拟地址 `vaddr`找到物理页并pin住（因为设备DMA是异步的，随时可能发生，物理页面不能交换出去），**然后找到 `Group`对应的 `Contex Entry`，建立页表项，页表项能够将 `iova`地址映射成上面pin住的物理页对应的物理地址上去**，这样对用户态程序完全屏蔽了物理地址，实现了用户空间驱动。IOVA地址的 `0 - 0x100000`对应DRAM地址 `0x10000000 - 0x10100000`，size为 `1024` * `1024`。一句话概述，`VFIO_IOMMU_MAP_DMA`这个命令就是将 `iova`通过 `IOMMU`映射到 `vaddr`对应的物理地址上去。

### 设备透传分析

**设备透传就是由虚机直接接管设备，虚机可以直接访问MMIO空间，VMM配置好IOMMU之后，设备DMA读写请求也无需VMM借入**，需要注意的是设备的配置空间没有透传，因为VMM已经配置好了BAR空间，如果将这部分空间也透传给虚机，虚机会对BAR空间再次配置，会导致设备无法正常工作。

### 虚机地址映射

在介绍透传之前，先看下虚机的GPA与HVA和HPA的关系，以及虚机是如何访问到真实的物理地址的，过程如下图。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230905220846.png)

一旦页表建立好后，整个映射过程都是硬件自动完成的，对于上图有如下几点说明：

1) 对于虚机内的页表，完成GVA到GPA的映射，虽然整个过程都是硬件自动完成，但有一点要注意下，在虚机的中各级页表也是存储在HPA中的，而CR3及各级页表中装的地址都是GPA，所以在访问页表时也需要借助EPT，上图中以虚线表示这个过程;
2) 利用虚机页表完成GVA到GPA的映射后，此时借助EPT实现GPA到HPA的映射，这里没有什么特殊的，就是一层层页表映射;
3) 看完上图，有没有发现少了点啥，是不是没有HVA。单从上图整个虚机寻址的映射过程来看，是不需要HVA借助的，硬件会自动完成GVA->GPA->HPA映射，那么HVA有什么用呢？这里从下面两方面来分析：
   + Qemu利用iotcl控制KVM实现EPT的映射，映射的过程中必然要申请物理页面。Qemu是应用程序，唯一可见的只是HVA，这时候又需要借助mmap了，Qemu会根据虚机的ram大小，即GPA大小范围，然后mmap出与之对应的大小，即HVA。通过KVM_SET_USER_MEMORY_REGION命令控制KVM，与这个命令一起传入的参数主要包括两个值，guest_phys_addr代表虚机GPA地址起始，userspace_addr代表上面mmap得到的首地址（HVA）。传入进去后，KVM就会为当前虚机GPA建立EPT映射表实现GPA->HPA，同时会为VMM建立HVA->HPA映射。
   + 当vm_exit发生时，VMM需要对异常进行处理，异常发生时VMM能够获取到GPA，有时VMM需要访问虚机GPA对应的HPA，VMM的映射和虚机的映射方式不同，是通过VMM完成HVA->HPA，且只能通过HVA才能访问HPA，这就需要VMM将GPA及HVA的对应关系维护起来，这个关系是Qemu维护的，这里先不管Qemu的具体实现（后面会有专门文档介绍），当前只需要知道给定一个虚机的GPA，虚机就能获取到GPA对应的HVA。

### 设备透传实现

在前面介绍VFIO的使用实例时，核心思想就是IOVA经过IOMMU映射出的物理地址与HVA经过MMU映射出的物理地址是同一个。对于设备透传的情况，先上图，然后看图说话。
![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230905221049.png)

+ [注]不是所有的虚拟化方案都有Qemu，仅仅是以Qemu+KVM方案举例。

![](https://raw.githubusercontent.com/ji92/markdown_picture/master/images/20230905221102.png)

先来分析一下设备的DMA透传的工作流程，**一旦设备透传给了虚机，虚机在配置设备DMA时直接使用GPA**。此时GPA经由EPT会映射成HPA1，GPA经由IOMMU映射的地址为HPA2，此时的HPA1和HPA2必须相等，设备的透传才有意义。下面介绍在配置IOMMU时如何保证HPA1和HPA2相等，在VFIO章节讲到了VFIO_IOMMU_MAP_DMA这个命令就是将iova通过IOMMU映射到vaddr对应的物理地址上去。对于IOMMU来讲，此时的GPA就是iova，我们知道GPA经由EPT会映射为HPA1，对于VMM来讲，这个HPA1对应的虚机地址为HVA，那样的话在传入VFIO_IOMMU_MAP_DMA命令时讲hva作为vaddr，IOMMU就会将GPA映射为HVA对应的物理地址及HPA1，即HPA1和HPA2相等。上述流程帮助理清整个映射关系，实际映射IOMMU的操作很简单，前面提到了qemu维护了GPA和HVA的关系，在映射IOMMU的时候也可以派上用场。注：IOMMU的映射在虚机启动时就已经建立好了，映射要涵盖整个GPA地址范围，同时虚机的HPA对应的物理页都不会交换出去（设备DMA交换是异步的）。

以网卡为例，配置虚机中的xml文件，指定host上的BDF号，再输入虚机中的网卡BDF号即可
如果是SR-IOV，驱动需要做对应的修改，增加vf驱动。驱动安装后，查询网口的时候自然会出现多个网口

# 模拟

# 参考资料

+ [深度剖析IOMMU与VFIO技术架构](https://zhuanlan.zhihu.com/p/550698319)
+ [虚拟化技术 — QEMU-KVM 基于内核的虚拟机](https://zhuanlan.zhihu.com/p/625828717)
+ [PCIe IOMMU 概述](https://zhuanlan.zhihu.com/p/348826888)
+ [VT-d 中断重映射分析](https://www.cnblogs.com/dream397/p/13628823.html)
+ [ARM SMMU原理与IOMMU技术](https://blog.csdn.net/Rong_Toa/article/details/108997226)
+ [虚拟化技术 - 内存虚拟化 [一]](https://zhuanlan.zhihu.com/p/69828213)
+ [虚拟化技术 - I/O虚拟化 [一]](https://zhuanlan.zhihu.com/p/69627614)
+ [虚拟化技术 - I/O虚拟化 [二]](https://zhuanlan.zhihu.com/p/75649223)
+ [再议 IOMMU](https://zhuanlan.zhihu.com/p/610416847)
+ [SR-IOV](https://www.pcietech.com/424.html/)
+ [Address Translation Services Specification](https://composter.com.ua/documents/ats_r1.1_26Jan09.pdf)
+ [Single Root I/O Virtualization and Sharing Specification Revision 1.1](https://composter.com.ua/documents/sr-iov1_1_20Jan10_cb.pdf)