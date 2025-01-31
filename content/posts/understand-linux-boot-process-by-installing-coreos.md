---
title: "通过安装 CoreOS 系统了解 Linux 启动流程"
date: 2021-10-20T00:25:25+08:00
draft: false
tags: ["OS", "Linux", "CoreOs"]
summary: "OpenShift 4.X 版本要求安装在操作系统为 CoreOS 的机器上，因此官方文档给出了使用 PXE 或 IPXE 引导 CoreOS 系统的方法。我们可以参考其操作流程，将一台 CentOS 7.X 的机器改写为 CoreOS 系统，步骤如下 ..."
---

## 前言

OpenShift 4.X 版本要求安装在操作系统为 CoreOS 的机器上，因此 [官方文档](https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html#installation-user-infra-machines-pxe_installing-restricted-networks-bare-metal) 给出了使用 PXE 或 IPXE 引导 CoreOS 系统的方法。我们可以参考其操作流程，将一台 CentOS 7.X 的机器改写为 CoreOS 系统，步骤如下：

1. 从 [镜像下载页](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.6/?extIdCarryOver=true&sc_cid=701f2000001Css5AAC) 下载安装所需版本的 kernel、initramfs 和 rootfs 文件，并将 rootfs 和点火文件（*.ign）上传到自建的 HTTP 服务器上；

2. 将 kernel 和 initramfs 文件拷贝到 CentOS 7.X 机器的 /boot 目录下；

3. 根据需求修改 /boot/grub2 目录下的 grub.cfg 文件；

4. 重启机器。

对于操作系统初学者（比如我）来说，很难想象仅依靠添加和修改文件就能改变一台计算机的操作系统。为了解其实现原理，我们将对 Linux 的启动流程进行讨论，并从中说明上述操作是如何影响操作系统的。

## Linux 启动流程

启动一台 Linux 机器的过程可以分为两个部分：Boot 和 Startup。其中，Boot 起始于计算机启动，在内核初始化完成且 systemd 进程开始加载后结束。紧接着， Startup 接管任务，使计算机达到一个用户可操作的状态。

![202110171642](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/202110171642.jpeg)

## Boot 阶段

如上图所示，Boot 阶段又可以细分为三个部分：

- BIOS POST
- Boot Loader
- 内核初始化

### BIOS POST

开机自检（Power On Self Test，POST）是 [基本输入输出系统](https://zh.wikipedia.org/wiki/BIOS)（Basic I/O System，BIOS）的一部分，也是启动 Linux 机器的第一个步骤。其工作对象是计算机硬件，因此对于任何操作系统都是相同的。**POST 检查硬件的基本可操作性**，若失败则 Boot 过程将会被终止。

POST 检查完毕后会发出一个 BIOS 中断调用 [INT 13H](https://en.wikipedia.org/wiki/INT_13H)，它将在任何可连接且可引导的磁盘上搜索含有有效引导记录的引导扇区（Boot Sector），通常是 [主引导扇区](https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%BC%95%E5%AF%BC%E8%AE%B0%E5%BD%95)。引导扇区中的主引导记录（Master Boot Record，MBR）将被加载到 RAM 中，然后控制权就会转移到其手中。

### Boot Loader

大多数 Linux 发行版使用三种 Boot Loader 程序：GRUB1、GRUB2 和 LILO，其中 GRUB2 是最新且使用最为广泛的。GRUB2 代表“GRand Unified Bootloader, version 2”，**它能够定位操作系统内核并将其加载到内存中**。GRUB2 还允许用户选择从几种不同的内核中引导计算机，如果更新的内核版本出现兼容性问题，我们就可以恢复到先前内核版本。

GRUB1 的引导过程可以分为三个阶段：stage 1、stage 1.5 和 stage 2。虽然 GRUB2 中并没有 stage 的概念，但两者的工作方式基本相同。为了方便说明，我们在讨论 GRUB2 时将沿用 GRUB1 中 stage 的说法。

#### stage 1

上文提到，BIOS 中断调用会定位主引导扇区，其结构如下图所示：

![20211015161614](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20211015161614.png)

主引导记录首部的引导代码便是 stage 1 文件 boot.img，它和 stage 1.5 文件 core.img 均位于 /boot/grub2/i386-pc 目录下：

```shell
[root@bastion ~]# du -b /boot/grub2/i386-pc/*.img 
512     /boot/grub2/i386-pc/boot.img
26664   /boot/grub2/i386-pc/core.img
```

它的作用是检查分区表是否正确，然后**定位和加载 stage 1.5** 文件。446 字节的 boot.img 放不下能够识别文件系统的代码，只能通过计算扇区的偏移量来寻找，因此 core.img 必须位于主引导记录和驱动器的第一个分区（partition）之间。第一个分区从扇区 63 开始，与位于扇区 0 的主引导记录之间有 62 个扇区（每个 512 字节），有足够的空间存储大小不足 30000 字节的 core.img 文件。当 core.img 文件加载到 RAM 后，控制权也随之转移。

#### stage 1.5

相比于只能读取原始扇区的 LILO，GRUB1 和 GRUB2 均可识别文件系统，这依赖于 stage 1.5 文件中内置的文件系统驱动程序。如果你拥有一台仍然使用 GRUB1 引导的 CentOS 6.X 机器，那么便可以在 /boot/grub/ 目录下找到这些适配不同文件系统的 stage 1.5 文件：

```shell
[root@centos6.5 ~]# du -b /boot/grub/* | grep stage1_5
13380    /boot/grub/e2fs_stage1_5
12620    /boot/grub/fat_stage1_5
11748    /boot/grub/ffs_stage1_5
11756    /boot/grub/iso9660_stage1_5
13268    /boot/grub/jfs_stage1_5
11956    /boot/grub/minix_stage1_5
14412    /boot/grub/reiserfs_stage1_5
12024    /boot/grub/ufs2_stage1_5
11364    /boot/grub/vstafs_stage1_5
13964    /boot/grub/xfs_stage1_5
```

GRUB2 中的 core.img 不仅整合了上述文件系统驱动，还新增了菜单处理等模块，这也是其优于 GRUB1 的地方。我们可以在 [GNU GRUB Manual 2.06: Images](https://www.gnu.org/software/grub/manual/grub/html_node/Images.html#Images) 中找到对各种 GRUB 镜像文件的详细介绍。

既然 core.img 文件可以识别文件系统，那么它就能够根据安装时确定的系统路径**定位和加载 stage 2** 文件。同样，当 stage 2 文件加载到 RAM 后，控制权也随之转移。

#### stage 2

stage 2 文件并非是一个 .img 的镜像，而是一些运行时内核模块：

```shell
[root@bastion ~]# ls /boot/grub2/i386-pc/ | grep .mod | head
acpi.mod
adler32.mod
affs.mod
afs.mod
ahci.mod
all_video.mod
aout.mod
appendedsig.mod
appended_signature_test.mod
archelp.mod
```

它们的任务是根据 grub.cfg 文件的配置**定位和加载内核文件**，然后将控制权转交给 Linux 内核。grub.cfg 文件存放在 /boot/grub2 目录下：

```shell
[root@bastion ~]# head /boot/grub2/grub.cfg -n 5
#
# DO NOT EDIT THIS FILE
#
# It is automatically generated by grub2-mkconfig using templates
# from /etc/grub.d and settings from /etc/default/grub
```

通过该文件的注释我们可以知道，它实际上是由 grub2-mkconfig 命令使用 /etc/grub.d 目录下的一些模板文件并根据 /etc/default/grub 文件中的设置生成的：

```shell
[root@bastion ~]# ls /etc/grub.d/
00_header  00_tuned  01_users  10_linux  20_linux_xen  20_ppc_terminfo  30_os-prober  40_custom  41_custom  README
```

40_custom 和 41_custom 文件常用于用户对 GRUB2 配置的修改，实际上我们对机器的操作也是从这里开始的。为了让 GRUB2 在机器启动时选择 CoreOS 系统内核而非默认的 CentOS，需要在原始 40_custom 文件末尾添加如下内容：

```shell
menuentry 'coreos' {
        set root='hd0,msdos1'
        linux16 /rhcos-live-kernel-x86_64 coreos.inst=yes coreos.inst.install_dev=vda rd.neednet=1 console=tty0 console=ttyS0 coreos.live.rootfs_url=http://{{HTTP-Server-Path}}/rhcos-live-rootfs.x86_64.img coreos.inst.ignition_url=http://{{HTTP-Server-Path}}/master.ign ip=dhcp
        initrd16 /rhcos-live-initramfs.x86_64.img
}
```

所示的 Menuentry 由三条 Shell 命令组成：

- `set root='hd0,msdos1'`
- `linux16 /rhcos-live-kernel-x86_64 ...`
- `initrd16 /rhcos-live-initramfs.x86_64.img`

第一条命令指定了 GRUB2 的根目录，也就是 /boot 所在分区在计算机硬件上的位置。既然我们已经将内核文件拷贝到了 /boot 目录下，那么能够识别文件系统的 GRUB2 便可以定位和加载它。本例中`hd`代表硬盘（hard drive），`0`代表第一块硬盘，`mosdos`代表分区格式，`1` 代表第一个分区。详细的硬件命名规范见 [Naming Convention](https://www.gnu.org/software/grub/manual/grub/grub.html#Naming-convention)。

第二条命令将从`rhcos-live-kernel-x86_64`（CoreOS 系统的内核文件）中以 16 位模式加载 Linux 内核映像，并通过`coreos.live.rootfs_url`和`coreos.inst.ignition_url`参数指定根文件系统（rootfs）的镜像文件和点火文件的下载链接。`ip=dhcp`代表该计算机网络将由 DHCP 服务器动态配置，也可以按`ip={{HostIP}}::{{Gateway}}:{{Genmask}}:{{Hostname}}::none nameserver={{DNSServer}}`的格式写入静态配置。

第三条命令将从`rhcos-live-initramfs.x86_64.img`中加载 RAM Filesystem。GRUB2 读取的内核文件实际上只包含了内核的核心模块，缺少硬件驱动模块的它无法完成 rootfs 的挂载。然而这些硬件驱动模块位于 /lib/modules/$(uname -r)/kernel/ 目录下，必须在 rootfs 挂载完毕后才能被识别和加载。为了解决这一问题，initramfs（前身为 initrd）应运而生。它是一个包含了必要驱动模块的临时 rootfs，内核可以从中加载所需的驱动程序。待真正的 rootfs 挂载完毕后，它便会从内存中移除。

除此之外我们还需要将 /etc/default/grub 文件中的 [GRUB_DEFAULT=saved](https://www.gnu.org/software/grub/manual/grub/grub.html#Simple-configuration) 修改为 GRUB_DEFAULT="coreos"，使其与 40_custom 文件中的`menuentry 'coreos'`对应。最后使用命令`grub2-mkconfig -o /boot/grub2/grub.cfg`来重新生成一份 grub.cfg 文件，这样计算机重启后 GRUB2 就会根据我们的配置去加载 CoreOS 系统的内核了。

至此我们已经明白了为什么“仅依靠添加和修改文件就能改变一台计算机的操作系统”，但计算机想要达到用户可操作状态还远不止于此。让我们再来看看内核被加载到内存后发生了什么。

### 内核初始化

不同内核及其相关文件位于 /boot 目录中，均以 vmlinuz 开头：

```shell
[root@bastion ~]# ls /boot/ | grep vmlinuz
vmlinuz-0-rescue-20210623110808105647395700239158
vmlinuz-4.18.0-305.12.1.el8_4.x86_64
vmlinuz-4.18.0-305.3.1.el8.x86_64
```

内核通过压缩自身来节省存储空间，所以当选定的内核被加载到内存中后，它首先需要进行解压缩（extracting）。一旦解压完成，内核便会开始**加载 systemd 并将控制权移交给它**。

## Startup 阶段

systemd 是所有进程之父，它负责**使计算机达到可以完成生产工作的状态**。其功能比过去的 init 程序要丰富得多，包括挂载文件系统、启动和管理计算机所需的系统服务。当然你也可以将一些应用（如 Docker）以 systemd 的方式启动，但它们与 Linux 的启动无关，因此不在本文的讨论范围之内。

首先，systemd 根据 /etc/fstab 文件中的配置挂载文件系统。然后读取 /etc 目录下的配置文件，包括其自身的配置文件 /etc/systemd/system/default.target。该文件指定了 systemd 需要引导计算机到达的最终目标和状态，实际上是一个软链接：

```shell
[root@bastion ~]# ls /etc/systemd/system/default.target -l
lrwxrwxrwx. 1 root root 37 Oct 17  2019 /etc/systemd/system/default.target -> /lib/systemd/system/multi-user.target
```

在我使用的 bastion 服务器上，它指向的是 multi-user.target；对于带有图形化界面的桌面工作站，它通常指向 graphics.target；而对于单用户模式的机器，它将指向 emergency.target。target 等效于过去 SystemV 中的 [运行级别](https://zh.wikipedia.org/wiki/%E8%BF%90%E8%A1%8C%E7%BA%A7%E5%88%AB)（Runlevel），它提供了别名以实现向后兼容性：

| SystemV Runlevel | systemd target    | systemd target alias | Description                                            |
| ---------------- | ----------------- | -------------------- | ------------------------------------------------------ |
|                  | halt.target       |                      | 在不关闭电源的情况下中止系统。                                        |
| 0                | poweroff.target   | runlevel0.target     | 中止系统并关闭电源。                                             |
| s                | emergency.target  |                      | 单用户模式。 没有服务正在运行，也未挂载文件系统。仅在主控制台上运行一个紧急 Shell，供用户与系统交互。 |
| 1                | rescue.target     | runlevel1.target     | 一个基本系统。文件系统已挂载，只运行最基本的服务和主控制台上的紧急 Shell。               |
| 2                |                   | runlevel2.target     | 多用户模式。虽然还没有网络连接，但不依赖网络的所有非 GUI 服务都已运行。                 |
| 3                | multi-user.target | runlevel3.target     | 所有服务都在运行，但只能使用命令行界面（CLI）。                              |
| 4                |                   | runlevel4.target     | 用户自定义                                                  |
| 5                | graphical.target  | runlevel5.target     | 所有服务都在运行，并且可以使用图形化界面（GUI）。                             |
| 6                | reboot.target     | runlevel6.target     | 重启系统                                                   |

每个 target 都在其配置文件中指定了一组依赖，由 systemd 负责启动。这些依赖是 Linux 达到某个运行级别所必须的服务（service）。换句话说，当一个 target 配置文件中的所有 service 都已成功加载，那么系统就达到了该 target 对应的运行级别。

下图展示了 systemd 启动过程中各 target 和 service 实现的一般顺序：

```shell
                   cryptsetup-pre.target veritysetup-pre.target
                                                  |
(various low-level                                v
 API VFS mounts:             (various cryptsetup/veritysetup devices...)
 mqueue, configfs,                                |    |
 debugfs, ...)                                    v    |
 |                                  cryptsetup.target  |
 |  (various swap                                 |    |    remote-fs-pre.target
 |   devices...)                                  |    |     |        |
 |    |                                           |    |     |        v
 |    v                       local-fs-pre.target |    |     |  (network file systems)
 |  swap.target                       |           |    v     v                 |
 |    |                               v           |  remote-cryptsetup.target  |
 |    |  (various low-level  (various mounts and  |  remote-veritysetup.target |
 |    |   services: udevd,    fsck services...)   |             |    remote-fs.target
 |    |   tmpfiles, random            |           |             |             /
 |    |   seed, sysctl, ...)          v           |             |            /
 |    |      |                 local-fs.target    |             |           /
 |    |      |                        |           |             |          /
 \____|______|_______________   ______|___________/             |         /
                             \ /                                |        /
                              v                                 |       /
                       sysinit.target                           |      /
                              |                                 |     /
       ______________________/|\_____________________           |    /
      /              |        |      |               \          |   /
      |              |        |      |               |          |  /
      v              v        |      v               |          | /
 (various       (various      |  (various            |          |/
  timers...)      paths...)   |   sockets...)        |          |
      |              |        |      |               |          |
      v              v        |      v               |          |
timers.target  paths.target   |  sockets.target      |          |
      |              |        |      |               v          |
      v              \_______ | _____/         rescue.service   |
                             \|/                     |          |
                              v                      v          |
                          basic.target         rescue.target    |
                              |                                 |
                      ________v____________________             |
                     /              |              \            |
                     |              |              |            |
                     v              v              v            |
                 display-    (various system   (various system  |
             manager.service     services        services)      |
                     |         required for        |            |
                     |        graphical UIs)       v            v
                     |              |            multi-user.target
emergency.service    |              |              |
        |            \_____________ | _____________/
        v                          \|/
emergency.target                    v
                              graphical.target
```

如上图所示，想要到达到某个 target，其依赖的所有 target 和 service 就必须已完成加载。如实现 sysinit.target，需要先挂载文件系统（local-fs.target）、设置交换文件（swap.target）、初始化 udev （various low-level services）和设置加密服务（cryptsetup.target）等。不过，同一个 target 的不同依赖项可以并行执行。

当计算机达到 multi-user.target 或 graphical.target 时，它的漫漫启动之路就走到了尽头。但为了满足用户多样的需求，它所面临的挑战其实才刚刚开始。

## Future Work

- 前言提到 RedHat 官方给出了 IPXE/PXE 引导 CoreOS 系统的方法，那么这项技术又是如何实现的呢？
- MBR 只有 446 个字节，可为什么 boot.img 文件却有 512 个字节？
- 目前已经有越来越多的计算机使用 UEFI 和 GPT 来代替 BIOS 和 MBR，其优势体现在哪？
- 我们该如何理解 systemd 的配置文件？如何使用 systemd 部署我们的应用？

## 参考文献

[Creating Red Hat Enterprise Linux CoreOS (RHCOS) machines by PXE or iPXE Booting](https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html#installation-user-infra-machines-pxe_installing-restricted-networks-bare-metal)

[BIOS - Wikipedia](https://zh.wikipedia.org/wiki/BIOS)

[INT 13H - Wikipedia](https://en.wikipedia.org/wiki/INT_13H)

[主引导记录 - Wikipedia](https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%BC%95%E5%AF%BC%E8%AE%B0%E5%BD%95)

[An Introduction To the Linux Boot and Startup Processes](https://opensource.com/article/17/2/linux-boot-and-startup)

[GNU GRUB Manual 2.06](https://www.gnu.org/software/grub/manual/grub/grub.html)

[Bootup(7) - Linux manual page](https://man7.org/linux/man-pages/man7/bootup.7.html)
