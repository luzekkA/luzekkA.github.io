# 使用SRAM Macro替换RTL代码中的仿真用RAM



[TOC]



## 1.根据处理器实际使用的RAM，使用MemoryCompiler生成RAM的Macro

如果你不清楚实际使用的RAM大小，可以先跑一次综合，然后在mapped文件夹中寻找对应的文件，然后在文件中寻找RAM的大小信息，例如：

```
module sirv_sim_ram_DP8192_FORCE_X2ZERO0_DW64_MW8_AW13 ( clk, din, addr, cs, 
        we, wem, dout );
  input [63:0] din;
  input [12:0] addr;
  input [7:0] wem;
  output [63:0] dout;
  input clk, cs, we;


endmodule


module sirv_sim_ram_DP16384_FORCE_X2ZERO1_DW32_MW4_AW14 ( clk, din, addr, cs, 
        we, wem, dout );
  input [31:0] din;
  input [13:0] addr;
  input [3:0] wem;
  output [31:0] dout;
  input clk, cs, we;

endmodule
```

从mapped的.v文件中可以找到以上信息，由此可以看出主要使用了两种尺寸的RAM

| 属性                | 值               |
| ------------------- | ---------------- |
| Instance Name       | SRAM_4096x64_MW8 |
| Number of Words     | 4096             |
| Number of Bits      | 64               |
| Frequency  \<MHz>   | 200              |
| Multiplexer Width   | 8                |
| Word Partition Size | 8                |

| 属性                | 值               |
| ------------------- | ---------------- |
| Instance Name       | SRAM_2048x32_MW4 |
| Number of Words     | 2048             |
| Number of Bits      | 32               |
| Frequency  \<MHz>   | 200              |
| Multiplexer Width   | 4                |
| Word Partition Size | 8                |

你可以使用上述参数，然后在PDK对应的MemoryCompiler中生成对应的.lib文件以及.v文件

在这个项目中采用的命名方式为SRAM[Number of Words]x[Number of Bits]_MW[Multiplexer Width].[lib or v or sth]

### 查看生成的.lib文件

在.lib文件中搜索area的相关内容，你应该可以找到这个内存块的相关面积信息，例如：

```
cell(SRAM_4096x32_MW8) {
	area		 : 413048.946;
	dont_use	 : TRUE;
	dont_touch	 : TRUE;
        interface_timing : TRUE;
	memory() {
		type : ram;
		address_width : 12;
		word_width : 32;
	}
```

其中`area		 : 413048.946;`就是模块对应的面积，可以对后面进行综合之后的Macro面积用作验证参考

## 2.使用生成的Memory IP对原本的RAM进行替换

如果MemoryCompiler生成的Memory的尺寸有比较大的限制，那么可以采用多块小型SRAM进行拼接的方式

(假设)如果你的MemoryCompiler最大只能生成4096x64尺寸的RAM,但是你需要8192x64尺寸的RAM，那么可以考虑使用两块4096x64尺寸的RAM进行拼接的方式解决这个问题。建议拿来拼接的小块RAM，只更改[Number of Words]以及[Number of Bits]这两个参数，其他参数尽量与原来的大块内存保持相同。

例如：

```
if (DP==8192 && DW==64 && MW==8) begin: g_ip_8192x64_by_4096x64
    // 用 2 颗 4096x64 (MW8) 宏实现：深度 2 倍分 bank，每 bank 4096×64
    // 地址拆分：A_word[0+:12] 为每 bank 内 64b 词地址；A_word[12] 为 bank 选择位
    localparam integer AW_WORD = (DP<=1) ? 1 : $clog2(DP); // 13
    localparam integer SHIFT   = (AW >= (AW_WORD + 2)) ? 2 : 0; // 若为字节地址则去掉低2位
    wire [AW_WORD-1:0] A_word = addr[SHIFT +: AW_WORD];

    localparam integer BANK_ADDR_W = 12; // 4096 深度 => 12 位词地址
    localparam integer NUM_BANKS   = 2;
    localparam integer BANK_SEL_W  = 1;
    
    wire [BANK_ADDR_W-1:0] A_in = A_word[0 +: BANK_ADDR_W];
    wire [BANK_SEL_W-1:0]  bank_idx = A_word[BANK_ADDR_W +: BANK_SEL_W];
    
    // 写掩码（8×8b，低有效）：仅选中 bank 参与
    wire [7:0] WEN_b0 = ~({8{cs & we & (bank_idx==1'b0)}} & wem[7:0]);
    wire [7:0] WEN_b1 = ~({8{cs & we & (bank_idx==1'b1)}} & wem[7:0]);
    
    wire CENA_b0 = ~(cs & (bank_idx==1'b0));
    wire CENA_b1 = ~(cs & (bank_idx==1'b1));
    
    wire [63:0] qa_b0, qa_b1;
    
    SRAM_4096x64_MW8 u_sram_b0 (
      .QA   (qa_b0),
      .QB   (),
      .CLKA (clk),
      .CENA (CENA_b0),
      .WENA (WEN_b0),
      .AA   (A_in),
      .DA   (din),
      .CLKB (1'b0), .CENB (1'b1), .WENB ({8{1'b1}}), .AB (12'b0), .DB (64'b0),
      .EMAA (3'b000), .EMAB (3'b000)
    );
    
    SRAM_4096x64_MW8 u_sram_b1 (
      .QA   (qa_b1),
      .QB   (),
      .CLKA (clk),
      .CENA (CENA_b1),
      .WENA (WEN_b1),
      .AA   (A_in),
      .DA   (din),
      .CLKB (1'b0), .CENB (1'b1), .WENB ({8{1'b1}}), .AB (12'b0), .DB (64'b0),
      .EMAA (3'b000), .EMAB (3'b000)
    );
    
    assign dout = (bank_idx==1'b1) ? qa_b1 : qa_b0;

  end
```

## 3.使用综合工具进行综合

### 如果你使用Design Compiler进行综合

#### 需要进行 PDK 的相关设定

如果你使用Design Compiler(简称DC)进行综合

你需要把对应的.lib文件，使用Lib Compiler生成对应的db文件，然后将对应的db文件添加到link_library中

例如：

```
set_app_var link_library [concat [get_app_var link_library] $SRAM_DB_PATH]
```

#### 验证

如果你成功进行了综合

你可以查看对应的doc/[top_module]/[top_module].rpt,这是对应模块的面积报告，如果正常结束了综合流程，那么生成的报告应该是类似这样的：

```
 
****************************************
Report : area
Design : e203_soc_top
Version: V-2023.12-SP3
Date   : Sun Oct 26 19:37:44 2025
****************************************
Library(s) Used:

    slow (File: /mnt/hgfs/processor_research/pdk/TSMC.90/aci/sc-x/synopsys/slow.db)
    USERLIB (File: /mnt/hgfs/processor_research/pdk/SRAM/db/SRAM_2048x32_MW4_ss_0.9_125.0_syn.db)
    USERLIB (File: /mnt/hgfs/processor_research/pdk/SRAM/db/SRAM_4096x64_MW8_ss_0.9_125.0_syn.db)

Number of ports:                        71309
Number of nets:                        130999
Number of cells:                        66938
Number of combinational cells:          51533
Number of sequential cells:             12821
Number of macros/black boxes:              10
Number of buf/inv:                      11492
Number of references:                       2

Combinational area:             261372.590763
Buf/Inv area:                    43510.824670
Noncombinational area:          195657.940908
Macro/Black Box area:          3509506.250000
Net Interconnect area:         8416049.557098

Total cell area:               3966536.781671
Total area:                   12382586.338770
1

```

在其中你可以看到`Macro/Black Box area:          3509506.250000`

你可以通过简单的计算得出，这就是我们所使用的SRAM的面积
