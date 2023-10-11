# virtio
+ 虚拟机和宿主机通信，类似客户端
+ vhost是客户端，两边使用socket建立通信（不是走内核协议栈，通过文件实现的socket）

# vhost

# vring
+ 虚拟机磁盘落盘是通过共享内存（vring），而不是virtio/vhost
+ 网卡收发的内存也是通过共享内存
+ 共享内存数据如何组织：
+ 

# 参考资料
[Vhost-user Protocol](https://qemu-project.gitlab.io/qemu/interop/vhost-user.html)
[Virtual i/o Device(VIRTIO) Version 1.2](https://docs.oasis-open.org/virtio/virtio/v1.2/csd01/virtio-v1.2-csd01.html)
