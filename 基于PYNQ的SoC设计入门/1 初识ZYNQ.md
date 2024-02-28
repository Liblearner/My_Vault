## 初识ZYNQ

芯片命名规则
![[Pasted image 20231117215546.png]]
### PS端
![[Pasted image 20231117215606.png]]
APU示意图：
![[Pasted image 20231117215618.png]]
对外连接接口
![[Pasted image 20231117215628.png]]
IO与MIO映射表：
![[Pasted image 20231117215639.png]]
### PL端

略

## AXI总线
![[Pasted image 20231117215709.png]]

上图使用AXI总线配置了一系列外设，搭建了PS和PL之间通信链路的ZYNQ核接口示意图。

协议为AXI4，支持以下接口：

- AXI4：高性能存储映射接口。支持突发传输，如处理器访问存储器等需要地址的高速数据传输场景。
- AXI4-Lite：简化版AXI4，少数据量。为外设提供单个数据传输，主要用于访问低速外设寄存器。
- AXI4-Stream：高速数据流传输，非存储映射。类似FIFO，主从之间直连进行传输，主要用于高速传输场合。

基于存储映射的协议，更多关于AXI总线的知识：
https://blog.csdn.net/bleauchat/article/details/96891619?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522170628259416800222855077%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=170628259416800222855077&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-96891619-null-null.142^v99^pc_search_result_base4&utm_term=AXI%E6%80%BB%E7%BA%BF&spm=1018.2226.3001.4187


![[Pasted image 20240127152628.png]]
读写架构：读地址、读数据、写地址、写数据、写响应一共五个通道；
地址通道：控制信息，用于描述被传输的数据属性
master->slave：写通道+写响应
slave->master：读通道

- **读/写地址通道**：读、写传输每个都有自己的地址通道，对应的地址通道承载着对应传输的地址控制信息；
- **读数据通道**：读数据通道承载着读数据和读响应信号包括数据总线（8/16/32/64/128/256/512/1024 bit）和指示读传输完成的读响应信号；
- **写数据通道**：写数据通道的数据信息被认为是缓冲（buffered）了的，master无需等待slave对上次写传输的确认即可发起一次新的写传输。写通道包括数据总线（8/16...1024 bit）和字节线（用于指示8 bit 数据信号的有效性）；
- **写响应通道**：slave使用写响应通道对写传输进行响应。所有的写传输需要写响应通道的完成信号；

主机发出的会话无论读写都会表明一个地址。此地址对应系统存储空间中的一个地址，表明是该存储空间的读写操作。

基于VAILD/READY的握手机制数据传输协议；
VAILD：传输源端使用，表明是地址/控制信号，数据是有效的
READY：表明自己可以接受信息

三种接口定义：
master/interconnect、slave/interconnect、master/slave。

几种典型拓扑：
+ 共享地址与数据总线
+ 共享地址总线，多数据总线（地址通道的带宽要求没有数据通道要求高，因此可以共用，多数据总线用于平衡系统性能与互联复杂度）
+ multilayer多层，多地址总线，多数据总线

协议内容如下：
![[Pasted image 20240127154701.png]]
![[Pasted image 20240127154712.png]]
![[Pasted image 20240127154722.png]]
![[Pasted image 20240127154732.png]]

![[Pasted image 20240127154741.png]]

读写时序如下：
![[Pasted image 20240127160247.png]]

通道传输顺序：



PS与PL之间的主要连接时通过一组9个AXI接口，每个AXI接口都有多个通道组成：
![[Pasted image 20231117215733.png]]
列表总结如下：
![[Pasted image 20231117215743.png]]
即分为三种：

- 4个AXI-GP：通用AXI,32bit总线，中低速通信，不带缓冲的透传，2个PS做主机，2个PL做主机
    
    如：M_AXI_GP0、M_AXI_GP0_ACLK
    
- 1个AXI-ACP：1加速器一致性接口AXI，64bit，PL与APU的SCU（Snoop Control Unit一致性控制的单元，在ARM核和二级Cache以及OCM存储器之间桥联，并负责与PL部分对接）之间的单个异步连接，用于实现APU Cache与PL单元一致性，PL主机
    
- 4个AXP-HP：高性能接口，32或64bit，带有FIFO缓冲，支持PL与PS中存储器单元的高速率通信
    
    如：S_AXI_HP0、S_AXI_HP0_FIFO_CTRL、S_AXI_HP0_ACLK
    

## IP核使用

### PLL

锁相环：常用于得到期望频率的时钟，优化已有时钟性能。主要根据外部反馈电压来修改内部环路振荡信号的频率与相位。可以对时钟进行任意分频、倍频、相位调整、占空比调整、优化时钟抖动。

Xilinx中的PLL：模拟锁相环。优缺点略。

Xilinx中的时钟资源：

时钟管理单元CMT（Clock Management Tile），每一个CMT由一个MMCM（一个PLL加上DCM的一部分以进行精细相移）和一个PLL组成。7z020中共有4个CMT。

调用IP核：

IP Catalog内，FPGA Features→clock→clock wizard

Customize IP中一些默认的时钟配置略，这里对时钟输出的驱动做简单介绍（涉及时钟驱动类型的选择）：

- BUFG：全局缓冲，如果时钟信号需要走全局网络则必须选择BUFG
- BUFH：区域水平缓冲器，驱动其所有水平区域（物理意义的水平）内的CLB，RAM，IOB
- BUFGCE：带有时钟使能端的全局缓冲器
- BUFHCE：带有时钟使能端的区域水平缓冲器
- No Buffer：输出时钟无需挂在全局时钟网络上时，可以选择无缓冲区

### 提到的其他IP

单双端口RAM

FIFO

串口

LCD

HDMI

### GPIO控制

### I2C控制器


### SPI控制器