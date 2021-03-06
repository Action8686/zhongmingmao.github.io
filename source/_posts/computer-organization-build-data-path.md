---
title: 计算机组成 -- 建立数据通路
mathjax: true
date: 2020-01-16 22:02:33
categories:
    - Computer Basics
    - Computer Organization
tags:
    - Computer Basics
    - Computer Organization
---

## 三种周期
### 指令周期
1. 执行一条指令的过程
   - **Fetch**（取得指令）
     - 从**PC寄存器**里面找到对应的**指令地址**，根据指令地址从**内存**里把具体的指令，**加载到指令寄存器**中
     - 然后把PC寄存器**自增**，便于未来执行下一条指令
   - **Decode**（指令译码）
     - 根据**指令寄存器**里面的指令，解析成要进行什么样的操作，是**R、I、J**中的哪一种指令
     - 具体要操作哪些**寄存器**、**数据**或者**内存地址**
   - **Execute**（执行指令）
     - **实际运行**对应的R、I、J这些特定的指令，进行**算术逻辑操作**、**数据传输**或者**直接的地址跳转**
   - 重复上面步骤
2. **指令周期**（Instruction Cycle）：**Fetch -> Decode -> Execute**

<!-- more -->

<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-instruction-cycle.jpg" width=800/>

#### 涉及的组件
1. **取指令**的阶段，指令是放在**存储器**里的
2. 通过**PC寄存器**和**指令寄存器**取出指令的过程，是由**控制器**（Control Unit）操作的
3. 指令的**解码**过程，也是由**控制器**进行的
4. 一旦到了**执行指令**阶段，R、I型指令都是由**算术逻辑单元**（ALU）操作
    - 进行**算术操作**、**逻辑操作**的**R型**指令
    - 进行**数据传输**、**条件分支**的**I型**指令
5. 如果是**简单的无条件地址跳转**，可以直接在**控制器**里面完成，不需要用到运算器

<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-instruction-cycle-component.jpg" width=800/>

### 机器周期
1. Machine Cycle：**机器周期**或者**CPU周期**
2. CPU内部的操作速度很快，但访问内存的速度却慢很多，每条指令都需要从内存里面加载而来
    - 一般把**从内存里读取一条指令的最短时间**，称为CPU周期

### 时钟周期
1. Clock Cycle：**时钟周期**（主频）

### 三者关系
1. 一个**CPU周期（机器周期）**，通常会由几个**时钟周期**累积起来
2. 对于一个**指令周期**来说，取出一条指令，然后执行它，至少需要**两个CPU周期**
    - 取出指令至少需要一个CPU周期，执行指令至少也需要一个CPU周期
      - **指令译码**只需要**组合逻辑电路**，**不需要一个完整的时钟周期** -- **时间很短**
    - 复杂的指令规则需要更多的CPU周期

<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-cycle.jpg" width=1000/>

## 建立数据通路
1. **数据通路**就是处理器单元，由两类元件组成：**操作元件**、**存储元件**
2. **操作元件**
   - 也称为**组合逻辑元件**，即**ALU**
   - 功能：在特定的输入下，根据**组合电路**的逻辑，生成特定的输出
3. **存储元件**
   - 也叫**状态元件**，如计算过程中用到的寄存器，不论是**通用寄存器**还是**状态寄存器**，都是存储元件
4. 通过**数据总线**的方式，把操作元件和存储元件连接起来，就可以完成数据的**存储**、**处理**和**传输**了，**建立了数据通路**

### 控制器
1. 控制器只是机械地重复**Fetch -> Decode -> Execute**循环中的前两个步骤
   - 然后把在最后一个步骤通过**控制器**产生的**控制信号**，交给**ALU**去处理
2. **所有CPU支持的指令**，都会在**控制器**里面，被解析成不同的**输出信号** -- **电路非常复杂**
3. 运算器里的ALU和各种组合逻辑电路，可以认为是一个固定功能的电路
   - 控制器翻译出来的就是不同的**控制信号**
   - 这些控制信号，告诉**ALU**去做不同的计算
4. **指令译码器**将输入的**机器码**（机器指令），解析成不同的**操作码**和**操作数**，然后传输给**ALU**进行计算

<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-controller.jpg" width=600/>

### 所需硬件电路
1. **ALU**
   - 一个**没有状态**的，**根据输入计算输出**的组合逻辑电路
2. **寄存器**
   - 一个能进行**状态读写**的电路元件
   - 这个电路能够**存储上一次的计算结果**
   - 常见实现：**锁存器**（Latch）、**D触发器**（Data/Delay Flip-flop）
3. 『**自动**』的电路
   - 按照固定的周期，不停地实现**PC寄存器**自增，自动去执行**Fetch -> Decode -> Execute**的步骤
   - **PC寄存器 = 程序计数器**
4. 『**译码**』的电路
   - 对**指令**进行**decode**
   - 拿到**内存地址**去**获取**对应的数据或者指令

## 时序逻辑电路
1. **组合**逻辑电路：**只需要给定输入，就能得到固定的输出**
2. **时序**逻辑电路解决的问题
   - **自动运行**
     - 时序电路接通之后可以不停地开启和关闭开关，进入一个自动运行的状态
     - 场景：控制器不停地让PC寄存器**自增读取**下一条指令
   - **存储**
     - 通过时序电路实现的**触发器**，能把计算结果**存储在特定的电路**里面
     - 不像组合逻辑电路那样，一旦输入有任何变化，对应的输出也会改变
   - 本质上解决了各个功能按照**时序协调**的问题
     - 无论是程序实现的软件指令，还是硬件层面的各种指令操作，都有先后的顺序要求
     - 时序电路使得不同的事件按照**时间顺序**发生

### 时钟信号
1. CPU的**主频**是由一个**晶体振荡器**来实现的，而这个晶体振荡器生成的**电路信号**，就是**时钟信号**
2. 开关A，一开始是断开的，由**手工控制**；另一个开关B，一开始是合上的；磁性线圈对准开关B
   - 一旦合上开关A，磁性线圈会通电，产生磁性，开关B就会从合上变成断开
   - 一旦开关B断开，电路中断，磁性线圈失去磁性，于是开关B又会**弹回到**合上的状态
   - 这样，电路就会来回不断地在**开启**、**关闭**两个状态中切换，对**下游电路**来说，就不断地产生**0**和**1**的信号
3. 这种按照**固定周期**不断在0和1之间切换的信号，就是**时钟信号**（Clock Signal）
4. **反馈电路**：_**把电路的输出信号作为输入信号，在回到当前电路**_
   - 通过**反相器**（Inverter）实现的时钟信号

<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-crystal-oscillator.jpg" width=800/>
<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-clock-signal.jpg" width=800/>
<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-feedback-circuit-inverter.jpg" width=800/>

### D触发器 -- 存储

#### RS触发器
<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-rs-trigger.jpg" width=1000/>

##### 或非门
| NOR | **0** | **1** |
| --- | --- | --- |
| **0** | 1 | 0 |
| **1** | 0 | 0 |

##### 过程
1. 电路一开始，输入开关都是关闭的
   - A的输入：<0,0>，A的输出：1
   - B的输入：<1,0>，B的输出：0，反馈到A，没有任何变化，Q的输出：0
2. 把A的开关R合上
   - A的输入：<0,1>，A的输出：0
   - B的输入：<0,0>，B的输出：1，反馈到A，Q的输出：1
   - A的输入：<1,1>，A的输出：0
   - 电路仍然是**稳定**的，不会像晶振那样震荡
3. 把A的开关R打开
   - A的输入：<1,0>，A的输出：0
   - B的输入：<0,0>，B的输出：1，反馈到A，Q的输出：1
   - 电路依然是**稳定**的，开关R、S的状态和第一步是一样的，但Q的输出是1（**保留上一步的输出**）
4. 把B的开关S合上
   - B的输入：<?,1>，B的输出：0，Q的输出：0
5. 小结
   - **接通开关R，输出变为1，即使断开开关，输出还是1；接通开始S，输出变为0，即使断开开关，输出还是0**
   - **当两个开关都断开的时候，最终的输出结果，取决于之前动作的输出结果 -- 记忆功能**
   - RS触发器也称为**复位置位**触发器（**Reset-Set Flip Flop**）

| R | S | Q |
| --- | --- | --- |
| 1 | 0 | 1 |
| 0 | 1 | 0 |
| 0 | 0 | Q |
| 1 | 1 | NA |

#### D触发器

##### 控制何时往Q写入数据
<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-rs-flip-flop-clk.jpg" width=1000/>

1. 在RS触发器的基础上，在R和S开关之后，加入两个**与门**，同时给这两个与门加入一个**时钟信号CLK**作为电路输入
2. 当时钟信号CLK在**低电平**的时候，与门的输入里有一个0，两个实际的R和S后的与门的输出**必然为0**
   - 无论怎么按R和S开关，根据R-S触发器的真值表，**对应的Q值都不会发生变化**
3. 当时钟信号CLK在**高电平**的时候，与门的一个输入为1，输出结果**完全取决于R和S的开关**
   - 此时可以通过开关R和S，来决定对应Q的输出
4. 小结
   - 通过一个**时钟信号**，可以在**特定的时间**对输出Q进行**写入**操作

##### D触发器
<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-rs-flip-flop-clk-d.jpg" width=1000/>

1. 让R和S的开关，通过一个**反相器**连起来，就是**通过同一个开关控制R和S**
2. 当CLK信号是1，R和S就可以设置输出Q；当CLK信号是0，无论R和S怎么设置，输出信号都不会改变
3. 用来控制R和S两个开关的信号，视为一个输入的**数据信号**，即**Data**，这就是D型触发器的由来
4. 小结：把R和S两个信号通过一个反相器合并，可以通过一个数据信号进行Q的写入操作
5. 一个D触发器，只能控制**1bit**的读写
   - 如果同时拿出多个D触发器并列在一起，并且把用**同一个CLK信号**控制所有D触发器的开关
   - 这样就变成了N位的D触发器，可以同时控制N位的读写
6. CPU里的**寄存器**可以直接通过**D触发器**来构造的

### PC寄存器
1. PC寄存器，也称为**程序计数器**（Program Counter）
2. 有了**时钟信号**，可以提供**定时的输入**，有了**D触发器**，可以在**时钟信号控制的时间点写入数据**
   - 把两者**组合**起来，就可以实现一个**自动的计数器**
3. 加法器的两个输入，一个**始终设置为1**，另外一个来自于一个D触发器，把加法器的输出结果，写到D触发器
   - 这样D触发器里面的数据会在**固定的时钟信号为1**的时候**更新一次**，每过一个时钟周期，就能**固定自增1**
   - 每次自增之后，可以去对应的D触发器里取值，即**下一条需要运行指令的地址**（同一程序的指令要**顺序**存放在内存里）
   - 因此同一程序**顺序**地存放指令，就是为了通过**程序计数器**就能**定时不断地**执行新指令

<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-pc.jpg" width=800/>

### 译码器
1. 数据能存储在**D触发器**里，把很多D触发器放在一起，就能形成一块很大的存储空间，甚至当成一块**内存**来使用
   - 在写入和读取数据时，怎么定位是操作哪一个Bit？ -- **寻址、译码器**
2. 实际使用的计算机内存使用的是**DRAM**，并非通过D触发器来实现的，而是使用**CMOS**芯片来实现 -- 但不影响理解译码器的原理

#### 2-1选择器
1. 把**寻址**退化到**最简单**的情况：在两个地址中，去选择一个地址，即**2-1选择器**
2. 2-1选择器的组成：一个**反相器**、两个**与门**、一个**或门**
   - 通过控制反相器的输入是0还是1，来决定对应的输出信号，是和地址A还是地址B的输入信号一致
3. 一个反相器只能有0和1两种状态，所以只能从两个地址中选择一个，如果输入的信号有三个不同的开关，称为**3-8译码器**
   - 现代计算器的CPU是64位，意味着**寻址空间**为**$2^{64}$**，需要一个有**64个开关的译码器**

<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-2-1-selector.jpg" width=800/>
<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-decoder-memory.jpg" width=800/>

#### 本质
1. 译码器的**本质**，是从输入的多个位的信号中，根据**一定的开关和电路组合**，选择出自己想要的信号
2. 除了**寻址**外，还可以通过译码器，找出**期望执行的指令**，即**opcod**e，以及后面对应的**操作数**或**寄存器地址**

## 简单的CPU
<img src="https://computer-composition-1253868755.cos.ap-guangzhou.myqcloud.com/computer-organization-build-data-path-cpu.jpg" width=800/>

1. **自动计数器**
   - 自动计数器会随着**时钟主频**不断地**自增**，作为**PC寄存器**
2. **地址译码器**
   - 地址译码器需要连着通过大量的**D触发器**组成的**内存**
3. 自动计数器随着时钟主频不断自增，从地址译码器当中，找到对应的计数器所表示的**内存地址**，然后读取出里面的**CPU指令**
4. 读取出来的CPU指令会通过**CPU时钟**的控制，写入到一个**由D触发器组成的寄存器**，即**指令寄存器**
5. **指令译码器**
   - 指令译码器不是用来寻址的，而是将拿到的指令，解析成**opcode**和对应的**操作数**
6. 拿到对应的opcode和操作数，对应的**输出线路**要连接**ALU**，开始进行各种**算术**和**逻辑**运算
   - 对应的**计算结果**，再**写回**到**D触发器组成的寄存器**或者**内存**中