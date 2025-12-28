---
title: DC运行流程
slug: dc-operation-process-2ktjse
url: /post/dc-operation-process-2ktjse.html
date: '2025-12-08 14:50:29+08:00'
lastmod: '2025-12-28 14:15:23+08:00'
toc: true
isCJKLanguage: true
---



# DC运行流程

```tcl
dc_shell -f run_syn.tcl | tee run_smic.log



```

仿真配置[^1]

总结配置[^2]

‍

```tcl
write -format verilog -hierarchy -output signed_vedic_mult_32bit_netlist.v
```

VCS 仿真命令[^4]

数字全流程虚拟机[^5]

```tcl
# 1. 检查设计是否有逻辑问题
check_design

# 2. (可选) 设置一个简单的时钟约束，防止DC随意优化
# 假设你的时钟端口叫 clk，目标频率是 100MHz (10ns)
create_clock -period 10 [get_ports clk]

# 3. 开始综合 (这是最关键的一步，把代码变成门电路)
compile -map_effort medium
# 或者使用更强的优化命令: compile_ultra

# 4. 修改命名规则 (非常重要！)
# DC生成的网表里可能会有反斜杠 \ 或方括号 []，VCS有时不喜欢，需要改名
change_names -rules verilog -hierarchy

# 5. 导出门级网表 (Netlist) -> 给 VCS 用
write -format verilog -hierarchy -output signed_vedic_mult_32bit_netlist.v

# 6. 导出标准延时文件 (SDF) -> 给 VCS 做时序仿真用
write_sdf -version 2.1 signed_vedic_mult_32bit.sdf

# 7. 导出约束文件 (SDC) -> 以后做布局布线 (P&R) 用
write_sdc signed_vedic_mult_32bit.sdc
```

1. read_file -format db /home/ic_libs/TSMC_013/synopsys/typical.db

    1. read_file -format db /home/ICer/project/synopsys/lib/tsmc018/osu018_stdcells.db
2. list_libs
3. check_library
4. remove\_design -designs

‍

---

这个库是后仿真用verilog库，dc综合后的网表仿真以及布局布线后网表仿真都用到它。一般随工艺库由foundry提供。举例来说，tower 0.18 工艺库，在主目录底下应该包含synopsys、astro等所用的库子目录。dc综合时用的工艺库应该包含在synopsys子目录下，同理，仿真用的verilog库应包含在名为verilog的子目录下，共包含2个文件，twr18.v 与 twr18\_neg.v，其中后一个包含了负时序（- holdtiming）信息，但仿真速度较慢，一般不采用。在对综合或布局布线后网表进行仿真时将这个.v文件与所得的sdf或者spef文件一起载入仿真器仿真即可。当然，不同的foundry提供的工艺库目录结构可能略有不同，但基本上遵循上述的目录结构。楼主请到相应的库目录下查找一下即可。

---

‍

‍

问题

1. 改变电压和输出芯片没变化，在亚阈值区
2. 怎么用向量测试
3. 我觉得还是先进行第二步，用modelsim看时序，写测试逻辑在所
4. 如何把32位乘法器模块化设计的更好些

‍

‍

‍

翻译加e入[^6]

‍

转换库[^7]

**网表 (Netlist)**  是由 **Design Compiler (DC)**  在运行完 `compile` 命令之后生成的。

在你的脚本 `run.tcl`​ 中，它是由 `write` 这个命令产生的。

### 1. 网表在哪里生成？ (Where)

在你提供的 run.tcl 文件末尾，有一行：

write -format ddc -hier -out unmapped/MY\_DESIGN.ddc

 **⚠️ 注意：**  这行生成的是 `.ddc`​ 文件（二进制数据库），只能给 DC 自己读，**Modelsim 和 Virtuoso 读不懂！**

你需要**手动添加**下面这一行命令到 `run.tcl`​ 的最后（在 `exit` 之前），才能生成通用的 Verilog 网表：

Tcl

```
# 必须添加这一行来生成 .v 文件
write -format verilog -hierarchy -output mult32_netlist.v
```

- **生成的时机：**  必须在 `compile`​ 命令**之后**执行。
- **生成的位置：**  会出现在你运行 DC 的当前文件夹下，名字叫 `mult32_netlist.v`。

---

### 2. 网表的格式是怎样的？ (Format)

网表虽然也是 `.v`​ 文件，但它和你要自己写的 `mult32.v` 长得完全不一样。

- **你的代码 (RTL):**  描述的是“功能” (人类能看懂)。
- **网表 (Netlist):**  描述的是“连接关系” (机器能看懂)。

#### 举个例子对比：

**A. 你写的代码 (Behavioral RTL):**

Verilog

```
module mult32 (A, B, P);
    input [31:0] A, B;
    output [63:0] P;

    assign P = A * B;  // 一句话搞定
endmodule
```

B. DC 生成的网表 (Gate-level Netlist):

(这就是 mult32\_netlist.v 里的内容，DC 把你的乘号打碎成了几千个门)

Verilog

```
module mult32 ( A, B, P );
  input [31:0] A;
  input [31:0] B;
  output [63:0] P;
  
  // 1. 定义了一堆乱七八糟的中间连线
  wire n1, n2, n3, n45, n_sys_12, net_334;

  // 2. 调用工艺库里的“标准单元” (Standard Cells)
  // AND2X1, XOR2X1 这些名字来自你的 SMIC .18 库
  AND2X1 U1 ( .A(A[0]), .B(B[0]), .Y(n1) );
  AND2X1 U2 ( .A(A[1]), .B(B[0]), .Y(n2) );
  
  // 3. 具体的连接关系
  XOR2X1 U3 ( .A(n1), .B(n_sys_12), .Y(P[0]) );
  INVX1  U4 ( .A(n45), .Y(net_334) );
  
  // ... 下面还有几千行这样的东西 ...
endmodule
```

### 3. 如何检查网表是否正确？

当你打开生成的 `mult32_netlist.v` 文件时：

1. **看名字：**  里面的模块名（比如 `AND2X1`​、`FAX1`​）必须是 **SMIC .18 工艺库**里的名字。如果看到 `GTECH_AND` 这种名字，说明综合没完全成功（没映射到具体工艺上）。
2. **看结构：**  里面不应该再出现 `assign P = A * B;` 这种语句。
3. **看端口：**  模块的输入输出端口（`A`​, `B`​, `P`）应该和你原始设计保持一致。

下一步操作建议：

修改你的 run.tcl，加上 write -format verilog ... 那行代码，重新运行一遍 DC，然后去文件夹里找生成的 .v 文件。

‍

[^1]: # 仿真配置

    ```tcl
    vcs -R -full64 -debug_access+all \
        tb_mult.v \
        signed_vedic_mult_32bit_netlist.v \
        -v /home/ICer/EDA/verilog/tsmc13.v \
        +neg_tchk +maxdelays
    ```
    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ‍

    ### . 关于 "Fatal: Error loading environment variables..."

    你 Log 的最后出现了这个严重错误：

    ‍

    ‍

    ‍

    驱动能力  5V

    ‍

    ‍

    Plaintext

    ```
    Fatal: Error loading environment variables from environment file.
    Terminating execution.
    ```
    **这意味着你的仿真环境配置（AMS Designer）挂了。**

    - ​**影响**​：如果你只是画图、跑 LVS/DRC，这可能暂时不影响你使用。但如果你要进行 ​**数模混合仿真 (AMS Simulation)** ，工具会无法启动。
    - ​**原因**​：通常是因为你的 Linux 环境变量配置有问题，或者工程目录下的 `.cdsinit`​ / `ams.env` 文件里指向的路径不对。
    - ​**建议**：

      - 如果 Virtuoso 没有闪退，可以先忽略它，专注解决上面的黄色框问题。
      - 如果后续跑仿真报错，建议联系实验室负责服务器的同学，检查一下 `AMS_DIR` 或相关的环境变量设置


[^2]: # 总结配置

    1. DC用sdl
    2. 需要配置symbol（没的话仿真不了）和schematic（重要）
    3. 网表要对应
    4. 模型库[^3]

    ‍

    ‍


[^3]: # 模型库

    这是一个非常关键的发现。从你的截图来看，`tran_models`​ 在你的 Windows 系统里显示为一个​**黄色文件夹**​，这在标准的 FreePDK45 结构中是不太正常的（通常它应该是一个名为 `tran_models.sp`​ 的​**文件**）。

    这很可能是解压或下载过程中产生的问题（例如多套了一层文件夹）。

    请按照以下 3 步操作来修复：

    ### 第一步：点进这个文件夹看看

    请双击打开截图中的这个 `tran_models`​ ​**文件夹**。

    - ​**情况 A**​：如果你在里面看到了一个叫 `tran_models.sp`​ 或者 `models.sp`​ 的文件，​**那个才是真正的模型文件**。
    - ​**情况 B**：如果你点进去是空的，或者是乱码文件，那么可能是文件丢失了。

    通常正确的路径结构应该是：

    .../ncsu\_basekit/models/hspice/tran\_models.sp (这是一个文本文件)

    ### 第二步：确认文件路径（Windows vs Linux）

    由于你在跑 Spectre 仿真（这是 Linux 软件），而你的截图是 Windows 界面，说明你用了虚拟机共享文件夹（`Virtual_share`）。

    **绝对不要**在 Cadence 里面直接填 Windows 的路径（如 `D:\Virtual_share...`），Cadence 读不懂。

    你需要找到这个文件在 ​**Linux 系统里的对应路径**。通常它长这样：

    - ​`/mnt/hgfs/Virtual_share/des_test/ncsu_basekit/models/hspice/tran_models/tran_models.sp`
    - 或者 `/home/你的用户名/Desktop/share/...`

    ### 第三步：在 ADE 中加载正确的文件

    1. 回到 Cadence ADE 的 **Model Libraries** 设置窗口。
    2. 点击 Browse（浏览），沿着 **Linux 的挂载路径** 找到那个真正的 `.sp` 文件。
    3. 加载后，**Section** 一栏​**留空**（不要填 tt）。

    如果你在文件夹里找不到 .sp 文件：

    你可以直接退回到上一级目录 .../ncsu\_basekit/models/hspice/，看看这一层有没有名为 hspice\_nom.include 或者 tran\_models.sp 的文件。如果有，直接用那个即可。

    **一句话总结：**  请点进那个黄色文件夹，找到里面的 `.sp` 文件，并在 Cadence 里通过 Linux 路径加载它。


[^4]: # VCS 仿真命令

    ```bash
    vcs -R -full64 \
      -debug_acc+all -debug_region+cell+encrypt \
      -timescale=1ns/1ps \
      +v2k \
      tsmc13.v \
      signed_vedic_mult_32bit_netlist.v \
      signed_vedic_mult_32bit_tb.v \
      -l func_sim.log
    ```
    ‍

    dve -vpd inter.vpd

    *注意*：这次运行后，你应该会看到生成了一个稍微大一点的 `inter.vpd`​ 或 `vcdplus.vpd` 文件。

    vcdplus.vpd

    dve -vcdplus.vpd

    ‍

    ./simv -gui &

    run

    ‍

    如果你做后端或验证，通常会用 Verdi 来查看网表，它的原理图追踪功能比 Design Vision 强大得多。

    看图

    ```bash
    verdi -sv signed_vedic_mult_32bit_netlist.v tsmc13.v &
    ```

[^5]: # 数字全流程虚拟机

    这是一个非常“豪华”且专业的IC设计环境（通常被称为EDA虚拟机），几乎涵盖了数字IC设计（前端、后端）、模拟IC设计以及FPGA开发的整个全流程工具。

    对于你的微电子科学与工程专业背景，这个环境非常适合学习和做项目。

    以下是根据你提供的目录截图整理的工具清单及其作用，以及关于替代ModelSim的回答：

    ### 1. 能替代 ModelSim 的数字仿真工具

    你问有没有能替代 ModelSim 的工具，**答案是：有，而且更强大、更符合工业界标准。**

    在这个虚拟机中，你有**三个**顶级的工业级仿真器，它们在性能和功能上都超越了普通的 ModelSim：

    - ​**​`vcs`​**​  **(Synopsys VCS):**

      - **地位：**  业界最主流的数字逻辑仿真器之一（通常与 Verdi 配合使用）。
      - **特点：**  它是编译型仿真器（Compile-based），速度比 ModelSim（解释型）快得多，尤其是在大规模设计时。
      - **用法：**  配合 **Verdi** 查看波形是数字IC设计工程师的标准工作流（不再是像 ModelSim 那样看自带的波形窗口）。
    - ​**​`questasim`​**​  **(Mentor Questasim):**

      - **地位：**  它是 ModelSim 的“大哥”或“高级版”。
      - **特点：**  界面和操作逻辑与 ModelSim 几乎一模一样，但支持更多的高级验证特性（如 SystemVerilog 断言、UVM 方法学等），且仿真速度更快。如果你习惯了 ModelSim，上手这个是零门槛的。
    - ​**​`INCISIVE`​**​  **(Cadence Incisive / IES):**

      - **地位：**  Cadence 家的数字仿真平台（命令通常是 `ncsim`​ 或 `irun`）。
      - **特点：**  同样是工业级强力仿真工具，常用于混合信号仿真（配合 Virtuoso）。

    **建议：**  作为微电子专业的学生，建议你**尽早开始学习使用** **​`VCS + Verdi`​** 的组合，这是目前数字后端和验证岗位最核心的技能之一。

    ---

    ### 2. 虚拟机内其他工具详细介绍

    我们将这些工具按照厂商和用途分类：

    #### **A. Synopsys 系列 (数字IC设计的核心力量)**

    截图路径：`synopsys` 目录下

    - ​**​`vcs`​**​  **/**  **​`vcs-mx`​**​ **:**  数字逻辑仿真器（前面已提）。
    - ​**​`verdi`​**​ **:**  ​**最强大的波形查看与调试工具**。它不产生波形，而是读取 VCS 跑出来的波形文件（.fsdb），用于追踪信号、查看原理图逻辑，是数字工程师每天都要用的神器。
    - ​**​`syn`​**​  **(Design Compiler - DC):**  ​**逻辑综合工具**。将你写的 Verilog RTL 代码转换成门级网表（Gate-level Netlist）。这是数字前端设计最重要的工具。
    - ​**​`icc2`​**​  **(IC Compiler II):**  ​**数字后端布局布线工具 (P&R)** 。用于将综合后的网表变成实际的版图（Layout）。
    - ​**​`pts`​**​  **(PrimeTime - PT):**  ​**静态时序分析工具 (STA)** 。用于检查芯片能否在指定频率下稳定工作，是Sign-off（签核）的金标准。
    - ​**​`SpyGlass`​**​ **:**  ​**RTL 检查工具**。用于在综合之前检查你的代码风格、跨时钟域（CDC）问题等，防止低级错误。
    - ​**​`lc`​**​  **(Library Compiler):**  库文件处理工具，用于将 .lib 转为 .db 格式供 DC 使用。

    #### **B. Cadence 系列 (模拟/数模混合设计霸主)**

    截图路径：`cadence` 目录下

    - ​**​`IC617`​**​  **(Virtuoso):**  ​**模拟/混合信号版图与原理图设计工具**。画 MOS 管、画版图、做 DRC/LVS 都要用到它。微电子学生应该非常熟悉这个界面。
    - ​**​`MMSIM151`​**​  **(Spectre):**  ​**模拟电路仿真器**。在 Virtuoso 里跑仿真（AC/DC/Tran）时，后台调用的就是它。
    - ​**​`INCISIVE152`​**​ **:**  数字仿真平台（前面已提）。

    #### **C. Mentor (现 Siemens EDA) 系列**

    截图路径：`mentor` 目录下

    - ​**​`Calibre2015`​**​ **:**  ​**物理验证金标准**。用于做 DRC（设计规则检查）、LVS（版图原理图一致性检查）和 PEX（寄生参数提取）。在 Virtuoso 画完版图后，必须用 Calibre 跑一遍才算完成。
    - ​**​`questasim`​**​ **:**  高级数字仿真器（前面已提）。

    #### **D. Xilinx 系列 (FPGA 开发)**

    截图路径：`Xilinx` 目录下

    - ​**​`Vivado 2019.1`​**​ **:**  Xilinx 的 FPGA 集成开发环境。包含综合、布局布线和仿真。
    - ​**​`SDK`​**​ **:**  嵌入式软件开发工具（通常用于 Zynq 系列里的 ARM 核开发）。

    #### **E. 开源/轻量级工具**

    - ​**​`iverilog`​**​  **(Icarus Verilog):**  开源的 Verilog 仿真器，轻量级，适合简单测试。
    - ​**​`gtkwave`​**​ **:**  开源的波形查看器，通常配合 iverilog 使用。
    - ​**​`Riscv_Tools`​**​ **:**  专门用于 RISC-V 处理器开发的工具链（编译器 gcc 等）。

    ### 总结

    这个虚拟机环境非常完善：

    1. **做数字IC：**  你有 VCS (仿真) + Verdi (调试) + DC (综合) + ICC2 (后端) + PrimeTime (时序)。
    2. **做模拟IC：**  你有 Virtuoso (设计) + Spectre (仿真) + Calibre (验证)。
    3. **做FPGA：**  你有 Vivado。

    **对于你的“替代 ModelSim”的需求，直接去** **​`synopsys`​**​ **目录下找** **​`vcs`​**​ **，或者在终端里配置好环境变量后尝试运行** **​`vcs`​**​ **或** **​`verdi`​**​ **即可。**


[^6]: # 翻译加入

    要在 Virtuoso (IC617) 中使用这个库，你需要把它“挂载”到 Library Manager 中。

    根据你提供的截图，你已经在 `osu_soc-master/cadence/lib/tsmc018` 路径下找到了库文件夹。

    请按照以下步骤操作：

    ### 第一步：确认 Linux 路径

    非常重要： 你的截图 显示的是 Windows 路径 (D:\\Virtual\_share...)。

    Virtuoso 运行在 Linux 中，它读不到 D 盘。你需要找到这个“共享文件夹”在 Linux 里的挂载路径。

    - 通常是在 `/mnt/hgfs/Virtual_share/`​ 或者 `/media/sf_Virtual_share/` 下。
    - 你需要定位到 Linux 下的这个文件夹：`.../osu_soc-master/cadence/lib/tsmc018/OSU_stdcells_tsmc018`

    ### 第二步：在 Library Manager 中添加库

    1. 打开 Virtuoso 的 **Library Manager** (库管理器)。
    2. 在顶部菜单栏点击 **Edit** -\> ​**Library Path...** 。
    3. 这会弹出一个表格编辑器。在最下面的一行空白处填写：

        - **Library (库名):**  `OSU_stdcells` (或者你自己喜欢的名字，只要记住就行)。
        - **Path (路径):**  点击单元格里的浏览按钮，找到上面提到的 Linux 路径，选中 `OSU_stdcells_tsmc018` 这个文件夹。
        -  *(注意：要选那个文件夹本身，不要进去了)*
    4. 点击 **File** -\> **Save** (保存 `cds.lib`)，然后关闭编辑窗口。

    ### 第三步：验证库是否有效

    1. 回到 Library Manager 主界面。
    2. 在左侧 Library 列表里，应该能看到你刚加的 `OSU_stdcells`。
    3. ​**点击它**，看中间的 Cell 列表：

        - **情况 A (成功):**  如果你看到全是 `AND2X1`​, `DFF`​, `INV` 等名字，恭喜你，挂载成功！
        - **情况 B (空库):**  如果里面是空的，或者 Virtuoso 报错说格式不对（OpenAccess版本问题），说明这个文件夹里的数据是旧版格式或者只是个空壳。**如果遇到这种情况，请看下面的“Plan B”。**

    ---

    ### 第四步：回到 Verilog In 界面 (解决你的最初问题)

    一旦第三步确认成功，你就可以回到“Verilog In”导入界面 进行修改了：

    1. **Target Library Name:**  建议填 `my_design` (你自己新建的库)。
    2. Reference Libraries: 这里填写：  
        OSU\_stdcells basic analogLib  
        (注意：OSU\_stdcells 必须和你第二步里起的 Library 名字一模一样)

    ---

    ### Plan B：如果直接挂载全是空的 (使用 GDS 导入)

    OSU 的这个包有时候库文件不完整，但它一定提供了 **GDSII** 版图文件。如果上面的方法不行，请用这个方法“重建”库：

    1. **新建库：**  Library Manager -\> File -\> New -\> Library。

        - 名字：`OSU_stdcells`。
        - Technology File: 选择 `Attach to an existing technology library`​ -\> 选择 `NCSU_TechLib_tsmc02`​ (或者是 tsmc03，如果没有就选 `Do not need process info`)。
    2. **导入 GDS：**

        - Virtuoso 主界面 -\> **File** -\> **Import** -\> ​**Stream**。
        - **Stream File:**  浏览找到 `.../osu_soc-master/cadence/lib/tsmc018/osu018_stdcells.gds2`​ (这个文件通常在 `lib` 文件夹同级或里面)。
        - **Library:**  选择刚才新建的 `OSU_stdcells`。
        - 点击 ​**Translate**。
    3. 这会自动把版图数据转换成 Virtuoso 的库。完成后，你再进行“第四步”即可。


[^7]: # 转换库

    ‍

    终端不能在同一个开启

    ‍

    这是一个非常典型的  **“操作位置冲突”**  错误。

    简单来说：你正在  **​`/root/Desktop/`​** ​ 目录下运行转换工具，而你要修改的 **​`cds.lib`​**​ 文件也在这个目录下。`cdb2oa`​ 工具为了保护文件不被写坏，**禁止**在包含 `cds.lib` 的同一级目录下运行它自己。

    ### ✅ 解决方案：换个地方运行命令

    你只需要退到其他目录（比如 `/tmp` 临时目录），然后“远程遥控”运行转换命令即可。

    请​**关闭 Virtuoso**，打开终端 (Terminal)，严格按照以下 3 步操作：

    #### 第一步：进入临时目录

    在终端输入：

    Bash

    ```
    cd /tmp
    ```
     *(这一步是为了离开 Desktop 目录，避开那个报错)*

    #### 第二步：运行转换命令

    复制并运行下面这行命令（这是根据你报错日志生成的修正命令）：

    Bash

    ```
    cdb2oa -lib OSU_CDB -cdslibpath /root/Desktop/cds.lib
    ```
    - **解释：**

      - ​`-lib OSU_CDB`：转换你刚才定义的那个旧版库。
      - ​`-cdslibpath ...`：告诉工具去哪里找你的配置文件。
    - **执行过程中：**  屏幕可能会滚动很多日志，如果没有报错 `ERROR`​，并且最后显示 `cdb2oa` 结束，那就是成功了。

    #### 第三步：重启 Virtuoso 验证

    1. 回到你的工作目录：
    2. 打开 ​**Library Manager**。
    3. 点击左侧的 **​`OSU_CDB`​** 库。
    4. **关键时刻：**  看中间的 **Cell** 列表（比如点 `AND2X1`​），现在右边的 **View** 列表里应该同时出现了 ​**​`layout`​**​, ​**​`schematic`​**​, 和 ​**​`symbol`​**！

    ---

    ### 🚀 最后一步：重新导入网表 (Verilog In)

    一旦你在 Library Manager 里看到了 ​**​`symbol`​**，就说明你已经拿到了“拼图的说明书”。

    1. 打开 ​**File ->**  **Import ->**  **Verilog**。
    2. **Reference Libraries** 必须修改为：

        ```
        OSU_CDB basic analogLib
        ```
         *(注意：一定要填刚才转换成功的库名* *​`OSU_CDB`​*​ *，不要再填那个只有 Layout 的* *​`OSU_stdcells`​*​ *了)*
    3. 点击 ​**OK**。

    这次导入生成的 Schematic，必定是连线完整的电路图！
