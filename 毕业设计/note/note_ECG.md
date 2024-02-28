- [[#ECG信号特征|ECG信号特征]]
	- [[#ECG信号特征#基础概念|基础概念]]
	- [[#ECG信号特征#常用数据集|常用数据集]]
	- [[#ECG信号特征#先进产品|先进产品]]
	- [[#ECG信号特征#处理算法|处理算法]]
		- [[#处理算法#小波|小波]]
- [[#硕士学位论文|硕士学位论文]]
	- [[#硕士学位论文#穿戴式心电监护设备的低功耗技术研究|穿戴式心电监护设备的低功耗技术研究]]
	- [[#硕士学位论文#心电信息采集及心率失常检测方法研究|心电信息采集及心率失常检测方法研究]]
	- [[#硕士学位论文#移动智能心电监测系统关键技术研究|移动智能心电监测系统关键技术研究]]
	- [[#硕士学位论文#穿戴式单导联心电监测系统及监测位置综合研究|穿戴式单导联心电监测系统及监测位置综合研究]]
- [[#ECG_Based machine-learing algorithms for heartbeat classification|ECG_Based machine-learing algorithms for heartbeat classification]]
	- [[#ECG_Based machine-learing algorithms for heartbeat classification#Abstract|Abstract]]
	- [[#ECG_Based machine-learing algorithms for heartbeat classification#信号处理流程：|信号处理流程：]]
		- [[#信号处理流程：#DWT（离散小波变换）：|DWT（离散小波变换）：]]
	- [[#ECG_Based machine-learing algorithms for heartbeat classification#R峰检测|R峰检测]]
	- [[#ECG_Based machine-learing algorithms for heartbeat classification#P和T峰检测|P和T峰检测]]
		- [[#P和T峰检测#特征提取的机器学习|特征提取的机器学习]]
	- [[#ECG_Based machine-learing algorithms for heartbeat classification#算法结果与对比|算法结果与对比]]
- [[#A generalizable electrocardiogram-base....|A generalizable electrocardiogram-base....]]
	- [[#A generalizable electrocardiogram-base....#Abstract|Abstract]]
- [[#1.06um 65nm|1.06um 65nm]]
	- [[#1.06um 65nm#Abstract|Abstract]]
	- [[#1.06um 65nm#滤波|滤波]]
	- [[#1.06um 65nm#R峰检测|R峰检测]]
	- [[#1.06um 65nm#离群值检测|离群值检测]]
	- [[#1.06um 65nm#网络处理|网络处理]]

## ECG信号特征

### 基础概念
维基百科：ECG介绍
[https://zh.wikipedia.org/wiki/%E5%BF%83%E7%94%B5%E5%9B%BE#](https://zh.wikipedia.org/wiki/%E5%BF%83%E7%94%B5%E5%9B%BE#)

**Cover公司**：他们出版了由D. [Dubin撰写的《心电图快速解读》](https://www.msdmanuals.cn/home/heart-and-blood-vessel-disorders/diagnosis-of-heart-and-blood-vessel-disorders/electrocardiography)[1](https://www.msdmanuals.cn/home/heart-and-blood-vessel-disorders/diagnosis-of-heart-and-blood-vessel-disorders/electrocardiography)。虽然他们主要是出版商，但他们的书籍在心电图领域具有广泛的影响力：
[https://www.msdmanuals.cn/home/heart-and-blood-vessel-disorders/diagnosis-of-heart-and-blood-vessel-disorders/electrocardiography](https://www.msdmanuals.cn/home/heart-and-blood-vessel-disorders/diagnosis-of-heart-and-blood-vessel-disorders/electrocardiography)

正常心电信号幅度范围：
		10uV-4mV，经典值1mV
	频谱：
		0.05Hz-100Hz
	常见干扰（没有提到U波，一般不用这个）：
![[Pasted image 20231215183740.png]]
**信号处理需求**
	**模拟前端：
		初级：仪表放大，搞CMRR
		后级：精密运放，高增益，有带通滤波**
	**硬件：
	AFE：（都集成了前端硬件滤波与ADC处理）
		AD8232
		BMD101
		ADAS1000
		ADS1292 
		KS1081
	硬件比较：https://blog.csdn.net/zilinhust/article/details/123768343


其他实现：
FPGA or ASIC的略
有一些通过自己搭建的两级运放进行硬件滤波

下面这个使用ADS8201进行硬件滤波与采样：
	![[Pasted image 20231215214102.png]]

### 常用数据集
[https://zhuanlan.zhihu.com/p/343265806](https://zhuanlan.zhihu.com/p/343265806)（附带数据集处理）
- MIT-BIH心电数据库：由美国麻省理工学院与Beth Israel医院联合建立；
- AHA心律失常心电数据库：由美国心脏学会建立（**需付费**下载）；
- CSE心电数据库：由欧盟建立（**需付费**下载）；
- 欧盟ST-T心电数据库。
其他尝试：
**PAF Prediction Challenge Database**
数据链接：[https://physionet.org/content/afpdb/1.0.0/](https://link.zhihu.com/?target=https%3A//physionet.org/content/afpdb/1.0.0/)
- 一个具有双通道的心电图记录数据库，是the Computers in Cardiology Challenge 2001的挑战赛数据集；
- 数据集共有**50组记录**，每一组记录包含**双通道**的**持续时间为30分钟**的数据；
- 但是数据主要关于“心房颤动”疾病。
**MIT-BIH Atrial Fibrillation Database**
- 数据链接：[https://physionet.org/content/afdb/1.0.0/](https://link.zhihu.com/?target=https%3A//physionet.org/content/afdb/1.0.0/)
- 一个具有双通道的心电图记录数据库；
- 该数据库共有**25组记录**，是25个长期**心房颤动（主要是阵发性）**人体受试者的心电图记录；
- 记录以**每秒250**个样本采样，每条记录**持续时间为10小时**；
- 部分数据**没有心拍注释**。
> 补充说明：导联方式 常规导联包括：肢体导联和胸壁导联两部分；
- 肢体导联分为三个标准导联（I、II、III）和三个加压单极导联（aVR、aVL、aVF）    
- 胸壁导联共15个（左边V2-V9；右边V1、V3R-V8R）


### 先进产品
**iRhythm Technologies的ZIO Patch**
https://www.irhythmtech.com/

ECG胸贴：
[**Y2 Dido智能手环**：虽然不是专门的ECG胸贴，但它具有医用级别的ECG心电图、心率、睡眠、血氧和血压监测功能](https://www.zhihu.com/question/401348775)[2](https://www.zhihu.com/question/401348775)。如果您寻找便携式的ECG监测设备，这可能是一个不错的选择。
![[Pasted image 20231215205044.png]]

### 处理算法

#### 降噪算法
带通滤波+小波阈值降噪



#### Pan_Tompkins
低通+高通：5-11Hz；消除低频信号漂移
求导：消除输入信号的直流分量，增强P，QRS与T
平方：增强 QRS波群的斜率并强调Q与S
移动平均：平滑输出，减小干扰波的影响（对200Hz，取窗口为30合适）
自适应阈值+回溯漏检

确定出R：之后定出Q与S，再确定P与T。
**获取到R之后，如何确定PT与QS？**
魔改PanTompkins
![[Pasted image 20240222175801.png]]
重要参数：
+ RR间期（OK）
+ 平均心率（OK）
+ PR间期
+ QRS波宽度
+ QRS波面积
+ R波幅值

Q与ST段的频率范围：0.05-2Hz
因此高通滤波的下限频率不可以太高，设置为0.05比较合适。

#### 呼吸率提取算法
可以用ECG，也可以用加速度。用ECG的如下：
EDR（ECG-Derived Respiratory Signal）**由单导联心电信号提取呼吸信息**；心电信号的复制变化总包含着呼吸信息，表现为吸气期间R波幅值减小，呼气期间R波幅值增大。
+ 由PanTompkins提取R波
+ 自然三次样条插值对心电信号的基线进行估计，去除基线漂移，获取心电信号R波
+ 自然样条插值算法得到R波幅值调制信号提取出呼吸信息

呼吸信号频率：0.1-0.4Hz
![[Pasted image 20240222203408.png]]
R峰波峰之间确定基点位置：作为插值的样点值
相减之后得到去除基线漂移之后的值

其中**自然三次样条插值**算法流程如下：
（关于插值与拟合：https://zhuanlan.zhihu.com/p/62860859）

三次样条插值：
简单来说就是已知了n个点的值，需要求穿过这n点的方程，插值用一个分段函数来表示要求解的方程。首先在数轴上的这一段分成n段区间，在每一个区间上分别用一个三次方程表示需要求解的方程，这个三次方程满足以下几个条件：
+ 满足插值条件，即S(x_i) = y_i
+ 曲线光滑，即S(x),S(x)的一阶与二阶导都连续
而后方程构造成如下形式 ：$$y = a_i + b_ix + c_ix^2 + d_ix^3$$
可以看出在n个区间一共四个未知数，有n个小区间，则有4n个未知数。需要4n个方程来求解。
其中4n个方程来自于：
+ 在端点处的值已知，一共可以列出2(n-1) + 2 个方程
+ 内部连接处的一阶导相等，有n-1个方程
+ 内部连接处的二阶导相等，与n-1个方程
+ 边界条件，最外侧两个点的二阶导数。若为自然边界则制定为0即可；固定边界和非扭结边界略

具体推导过程可以看上面链接。
python代码实现：
https://zhuanlan.zhihu.com/p/672601034
c代码实现：
https://blog.csdn.net/weixin_51472360/article/details/112383805?ops_request_misc=&request_id=&biz_id=102&utm_term=%E8%87%AA%E7%84%B6%E4%B8%89%E6%AC%A1%E6%A0%B7%E6%9D%A1%E6%8F%92%E5%80%BC%20C&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-1-112383805.142^v99^pc_search_result_base4&spm=1018.2226.3001.4187




关于追赶法进行矩阵运算：
![[Pasted image 20240222211713.png]]

加速度提取呼吸率：
![[Pasted image 20240228165227.png]]
三轴加速度数据（采样数据10Hz）：
首先经过窗口为10的移动平均滤波器
![[Pasted image 20240228165904.png]]
再存在尺寸大小为100的互斥buffer中（详情见https://zhuanlan.zhihu.com/p/598993031）、
每100个数据进行归一化处理
![[Pasted image 20240228165921.png]]
归一化处理的数据在matlab中计算出RR
处理时，首先：
窗口大小为15的移动平均平均滤波
然后输入曲线拟合：matlab中的样条插值
从曲线中可以读出检测到的峰值的位置与数值；峰值就代表呼吸呼气阶段形成的峰值。
计算RR就是需要考虑在1min之内观察到的峰值数量

![[Pasted image 20240228171453.png]]
![[Pasted image 20240228171622.png]]

#### 小波
可以尝试基于db8的小波，在对应频段去除基线漂移和工频干扰：

算法原理：
https://blog.csdn.net/weixin_39852875/article/details/102727474
![[Pasted image 20231203193321.png]]
优势：可以在不同尺度上对信号进行分解。
低频信号近似原来的信号，高频信号用于体现信号的细节。

```
散小波变换是对基本小波的尺度和平移进行离散化。在图像处理中，常采用二进小波作为小波变换函数，即使用2的整数次幂进行划分。
余弦变换是经典的谱分析工具，他考察的是整个时域过程的频域特征或整个频域过程的时域特征，因此对于平稳过程，他有很好的效果，但对于非平稳过程，他却有诸多不足。
小波分解的意义就在于能够在不同尺度上对信号进行分解，而且对不同尺度的选择可以根据不同的目标来确定。
通过不断的分解过程，将近似信号连续分解，就可以将信号分解成许多低分辨率成分。理论上分解可以无限制的进行下去，但事实上，分解可以进行到细节（高频）只包含单个样本为止。因此，在实际应用中，一般依据信号的特征或者合适的标准来选择适当的分解层数。
即低频至近似；高频指细节。
```
小波分析进行消噪处理一般有下述3种方法。

(1)默认阈值消噪处理。该方法利用函数ddencmp生成信号的默认阈值，然后利用函数wdencmp进行消噪处理。

(2)给定阈值消噪处理。在实际的消噪处理过程中，阈值往往可通过经验公式获得，且这种阈值比默认阈值的可信度高。在进行阈值量化处理时可利用函数wthresh。

(3)强制消噪处理。该方法是将小波分解结构中的高频系数全部置为0，即滤掉所有高频部分，然后对信号进行小波重构。这种方法比较简单，且消噪后的信号比较平滑，但是容易丢失信号中的有用成分。
matlab实操：
wavedec：分解
```
% 用db1小波对原始信号进行3层分解并提取系数
[c,l]=wavedec(s,3,'db1');   %这里c的长度为1453，比s（1451）多，l的长度为5
 
% function [c,l] = wavedec(x,n,IN3,IN4)：多层一维小波分解。
% WAVEDEC 使用一个特定的小波“wname”或一组特定的小波分解滤波器进行多层一维小波分析。
% [C,L] = WAVEDEC(X,N，'wname') 使用'wname'返回信号X在N级的小波分解。
% 输出分解结构包含小波 the wavelet decomposition vector C和 the bookkeeping vector L
% N 必须是一个严格的正整数。
————————————————
版权声明：本文为CSDN博主「Cheeky_man」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_36045093/article/details/114991235
```
appcoef：提取近似系数（低频系数）
```
% 4、对[c,l]提取近似系数
a3=appcoef(c,l,'db1',3);
% APPCOEF: 提取一维小波变换近似系数。
% A = APPCOEF(C,L,'wname',N) 使用小波分解结构计算N级的近似系数[C,L]
% Level N must be an integer such that 0 <= N <= length(L)-2. 
% A = APPCOEF(C,L,'wname') extracts（提取） the approximation coefficients（近似
```
detcoef：提取细节系数（高频系数）

```
% 5、对[c,l]提取细节系数
% 5.1、提取3级细节系数
d3=detcoef(c,l,3);
%  DETCOEF提取一维细节系数。
%  D = DETCOEF(C,L,N)从小波分解结构[C,L]中提取出N级的细节系数(the detail coefficients)。
%  Level N must be an integer such that 1 <= N <= NMAX where NMAX = length(L)-2.
%  D = DETCOEF(C,L)提取最后一级NMAX的细节系数。
%  If N is a vector of integers such that 1 <= N(j) <= NMAX:
```

遗留问题：不知道怎么进行高频滤除与重组，只能滤除低频。

## 硕士学位论文

### 基于BP神经网络的ECG分类及应用研究
PT+BP神经网络针对房性早搏与室性早搏

1. 心电信号预处理
2. 心电波形检测（重点是QRS波）
+ 滤波法
	+ 1. 0.5-35Hz带通
	+ 2. 数字滤波滤除工频干扰
	+ 3. 最佳信噪比输出的滤波器
	+ 4. QRS检测：自适应双边界阈值方法
+ 模版匹配
	+ QRS波群与T波作为固定模版，通过频率范围内的模版能量浓度匹配检测QRS波群
+ 小波（时域与局部特征）
	+ 小波分解+自适应滤波，重构输出多个尺度信号
+ 神经网络
3. 特征提取与选择
4. 分类诊断

PT：得到完整QRS波并且获得特征参数；用到的特征参数是：$$RR_{n-1}，RR_n,以及QRS波面积与QRS波宽度$$




### 穿戴式心电监护设备的低功耗技术研究

ECG便携设备研究历程：
目前比较受欢迎的检测设备（主要分为胸部和手部）：
Zio Patch（单导联）
实现低功耗：压缩
![[Pasted image 20231215193906.png]]
我国普及：
Holter：动态心电图监测技术；但使用不便利，因而需求可穿戴式
硬件滤波也常用于大型医疗设备

### 心电信息采集及心率失常检测方法研究

硬件前端：
BMD101

信号提取：
去噪：带通滤波与小波阈值去噪方法
检测R与QRS：改进的差分自适应阈值检测算法

处理与检测：
R波：计算出心率、心率变异性、身体疲劳度和精神疲劳度、5种常见的心律失常情况判定
高斯核SVM和CNN分类：高斯核SVM检测成功率更高
**这篇论文中还介绍了诸多计算生理特征的计算式，可以试试**

关于**去噪和特征提取**的研究现状：
	Alhelal D：基于FPGA的滤波与QRS检测，对MIT-BIH中节拍的准确率到98%
	Garrmouri S：多层感知神经网络对卡尔曼进行扩展，在噪声环境中的提取效果较好
	传统方式：Wiener，小波，中值，最小均方等等
	Verma AK：分数阶差分窗口滤波器AFDW
	Varfas RN：普通离散小波：结合隐马尔科夫模型的心电数据降噪方法
	Velmurugan S：中值滤波， Gabor小波多线性判别：提取PT与QRS
	陈耿铎：自适应差分阈值法：对R波检测率达到99.5%

论文的对比结论：**小波变换类算法和神经网络类算法的检测准确率较高，对去噪声效果较好；差分阈值类型对R波提取方面最好**

论文中还提到了对于心律失常技术的研究现状，在此略，但结论是**支持向量积累算法和神经网络类算法。**

### 移动智能心电监测系统关键技术研究

硬件平台：
自己搭建的两级运放滤波，MCU：nRF51822（Cortx-M0，只负责发送）
原型机形状类似乐普
预处理（硬件）-小波变换（识别QRS）-KF（去除肌电）

心电信号提取技术：
1. 工频干扰：50-60Hz噪声。常用传统数字显波器与自适应滤波。Levkov滤波方法
2. 基线漂移：数字高通、三次样条估计、自适应滤波、滑动均值滤波、独立成分分析（ICA）、差值、经验模态分解、小波分解。其中基于小波的效果最好，对于心电ST水平评估的影响最小
3. 肌电噪声：频谱分解估计、基于db8小波、扩展KF
4. 其他噪声：硬件方式去除

### 穿戴式单导联心电监测系统及监测位置综合研究

硬件：MCU：nRF51882
预处理：
	低通（10阶IIR巴特沃斯，Fc = 45Hz）-高通（4阶IIR巴特沃斯，Fc = 0.5Hz）-带阻（8阶IIR巴特沃斯,Fc = 51.5Hz）-小波(8层分解，去除其中的基线漂移)-全局阈值处理（消除高频噪声）
特征检测与处理：
	R峰：斜率阈值
	Q与S类似，在检测出R峰之后搜索附近波形特征


## ECG_Based machine-learing algorithms for heartbeat classification
### Abstract
ECG 信号波形特征：
由R、QRS、T组成。每个波形的持续时间和形状以及不同峰值之间的距离可以用于诊断心脏疾病。
![[Pasted image 20231203184655.png]]

本文中分析心电信号的算法：
TERNA：两事件相关移动平均。针对特定信号段来提取峰值
FrFT：分数FT在时频平面上旋转以显示各种峰值的位置
并使用不同峰值、峰值间距与其他特征来训练机器学习模型。

R峰检测：
检测R峰的快速斜坡有效算法：
Adeluyi , O. & Lee, J. A. R-reader: A lightweight algorithm for rapid detection of ECG signal R-peaks. In IEEE International Confer‑ ence on Engineering and Industries (ICEI), pp. 1–5 (2011).

TERMA检测P峰和T峰：
Elgendi, M., Jonkman, M. & De Boer, F. R wave detection using coifets wavelets. In IEEE 35th Annual Northeast Bioengineering Conference, pp. 1–2 (2009).
11. Elgendi, M. Fast QRS detection with an optimized knowledge-based method: Evaluation on 11 standard ECG databases. PLOS ONE 8(9), 1–18 (2013). 12. Elgendi, M. Terma framework for Biomedical signal analysis: An economic-inspired approach. Biosensors 6(4), 55–69 (2016). 13. Elgendi, M., Meo, M. & Abbott, D. A proof-of-concept study: Simple and efective detection of P and T waves in arrhythmic ECG signals. Bioengineering 3(4), 26–40 (2016).


### 信号处理流程：

1. 小波变换：去除噪声和运动伪差
2. TERMA和FrFT融合检测P、QRS和T
3. 机器学习分类

ECG -> （DWT -> 去除基线漂移对应系数 -> IDWT ）-> 高频滤波 -> 滑动均值提取R峰 -> R峰归0 -> 滑动平均 -> 提取P/T峰

#### DWT（离散小波变换）：
用于滤除基线漂移和高频噪声。是一种常用的降噪算法：

x(t)离散小波变换的近似系数与详细系数定义如下：
![[Pasted image 20231203191638.png]]

j >= j0， j0为起始尺度，上φ为缩放函数。下φ为小波函数

反小波变换（IDWT）,用于重建信号
![[Pasted image 20231203191812.png]]



① 先计算中心频率Fc（即基线漂移）其与信号与所选小波之间的相似度有关，范围在0-1；
②心电信号中的Daubichie-4(db4，是一种小波信号https://blog.csdn.net/weixin_47697584/article/details/121953827)的Fc系数（？）最高，约为0.7，接下来使用下面的公式计算每一个pseudo-frequency（伪频率）：
$$ F_a  =  \frac{F_cF_s}{2^a} $$
Fa是基线漂移频率（为0.5），a与Fs是ECG信号的小波分解层数与采样率。对于不同的数据集Fs（MIT_BIH的为360）不同。
$$ 0.5 = 0.7*360/2^9$$

最大分解尺度计算得出为9。
③获取QRS波形：
大部分QRS的波形能量集中在8-20Hz。
需要进行6级分解（以360为中心对半分解6次）。并使用4-6级的详细系数和6级的近似系数重建信号。

1-3级包含：50-100Hz，属于肌肉收缩噪声。舍弃了详细系数，保留近似系数。

![[Pasted image 20231203200546.png]]

### R峰检测

ECG信号最大的频率变化就发生在R峰附近。
①FrFT的数学原理不看了。
![[Pasted image 20231203202324.png]]
当α为0.01时可以适当增强R峰。
②在FrFT之后再进行平方处理进一步增强R峰
③基于事件和周期的两个移动平均值计算：
![[Pasted image 20231203202052.png]]

其中：W1取决于QRS复合波的持续时间，而W2取决于心跳持续时间。
计算增强信号的平均值（µ），并乘以因子（β），该因子的最佳值通过点击试验法选择。
输出数用γ=βµ表示，并添加到MAcyclic中以生成阈值。
将MAevent的每个值与相应的阈值进行比较。如果MAevent（n）大于第n个阈值，则分配一个。否则，将在新矢量中指定零。这样，就产生了一系列不均匀的矩形脉冲。最后，宽度等于W1的脉冲是包含所需事件的块。

### P和T峰检测
① 去除检测出的R峰
② 使用FrFT对P和T进行增强
③ 使用移动平均滤波，类似于检测R的方法对P和T进行检测

![[Pasted image 20231203203055.png]]
检测结果如下：
![[Pasted image 20231203203214.png]]
一般P波维持时间：100±20ms；QT间隔为400±40ms。

最后使用阈值检测（但这个阈值的生成方式和R峰的有所不同，更加简单），加上对信号时间的检测就可以判定P和T峰。
但不适用于T峰叠加U波。


#### 特征提取的机器学习

① 特征提取：
从ECG中提取的不同特征：比如R峰的间隔、ECG的AR（自回归模型）等等
其他可以提取的特征：小波变化系数、均值、年龄、方差、性别和累计值
用于对ECG进行CVD（心脑血管疾病）分类。
数据集：MIT-BIHGeorge, M., & Roger, M. MIT-BIH arrhythmia database. https://www.physionet.org/content/mitdb/1.0.0/.
和：
Zheng, J. A 12-lead Electrocardiogram Database for Arrhythmia Research covering more than 10,000 Patients (2019). https://fgsh are.com/collections/ChapmanECG/4560497/2.

② 特征矩阵
根据上述特征写成矩阵

③ 监督的机器学习
SVM：对于高维空间大于样本数时非常有效
MLP：ANN。


### 算法结果与对比
略

## A generalizable electrocardiogram-base....
### Abstract
一种HF（心力衰竭）检测模型。

采用ARIC建立6个比较模型：
1. 预测HF风险的ECG-AI模型
2. 其他略
3. 预测和区分10年内准确率最高的模型：ECG-AI-cox



首先提取ECG信号的特征：
来自 ARI C 数 据的 12 个 导联 的 P、 Q、 R、S 和 T 波的 持续 时 间和 幅度 相 关的 2 88 个波 形 EC G 特征 。
QRS 复合 体的 持 续时 间
ST 段的 最小 振幅 和 最大 振幅
S T 仰 角
J 点处 的 S T 电平
ST 段 范围 的中 间
P-、 R-和 t 波 轴
PR、 Q T 间期 (和 校正 后 的 Q T 间期) 和 QT 间期 指 数(由 Q T 间 期 延长 的百 分 比推 导 而来)

数据集：ARIC数据集，

特征：从GE MUSEv7中直接导出
缺点：**ECG信号的特征需要非常精确地检测每个波的起始与偏移，但这样的计算因供应商而异，并且容易受噪声干扰，但在深度学习的框架中不存在上述问题**


全部都是用网络直接分析。
没有信号提取的部分

## 1.06um 65nm
### Abstract
功能：生物认证、检测心律失常等异常情况

系统框图：
![[Pasted image 20231203155533.png]]

算法框架：
![[Pasted image 20231203155613.png]]


### 滤波

BPF：筛选出了不同的频率范围1-40 5-40 5-50
NRF：1-40，滤掉高频和直流漂移
HPF：对NRF输出进一步滤波
BPF + DIFF + LPF ：对HPF后进一步滤波

四个采样率为256的滤波器输出将作为NN的输入
### R峰检测

HPF的输出将被用于定义ECG波形中的峰值
LPF输出用于定义ECG中R峰检测的动态阈值
R峰检测到之后会从buffer中提取同一时间范围内几路BPF、NRF的输出存于memory中进行离群值检测

### 离群值检测

在认证与检测阶段的离群值检硬件模块复用，但软件处理不同：
认证：去除异常值，用于认证
检测：保留，可能有潜在风险


检测方式：
![[Pasted image 20231203163107.png]]
具体含义略，论文中有解释

### 网络处理

在进网络之前之前先要进行归一化
网络结构如下：
![[Pasted image 20231203163539.png]]

输入层：向量数随滤波器的采样阶段不同
隐藏层：4 * 100
输出层：1146

十倍交叉验证
激活函数：tanh(x)
损失函数：
![[Pasted image 20231203163916.png]]
但看不懂......