  
数据流图：

![[Untitled.png]]
其中深色的部分为一个flip-flop

PC取值后+4

指令存储器中指令进行译码，得到寄存器地址与源操作数，再送入ALU执行，最后写回即可。

## ori指令详解

ori指令结构：

31-26 25-21 20-16 15-0

指令码 rs地址 rt地址 16bit立即数（进行无符号扩展，即高16bit都置0）

001101

指令用法：

ori rs, rt, immediate

## 结构框图
![[1.png]]

## 编写过程

疑问：

- 注意到在链接ID与IF、ID与EX以及之后阶段中都有寄存器用来传递值。是为了方便后续实现流水线么
    
- 在读写寄存器时，有一下几种情况会为读出的寄存器值赋0：
    
    - 首先是复位，这个好理解
        
    - 其次是raddr为0时读出0，是因为0寄存器
        
- 读写寄存器的enable由谁接入？

```
 always @(*) begin  
     if(rst == `RstEnable) begin  
         rData1 <= 32'b0;  
     end  
     else if(rAddr1 == `RegNumLog2'b0)begin  
         rData1 <= 32'b0;  
     end      
     else if(re1 == `ReadEnable) begin  
         rData1 <= regs[rAddr1];  
     end  
 end
```


- id.v中
    
    - 模块的输入与输出都涉及到Regfile的读写，注意数据通路
        
- 其他模块按照系统框图编写即可
    
- 初次之外，验证还需要实现一个ROM：
    
```
 //使用rom.data来initial ROM  
 initial $readmemh ("rom.data", rom)
```


且需要按字寻址，因此右移之后（通过索引实现）再索引：
```
 inst <= inst_mem[addr[`InstMemNumLog2 + 1 : 2]];
```


其中InstMemBumLog为17，即为地址线宽度。

由此实现读写。但此处的initial只可以用于模拟仿真，通常也不被综合工具支持，因此需要修改这里初始化的方法。

- 最终编写顶层模块，包含openmips与inst_mem。整个系统的输入目前只有clk与rst
    

至此编写过程完毕。

## 验证

验证使用modelsim即可完成。但在考虑要不要结合vivado工程实现。

记录一下第一次编译的158errorhhh：
![[2.png]]

![[3.png]]
需要注意，tb.v同样需要在此处过编译

等等，不再赘述。报错的主要问题集中收集在下：

- include问题所导致，需要为每一个文件设置include路径才可以。
    
- illeague引用：
    
    - always语句块中output不可以用wire,必须用reg
        
    - 组合逻辑assign输出可以用wire
        
![[4.png]]
编译之后根据编译结果进行仿真，报错如下：

![[5.png]]
即编译时无法检查出的模块端口不对应的问题被查出，进行对应即可。

修改后，开始仿真，现将仿真中的问题记录如下：

- 仿真时通过add wave来添加子波形，只能看顶层模块以及子模块中输入输出波形；module内部信号将是xxxx（红色波浪线）
    
- 波形都需要在添加之后再run才可以看到，否则就是Hiz
    

debug过程：

1. 首先可以发现PC功能正确，可以传递正确的Instruction
    
![[6.png]]
但regfile并没有正常写入。

2. 接下来检查ex阶段的输入与输出
    ![[7.png]]

观察得到波形中，alusel与aluop值正确，表明id阶段没有出错，且reg2的值可以正常读取。
![[8.png]]
但reg1值为0，即从regfile中读出的$0值无效（本应为0）。也导致输出的wdata_o始终为0。这一点与书中的现象也不同。需要排查原因

3. 排查原因如下
    ![[9.png]]

由于$0值始终为0，但又不可以用通过initial的方式完成。因此需要在regfile读取时设计判断：

上图中，左为仿真code，右图为实例code，区别仅仅在于对RAddr1的位数描述有区别，左为b右为h。
![[10.png]]
对比发现，原来是在设计regfile时，端口的输入输出属性错误，导致无法正确读写。

 **module内部的input或者inout必须是wire。调用模块时连接输出的端口必须是wire。**
![[11.png]]
修改后，可以正常获得两个操作数。
![[12.png]]
但观察Dataflow，可以看到ex获得了正确的操作数，但没有得到正确的输出。

最终对比之后发现原因，是ALU没有获得正确的操作码。

ORI对应的操作码为·EXE_OR_OP，而非EXE_ORI_OP。

原因：

书写时前后不对应，在id中赋值为EXE_OR_OP，导致运算类型不匹配。

最终修改这些错误后，正确的数据流如下：
![[13.png]]
## vivado实现

新建工程

## 总结

- 复健了部分verilog语法知识
    
- 熟悉了Verilog从设计到前仿的工具链
    
- 熟悉了MIPS CPU内部Dataflow
    
- 此外，发现教材中讲解给出的实现思路与从网上下载得到的code结构不同；从网上