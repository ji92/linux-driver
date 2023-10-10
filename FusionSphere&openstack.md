# FusionSphere
+ FusionSphere其实就是虚拟化层（hyperviser），对下面做资源池化，对上面做虚拟机的创建和管理
+ FusionSphere底层是基于KVM实现的
+ 虚拟机创建有手动创建、`模板创建`、虚拟机克隆等方式
+ 虚拟创建后，host上对应目录下会产生磁盘文件，磁盘文件是可以迁移的。
## 使用模板部署虚拟机
+ 手动安装效率低，批量部署使用模板
+ 模板是虚拟机的一个副本，包含os，应用软件和虚拟机规格配置，使用模板可以大幅节省配置新虚拟机和安装os的时间
+ 使用系统内已有的虚拟机制作模板，可以创建与模板设置一致的虚拟机，实现快速部署的功能
+ 制作模板的方式：
  + 虚拟机转为模板
  + 虚拟机克隆为模板
  + 模板克隆为模板
+ 详见FusionSphere产品文档
+ 虚拟机镜像(img):只用于虚拟机
+ os镜像(iso)：虚拟机和物理机都可以用
## FusionSphere：PCI直通到虚拟机
[FS6.0 PCI直通虚拟机](http://w3.huawei.com/unisearch/index.html?keyword=pci%E7%9B%B4%E9%80%9A%E8%99%9A%E6%8B%9F%E6%9C%BA%E9%85%8D%E7%BD%AE&lang=zh#lang=zh&newKeyword=pci%E7%9B%B4%E9%80%9A%E8%99%9A%E6%8B%9F%E6%9C%BA%E9%85%8D%E7%BD%AE)
## 虚拟机迁移
+ HA保证，虚拟迁移后关闭物理主机
## 快照技术
+ 快照恢复
## 虚拟机克隆
## vCPU、内存在线调整
## 虚拟换在线扩容

# 参考资料
[华为FusionSphere之虚拟机和模版---后半段](https://www.bilibili.com/video/BV1pt411j7pg/?spm_id_from=333.337.search-card.all.click&vd_source=00c7bb189a105f317a347bc7d83911b5)

[FusionSphere openstack](https://www.bilibili.com/video/BV17b411L7iL/?spm_id_from=333.337.search-card.all.click&vd_source=00c7bb189a105f317a347bc7d83911b5)

[FusionSphere产品文档](https://support.huawei.com/hedex/hdx.do?docid=EDOC1100092091&id=ZH-CN_TOPIC_0239789907)

[PCI Passthrough（PCI直通）虚拟机的配置方法](http://3ms.huawei.com/km/blogs/details/5984789)
