## 原理
使用系统：
	线性（叠加性与齐次性）+高斯（噪声正态分布）

宏观意义：
	滤波即加权
	低通滤波：实际信号 = 信号 * 1 + 噪声 * 0
	卡尔曼：    修正信号 = 估计值 * k1 + 观测值 * k2
	选择估计值与观测值的权重
状态空间表达式：
	状态方程：$$x_k = Ax_{k-1} + Bu_k + w_k$$
	其中：
	x是我们所关心的状态量
	u是我们施加的量（输入）
	w是过程噪声
	A与B表示某种作用残留或者产生的影响
	观测方程：
	$$y_k = Cx_k + v_k$$
	其中：
	y是观测量
	v是观测器的误差（噪声）
	![[2660911710337000..jpg]]

	参数分析：

	![[B24CBF163E0DDA831AD798E4A61569C9.png]]
	w与v分别满足以0为均值，Q、R为标准差的正态分布
	协方差（两个变量之间变化趋势相同程度的大小，正表示同时增大，负表示同时变小）

超参数：
	主要需要调这两个：过程噪声与观测噪声Q、R
直观图解：
	最优估计值（黑色）、先验估计值（黄色）、观测值（紫色）
	![[AB7C521551E6F0DB53FCFEC5E6DC56D1.jpg]]
		$$\hat{X}_{k-1}：上一时刻的最优估计值，上一时刻的卡尔曼滤波值$$
$$\hat{X}_k-：（黄色）根据上一个时刻的最优估计值得出的本时刻估计值，y_k：观测值$$

而后得到当前时刻的最优估计值x_k
**先由上一时刻最优估计值预测得出这一时刻先验估计值，在用观测值修正**
预测与更新过程（黄金五式）

![[40F7DAE4A7048631AF27599DFD320445.jpg]]
以一个匀加速直线运动来说明：
1.  先验估计值
	![[26B75FA0B44C9728E61D06A9A45229AD.jpg]]
	F：状态转移矩阵
	B：控制矩阵
	u_{t-1}：外力作用
2. 先验估计协方差
	![[1241809AEB165C3F3F3CFEAD7D3374FE.jpg]]
	Q：过程噪声的协方差矩阵
3. 测量方程
	![[76AD0B6031667B905DF72925CF978AC1.jpg]]
	H：测量矩阵
4. 更新的三个公式不用推导


参数调节：
更信任观测值（传感器精度）：Q增大，R减小（导致K增大）
更信任被观测对象运动过程（平滑无摩擦等）：Q减小，R增大（导致K减小）
Q：过程噪声；R：观测噪声
P_0（状态协方差矩阵）与x_0（初始状态）取值：
一般x_0取0，P往小取，方便收敛

卡尔曼滤波的使用：
+ 选择状态量与观测量
+ 构建方程
+ 初始化参数
+ 带入公式迭代
+ 调节超参数


## 使用例子
### 陀螺仪



### 装甲板跟随





## C实现
KF结构体
```C
typedef struct kf_t

{

    float *FilteredValue;

    float *MeasuredVector;

    float *ControlVector;

  

    uint8_t xhatSize;

    uint8_t uSize;

    uint8_t zSize;

  

    uint8_t UseAutoAdjustment;

    uint8_t MeasurementValidNum;

  

    uint8_t *MeasurementMap;      // 量测与状态的关系 how measurement relates to the state

    float *MeasurementDegree;     // 测量值对应H矩阵元素值 elements of each measurement in H

    float *MatR_DiagonalElements; // 量测方差 variance for each measurement

    float *StateMinVariance;      // 最小方差 避免方差过度收敛 suppress filter excessive convergence

    uint8_t *temp;

  

    // 配合用户定义函数使用,作为标志位用于判断是否要跳过标准KF中五个环节中的任意一个

    uint8_t SkipEq1, SkipEq2, SkipEq3, SkipEq4, SkipEq5;

  

    // definiion of struct mat: rows & cols & pointer to vars

    mat xhat;      // x(k|k)

    mat xhatminus; // x(k|k-1)

    mat u;         // control vector u

    mat z;         // measurement vector z

    mat P;         // covariance matrix P(k|k)

    mat Pminus;    // covariance matrix P(k|k-1)

    mat F, FT;     // state transition matrix F FT

    mat B;         // control matrix B

    mat H, HT;     // measurement matrix H

    mat Q;         // process noise covariance matrix Q

    mat R;         // measurement noise covariance matrix R

    mat K;         // kalman gain  K

    mat S, temp_matrix, temp_matrix1, temp_vector, temp_vector1;

  

    int8_t MatStatus;

  

    // 用户定义函数,可以替换或扩展基准KF的功能

    void (*User_Func0_f)(struct kf_t *kf);

    void (*User_Func1_f)(struct kf_t *kf);

    void (*User_Func2_f)(struct kf_t *kf);

    void (*User_Func3_f)(struct kf_t *kf);

    void (*User_Func4_f)(struct kf_t *kf);

    void (*User_Func5_f)(struct kf_t *kf);

    void (*User_Func6_f)(struct kf_t *kf);

    // 矩阵存储空间指针

    float *xhat_data, *xhatminus_data;

    float *u_data;

    float *z_data;

    float *P_data, *Pminus_data;

    float *F_data, *FT_data;

    float *B_data;

    float *H_data, *HT_data;

    float *Q_data;

    float *R_data;

    float *K_data;

    float *S_data, *temp_matrix_data, *temp_matrix_data1, *temp_vector_data, *temp_vector_data1;

} KalmanFilter_t;
```

