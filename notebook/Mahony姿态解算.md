
## 前言

讲解帖子：
https://zhuanlan.zhihu.com/p/342703388
官网源码：
https://x-io.co.uk/open-source-imu-and-ahrs-algorithms/
四元数文档：
https://krasjet.github.io/quaternion/quaternion.pdf

融合加速度计，磁力计，陀螺仪共9轴数据，融合结算得到四元数
也可以只融合6轴数据
## 概念
### 四元数
表示形式
![[Pasted image 20240306205248.png]]
运算规则
模长
加减与标量乘法
四元数乘法（不遵守交换律）
![[Pasted image 20240306205558.png]]

四元数表示姿态：
https://krasjet.github.io/quaternion/quaternion.pdf
四元数表示旋转：
![[Pasted image 20240306205822.png]]
四元数表示姿态矩阵：
![[Pasted image 20240306210220.png]]
![[Pasted image 20240306210207.png]]
四元数反解欧拉角：
![[Pasted image 20240306210135.png]]

### 传感器数据融合

如何求解与更新四元数：
四元数关于时间的微分方程
![[Pasted image 20240306211706.png]]

六轴数据计算角度的两种方式：
+ 角速度积分得到角度
	+ 不足：误差容易被放大
+ 加速度正交分解得到角度
	+ 传感器笔记比较敏感，包含高频噪声

**补偿方式：通过加速度计补偿角速度，融合两种数据以获得准确姿态**

计算过程：误差补偿+反解欧拉角




## 代码部分

### 解算流程

核心分为两个线程：CPU温控与VOFA上位机
```C
  /* Create the thread(s) */

  /* definition and creation of TEMP_IMU */

  osThreadStaticDef(TEMP_IMU, imu_temp_control_task, osPriorityRealtime, 0, 1024, TEMP_IMUBuffer, &TEMP_IMUControlBlock);

  TEMP_IMUHandle = osThreadCreate(osThread(TEMP_IMU), NULL);

  

  /* definition and creation of Vofa */

  osThreadStaticDef(Vofa, Vofa_Task, osPriorityIdle, 0, 512, VofaBuffer, &VofaControlBlock);

  VofaHandle = osThreadCreate(osThread(Vofa), NULL);
```

imu_temp_control_task中进行如下任务：

```C
__weak void imu_temp_control_task(void const * argument)

{

  /* init code for USB_DEVICE */

  MX_USB_DEVICE_Init();

  /* USER CODE BEGIN imu_temp_control_task */

    Imu_Init();

  /* Infinite loop */

  for(;;)

  {

    INS_Task();//调用姿态解算

    osDelay(1);

  }

  /* USER CODE END imu_temp_control_task */

}
```

Imu_init()：

```C 
/*EKF初始化
温控初始化
Mahony初始化
*/
void Imu_Init()

{

  IMU_QuaternionEKF_Init(10, 0.001, 10000000, 1, 0.001f,0); //ekf初始化

  PID_init(&imu_temp_pid, PID_POSITION, imu_temp_PID, TEMPERATURE_PID_MAX_OUT, TEMPERATURE_PID_MAX_IOUT);

  Mahony_Init(1000);  //mahony姿态解算初始化

  HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_2);

  while(BMI088_init()){}

    //set spi frequency

//  hspi2.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_4;

//  if (HAL_SPI_Init(&hspi2) != HAL_OK)

//  {

//    Error_Handler();

//  }

}
```
INS_Task();

姿态解算部分：
+ 1s的陀螺仪0漂初始化
姿态解算
+ 陀螺仪读+Mahony初始化
+ EKF解算
	+ 去0漂
	+ 输入六轴，输出更新后的四元数
	+ Mahony姿态解算
	+ 角度计算，获得roll，yaw，pitch


```C
void INS_Task()

{

    static uint32_t count = 0;

  

    // ins update

    if ((count % 1) == 0)

    {

        BMI088_read(gyro, accel, &temp);

        if(first_mahony==0)

        {

          first_mahony++;

          MahonyAHRSinit(accel[0],accel[1],accel[2],0,0,0);  

        }

        if(attitude_flag==2)  //ekf的姿态解算

        {

          gyro[0]-=gyro_correct[0];   //减去陀螺仪0飘

          gyro[1]-=gyro_correct[1];

          gyro[2]-=gyro_correct[2];

          //===========================================================================

            //ekf姿态解算部分

          //HAL_GPIO_WritePin(GPIOC,GPIO_PIN_9,GPIO_PIN_SET);

          IMU_QuaternionEKF_Update(gyro[0],gyro[1],gyro[2],accel[0],accel[1],accel[2]);

          //HAL_GPIO_WritePin(GPIOC,GPIO_PIN_9,GPIO_PIN_RESET);

          //===============================================================================

          //=================================================================================

          //mahony姿态解算部分

          //HAL_GPIO_WritePin(GPIOE,GPIO_PIN_13,GPIO_PIN_SET);

          Mahony_update(gyro[0],gyro[1],gyro[2],accel[0],accel[1],accel[2],0,0,0);

          Mahony_computeAngles(); //角度计算   移植到别的平台需要替换掉对应的arm_atan2_f32 和 arm_asin

          //HAL_GPIO_WritePin(GPIOE,GPIO_PIN_13,GPIO_PIN_RESET);

          //=============================================================================

          //ekf获取姿态角度函数

          pitch=Get_Pitch(); //获得pitch

          roll=Get_Roll();//获得roll

          yaw=Get_Yaw();//获得yaw

          //==============================================================================

        }

        else if(attitude_flag==1)   //状态1 开始1000次的陀螺仪0飘初始化

        {

            //gyro correct

            gyro_correct[0]+= gyro[0];

            gyro_correct[1]+= gyro[1];

            gyro_correct[2]+= gyro[2];

            correct_times++;

            if(correct_times>=correct_Time_define)

            {

              gyro_correct[0]/=correct_Time_define;

              gyro_correct[1]/=correct_Time_define;

              gyro_correct[2]/=correct_Time_define;

              attitude_flag=2; //go to 2 state

            }

        }

    }

// temperature control

    if ((count % 10) == 0)

    {

        // 100hz 的温度控制pid

        IMU_Temperature_Ctrl();

        static uint32_t temp_Ticks=0;

        if((fabsf(temp-Destination_TEMPERATURE)<0.5f)&&attitude_flag==0) //接近额定温度之差小于0.5° 开始计数

        {

          temp_Ticks++;

          if(temp_Ticks>temp_times)   //计数达到一定次数后 才进入0飘初始化 说明温度已经达到目标

          {

            attitude_flag=1;  //go to correct state

          }

        }

    }

    count++;

}
```


### 运算库
MahonyAHRS.h：
姿态解算库
```C
#ifndef MahonyAHRS_h

#define MahonyAHRS_h

void Mahony_update(float gx, float gy, float gz, float ax, float ay, float az, float mx, float my, float mz);

void Mahony_Init(float sampleFrequency);

void MahonyAHRSupdateIMU(float gx, float gy, float gz, float ax, float ay, float az);

void Mahony_computeAngles(void);

void MahonyAHRSinit(float ax, float ay, float az, float mx, float my, float mz);

//获取角度制角度
float getRoll(void);

float getPitch(void);

float getYaw(void);
//获取弧度制角度
float getRollRadians(void);

float getPitchRadians(void);

float getYawRadians(void);

#endif
```

Quaernion.h
EKF应用到四元数的库
```C
typedef struct

{

    uint8_t Initialized;

    KalmanFilter_t IMU_QuaternionEKF;

    uint8_t ConvergeFlag;

    uint8_t StableFlag;

    uint64_t ErrorCount;

    uint64_t UpdateCount;

  

    float q[4];        // 四元数估计值

    float GyroBias[3]; // 陀螺仪零偏估计值

  

    float Gyro[3];

    float Accel[3];

  

    float OrientationCosine[3];

  

    float accLPFcoef;

    float gyro_norm;

    float accl_norm;

    float AdaptiveGainScale;

  

    float Roll;

    float Pitch;

    float Yaw;

  

    float YawTotalAngle;

  

    float Q1; // 四元数更新过程噪声

    float Q2; // 陀螺仪零偏过程噪声

    float R;  // 加速度计量测噪声

  

    float dt; // 姿态更新周期

    mat ChiSquare;

    float ChiSquare_Data[1];      // 卡方检验检测函数

    float ChiSquareTestThreshold; // 卡方检验阈值

    float lambda;                 // 渐消因子

  

    int16_t YawRoundCount;

  

    float YawAngleLast;

} QEKF_INS_t;

  

extern QEKF_INS_t QEKF_INS;

extern float chiSquare;

extern float ChiSquareTestThreshold;

void IMU_QuaternionEKF_Init(float process_noise1, float process_noise2, float measure_noise, float lambda, float dt, float lpf);

void IMU_QuaternionEKF_Update(float gx, float gy, float gz, float ax, float ay, float az);

void IMU_QuaternionEKF_Reset(void);

  

float Get_Pitch(void);//get pitch

float Get_Roll(void);//get roll

float Get_Yaw(void);//get yaw
```