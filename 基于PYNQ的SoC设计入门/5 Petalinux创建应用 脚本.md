## Vitis开发Linux应用

Vitis中选择Create Platform Project

在选项中配置硬件 软件基础系统 平台信息 也有

BIF文件

编译完成后使用elf文件运行即可，有三种方式在linux上运行.elf。但三种方式都需要板子ping通虚拟机：

1. TCF Agent：类似于debug的形式，在vitis终端中调试
2. NFS共享方式：通过NFS共享的形式将虚拟机的.elf共享到板子运行
3. SSH方式：即将.elf直接上传到板子，在板子中使用./xxx.elf执行即可

## 获取交叉编译链

交叉编译链：

简单理解即为跨架构与OS的编译链

Linux嵌入式开发所需要的便是用Petalinux构建SDK，安装SDK。

- 首先是定制根文件系统，各种库文件便存在于根文件系统中
    
    根文件系统的要求：
    
    SD卡格式分区 第二分区为EXT4
    
    根文件系统配置命令为：
    
    ```bash
    petalinux-config -c rootfs
    ```
    
- 接下来在界面中添加对应的包即可
    
    在Petalinux Package Groups下进行选择接口。
    
    包中有无后缀、后缀为dev和opt的版本，选择无后缀版即可。
    
- 最后根文件系统需要编译以生成SDK
    
    ```bash
    petalinux-build --sdk
    ```
    
    构建出sdk的安装文件./sdk.sh 并在petalinux工程目录的linux/image下进行安装默认安装环境在/opt/petalinux/2020.2下
    
    之后在每一个新终端中使用sdk时都需要执行
    
    ```bash
    ./opt/petalinux/2020.2/environment-setup-cortexa9t2f-neon-xilinx-linux-gnueabi
    ```
    
    安装sdk之后便获得了arm交叉编译工具链 从此之后可以直接使用此交叉编译链编译ubootlinux内核 linux应用等等