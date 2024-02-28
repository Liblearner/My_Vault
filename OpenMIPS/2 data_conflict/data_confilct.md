## 一、流水线中的相关
+ 1. 结构相关：硬件资源无法满足需求，如在一个指令周期内数据与指令需要同时访问存储
+ 2. 数据相关：一条指令依赖于后一条指令的结果
	+ RAW：read after write。指令2需要读指令1写入的数据，但是流水导致指令2在1写入之前就先读。以下讨论5级流水中如何解决：
		+ ①：相邻指令之间存在数据相关
		+ ②：相隔一条指令存在数据相关
		+ ③：相隔两条指令存在数据相关
		+ tips:load指令所导致的问题在这里暂不讨论。
	+ WAR：write after read。指令2需要在指令1读出数据之后才能写，但是在其读之前就写入数据。导致1读出的数据错误。
	+ WAW：write after write。指令2需要在指令1写之后才能写，但是指令2先写，导致数据错误
+ 3. 控制相关

## 二、解决方式
对于数据相关中，RAW成因③的解决方式：
在regfile中即可解决：
```
module regfile(
······
);
always@(*)begin
	······
	end
	//当读与写访问地址相同且都使能的情况下，使用数据旁路即可
	else if(raddar1 == waddr) && (we == `WriteEnable) && (re1 == ReadENable)
	rdata1 <= wdata;
	//reg2的操作同理
end
```
而在相邻或只相隔一条指令的情况下，检测到相关时的解决方式有以下三种：
1. bubble：插入暂停周期，延迟直到顺序执行时数据正确
2. 编译器调度：通过改变指令执行顺序避免数据冲突
3. 数据旁路：数据前推：避免流水线暂停。将计算结果从产生处直接送入其他指令所需处或者所有所需要的功能单元处。
![[Pasted image 20231123200604.png]]

OpenMIPS使用数据旁路方式解决这个问题，修改过后module的连接图如下所示：
![[Pasted image 20231123200842.png]]
①：将执行阶段的结果传递到译码阶段（即对应上上图中第一条竖线），包括写目的寄存器信号wreg_o，地址wd_o，写入寄存器的数据wdata_o。
②：将访存阶段的结果传递到姨妈阶段（即对应第二条），包括写使能，地址与数据。
因此module id需要增加接口用于接收上述数据。
具体修改如下：
```
`include "defines.v"
module id(
    input wire                                      rst,
    input wire[`InstAddrBus]            pc_i,
    input wire[`InstBus]          inst_i,
    //处于执行阶段的指令要写入的目的寄存器信息
    input wire                                      ex_wreg_i,
    input wire[`RegBus]                     ex_wdata_i,
    input wire[`RegAddrBus]       ex_wd_i,
    //处于访存阶段的指令要写入的目的寄存器信息
    input wire                                      mem_wreg_i,
    input wire[`RegBus]                     mem_wdata_i,
    input wire[`RegAddrBus]       mem_wd_i,
    input wire[`RegBus]           reg1_data_i,
    input wire[`RegBus]           reg2_data_i,

    //送到regfile的信息
    output reg                    reg1_read_o,
    output reg                    reg2_read_o,    
    output reg[`RegAddrBus]       reg1_addr_o,
    output reg[`RegAddrBus]       reg2_addr_o,        
    //送到执行阶段的信息
    output reg[`AluOpBus]         aluop_o,
    output reg[`AluSelBus]        alusel_o,
    output reg[`RegBus]           reg1_o,
    output reg[`RegBus]           reg2_o,
    output reg[`RegAddrBus]       wd_o,
    output reg                    wreg_o
);

  

  wire[5:0] op = inst_i[31:26];

  wire[4:0] op2 = inst_i[10:6];

  wire[5:0] op3 = inst_i[5:0];

  wire[4:0] op4 = inst_i[20:16];

  reg[`RegBus]  imm;

  reg instvalid;

    always @ (*) begin  

        if (rst == `RstEnable) begin

            aluop_o <= `EXE_NOP_OP;

            alusel_o <= `EXE_RES_NOP;

            wd_o <= `NOPRegAddr;

            wreg_o <= `WriteDisable;

            instvalid <= `InstValid;

            reg1_read_o <= 1'b0;

            reg2_read_o <= 1'b0;

            reg1_addr_o <= `NOPRegAddr;

            reg2_addr_o <= `NOPRegAddr;

            imm <= 32'h0;          

      end else begin

            aluop_o <= `EXE_NOP_OP;

            alusel_o <= `EXE_RES_NOP;

            wd_o <= inst_i[15:11];

            wreg_o <= `WriteDisable;

            instvalid <= `InstInvalid;    

            reg1_read_o <= 1'b0;

            reg2_read_o <= 1'b0;

            reg1_addr_o <= inst_i[25:21];

            reg2_addr_o <= inst_i[20:16];      

            imm <= `ZeroWord;

          case (op)

            `EXE_ORI:           begin                        //ORI指令

                wreg_o <= `WriteEnable;     aluop_o <= `EXE_OR_OP;

                alusel_o <= `EXE_RES_LOGIC; reg1_read_o <= 1'b1;    reg2_read_o <= 1'b0;        

                    imm <= {16'h0, inst_i[15:0]};       wd_o <= inst_i[20:16];

                    instvalid <= `InstValid;    

            end

            default:            begin

            end

          endcase         //case op

        end       //if

    end         //always

  

    always @ (*) begin

        if(rst == `RstEnable) begin

            reg1_o <= `ZeroWord;        
//如果译码得出要读的寄存器与EX极端要写入的寄存器地址相同
//如果译码得到要读的寄存器与WB阶段要写入的寄存器地址相同
//则表明有数据冲突，此时直接写入
        end else if((reg1_read_o == 1'b1) && (ex_wreg_i == 1'b1)

                                && (ex_wd_i == reg1_addr_o)) begin

            reg1_o <= ex_wdata_i;

        end else if((reg1_read_o == 1'b1) && (mem_wreg_i == 1'b1)

                                && (mem_wd_i == reg1_addr_o)) begin

            reg1_o <= mem_wdata_i;          

      end else if(reg1_read_o == 1'b1) begin

        reg1_o <= reg1_data_i;

      end else if(reg1_read_o == 1'b0) begin

        reg1_o <= imm;

      end else begin

        reg1_o <= `ZeroWord;

      end

    end

//对reg2同理
    always @ (*) begin

        if(rst == `RstEnable) begin

            reg2_o <= `ZeroWord;

        end else if((reg2_read_o == 1'b1) && (ex_wreg_i == 1'b1)

                                && (ex_wd_i == reg2_addr_o)) begin

            reg2_o <= ex_wdata_i;

        end else if((reg2_read_o == 1'b1) && (mem_wreg_i == 1'b1)

                                && (mem_wd_i == reg2_addr_o)) begin

            reg2_o <= mem_wdata_i;          

      end else if(reg2_read_o == 1'b1) begin

        reg2_o <= reg2_data_i;

      end else if(reg2_read_o == 1'b0) begin

        reg2_o <= imm;

      end else begin

        reg2_o <= `ZeroWord;

      end

    end

  

endmodule

```

三、仿真结果
可以正确读写
**Tips:一定要将"inst_mem.data"放在Modelsim目录下其才可以正确读写！**
![[Pasted image 20231123220105.png]]