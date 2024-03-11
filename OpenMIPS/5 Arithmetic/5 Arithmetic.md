## 概念
实现所有的算术操作指令；共21条：

+ 简单算术指令：共15条，加减乘除等；
	实现思路：与3 calculate相似，只需要一个时钟周期，修改ID与EX即可

+ 乘累加、乘累减指令：共4条
	乘累加madd、无符号乘累加maddu；表示操作数相乘之后，再与HI、LO寄存器的值相加；
	乘累减msub、无符号乘累减msubu；表示操作数相乘之后，再与HI、LO寄存器的值相减；
	这4条指令都需要做一次乘法与一次加、减法。为了不延长流水线的执行时间，在OpenMIPS设计时采用两个时钟周期完成这类指令，一个时钟周期进行乘法，下一个进行加或减法。
	**涉及到使流水线停止的办法。**

+ 除法指令：共2条，
	有符号除法div与无符号除法divu，OpenMIPS采用试商法完成除法；而对于32位的除法，流水线就至少需要32个时钟周期。因而需要多个时钟周期才能完成。

### 简单算术指令
1. 加减与比较
六条均为R型指令，SPECIAL类，根据功能码进行区分
![[Pasted image 20240207185617.png]]
用法如下：
+ add rd, rs, rt      rd <- rs + rt（如果溢出则产生溢出异常，并且不保存结果）
+ addu rd, rs rt     同上，无符号运算，不检查溢出，总是保存结果到rd
+ sub rd, rs, rt       rd <- rs - rt（如果溢出则产生溢出异常，并且不保存结果）
+ subu rd, rs, rt     同上，无符号运算，不检查溢出，总是保存结果到rd
+ slt rd, rs, rt          rd <- (rs < rt) 比较rs与rt存放的有符号数，小于则存1到rd，否则存0
+ sltu rd, rs, rt        rd <- (rs < rt)同上，但按照无符号数比较，保存结果相同
2. 立即数加减与比较
I型指令，根据指令码判断指令类型

![[Pasted image 20240207190656.png]]
用法如下：
+ addi rt, rs, imm       rt <- rs + (sign_extended)imm，将指令中的16bit立即数进行符号扩展，与地址为rs的值加法，并存到rt，当加法溢出时产生溢出异常，且不保存结果
+ addiu rt, rs, imm     rt <- rs + (sign_extended)imm，将指令中的16bit立即数进行符号扩展，与地址为rs的值加法，并存到rt，但不进行溢出检查，总是保持结果
+ slti rt, rs, imm          rt <- (rs < (sign_extended)imm) 按符号扩展16bit立即数之后，按照有符号数与rs进行比较，小于则存1到rt，否则存0
+ sltiu rt, rs, imm        rt <- (rs < (sign_extended)imm) 按符号扩展16bit立即数之后，按照无符号数与rs进行比较，小于则存1到rt，否则存0
3. clo，clz指令
![[Pasted image 20240207191301.png]]
R型，SPECIAL2类型；根据功能码判断指令类型
+ clz rd, rs        rd <- coun_leading_zeros rs ；作用为从高位开始，检查rs的值，直到某一位为1停止，则将该位之前的0的个数存在rd中
+ clo rd, rs       rd <- coun_leading_ones rs；  作用基本同上，但是检查1的个数并保存

4. 乘法
![[Pasted image 20240207191531.png]]

R型指令，类型为SPECIAL2。用法如下：
+ mul rd, rs, rt   rd <- rs * rt      将rs与rt值作为有符号数相乘，结果的低32bit保存在rd中
+ mult rs, rt       {hi, lo} <- rs * rt，将rs与rt作为有符号数相乘，保存结果到HI与LO，分别存高32bit与低32bit
+ multu rs, rt     {hi, lo} <- rs * rt   将rs与rt作为无符号数相乘，保存结果到HI与LO中，分别存高32bit与低32bit

实现思路：没有太多改动，只需要在ID与EX阶段加入对应的操作即可
如ID阶段：
![[Pasted image 20240207201009.png]]
完整译码如下：
  wire[5:0] op = inst_i[31:26];

  wire[4:0] op2 = inst_i[10:6];

  wire[5:0] op3 = inst_i[5:0];

  wire[4:0] op4 = inst_i[20:16];
 case(inst[31:26])
	 SPECIAL:
		 case(inst[10:6])
			 5'b00000:
				 case(inst[5:0]):
					 OR
					 AND
					 XOR
					 NOR
					 SLLV
					 SRLV
					 SRAV
					 MFHI
					 MFLO
					 MTHI
					 MTLO
					 MOVN
					 MOVZ
					 SLT
					 SLTU
					 ADD
					 ADDU
					 SUB
					 SUBU
					 MULT
					 MULTU
				default:
	ORI:
	ANDI:
	XORI:
	LUI:
	SLTI:
	SLTIU:
	ADDI:
	ADDIU:
	SPECIAL2:
		case(inst[5:0])
			CLZ:
			CLO:
			MUL:

### 流水线暂停机制
为后续的乘累加、乘累减、除法等占用多个时钟周期的指令做准备
暂停方式有以下两种：
+ 直观：保持PC值不变，也保持流水线寄存器的值不变（IF/ID、ID/EX、EX/MEM、MEM/WB）
+ 改进：保持PC值不变，保持流水线第n阶段、第n阶段之前的各个阶段寄存器不变，第n阶段之后的指令继续运行。比如在EX阶段请求暂停，则EX及之前的寄存器不变，但允许MEM与WB继续运行

增加ctrl模块，用于接受各阶段传递来的流水线暂停信号，控制流水线阶段运行
输入：
ID与EX阶段的请求暂停信号。IF、MEM与WB的不需要因为其都可以在一个指令周期之内完成
输出：
流水线暂停信号stall，输出到PC、IF/ID、ID/EX等流水线寄存器模块；从而进行控制。
![[Pasted image 20240228211811.png]]

### 乘累命令
共4条：madd、maddu、msub、msubu；格式如下
![[Pasted image 20240229192759.png]]
+ madd rs, rt  {HI, LO} <= {HI, LO} + rs * rt。有符号数乘法
+ maddu rs, rt {HI, LO} <= {HI, LO} + rs * rt。 无符号数乘法
+ msub rs, rt {HI, LO} <= {HI, LO} - rs * rt。有符号数乘法
+ msubu rs, rt {HI, LO} <= {HI, LO} - rs * rt。无符号乘法

实现时分两个周期进行运算。第一个周期完成乘法，第二个周期完成加减法、
需要保存两个信息：
+ 当前是此命令的第几个周期 cnt
+ 乘法结果 hilo_temp
通过在流水线暂停时，将计算结果送出，并使用EX内部的计数器控制运算
![[Pasted image 20240229193304.png]]


### 除命令
![[Pasted image 20240229212247.png]]
+ div rs, rt {HI, LO} <- rs/rt。有符号除法
+ divu rs, rt {HI, LO} <- rs/rt。无符号除法

OpenMIPS使用试商法，对于32位的除法，至少需要32个时钟周期才能得到除法结果。算法过程描述略，类似于竖式计算过程：
![[Pasted image 20240229213057.png]]

![[Pasted image 20240229213151.png]]
需要新增模块以完成试商法的实现
![[Pasted image 20240229213521.png]]

## 实现
### 基本算数指令的计算如下。
指令基本使用verilog的运算就可以实现。注意以下几点就可以：
+ 在减法以及slt等用减法实现的指令用到了reg2的补码
+ 溢出的标志位
+ clo与clz指令使用嵌套的三元表达式实现的。且用到了reg1的反码
+ 乘法结果要存储在HI与LO中，因此需要一个寄存器来暂存
+ 在有符号乘法实现时，注意在乘法结果传递之后需要取补码以还原


```verilog
  
  

    reg[`RegBus] logicout;

    reg[`RegBus] shiftres;

    reg[`RegBus] moveres;

    reg[`RegBus] HI;

    reg[`RegBus] LO;

  

    //运算用的变量

    wire ov_sum;                    //保存溢出情况

    wire reg1_eq_reg2;              //第一个操作数是否等于第二个操作数

    wire reg1_lt_reg2;              //第一个操作数是否小于第二个操作数

    reg[`RegBus] arithmeticres;     //算数运算结果

    wire[`RegBus] reg2_mux;         //保存输入的第二个操作数reg2的补码

    wire[`RegBus] reg1_not;         //保存输入的第一个操作数reg1取反的值

    wire[`RegBus] result_sum;       //保存假发结果

    wire[`RegBus] opdata1_mult;     //乘法中的被乘数

    wire[`RegBus] opdata2_mult;     //乘法中的乘数

    wire[`DoubleRegBus] hilo_temp;  //临时保存乘法结果，宽度64bit

    reg[`DoubleRegBus] mulres;      //保存乘法结果，宽度64bit

  
  

//第一阶段，根据aluop码进行运算或处理,logic,shift.move与arithmetic分always块

  
  

//取补码的情况：减法，此外SLT的执行也用到了减法

assign reg2_mux = ((aluop == `EXE_SUB_OP) || (aluop == `EXE_SUBU_OP) ||

                    (aluop == `EXE_SLT_OP))?(~reg2) + 1 : reg2;

  

//求和

assign result_sum = reg1 + reg2_mux;

  

//求和溢出的情况：负数+负数=正数，正数+正数=负数

assign ov_sum = ((!reg1[31] && !reg2_mux[31] && result_sum[31])) ||

                (reg1[31] && reg2[31] && result_sum[31]);

  

//比较大小运算,不是SLT运算的部分是？

assign reg1_lt_reg2 = (aluop == `EXE_SLT_OP)?

                        ((reg1[31] && !reg2[31])||(!reg1[31] && !reg2[31] && result_sum[31])

                        ||(reg1[31] && reg2[31] && result_sum[31])) : (reg1 < reg2);

  

assign reg1_not = ~reg1;

  
  

always @(*) begin

    if(rst == `RstEnable)begin

        arithmeticres <= 32'b0;

    end

    else begin

        case (aluop):

        `EXE_SLT_OP, `EXE_SLTU_OP:

            arithmeticres <= reg1_lt_reg2;

        `EXE_ADD_OP, `EXE_ADDU_OP,`EXE_ADDI_OP,`EXE_ADDIU_OP:

            arithmeticres <= result_sum;

        `EXE_SUB_OP, `EXE_SUBU_OP:

            arithmeticres <= result_sum;

        `EXE_CLZ_OP://数0的个数，到1停止

            arithmeticres <= (reg1[31] ? 0 : reg1[30] ? 1 : reg1[29] ? 2:

            reg1[28] ? 3 : reg1[27] ? 4 : reg1[26] ? 5 : reg1[25] ? 6 :

            reg1[24] ? 7 : reg1[23] ? 8 : reg1[22] ? 9 : reg1[21] ? 10:

            reg1[20] ? 11 : reg1[19] ? 12 : reg1[18] ? 13 : reg1[17] ? 14:

            reg1[16] ? 15 : reg1[15] ? 16 : reg1[14] ? 17 : reg1[13] ? 18:

            reg1[12] ? 19 : reg1[11] ? 20 : reg1[10] ? 21 : reg1[9] ? 22:

            reg1[8] ? 23 : reg1[7] ? 24 : reg1[6] ? 25 : reg1[5] ? 26 :

            reg1[4] ? 27 : reg1[3] ? 28 : reg1[2] ? 29: reg1[1] ? 30 :

            reg1[0] ? 31 : 32);

        `EXE_CLO_OP://数1的个数，到0停止，可以利用取非来判断

            arithmeticres <= (reg1_not[31] ? 0 : reg1_not[30] ? 1 : reg1_not[29] ? 2:

            reg1_not[28] ? 3 : reg1_not[27] ? 4 : reg1_not[26] ? 5 : reg1_not[25] ? 6 :

            reg1_not[24] ? 7 : reg1_not[23] ? 8 : reg1_not[22] ? 9 : reg1_not[21] ? 10:

            reg1_not[20] ? 11 : reg1_not[19] ? 12 : reg1_not[18] ? 13 : reg1_not[17] ? 14:

            reg1_not[16] ? 15 : reg1_not[15] ? 16 : reg1_not[14] ? 17 : reg1_not[13] ? 18:

            reg1_not[12] ? 19 : reg1_not[11] ? 20 : reg1_not[10] ? 21 : reg1_not[9] ? 22:

            reg1_not[8] ? 23 : reg1_not[7] ? 24 : reg1_not[6] ? 25 : reg1_not[5] ? 26 :

            reg1_not[4] ? 27 : reg1_not[3] ? 28 : reg1_not[2] ? 29: reg1_not[1] ? 30 :

            reg1_not[0] ? 31 : 32);

        default:

            arithmeticres <= 32'b0;

        endcase

    end

end

  

//取得乘法操作的操作数，如果是有符号除法且操作数是负数，则取反+1

assign opdata1_mult = (((aluop == `EXE_MUL_OP) || (aluop == `EXE_MULT_OP))

                        && (reg1[31] == 1'b1)) ? (~reg1 + 1) : reg1;

assign opdata2_mult = (((aluop == `EXE_MUL_OP) || (aluop == `EXE_MULT_OP))

                        && (reg2[31] == 1'b1)) ? (~reg2 + 1) : reg2;

assign hilo_temp = opdata1_mult * opdata2_mult;

  

always@(*) begin

    if(rst == `RstEnable)

        mulres <= 64'b0;

    else if((aluop == `EXE_MUL_OP) || (aluop == `EXE_MULT_OP))begin

        //最高位异或，不一样说明是异号相乘

        if(reg1[31] ^ reg2[31] == 1'b1)

            mulres <= ~hilo_temp + 1;

        else

            mulres <= hilo_temp;

    end

    else

        mulres <= hilo_temp;

end

  
  
  

always@(*) begin

    if(rst == `RstEnable)begin

    logicout <= 32'b0;

    end

    else begin

        //同类型带I与不带I的可以合并，LUI也可以合并，节省硬件开销

        case(aluop)

        `EXE_OR_OP:begin

            logicout <= reg1 | reg2;

        end

        `EXE_ORI_OP:begin

            logicout <= reg1 | reg2;

        end

        `EXE_AND_OP:begin

            logicout <= reg1 & reg2;

        end

        `EXE_ANDI_OP:begin

            logicout <= reg1 & reg2;

        end

        `EXE_NOR_OP:begin

            logicout <= ~(reg1 & reg2);

        end

        `EXE_XOR_OP:begin

            logicout <= reg1 ^ reg2;

        end

        `EXE_XORI_OP:begin

            logicout <= reg1 ^ reg2;

        end

        `EXE_LUI_OP:begin

            logicout <= reg1 | reg2;//reg1是$0，reg2高16bit是寄存器指令中值

        end

        default:begin

            logicout <= 32'b0;

        end

        endcase

    end

end

  

always@(*) begin

    if(rst == `RstEnable)begin

    shiftres <= 32'b0;

    end

    else begin

        case(aluop)

        `EXE_SLL_OP:begin

            shiftres <= reg2 << reg1[4:0];

        end

        `EXE_SLLV_OP:begin

            shiftres <= reg2 << reg1[4:0];

        end

        `EXE_SRL_OP:begin

            shiftres <= reg2 >> reg1[4:0];

        end

        `EXE_SRLV_OP:begin

            shiftres <= reg2 >> reg1[4:0];            

        end

        `EXE_SRA_OP:begin//算术移略显特殊

            shiftres <= ({32{reg2[31]}} << (6'd32-{1'b0, reg1[4:0]})) | reg2 >> reg1[4:0];

        end

        `EXE_SRAV_OP:begin

            shiftres <= ({32{reg2[31]}} << (6'd32-{1'b0, reg1[4:0]})) | reg2 >> reg1[4:0];      

        end

        default:begin

            shiftres <= 32'b0;

        end

        endcase

    end

end

  

always @ (*) begin

    if(rst == `RstEnable) begin

    moveres <= 32'b0;

  end else begin

    moveres <= 32'b0;

   case (aluop)

    `EXE_MFHI_OP:       begin

        moveres <= HI;

    end

    `EXE_MFLO_OP:       begin

        moveres <= LO;

    end

    `EXE_MOVZ_OP:       begin

        moveres <= reg1;

    end

    `EXE_MOVN_OP:       begin

        moveres <= reg1;

    end

    default : begin

    end

   endcase

  end

end  

  

//得到最新的HI、LO寄存器的值，并在这里解决指令数据相关问题

always @(*) begin

    if(rst == `RstEnable) begin

        {HI,LO} <= {32'b0, 32'b0};

    end

    else if(mem_whilo_i == `WriteEnable) begin

        {HI,LO} <= {mem_hi_i,mem_lo_i};

    end

    else if(wb_whilo_i == `WriteEnable) begin

        {HI,LO} <= {wb_hi_i,wb_lo_i};

    end

    else

        {HI,LO} <= {hi_i,lo_i};

end

  
  
  
  

//第二阶段，根据alusel制定类型选择一个结果作为最终结果

always @(*) begin

    wd_o <= wd_i;

    wreg_o <= wreg_i;

  

    if(((aluop == `EXE_ADD_OP) || (aluop == `EXE_ADDI_OP) ||

            (aluop == `EXE_SUB_OP)) && (ov_sum == 1'b1))

        wreg_o <= `WriteDisable;

    else

        wreg_o <= wreg_i;

  

    case(alusel)

    `EXE_RES_LOGIC:begin

        wdata <= logicout;

    end

    `EXE_RES_SHIFT:begin

        wdata <= shiftres;

    end

    `EXE_RES_MOVE:begin

        wdata <= moveres;

    end

    `EXE_RES_ARITHMETIC:begin

        wdata <= arithmeticres;

    end

    `EXE_RES_MUL:begin

        wdata <= mulres[31:0];

    end

    default:

        wdata <= 32'b0;

    endcase

end

  

//对于MTHI与MTLO，实现whilo_o与HI与LO的写输出

always @ (*) begin

    if(rst == `RstEnable) begin

        whilo_o <= `WriteDisable;

        hi_o <= 32'b0;

        lo_o <= 32'b0;      

    end else if((aluop == `EXE_MULT_OP) || (aluop == `EXE_MULTU_OP)) begin

        whilo_o <= `WriteEnable;

        hi_o <= mulres[63:32];

        lo_o <= mulres[31:0];

    end else if(aluop == `EXE_MTHI_OP) begin

        whilo_o <= `WriteEnable;

        hi_o <= reg1;

        lo_o <= LO;

    end else if(aluop == `EXE_MTLO_OP) begin

        whilo_o <= `WriteEnable;

        hi_o <= HI;

        lo_o <= reg1;

    end else begin

        whilo_o <= `WriteDisable;

        hi_o <= 32'b0;

        lo_o <= 32'b0;

    end            

end
```


### 流水线暂停机制
ctrl实现如下：

![[Pasted image 20240228212628.png]]

PC修改如下：
Q：这样的写法不会生成锁存器？还是说这里保持PC的值不变也会生成锁存器？
```verilog
//then for the behavior of pc,reset or plus 4

always @(posedge clk) begin

    if(ce == `ChipDisable)

        pc <= 32'b0;

    else if(stall[0] == `NoStop)

        pc <= pc+4'h4;

  

end
```
IF/ID修改如下：
疑问同上，所有的流水线寄存器修改都类似
```verilog

else if(stall[1] == `Stop && stall[2] == `NoStop)begin

    id_pc <= `ZeroWord;

    id_inst <= `ZeroWord;

end

else if(stall[1] == `NoStop)begin

    id_pc <= if_pc;

    id_inst <= if_inst;

end

end
```

### 累乘
ID 部分：
```verilog
                    `MADD:begin

                    aluop_o <= `EXE_MADD_OP;

                    alusel_o <= `EXE_RES_MUL;

                    reg1_read_o <= 1'b1;

                    reg2_read_o <= 1'b1;

                    wreg_o <= `WriteDisable;

                    instvaild <= `Instvaild;

                    end

                    `MADDU:begin

                    aluop_o <= `EXE_MADDU_OP;

                    alusel_o <= `EXE_RES_MUL;

                    reg1_read_o <= 1'b1;

                    reg2_read_o <= 1'b1;

                    wreg_o <= `WriteDisable;

                    instvaild <= `Instvaild;

                    end

                    `MSUB:begin

                    aluop_o <= `EXE_SUB_OP;

                    alusel_o <= `EXE_RES_MUL;

                    reg1_read_o <= 1'b1;

                    reg2_read_o <= 1'b1;

                    wreg_o <= `WriteDisable;

                    instvaild <= `Instvaild;

                    end

                    `MSUBU:begin

                    aluop_o <= `EXE_SUBU_OP;

                    alusel_o <= `EXE_RES_MUL;

                    reg1_read_o <= 1'b1;

                    reg2_read_o <= 1'b1;

                    wreg_o <= `WriteDisable;

                    instvaild <= `InstVaild;

                    end
```
EX部分：
增加了指令周期计数与乘法结果暂存接口
使用流水线暂停以实现多周期运算：
```verilog
always @(*) begin

    if(rst == `RstEnable)begin

        hilo_temp_o <= {`ZeroWord, `ZeroWord};

        cnt_o <= 2'b00;

        stallreq_for_madd_msub <= `NoStop;

    end

    else begin

        case(aluop)

        `EXE_MADD_OP, `EXE_MADDU_OP:begin

            //双周期计算的第一个周期

            if(cnt_i == 2'b00) begin

                hilo_temp_o <= mulres;

                cnt_o <= 2'b01;

                hilo_temp1 <= {`ZeroWord, `ZeroWord};

                stallreq_for_madd_msub <= `Stop;

            end

            //双周期计算的第二个周期

            else if(cnt_i == 2'b01) begin

                hilo_temp_o <= {`ZeroWord, `ZeroWord};

                cnt_o <= 2'b10;

                hilo_temp1 <= hilo_temp_i + {HI, LO};

                stallreq_for_madd_msub <= `NoStop;

            end

        end

        `EXE_MSUB_OP, `EXE_MSUBU_OP:begin

            if(cnt_i == 2'b00) begin

                hilo_temp_o <= ~mulres + 1;

                cnt_o <= 2'b01;

                hilo_temp1 <= {`ZeroWord, `ZeroWord};

                stallreq_for_madd_msub <= `Stop;

            end

            else if(cnt_i == 2'b01)begin

                hilo_temp_o <= {`ZeroWord, `ZeroWord};

                cnt_o <= 2'b01;

                hilo_temp1 <= hilo_temp_i + {HI, LO};

                stallreq_for_madd_msub <= `Stop;

            end

        end

        default:begin

            hilo_temp_o <= {`ZeroWord, `ZeroWord};

            cnt_o <= 2'b00;

            stallreq_for_madd_msub <= `NoStop;

        end

        endcase

    end

end
```
EX/MEM的改动如下：
在流水线暂停时，将输入信号通过输出接口hilo_o送出，输入信号cnt_i也从输出接口cnt_o送出
其余时刻置0
```verilog
    always @(posedge clk) begin

        if(rst == `RstEnable) begin

        mem_wd <= `RegNopAddr;

        mem_wreg <= 1'b0;

        mem_wdata <= `RegNopData;

  

        mem_hi <= 32'b0;

        mem_lo <= 32'b0;

        mem_whilo <= 1'b0;

  

        hilo_o <= {`ZeroWord, `ZeroWord};

        cnt_o <= 2'b00;

        end

  

        else if(stall[3] == `Stop && stall[4] == `NoStop)begin

        mem_wd <= `RegNopAddr;

        mem_wreg <= `WriteDisable;

        mem_wdata <= `ZeroWord;

  

        mem_hi <= `ZeroWord;

        mem_Lo <= `ZeroWord;

        mem_whilo <= `WriteDisable;

  

        hilo_o <= hilo_i;

        cnt_o <= cnt_i;

        end

        else if(stall[3] == `NoStop)begin

        mem_wd <= ex_wd;

        mem_wdata <= ex_wdata;

        mem_wreg <= ex_wreg;

  

        mem_hi <= ex_hi;

        mem_lo <= ex_lo;

        mem_whilo <= ex_whilo;

  

        hilo_o <= hilo_i;

        cnt_o <= cnt_i;

        end

    end
```
### 除法
新模块，试商法实现32bit除法运算



## 仿真
![[Pasted image 20240304221250.png]]
2024.3.4
结果有，但是部分信号x，后面再查查问题