# dcoker
+ docker只是一个管理工具，方便容器的增删改查，容器技术本来是linux支持的
+ linux容器技术，是基于namespace/cgroup特性实现的，每个容器有独立的进程空间、网络资源和文件系统
+ 容器本质还是个进程，内部封装了业务进程（如nginx），提供独立的运行环境
+ 容器的网络访问，通过网桥
+ `传统无容器化部署`：开发交付件是源码，开发测试运维的环境一致性问题，部署效率低，交付件由github/gitee等源码托管平台管理。
+ `虚机化部署`：开发交付完整镜像（包含源码、依赖、内核、发行版），非常厚重
+ `容器化部署`：开发交付容器镜像（包含源码、依赖和发行版，内核是和宿主机公用的），较虚机化部署轻量化，部署效率高，镜像由docker镜像仓库统一管理。
+ devops部署流水线，实现环境一致性

# K8S
+ K8S是个容器编排系统，解决复杂场景下容器的管理问题
+ K8S是master-slave模式，master和slave运行在不同node上面(node可以是一个虚拟机，也可以是一个物理机)。
+ 运行master的是控制节点，运行slave的是数据节点，在生产环境为了实现高可用可能会有两个控制节点，一主一备。
+ master负责整体的管理，包括容器创建、负载均衡、容器热迁移、容器删除等等。slave上跑一个个容器。
+ K8S有很多组件，包括scheduler、ETCD、contriller-management、api-server、kube-proxy、kubelet、pod等
+ K8S以pod为单位进行容器管理，一个pod内是一组相同或者紧密相关的容器
+ K8S对下还提供硬件资源的发现、管理、调度等，对于不识别的硬件，需要使用device plugin，进行硬件资源的发现和管理。这个插件本质上也是一个容器

# 参考资料
[最新docker入门教程 入门到精通一套教程就够了 入门必看！](https://www.bilibili.com/video/BV1xs4y1T7Ns?p=1&vd_source=00c7bb189a105f317a347bc7d83911b5)

[2023最新k8s教程 大佬超详细讲解 通俗易懂 小白必备！](https://www.bilibili.com/video/BV1Uo4y1T7qF/?p=8&spm_id_from=pageDriver&vd_source=00c7bb189a105f317a347bc7d83911b5)

[docker从入门到实践](https://yeasy.gitbook.io/docker_practice/)
