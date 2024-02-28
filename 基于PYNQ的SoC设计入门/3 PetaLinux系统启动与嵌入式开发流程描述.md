## Linux系统启动需要

构成一个可以正常使用、功能完善的Linux系统，需要U-Boot、Linux kernel与rootfs。

一般在搭建一个linux系统时，都需要移植linux。而在移植linux之前，都需要先移植一个bootloader代码，bootloader用于启动linux内核，常用的bootloader便是U-Boot。接下来正常启动linux还需要一个完整的文件系统，这便是rootfs，即根文件系统。

对于Boot.BIN，其文件格式有明确详细的说明，主要包括bootrom的说明、fsbl的说明与其他校验值等等，详细内容略过。

## Yocto

自定义Linux系统的开源项目。petalinux从本质上即使用了Yocto，以下是Yocto的wiki：[https://docs.yoctoproject.org/#creating-a-general-layer-using-the-yocto-layer-script](https://docs.yoctoproject.org/#creating-a-general-layer-using-the-yocto-layer-script)

Yocto的核心是Poky，他包含了它包含了 BitBake工具、编译工具链、BSP、诸多程序包或层。由Poky得到的默认参考Linux发行版也叫Poky。
![[Pasted image 20231118161059.png]]
BootROM核心任务是将FSBL从SD、SPI FLASH等存储器中源码通过总线转移到CPU运行

FSBL(first stage boot loader)的核心任务：完成PL的布置、PS端的初始化、初始化DDR时钟等，将SSBL从SD卡中转移到DDR中存放

SSBL：为Linux的启动文件、类似于U-Boot，用于加载UImage（即操作系统内核）

UImage：操作系统内核

device tree：整体的设备树文件、本质上是硬件结构描述文件、描述了CPU、总线、设备等硬件信息，内核会通过解析设备数文件来的得到各个设备所需要的中断资源，然后分配。例如：在PL端配置外设时，需要申请终中断资源，将中断信号连接到内核的中断口上，这样内核才可以接收到PL外外设的中断信号。

## Linux嵌入式开发全流程

因而可以得到在Linux中进行嵌入式开发的全流程如下：

1. 观看外设硬件原理图以及配置的物理地址
2. 通过vivado得到硬件文件
3. 修改设备树文件，在sys-top.dts中添加对应结点
4. 编写内核空间的驱动代码、调用底层读写函数
5. 编写用户空间的应有测试程序（APP）
6. 编写makefile，利用主机的arm交叉编译工具链编译驱动C文件和用户测试程序，得到KO文件与App文件
7. 将驱动代码拷贝到Linux系统的根文件系统下，在板子上运行操作系统，通过加载驱动模块测试驱动是否正确，在调试时需要利用好应用在文件中的print输出有效的外设寄存器信息以更好的debug。