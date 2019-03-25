虚拟化virtualizations 是一种资源管理技术，将计算机各种实体资源(如服务器、网络、存储、内存等)，予以抽象、转换后呈现出来。打破实体结构间的不可切割障碍。使用户可以比原本更好的方式来应用这些资源。

虚拟化分类

根据虚拟化层是通过纯软件的方法，还是利用物理资源提供的机制来实现这种“截获并重定向”，分为软件虚拟化和硬件虚拟化，纯软件虚拟化：所有指令都是纯软件模拟的，例如早期QEMU/VMware workstation

根据是否改动操作系统，虚拟化分为全虚拟化和准虚拟化/半虚拟化
全虚拟化：提供完整的操作系统。例如基于硬件辅助的KVM
半虚拟化:改动guest操作系统来与VMM协调，例如XEN(特点:guest知道自己在虚拟化环境中，是通过修改操作系统实现的，需要安装特定的驱动)

Hypervisor

Hypervisor类型主要有2类：
1. 宿主型 实现虚拟化的这个软件需要安装在一个操作系统上，它本身没有管理硬件的功能，而操作系统有，因此依赖于操作系统，例如KVM
2. 裸机型 实现虚拟化的这个软件不需要安装在操作系统上，它本身就有管理硬件的功能，例如EXSi

KVM Kernel-base virtual Machine

KVM虚拟化技术的前提
1. 基于内核kvm模块，linux 内核2.6.34以后开始有kvm模块，查看lsmod |grep kvm
2. CPU支持,Intel VT,AMD-V
3. 相关软件,例如virt-manager 、libvirtd、virsh

为什么要打开VT,因为KVM是全虚拟化能模拟出的完整操作系统，不需要改动，并且结合硬件辅助，在虚拟机中的速度会更快，操作虚拟机和真机一样

带有虚拟技术的处理器具有额外的指令集，Virtual Machine Extensions 简称VMX，cat /proc/cpuinfo |grep -E 'vmx|svm'

X86服务器有4个特权级别Ring0 ~ Ring3

只有运行在Ring 0 ~ 2级时，处理器才可以访问特权资源或执行特权指令，运行在Ring 0级时，处理器可以访问所有的特权状态，x86平台的操作系统一般只使用Ring0和Ring3这两个级别，操作系统运行在Ring0级，用户运行在Ring3级，同时为了避免Guest OS控制系统资源，为了满足第一个充分条件-GuestOS运行在Ring1或者Ring3级(Ring2不使用)

```txt
qemu-img create -f qcow2 /var/lib/libvirt/images/vm1.qcow2 10G
chown qemu:qemu  /var/lib/libvirt/images/vm1.qcow2
yum install -y *vnc*
yum install lsof
virt-install --name=vm1 --disk path=/var/lib/libvirt/images/vm1.qcow2 --graphics vnc,listen=192.168.5.120 --vcpus=2 --ram=2048 --location=/mount/rhel-server-7.0.iso

lsof -i:5900

yum install virt-manager
```

KVM只是模块，它可以和硬件打交道，和用户打交道的是qemu-kvm,这个工具将用户操作交给KVM模块

vCPU在KVM中三种模式
```txt
----Userspace------|--------Kernel---------|------Guest----------------|
 |-----ioctl()----------------|
 |                            |
 |                   |-Swich to Guest mode-----------|
 |                   |                         Native Guest execution
 |                   kernel exit Handler<------------|
 |-Userspace exit Handler<--------|
```

SMP系统(Symmetric Multi-Processor对称多处理器)是指在一个计算机上汇集了一组处理器(多CPU)，各CPU之间共享子系统以及总线架构，在SMP系统中，多个程序(进程)可以做到真正的并行，而且单个进程的多个线程也可以并行执行，提高了性能
```txt
rich@R:~$ uname -a
Linux R 4.15.0-29deepin-generic #31 SMP Fri Jul 27 07:12:08 UTC 2018 x86_64 GNU/Linux
```

内存气泡
---

我们为什么可以动态调整内存呢？因为有内存气泡，那么什么是内存气泡？它是通过virtio_balloon驱动来实现宿主机Hypervisor和客户机之间的协作

使用的时候，虚拟机安装virt balloon的驱动，内核开启CONFIG_VIRTIO_BALLOON，RHEL7默认已开启，并且默认已经安装virt balloon驱动

气泡里面为: 物理机可使用的内存，也就是说当虚拟机内存被回收后，会到气泡里，然后物理机再分配给其它进程。

在虚拟机中查看virt balloon，可以看到virtio_balloon模块，如果没有这个模块，我们无法动态调整内存
```txt
#lsmod |grep virt
virtio_balloon
```

共享页
---

宿主机内存压缩主要采用KSM(Kernel SamePage Merging)技术，原理和压缩类似，就是将相同的内存页进行合并，如果内存发生变化，则将内存单独复制出来独立运行

KSM原理： KSM作为内核守护进程(ksmd)存在，它定期执行内存页面扫描，识别副本页面并合并副本，释放这些页面供其它用，因此在多个进程中，Linux将内核相似的内存页合并成为一个内存页，提高内存使用效率，由于内存是共享的，所以多个虚拟机使用的内存就少了。这个特性对于虚拟机使用相同镜像和操作系统时，效果更加明显，但是用这个特性增加了内核开销，用时间换空间，如果追求效率则可以将这个特性关闭。


```txt
systemctl status ksm
systemctl status ksmtuned
如果不想使用该服务，关闭该服务即可

root@R:~# ls /sys/kernel/mm/ksm/
full_scans	    pages_to_scan    stable_node_chains
max_page_sharing    pages_unshared   stable_node_chains_prune_millisecs
merge_across_nodes  pages_volatile   stable_node_dups
pages_shared	    run		     use_zero_pages
pages_sharing	    sleep_millisecs
root@R:~# cd /sys/kernel/mm/ksm/
root@R:/sys/kernel/mm/ksm# cat *
0
256
1
0
0
100
0
0
0
20
0
2000

```


