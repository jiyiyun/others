虚拟化架构
---

寄居虚拟化 
- 优点：简单，易于实现
- 缺点：安装和运行应用程序依赖于主机操作系统对设备的支持；管理开销大，性能损耗大
- 厂家：VMware WorkStation

裸金属虚拟化
- 优点：虚拟机不依赖于操作系统；支持多种操作系统 多种应用
- 缺点：虚拟层内核开发难度大
- 厂家：VMware ESXServer、Ctrix XenServer、华为FusionSphere

操作系统虚拟化 
- 优点：简单易于实现；管理开销低
- 缺点：隔离性差，多容器共享同一操作系统
- 厂家：virtuozzo

混合虚拟化
- 优点：相对于寄居虚拟化架构没有冗余性能高；可支持多种操作系统
- 缺点：需底层硬件支持虚拟化拓展功能
- 厂家：Readhat KVM

寄居虚拟化：寄居虚拟化架构指在宿主操作系统之上安装和运行虚拟化程序，依赖于宿主操作系统对设备的支持和物理资源管理。虚拟化管理软件作为底层操作系统(Windows或者Linux等)上的一个普通应用程序，然后通过其创建相应的虚拟机，共享底层服务器资源。也可以理解为宿主操作系统之上安装和运行虚拟化程序，依赖于宿主操作系统对设备的支持和物理资源的管理。

裸金属虚拟化：裸金属虚拟化架构指直接在硬件上面安装虚拟化软件，再在其上安装操作系统和应用，依赖于虚拟层内核和服务器控制台进行管理。Hypervisor是指直接运行与物理硬件上的虚拟机监控程序。它主要实现两个基本功能：首先是识别、捕获、响应虚拟机发出的CPU特权指令或保护指令；其次，它负责处理虚拟机队列和调度并将物理硬件的处理结果返回给相应的虚拟机

华为格式的统一虚拟化平台使用的是裸金属虚拟化结构

操作系统虚拟化：操作系统虚拟化架构在操作系统层面增加虚拟服务器功能。操作系统虚拟化架构把单个的操作系统划分为多个容器，使用容器管理器来进行管理。宿主操作系统负责在多个虚拟服务器(即容器)之间分配硬件资源，并且让这些服务器彼此独立

没有独立的hypervisor层，一个明显区别是，如果使用操作系统虚拟化，所有虚拟服务器必须运行同一操作系统，(不过每个实例有各自的应用程序和用户账户)

混合虚拟化：将一个内核级驱动器插入到宿主操作系统内核。这个驱动器作为虚拟硬件管理器来协调虚拟机和操作系统之间的硬件访问。混合虚拟化需要底层硬件支持虚拟化扩展功能(Intel-VT，AMD-V)。


根据Hypervisor的实现方式和所处位置
```txt
 1型                    2型
GuestOS  GuestOS     |  GuestOS  GuestOS
Hypervisor(ESXi Xen) |  Hypervisor(KVM)
                     |  Linux OS
                     |
Server Hardware      |  Server Hardware
```
裸机型Hypervisor  Bare-metal(业界有时候称为Type 1 Hypervisor):最常见，直接安装在硬件计算资源上，直接管理和调用硬件资源，不需要底层操作系统，可以理解为Hypervisor被做成了一个很薄的操作系统。操作系统安装并且运行在Hypervisor之上，主流虚拟化产品都使用Hypervisor,其中包括VMware ESX Server、Microsoft Hyper-V 、Ctrix XenServer,此种方案的性能出于主机虚拟化和操作系统虚拟化之间。

主机型的Hypervisor Hosted(业界有时候称为Type2 Hypervisor):也有一些这样的Hypervisor可以内嵌在硬件计算资源的固件套装中--和主板BIOS位于同一级别。也就是跑在操作系统上的应用软件。托管型/主机型的Hypervisor运行在基础操作系统之上，构建出一整套虚拟硬件平台(CPU/Memory/Storage/Adapter)，使用者根据需要安装新的操作系统和应用软件。低层和上层的操作系统可以完全无关化，如windows运行Liunx系统。主机虚拟化中VM的应用程序调用硬件资源时需要经过VM内核-->Hypervisor-->主机内核，因此相对来说，性能是虚拟化中最差的，与使用这种方式的有Hitachi Virtage、VMware ESXi和Linux KVM--基于内核的虚拟机。而宿主型的Hypervisor是运行在操作系统内部的应用程序，其它操作系统和应用程序运行在VMware Server和Microsoft Virtual Server之上，以及其它基于终端的虚拟化平台，诸如VMware Workstation、Microsoft Virtual PC和Parallels Workstation,这些都是宿主型的Hypervisor

1型虚拟化：Hypervisor直接安装在物理机上，多个虚拟机在Hypervisor上运行，Hypervisor实现方式一般是一个特殊定制的Linux系统，Xen和VMware ESXi属于这个类型

2型虚拟化：物理机上首先安装常规的操作系统，Hypervisor作为OS上的一个程序模块运行，并对管理虚拟机进行管理。KVM 、VirtualBox、VMware Workstation都属于这个类型

理论上讲：1型虚拟化一般对虚拟化进行了特定优化，性能比2型高；2型虚拟化因为基于普通操作系统，比较灵活，支持虚拟嵌套，嵌套意味着可以在KVM虚拟机中再运行KVM

虚拟机与VMM
---

虚拟机(Virtual Machine)是由虚拟化层提供高效、独立的虚拟计算机系统，其拥有自己的虚拟硬件(CPU,内存，IO设备)。

通过虚拟化层的模拟，虚拟机在上层软件看来，其就是一个真实在机器。这个虚拟化层一般称为虚拟机监控器(Virtual Machine Monitor VMM)也称为Hypervisor

VM

1. 虚拟资源  VMM利用底层硬件资源来构建一个包含虚拟CPU、内存和外设等的虚拟环境。在这个环境中，GuestOS认为自己运行在一台真实的计算机上，并唯一拥有这台“虚拟”机器上所有资源

2. 虚拟环境的调度  VMM可以同时构建多个虚拟机环境，从而允许多个GuestOS并非执行，VMM利用一套策略来有效的调度资源

3. 虚拟化环境的管理接口  VMM提供一组完备的管理接口，来支持虚拟环境的创建、删除、暂停和迁移等功能。上层的管理程序通过调用VMM提供的管理接口，为用户提供管理界面

KVM架构 VS Xen架构
---

KVM  
- 内核模块，使内核称为Hypervisor(呈现给用户空间字符设备/dev/kvm;用户空间通过ioctl访问)
- Guest是一个普通的进程（vcpu是一个线程）
- 充分利用linux内核支持(调度，内存共享，Qos，电源管理等)
- 仅支持全虚拟化(Intel VT/AMD-V)

Xen
- 轻量级Hypervisor(直接运行在硬件上，CPU虚拟化，内存虚拟化)
- 特权虚拟机Domain0(IO虚拟化，虚拟机管理)
- 支持全虚拟化+半虚拟化(非硬件虚拟化平台PV Guest)

KVM是一个独特的管理程序，让Linux内核自身变成一个管理程序，KVM作为一个内核模块实现，在虚拟环境下Linux内核集成管理程序将其作为一个可加载的模块，可以简化管理及提升性能。KVM使用标准Linux调度程序、内存和其它服务。将虚拟技术建立在内核上而不是去替换内核。

Xen作为优秀的半虚拟化引擎，在基于硬件的虚拟化帮助下，现在也完全支持虚拟化MS Windows,被设计成一个独立的内核，它只需要linux执行I/O。这样使得它非常大，并且有自己的调度程序、内存管理器、计时器和机器初始化程序。Domain 0(特权虚拟机)是其它虚拟机的管理者和控制者，可以构建更多的Domain,并管理虚拟设备，它还能执行管理任务(比如虚拟机休眠、唤醒和迁移其它虚拟机)。此外还有个Domain U,这个是指除了Domain 0之外的普通虚拟机。

Xen 和KVM架构各有所长
---

- Xen平台架构侧重安全性  为保证安全性，各Domain之间对共享区域的访问和映射必须通过Hypervisor授权(Xen以及针对上述问题有改进优化方案)
- KVM平台架构侧重性能  VM之间以及与Host Kernel之间对共享区域的访问和映射无需Hypervisor授权，故整个访问路径较短，使用Linux baremetal内核，无pvops性能损耗。

```txt
User-space(applications)
Guest OS(Virtual Machine)  QEMU
/dev/kvm   Hypervisor(Virtual machine monitor)
Hardware

guest:客户系统(包括vCPU、内存、驱动(Console、网卡、I/O设备驱动等))被KVM置于一种受限的CPU模式下运行
KVM:运行在内核空间，提供CPU和内存虚拟化，以及客户机的I/O拦截，Guest的I/O被KVM拦截后，交给QEMU处理。
QEMU:修改过的为KVM虚拟机使用的QEMU代码，运行在用户空间，提供硬件I/O虚拟化。通过IOCTL /dev/kvm设备和KVM交互。
```

- 在x86平台运用最广泛的虚拟化方案莫过于KVM了，OpenStack对其支持也最好
- KVM全称Kernel-based Virtual Machine,也就是说KVM是嵌入在Linux操作系统内核中的一个虚拟化模块，是基于Linux内核实现的。KVM有一个内核模块叫kvm.ko只用于管理虚拟CPU和内存
- KVM能够将一个Linux标准内核转换成为一个VMM,嵌有KVM模块的Linux标准内核可以支持通过KVM tools来进行加载GuestOS,所以在这样的操作系统平台下，计算机物理硬件层上直接就是VMM虚拟化层，而没有独立出来的HostOS操作系统层，在这样的环境中Host OS就是一个VMM。Guest OS的指令不用再经过Qemu转译，直接运行，大大提高了速度，KVM通过/dev/kvm暴露接口，用户态程序可以通过ioctl函数来访问这个接口
- KVM内核模块本身只能提供CPU和内存的虚拟化，所以它必须结合QEMU才能构成一个完整的虚拟化技术，这就是qemu-kvm
- 作为一个Hypervisor,KVM本身只关注虚拟机调度和内存管理这两个方面，IO外设的任务交给Linux内核和Qemu,IO的虚拟化，比如存储和网络设备就交给Linux内核和Qemu来实现
- Qemu将KVM整合进来，通过ioctl调用/dev/kvm接口，将有关CPU指令的部分交由内核模块来做。KVM负责CPU虚拟化+内存虚拟化，实现了CPU和内存虚拟化，但KVM不能模拟其它设备，Qemu模拟IO设备(网卡，磁盘等)，KVM+Qemu才是真正意义上的服务器虚拟化，Qemu是一个模拟器，它向Guest OS模拟CPU和其它硬件，Guest OS认为直接和硬件直接打交道，其实是Qemu模拟出的硬件打交道，Qemu将这些指令转译给真正的硬件。由于所有指令要Qemu转译，因此性能比较差

Qemu模拟其它硬件，如Network,disk,同样会影响这些设备的性能，于是产生了pass through半虚拟化设备virtio_blk,virtio_net,提高设备性能

XEN
---

```txt
Linux Domain0   Linux application
Xen
Hardware

Xen Hypervisor:直接运行在硬件上，是Xen客户操作系统与硬件资源之间的访问接口，通过将客户操作系统与硬件进行分类，Xen管理系统可以允许客户操作系统安全、独立的运行在相同硬件环境之上。
Domain 0:运行在Xen管理程序之上，具有直接访问硬件和管理其它操作系统的特权的客户操作系统
Domain U:运行在Xen管理程序之上的普通客户操作系统或业务操作系统，不能直接访问硬件资源，但可以独立并行存在多个。
```
Xen的VMM(Xen Hypervisor)位于操作系统和硬件之间，负责为上层的操作系统内核提供虚拟化的硬件资源，负责管理和分配这些资源，并且确保上层虚拟机(Domain)之间相互隔离，Xen采用混合模式，因而设定了一个特权域用以辅助Xen管理其它的域，并提供虚拟化的资源服务，特权域称为Domain 0,其余域称为Domain U

```txt
Domain 0              |  VM1                 | VM2           |  VM3
设备管理与控制软件       |未修改的应用程序         |未修改的应用程序 |未修改的应用程序 
GuestOS(XenLinux)     |GuestOS               |GuestOS        |运行在支持硬件虚拟化的处理器
后端驱动 原生设备驱动程序 |后端驱动 原生设备驱动程序 |SMP 前端驱动程序 |前端驱动程序
Xen虚拟机监视器(Xen VMM) 控制接口、硬件安全访问接口、事件通道、虚拟处理器、虚拟内存管理单元
Hardware
```
- Xen通过Hypervisor软件访问物理层硬件，实现在一台单独的计算机上运行多个各自独立、彼此隔离的子操作系统。Hypervisor指挥硬件访问和协调来自各子系统的请求
- Xen环境中，主要是虚拟机控制器(VMM),也叫Hypervisor,Hypervisor层硬件位于虚拟机之间，是最先被载入硬件的第一层。Hypervisor载入就可以部署虚拟机。在Xen中的Domain(域)由Xen控制，可高效的利用CPU资源，Domain可以分为Domain 0 和Domain U
- Domian 0 :属于特权Domain,在虚拟机中，Domain 0有很高的特权，负责一些专门的工作，并提供虚拟的资源服务，由于Hypervisor中不包含任何与硬件对话的驱动，也没有与管理对话的接口，这些驱动就由Domain 0完成。
- Domain U 属于无特权Domain，管理员利用Xen工具通过Domain 0创建其它虚拟机Domain 1,Domain ...
- Domain 0是其它虚拟主机的管理者和控制者，Domain0可以构建其它Domain并管理虚拟设备

Xen架构简介
---

Domain 内部包含了真正的驱动设备，可以直接访问物理硬件，负责与Xen提供的管理API交互，并通过用户模式下的管理工具来管理Xen的虚拟机环境。

- 控制接口 控制接口仅能被Domain0 使用，用于帮助Domain 0控制管理其它域，控制接口提供的具体功能包括Domain的创建、销毁、暂停、恢复、迁移，以及对其它Domain的CPU调度、内存分配以及设备访问
- 安全硬件接口  提供除虚拟机CPU和MMU之外的所有硬件虚拟化工作，包括DMA/IO、驱动程序、虚拟的PCI地址配置、虚拟硬件中断等，该接口只具有原生设备驱动的Domain使用，而其它Domian则仅能通过设备通道提供虚拟硬件服务
- 事件通道  事件通道是子域通信的事件提示机制。事件通道用于Domain和Xen之间、Domain相互之间的一种异步事件通知机制，用于处理GuestOS中虚拟中断、物理中断、以及Domain之间的通信
- 虚拟处理器  vCPU为每个Domain建立了vCPU结构，用于接收GuestOS中传递的指令，其中大部分的指令被vCPU直接交到物理CPU执行，而对于特权指令则需要经过确认和提交Xen代为执行
- 虚拟内存管理单元  虚拟MMU用于帮助GuestOS完成虚拟地址到机器地址的转换。Xen系统中增加了客户物理地址层，因而地址由原来的二层结构变为三层结构。Xen通过虚拟的MMU仍能使用硬件MMU来完成地址转换

