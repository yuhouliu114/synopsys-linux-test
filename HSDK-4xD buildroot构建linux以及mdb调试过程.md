# 1： buildroot构建kernel -> vmlinux
## 1.1： buiroot需要的环境工具
### 1.1.1： 基于RedHat的操作系统（RedHat、CentOS等）
在终端命令行中输入以下命令：
```SHELL
sudo dnf install subversion binutils bzip2 gcc gcc-c++ gawk gettext flex ncurses-devel zlib-devel make patch unzip perl-ExtUtils-MakeMaker glibc glibc-devel glibc-static quilt ncurses-lib sed sdcc intltool sharutils bison wget perl-devel
```
### 1.1.2： 基于Debian的操作系统（Debian、Ubuntu等）
在命令行中输入以下命令：
```SHELL
sudo apt-get install build-essential subversion libncurses5-dev zlib1g-dev gawk gcc-multilib flex git-core gettext libssl-dev
```
## 1.2： 获取buildroot源码
用git工具从buildroot官网的release中获取最新版本的builroot源码，并进入到buildroot目录下，在终端命令行中输入：
```shell
$ git clone https://git.busybox.net/buildroot
$ cd buildroot
```
## 1.3： 修改HSDK的默认配置
 我们需要修改以下HSDK的默认配置文件:`./configs/snps_archs38_hsdk_defconfig` 。
 在终端中输入以下命令：
 ```shell
 vim configs/snps_archs38_hsdk_defconfig
 ```
 可以看到原始的配置信息如下：
 ```vim
 BR2_arcle=y
BR2_archs38_full=y
BR2_TOOLCHAIN_BUILDROOT_GLIBC=y
BR2_PACKAGE_HOST_LINUX_HEADERS_CUSTOM_5_16=y
BR2_PACKAGE_GLIBC_UTILS=y
BR2_TOOLCHAIN_BUILDROOT_CXX=y
BR2_TARGET_OPTIMIZATION="-mfpu=:fpud_all"
BR2_TARGET_GENERIC_HOSTNAME="hsdk"
BR2_TARGET_GENERIC_ISSUE="Welcome to the HSDK Platform"
BR2_SYSTEM_DHCP="eth0"
BR2_ROOTFS_POST_IMAGE_SCRIPT="support/scripts/genimage.sh"
BR2_ROOTFS_POST_SCRIPT_ARGS="-c board/synopsys/hsdk/genimage.cfg"
BR2_LINUX_KERNEL=y
BR2_LINUX_KERNEL_CUSTOM_VERSION=y
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="5.16"
BR2_LINUX_KERNEL_DEFCONFIG="hsdk"
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES="board/synopsys/hsdk/linux.fragment"
BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y
# BR2_TARGET_ROOTFS_TAR is not set
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_BUILD_SYSTEM_KCONFIG=y
BR2_TARGET_UBOOT_CUSTOM_VERSION=y
BR2_TARGET_UBOOT_CUSTOM_VERSION_VALUE="2022.01"
BR2_TARGET_UBOOT_BOARD_DEFCONFIG="hsdk"
BR2_TARGET_UBOOT_NEEDS_DTC=y
BR2_TARGET_UBOOT_FORMAT_ELF=y
BR2_TARGET_UBOOT_NEEDS_OPENSSL=y
BR2_PACKAGE_HOST_DOSFSTOOLS=y
BR2_PACKAGE_HOST_GENIMAGE=y
BR2_PACKAGE_HOST_MTOOLS=y
BR2_PACKAGE_HOST_UBOOT_TOOLS_ENVIMAGE=y
BR2_PACKAGE_HOST_UBOOT_TOOLS_ENVIMAGE_SOURCE="board/synopsys/hsdk/uboot.env.txt"
BR2_PACKAGE_HOST_UBOOT_TOOLS_ENVIMAGE_SIZE="0x4000"
 ```
为了成功地构建出可以运行的`vmlinux`，应该对`defconfig`文件作出以下修改：

1. 找到`BR2_TARGET_UBOOT_CUSTOM_VERSION_VALUE="2022.01"`选项， 并将其修改为`BR2_TARGET_UBOOT_CUSTOM_VERSION_VALUE="2020.01"`, 因为目前u-boot最新版本还是2021.09，。
2. 目前这份`defconfig`文件还没有构建`rootfs`，仅仅是对`kernel`进行了配置，为了顺利的启动内核，加载文件系统，使用`shell`命令,还需要添加以下配置选项：
   ```vim
   BR2_PACKAGE_BUSYBOX_CONFIG="package/busybox/busybox.config"
   BR2_PACKAGE_BUSYBOX_SHOW_OTHERS=y
   BR2_TARGET_ROOTFS_INITRAMFS=y
   BR2_TARGET_ROOTFS_TAR_GZIP=y

   ```
   
## 1.4：编译buildroot
1. 将修改后的`defconfig`文件写入`.config`中 -> 在终端中输入`make snps_archs38_hsdk_defconfig` 。
2. 编译 -> 在终端中输入`make`。
3. 编译完成后，在路径：`./output/build/linux-5.16/vmlinux`中能找到中间生成文件`vmlinux`,这也是后面`mdb`能够调试运行的静态可连接文件。

# 2：mdb调试vmlinux
1. 首先，需要在运行环境中安装`metaware`开发工具链。
2. 在`buildroot`根目录下执行以下`mdb`调试命令：
   ```shell
   mdb_opts="-off=cr_for_more -on=mmux -OK -c"
   mdb -digilent -prop=dig_device=${DIGILENT_DEVICE} ${mdb_opts} ${VMLINUX_PATH}
   ```
3. 打开`putty`串口工具，找到`HSDK-4xD`连接的端口号，串口配置为：`115200 、8 、1 、none`,配置完之后，能看到`kernel`的启动信息。
4. 进入`mdb`后,依次输入以下命令：
   ```shell
   evaluate $put_bank_reg(1, 0x700, 1)
   continue -detach
   wait 10
   quit
   ```
5. 在`putty`串口中能看到以下输入日志：
   ```vim
   Linux version 5.17.0-rc4 (arcoss@2393acda25a0) (arc-buildroot-linux-gnu-gcc.br_real (Buildroot 2021.11-20-g57d993263b) 11.2.0, GNU ld (GNU Binutils) 2.37) #2 SMP PREEMPT Mon Feb 14 03:21:12 UTC 2022
   Memory @ 80000000 [1024M] 
   OF: fdt: Machine model: snps,hsdk
   earlycon: uart8250 at MMIO32 0xf0005000 (options '115200n8')
   printk: bootconsole [uart8250] enabled
   Failed to get possible-cpus from dtb, pretending all 4 cpus exist
   archs-intc	: 2 priority levels (default 1) FIRQ (not used)
   
   IDENTITY	: ARCVER [0x52] ARCNUM [0x0] CHIPID [ 0x0]
   processor [0]	: HS38 R2.1 (ARCv2 ISA) 
   Timers		: Timer0 Timer1 RTC [UP 64-bit] GFRC [SMP 64-bit] 
   ISA Extn	: atomic ll64 unalign mpy[opt 9] div_rem 
   BPU		: full match, cache:2048, Predict Table:16384 Return stk: 8
   MMU [v4]	: 8k PAGE, 2M Super Page (not used) , swalk 2 lvl, JTLB 1024    (256x4), uDTLB 8, uITLB 4, PAE40 (not used) 
   I-Cache		: 64K, 4way/set, 64B Line, VIPT aliasing
   D-Cache		: 64K, 2way/set, 64B Line, PIPT
   SLC		: 512K, 128B Line
   Peripherals	: 0xf0000000, IO-Coherency (per-device) 
   Vector Table	: 0x90000000
   FPU		: SP DP 
   DEBUG		: smaRT RTT ActionPoint 8/min
   Extn [SMP]	: ARConnect (v2): 4 cores with IPI IDU DEBUG GFRC
   Zone ranges:
   ......//省略
   udhcpc: started, v1.34.1
   udhcpc: broadcasting discover
   usb 1-1.1: new high-speed USB device number 3 using ehci-platform
   usb-storage 1-1.1:1.0: USB Mass Storage device detected
   scsi host0: usb-storage 1-1.1:1.0
   scsi 0:0:0:0: Direct-Access     Generic  STORAGE DEVICE   0272 PQ: 0 ANSI: 0
   sd 0:0:0:0: [sda] 15759360 512-byte logical blocks: (8.07 GB/7.51 GiB)
   sd 0:0:0:0: [sda] Write Protect is off
   sd 0:0:0:0: [sda] Mode Sense: 0b 00 00 08
   sd 0:0:0:0: [sda] No Caching mode page found
   sd 0:0:0:0: [sda] Assuming drive cache: write through
    sda: sda1 sda2 sda3 sda4
   sd 0:0:0:0: [sda] Attached SCSI removable disk
   udhcpc: broadcasting discover
   stmmaceth f0008000.ethernet eth0: Link is Up - 1Gbps/Full - flow control rx/   tx
   IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
   udhcpc: broadcasting discover
   udhcpc: broadcasting select for 10.0.0.91, server 10.0.0.1
   udhcpc: lease of 10.0.0.91 obtained from 10.0.0.1, lease time 600
   OK
   
   Welcome to Buildroot
   buildroot login:

   ```
6. 看到`buildroot login` 后,输入`root`便可登入。