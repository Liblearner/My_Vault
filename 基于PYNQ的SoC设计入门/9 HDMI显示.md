- [[#基本功能|基本功能]]
- [[#实现|实现]]
		- [[#dvi_encoder|dvi_encoder]]
		- [[#Serializer模块设计|Serializer模块设计]]
	- [[#实现#上板|上板]]
- [[#扩展|扩展]]

## 基本功能
A型HDMI如下：
HDMI向下兼容DVI，还可以多传输音频
![[Pasted image 20231117215803.png]]
DVI与HDMI接口协议在物理层实现TMDS标准传输音视频数据

以下是TMDS协议的驱动逻辑：
![[Pasted image 20231117215829.png]]
简单来讲，DVI或者HDMI视频传输所使用的TMDS连接通过四个串行通道实现。DVI和HDMI都使用其中的三个通道来分别传输RGB，同时也可以传输像素点的亮度和色度信息。第四个通道是时钟通道，用于传输像素时钟，用于接收端数据同步。

数据传输过程：

8bit→10bit（2bit控制位：行同步、帧同步）→串转并（波特率不变）→TMDS数据通道发送

**HDMI与DVI的区别与联系：**
![[Pasted image 20231222161943.png]]

这里为了简单直接把HDMI当做DVI进行驱动

并串转换的资源直接使用专门的转换器-OSERDESE2。可以实现8:1、10:1、14:1

原理图如下：
![[Pasted image 20231117215848.png]]
接口中的每一对DATA+-直接与PL端的差分信号相连，并在每一pin上接瞬态抑制二极管，用于保护高速数据接口。热拔插端（HDMI_HPD）与HDMI的数据显示通道（DDC）以及IIC接口不再赘述。

这里的实现：
LCD彩条显示+RGB2DVI模块，用于将RGB888转成TMDS输出，其余相同
## 实现
**RCG2DVI**
![[Pasted image 20231222162226.png]]

Encoder：数据编码
Serializer：串并转换
OBUFDS：转差分输出

系统时钟：
1. 像素时钟
2. 5倍像素时钟：由于串转并中需要10倍传输速率，而OSERDESE2自带2倍DDR，因此需要5倍频
![[Pasted image 20231222163754.png]]
#### dvi_encoder
实现8b转10b
代码由Xilinx官方提供，

![[Pasted image 20231222163938.png]]
**TMDS编码算法**

![[Pasted image 20231222163954.png]]
TMDS 通过逻辑算法将 8 位字符数据通过编码转换为 10 位字符数据，前 8 位数据由原始信号经运算后 获得，第 9 位表示运算的方式，1 表示异或 0 表示异或非。经过 DC 平衡后（第 10 位），采用差分信号传 输数据。第 10 位实际是一个反转标志位，1 表示进行了反转而 0 表示没有反转，从而达到 DC 平衡。 接收端在收到信号后，再进行相反的运算。TMDS 和 LVDS、TTL 相比有较好的电磁兼容性能。这种算 法可以减小传输信号过程的上冲和下冲，而 DC 平衡使信号对传输线的电磁干扰减少，可以用低成本的专用 电缆实现长距离、高质量的数字信号传输。
具体代码原理略。
#### Serializer模块设计
采用Xilinx官方提供的OSERDESE2原语
功能框图：
![[Pasted image 20231222165929.png]]
使用原语步骤：
![[Pasted image 20231222170025.png]]
1-10时需要两个连接扩展

### 上板

对于约束：差分信号的约束，只需要给定其正极信号即可，自动分配负极。

啊啊啊屏幕不亮啊啊啊
## 扩展
在领航者给好的板子工程中 ，看到HDMI控制器IP核构成如下：
![[Pasted image 20231117215901.png]]
包括几个模块：
![[Pasted image 20231117215913.png]]
分别是：

- AXI接口模块
- ？
- 分频模块
- VTC模块
- reset模块
- 输出驱动模块（AXI连接）
- 缓冲FIFO
- ？
![[Pasted image 20231117215924.png]]
LCD的这5个模块分别为：id识别，clk分频，lcd显示与lcd驱动。以及最终包装成的顶层模块。
