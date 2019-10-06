# FPGA Scan_Bist 测试板交互
## Revision: RK826_verC_FPGS_test_2019/10/06


1. **IIC 数据格式**

**--Device ID (7bit)** | **--寄存器地址 (16bit)** | **--Seq_num (8bit)** | **--Operation_ID (8bit)** | **--配置数据 (16bit)**

   1.1 写控制寄存器
|    --0xA8           |       --0x0020          |   --0x00 ~ 0xFF      |  --0x0C,0x08,0x09,0xAC   |   --x0000 ~ 0xFFFF   |

   1.2 读状态寄存器
|    --0xA8           |       --0x0010          |   --0x00 ~ 0xFF      |               --24bit 状态信息                  |


2. **配置说明**

**(寄存器的配置 Device ID 和 寄存器地址格式固定，Seq_num 从 0 ~ 255 循环，直到状态寄存器的 Seq_num 域读到配置中的 Seq_num 代表一次配置完成并相应)**

| **--Operation_ID (8bit)** |

| **--0x0C**                |
| **配置装入的Pattern信息，scan/bist时钟模式配置， 初始化位配置，ScanIO的方向配置** |
|  --bit15 -- bit12     |  --bit11,bit10    |  --bit9                 |   --bit8           |    --bit7 -- bit0    |
|  --pattern ID        |  --Reserved       |  --scan/bist时钟模式     |   --初始化位 W1C    |    --IO方向寄存器    |

| **--0x08**                |
|    --bit15 -- bit8             |    --bit7 -- bit0         |
|    --scan时钟上升沿位置计数     |    --scan时钟周期计数     |

| **--0x09**                |
|    --bit15 -- bit8             |    --bit7 -- bit0              |
|    --scan时钟采样点位置计数     |    --scan时钟下降沿位置计数     |

| **--0xAC**                |
| **配置完毕,开始发送pattern,数据信息保留，可用作其它配置** |


3. **编程模型(scan/bist过程)**

   Step1. 读状态寄存器，查询当前Seq_ID
      Seq_ID = IIC_READ(0x0010);
   
   Step2. Seq_ID++ 进行下一操作的配置
      Seq_ID++;
   
   Step3. 配置0x0C
      IIC_WRITE(0x0020, 0x<Seq_ID>0C<pattern_id><rsv><self_test_clk_en><init><io_dir>);

   Step4. Seq_ID++ 进行下一操作的配置
      Seq_ID++;

   Step5. 配置0x08
      IIC_WRITE(0x0020, 0x<Seq_ID>08<rise><cycle>);

   Step6. Seq_ID++ 进行下一操作的配置
      Seq_ID++;

   Step7. 配置0x09
      IIC_WRITE(0x0020, 0x<Seq_ID>09<capture><down>);

   Step8. Seq_ID++ 进行下一操作的配置
      Seq_ID++;

   Step9. 启动Pattern/Bist
      IIC_WRITE(0x0020, 0x<Seq_ID>AC<rsv>);

   Step10. 读状态寄存器，查询当前Seq_ID
      Seq_ID = IIC_READ(0x0010);

   Step11. 若 Seq_ID == Step9 中发下的 Seq_ID 则 判定结果，否则继续重复步骤Step10 过程查询
      若读出数据为0x00, 则 site0,site1 Pass;
      若读出数据为0x02, 则 site0 Fail;
      若读出数据为0x20, 则 site1 Fail;

   回到Step2. 进入下一操作


4. Tips

   IIC寄存器配置过程与RK826相同，Operation_ID 可扩展，目前只用到 0x0C,0x08,0x09,0xAC;
   通过Seq_ID进行流程交互，如遇到超时无响应状态，则回到Step1，查询Seq_ID后继续交互；
