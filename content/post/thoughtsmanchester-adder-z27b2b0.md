---
title: 思考_曼切斯特加法器
slug: thoughtsmanchester-adder-z27b2b0
url: /post/thoughtsmanchester-adder-z27b2b0.html
date: '2025-11-20 08:42:19+08:00'
lastmod: '2025-12-28 13:59:49+08:00'
toc: true
isCJKLanguage: true
---



# 思考_曼切斯特加法器

传统的行波进位加法器（RCA）慢，是因为进位信号必须经过每一级的复杂逻辑门（与非门、或非门等）才能传到下一级。

曼彻斯特进位链通过定义三个状态，利用开关电路直接导通进位：

- ​**生成 (Generate,**  **$G_i$**​ **)** : $A_i = 1, B_i = 1$。无论低位进位如何，当前位必然产生进位。
- ​**传播 (Propagate,**  **$P_i$**​ **)** : $A_i \oplus B_i = 1$ (即其中一个是 1)。如果低位有进位，直接通过当前位传给高位。
- **消除 (Kill/Delete,**  **$K_i$**​ **)** : $A_i = 0, B_i = 0$。无论低位进位如何，进位在此截断。

逻辑公式：

$$
C_{i+1} = G_i + (P_i \cdot C_i)
$$

#### 结构组成：

1. ​**预充电路（Precharge）** : 在时钟 $\phi$ 为低电平时，将进位节点预充电到高电平（VDD）。
2. ​**进位链（The Chain）** :

    - ​**传输开关**: 由 $P_i$ 控制的 NMOS 传输管串联在一起。如果 $P_i$ 为高，进位信号 $C_i$ 直接“流”向 $C_{i+1}$。
    - ​**下拉路径**: 由 $G_i$ 控制的 NMOS 管接地。如果 $G_i$ 为高，直接将 $C_{i+1}$ 节点拉低（表示产生反向逻辑的进位，或者根据具体逻辑极性设计）。
3. ​**求值（Evaluate）** : 时钟 $\phi$ 变为高电平时，电路根据输入 $A$ 和 $B$ 的状态决定节点电平。

‍

![image](/images/image-20251120090558-my1zh3z.png)

预充电

![image](/images/image-20251120090709-8ed8s08.png)

求值 CLK = 1

GI=0 ; PI=0

‍

![image](/images/image-20251120090811-twhp7hx.png)

GI=1;PI=0

![image](/images/image-20251120091103-if8csn6.png)

GI=0;PI=1;CARRY IN =0

‍

‍

‍

总的电路

真值表：动态门 推出逻辑表达式

![image](/images/image-20251120084828-ukxzm8o.png)

schematic

‍

![20251126-vmware-914](/images/20251126-vmware-914-20251126220440-11vhpzr.png)

1bit 曼切斯特加法器原理图

![image](/images/image-20251126220504-ctbwkte.png)

仿真波形图

![fd39e0e31ad0ffd84be6d863bfe3d9da](/images/fd39e0e31ad0ffd84be6d863bfe3d9da-20251120084317-gx7n30o.png)

4bit 曼切斯特加法器原理图

‍

‍

![image](/images/image-20251120085527-8wly4vw.png)

电阻

‍

‍

![image](/images/image-20251120085604-nhtweqo.png)

延时

4R2C 如何来的

‍

![image](/images/image-20251120090230-1ci7a0l.png)

平方关系如何计算

‍

![image](/images/image-20251120090443-cx0lxz0.png)

这俩个图有什么不同，延时为什么不一样[^1]

‍

[^1]: ### 1. 两个图的核心区别

    #### 右图：常规静态 CLA (Standard Static CLA)

    - ​**标注**​：`Carry lookahead logic OR / AND`
    - ​**核心结构**​：使用的是​**传统的静态 CMOS 逻辑门**（与门、或门、非门，或者更复杂的 AOI/OAI 复合门）。
    - 工作方式：  
      它通过纯粹的布尔代数公式计算进位。例如，\$C\_4\$ 的逻辑表达式是：

      $$
      C_4 = G_3 + P_3G_2 + P_3P_2G_1 + P_3P_2P_1G_0 + P_3P_2P_1P_0C_0
      $$
    - ​**代价**​：为了实现这个长长的公式，需要多级逻辑门，或者堆叠层数很高的晶体管（Fan-in 很大）。这意味着​**节点电容大**，且信号要经过好几级门的翻转。

    #### 左图：曼彻斯特 CLA (MCC - Manchester Carry Chain)

    - ​**标注**​：`Carry lookahead logic MCC`
    - ​**红圈含义**：红圈圈住了 Bit 模块，暗示进位链是穿插在这些模块之间或者是利用传输管级联的。
    - ​**核心结构**​：使用的是我们刚才讨论的​**传输管（Pass Transistor）或动态逻辑开关**。
    - 工作方式：  
      它不像右边那样硬算公式，而是利用开关网络让电流直接流过。如果 \$P\_0, P\_1, P\_2, P\_3\$ 都为 1，电流就直接像穿过水管一样从头流到尾。

    ---
