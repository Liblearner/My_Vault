
## 基本概念

TFT-LCD：
Thin Film Transistor-Liquid Crystal Display
一些参数如分辨率（720p、1080p）、像素格式（RGB888/RGB565）、时间参数等等就不再赘述。
目前找到的一块LCD屏参数：
（暂时没找到hhhh）
正点原子历程中给出的：
RGBLCD
![[Pasted image 20231221195041.png]]
![[Pasted image 20231221195111.png]]
行同步信号：HSYNC，产生时表示开始新的一行
帧同步信号：VSYNC，产生时表示开始新的一帧图
扫描顺序：默认从左往右，从上往下（横屏显示）
HBP、HFP、VPB、VFP：由IC切换行或者帧时产生，是扫描完关闭显示与产生同步信号或者等待同步信号产生时的延时，用于锁定有效的像素数据，

行显示时序图如下：
![[Pasted image 20231221200034.png]]
行同步信号产生后，需要持续**行同步信号宽度+行显示后沿（HSPW+HBP）** 的时间，才可以接受有效的像素数据
当一行显示完之后，需要等待**HFP**时间才可以发出下一个HSYNC信号
一行的显示总耗时：
HSPW+HBP+HOZVAL+HFP


帧显示时序：
![[Pasted image 20231221200403.png]]
显示一帧的时间：
（VSPW+VBP+LINE+VFP）* （HSPW+HBP+HOZVAL+HFP）

（帧同步宽度+帧显示后沿+行数+帧显示前沿 ）* （行时间）？
像素时钟：即显示一帧图像需要的时钟数 * 刷新率：
635 * 1344 * 60 = 51.2M

可以不用严格按照此计算，提供一个相近频率就可以达到接近的刷新率

这里提供了不同显示分辨率的参数：
![[Pasted image 20231221200939.png]]

硬件部分：
RGB888一边是有效数据，另一边三个最高位也用于判定LCD的ID，所以对于ZYNQ这是一个双向的引脚

## 实现流程

## RTL

![[Pasted image 20231221201127.png]]
共5个模块：
顶层：lcd_rgb_colorbar
读取ID：rd_id
分频：clk_div
显示：lcd_display
驱动：lcd_driver
### rd_id

模块用于读取rgb-lcd的id，简单的译码电路。
### clk_div
分频模块，这里只涉及到了2和4分频，实现比较简洁。以后可以参考
### lcd_driver
主要实现上述原理中的时序功能，讲准备好的pix数据按照时序输出显示在lcd屏，这里附上时序参数与波形图：
![[Pasted image 20231221212701.png]]
![[Pasted image 20231221212721.png]]
注意，代码里给的和文档略有区别
### lcd_display
lcd显示，用于按照时序送入xy左边与像素点数据
### lcd_colorbar
顶层模块，略
