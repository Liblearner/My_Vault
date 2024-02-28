# 启动最小系统（仅PS核）
## petalinux部分
这些工作主要在9月份完成，可以简述一下这一个月内做的事。

首先接触了ZYNQ的7000系列芯片，了解了ZYNQ由PS+PL组成，PS由双核ARM-A9加上可配置的SOC构成，拥有独立外设，可以独立运行一个嵌入式系统；而PL则是芯片中外围的FPGA资源，可以进行自定义IP核的开发。

工具链主要包括vivado、vitis与petalinux。其中vivado主要负责硬件的设计与配置、最终导出并生成.xsa文件，以供后续在vitis中与petalinux中进行嵌入式开发或是linux系统的开发。

在安装过程中的小坑就不再赘述、比如在安装工具链时最好腾出200G左右的空间较为有余量。以及xilinx下工具链使用时版本需要一致（比如我的Ubuntu20.04.5+vivado2020.2+petalinux2020）是适配可用的，其他版本可以参考

第一个任务是在开发板中运行一个最小的linux嵌入式系统，磨磨蹭蹭大概花费了一个月左右的时间。首先第一个坑便是在板子 @的硬件中，PYNQ_Z2与正点原子的板子是不适配的也就是说 想要使用开发板上的资源必须从新建vivado工程开始 走一遍选板子和芯片加入IP核并且配置的流程。这里贴上在平板上之前做的笔记。
![[Screenshot_20231007_001947_com.newskyer.draw.jpg]]

![[Screenshot_20231007_001954_com.newskyer.draw.jpg]]

![[Screenshot_20231007_002007_com.newskyer.draw.jpg]]
在vivado新建工程时可以从网上寻找ZYNQ的board file来进行配置 而不巧的是xilinx官网上缺失了很多很多关于Z2这块板子的资料 因此需要多找几次。而启动最小系统的过程只需要配置7000system这个IP即可 它便包含了ZYNQ中的PS部分。

配置完成 并且走完一遍综合的流程之后得到.xsa文件接下来的步骤参考正点原子教程即可 这里省略具体步骤 只贴上几个关键的命令行

新建工程 (参数名略)

petalinux -create -t project —template ZYNQ -n PROJECT-NAME

获取硬件配置

petalinux-config —get-hw-description ../route of xsa

注意重新配置vivado之后重新执行此命令即可 不需要重新新建工程再进行配置 但需要重新编译

接下来便弹出petalinux工程配置界面。 在启动最小系统的过程中大部分保持默认即可 需要配置的是板

子的串口输出信息 在上电后会生成命令行终端

引导启动镜像和内核镜像的存储媒介选择为Primary SD

根文件系统文件格式 由于我们运行的是ubuntu 所以需要将其配置成EXT4 (SD eMMC SATA USB)NFS挂载启动时需要配制成NFS。

在这个过程中petalinux进行的工作是根据配置来解析xsa文件来获取更新设备树 UBOOT配资与内核配置文件的硬件信息。

在后期想重新配置时petalinux-config即可

定制linux内核

petalinux-config -c kernel

使用默认配置即可

根文件系统配置

petalinux-config -c rootfs

一般默认设置即可

设备树配置

一般来说 设备树的配置来源于两种 一种是由vivado中使用的xilinx IP 那么在编译过程中驱动就会解析出这些IP并且加入到设备树中并附上驱动。而如果是自己写的IP 就需要自己来编写设备树与驱动。

一般使用默认生成的设备树即可 自定义的设备树可以在system-user.dtsi中编写 设备树的编写涉及到了Linux驱动部分的相关知识 一般不使用 保持默认即可

工程编译

petalinux-build

编译过程较长 可以泡杯茶

生成ZYNQ启动文件

petalinux-package —boot —fslb—FPGA --u-boot —force

三个参数分别指定了fsbl bitstream与 uboot文件的位置

接下来便是适应SD卡引导linux系统启动

SD卡需要两个分区 500MB的FAT32放置镜像文件 剩余空间使用EXT4存放根文件系统

如何配置SD不再赘述 使用fdisk命令即可

需要注意根文件系统为rootfs.cpio 需要完全解压

最终成功登入系统，可以查看到串口发送到MobaXterm上的信息。


## 硬件vivado配置部分

先看看正点原子用到了哪些IP：
**rst_ps_7_0_100M(processor system reset)**
![[Pasted image 20240123222937.png]]
作用：输入不同的信号，为系统产生不同的复位信号
输入信号：全时钟域中最慢的同步时钟、各个复位产生指示信号
输出信号：debug、总线、互连、外设的复位信号
可以调节输入信号的有效时长，复位信号的极性
输出信号会按照一个序列顺序从先往后变得有效

具体使用细节略，由图指导。
在正点原子工程中：
ext_reset_in由ZYNQ7 Processing System的FCLK_RESET0_N产生。 
slowest_sync_clk由 ZYNQ7 Processingde的FCLK_CLK0产生，同时也是AXI总线时钟；并同时送到了PLL作为输入。
peripheral_aresetn[0:0]产生后接入到了AXI Interconnect，以及各个AXI总线外设的aresten中


**ps7_0_axi_periph(AXI Interconnect)**
![[Pasted image 20240123222948.png]]
如图中，这个AXI互连的IP互连了一个Slave AXI，15个Master AXI
主要作用是互连，将一个或者多个AXI存储器映射的主器件连接到一个或者多个存储器映射的从器件。AXI接口符合AMBA的AXI第四版规范，包括了AXI4-Lite控制寄存器接口子集。仅用于存储器映射传输；AXI4-Stream传输不适用。
内部互连结构：
![[Screenshot_20240126_205012_com.newskyer.draw.jpg]]


在这个block diagram中，AXI总线上设备的连接如下：
核为从设备（核其实同时也为主设备，只是没有映射出来）；
![[F0343066D46000A2CD82FA9E6F5320E0.jpg]]

核为主设备：
![[Pasted image 20240126230141.png]]

看可以看到，在内部总线中，这里的主与从采取了1-N的连接方式。

IP核接口如下：
ACLK、ARESETN
一组Master总线接口包括：
+ Mxx_ACLK
+ Mxx_ARESETN
+ Mxx_AXI
一组Slave总线接口包括：
+ Sxx_AXI
+ Sxx_ACLK
+ Sxx_ARESETN

其中clk与reset的产生与互联基本都公用，不再描述，下面是AXI数据总线的连接情况：

N个设备如下：
module中lcd_out：
M00：S_AXI
M01：PWM_AXI
M02：ctrl
M03：s_axi_CTRL

module中clk_wiz_0：
M04：s_axi_lite

module中camera_in：
M05：s_axi_ctrl

module中axi_gpio_0
M06：S_AXI

module中i2c2：
M07：S_AXI

module中rest_gpio：
M08：S_AXI

module中audio：
M09：s_axi_lite
M10：s_axi_ctrl
M11：s_axi_ctrl1

module中hdmi_out：
M12：s00_axi
M13：ctrl
M14：S_AXI_LITE


IP核的功能简介：
主从接口各16个：
一主一从连接
Slave Interface接Master设备
Master Interface接Slave设备

总线地址自检（12bit到64bit）
传输位数转换（32到1024可调，向上兼容缝合与向下拆分,上与下的依据是由从到主）
时钟频率转换（每个连接独立，同步时钟1：N到N：1，异步时钟也可以转换，但需要消耗更多资源，延时更长）
协议转换（AXI与AXI-Lite兼容）
数据路径缓冲

**clocking wizard(clocking wizard)**
![[Pasted image 20240123222958.png]]





**audio**
![[Pasted image 20240123223006.png]]


**rest_gpio(AXI GPIO)**
![[Pasted image 20240123223014.png]]


**hdmi_out**
![[Pasted image 20240123223022.png]]


**xlconcat_0(concat)**
![[Pasted image 20240123223028.png]]



**camera_in**
![[Pasted image 20240123223035.png]]


**lcd_out**
![[Pasted image 20240123223045.png]]
**processing_system7_0(ZYNQ7 Processing System)**
![[Pasted image 20240123223056.png]]
**gmii2rgmii_0(gmii to rgmii V1.0)**
![[Pasted image 20240123223107.png]]
**i2c2(AXI IIC)**
![[Pasted image 20240123223114.png]]
**axi_gpio_0(AXI GPIO)**
![[Pasted image 20240123223120.png]]


