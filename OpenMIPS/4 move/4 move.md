## 概念
共6条移动指令；后面四条涉及到了特殊寄存器LO、HI的使用；
![[Pasted image 20240205172052.png]]
6条均为R形指令；且指令码均为6'b000000，都是SPECIAl类型；功能码各不相同。
HI、LO的用途：在乘除法中用于保存结果的高位低位，以及余数与商；
各个指令用途如下：
+ movn rd, rs, rt 当rt值作为地址的寄存器的值非0时，将rs的值传递到rd中；（move conditional on Not zero）
+ movz rd, rs, rt 当rt只作为地址的寄存器的值为0时，将rs的值传递到rd中；（move conditional on zero）
+ mfhi rd 将HI的值传递给rd
+ mflo rd 将LO的值传递给rd
+ mthi rd 将rd的值传递给HI
+ mtlo rd 将rd的值传递给LO

实现思路：
1. 第1/2条指令与运算阶段的实现类似；
	+ ID：给出运算类型、子类型，写寄存器的地址，读取通用寄存器的值；**并判断其是否为0**
	+ EX：确定写入寄存器的信息，是否写，写入地址，写入的值
	+ 传递到MEM与WB阶段；最终决定对寄存器的修改
2. 第5、6条指令写特殊寄存器HI、LO：
	+ ID：给出运算类型、子类型，**写寄存器地址与写使能都关闭，因为其不在通用寄存器组**；读出rs的值
	+ EX：确定HI、LO的值，传递信息
	+ MEM与WB阶段：修改HI/LO
3. 第3/4条指令读特殊寄存器HI、LO
	+ ID：给出运算类型、运算子类型，写寄存器rd的地址
	+ EX：获取HI、LO的值；并将这些信息传递
	+ MEM、WB：根据信息修改HI、LO


新增之后的数据流图：
![[Pasted image 20240205173044.png]]
+ 增加了HI、LO模块，用于选择要参与运算的数据，选择是否从HI与LO来的数据

在涉及到HI、LO之后，也可能会出现类似于之前数据冲突的问题；因此需要再后面的写回阶段添加判断条件，使用数据旁路避免冲突。
问题举例：
![[Pasted image 20240205173951.png]]
如图，在执行（读）与访存（写）之间发生RAW冲突；因此需要将访存于回写阶段的信息反馈到执行阶段；判断上一条执行的是不是mfhi/mflo指令，如果是则直接从上一条指令中获取值。

修改之后的数据流图与系统框图如下：
![[Pasted image 20240205174155.png]]
![[Pasted image 20240205174202.png]]

特殊寄存器单独用一个module实现、

## 仿真
过程略；
注意点如下：
1. movz与movn判断0是在ex阶段进行判定，根据读出的reg值可以得出
2. 各个连接处信号的命名：不包括流水线过渡阶段的几个模块，以几个主要阶段进行命名。流水线阶段+信号名称+i/o
3. HI与LO寄存器的实现是在EX阶段，而HILO模块只负责在WB阶段判断是否对HI与LO进行读写
![[Pasted image 20240206214934.png]]