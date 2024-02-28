
一、指令说明：
1. 逻辑操作： and or xor nor
R型，rd<-rs AND rt
![[Pasted image 20231124161648.png]]
I型，rt <- rs ANDI (zero_extended)imm
![[Pasted image 20231201160746.png]]
![[Pasted image 20231124161843.png]]
lui，同样也是I型，将16bit立即数保存在rt高16bit，低16bit用0填充

![[Pasted image 20231124162008.png]]

移位操作： sll sllv sra srav srl srlv
都是R型，指令码为6'b000000
![[Pasted image 20231124162203.png]]
sll（逻辑左移）rt右移sa位，空位用0填充，存于rd。rd<- rt<<sa(logic)
srl（逻辑右移）同上

sllv  (逻辑左移)：rt右移rs[4:0]位，空位0填充，存于rd。rd<- rt>>rs:[4:0] (logic)
srlv（逻辑右移）：同上
tips：没有算术左移
sra（算术右移）：rt右移sa位，空位用rt[31:0]值填充，存于rd。rd<- rt>>sa(arithmetic)
srav（算术右移）：rt右移rs[4:0]，空位用rt[31]填充，存于rd。rd <- rt>>rs[4:0](arithmetic) 
2. 空指令：
nop ssnop（多指令发射CPU中确保单独占用一个发射周期，但对于OpenMIPS这个标量处理器来说是一样的）
sync（用于保证加载、存储操作的顺序，但对于OpenMIPS来说其本来就是按照顺序严格执行的。所以也可以按照nop来处理）pref（用于缓存预取，OpenMIPS没有实现缓存，所以也是空指令）
![[Pasted image 20231124162848.png]]
Q&A:
处理nop与ssnop时，其二进制译码与sll $0,$0,0/1一样。
无需特备处理，两者结果都等同于什么都没做。

二、修改说明
1. 修改id模块，从而实现上述指令译码
+ andi、xori的实现不再赘述，和ori类似，仅有alu操作以及选择码上的不同
+ lui指令对立即数的处理略有不同，其将指令中的立即数放在了高16bit，并送往ALU进行or操作
+ 关于and/or/xor等非立即数的操作指令，涉及到了一个case的包含关系：

```
            `SPECIAL:begin

            begin

                //书中先直接对inst[10:6]进行了判断，这一判断可以用于区分算术和逻辑位移指令

                //但case的包含关系竟然是小类别的在前

                case(op2):

                    5'b00000:begin

                        case(op3):begin

                        end

                        endcase

                    end

                    default: begin

                    end

                endcase

            end

            end
```

及inst[10:6]的判断先于inst[5:0]的判断。用于区分普通的逻辑运算与唯一以及算术运算

2.  修改EX模块，从而进行运算
其中，SRA与SRAV的运算过程需要特别注意，算术右移：
```
`EXE_SRA_OP:begin
			shiftres <= ({32{reg2[31]}} << (6'd32-{1'b0, reg1[4:0]})) | reg2 >> reg1[4:0];
        end
        `EXE_SRAV_OP:begin
            shiftres <= ({32{reg2[31]}} << (6'd32-{1'b0, reg1[4:0]})) | reg2 >> reg1[4:0];      
        end
```

Q:此处的ALU只实现了逻辑运算，那么实现加减乘除的数值运算的ALU在哪里实现？


仿真结果如下：
![[Pasted image 20231203111715.png]]

![[Pasted image 20231203112109.png]]

逻辑指令测试正确；位移操作测试略