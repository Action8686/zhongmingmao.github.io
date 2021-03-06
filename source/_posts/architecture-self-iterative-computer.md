---
title: 架构 -- 自我迭代的计算机
date: 2019-05-02 08:24:21
categories:
    - Architecture
tags:
    - Architecture
---

## MVP
1. 驱动程序：**键盘**和**显示器**
2. 驱动程序：**外置存储**
3. 汇编程序**编辑器**：可以从存储中读取汇编程序代码，修改并保存到存储中
4. 汇编程序**编译器**：可以将汇编代码编译成机器代码，并保存在存储中
5. **执行机器代码的程序**：可以执行一段保存在外置存储设备中的机器代码

<!-- more -->

## 需求分析
1. _**准确的需求分析是做出良好架构设计的基础**_，在整个架构的过程中，至少应该花费**1/3**的精力在需求分析上
2. 在需求分析时，需要区分需求的**变化点**和**稳定点**：稳定点是系统的**核心能力**，变化点则需要对应地去考虑**扩展性**上的设计

## 核心部件
1. **中央处理器**：计算能力的核心
2. **存储**：除了作为计算的输入输出，还可作为**计算本身**的承载（程序，**主要变数**，主要分两类：**BIOS** + **外置存储的程序**）
    - 需要考虑BIOS和外置存储的程序的**具体职责**是什么
3. **输入输出设备**：键盘 + 显示器 + 外置存储（**主要变数**）
    - 对于键盘和显示器，只需要准备好驱动程序即可
    - 对于外置存储，准备好了驱动程序以后，需要考虑如何设计外置存储的**数据格式**

## 外置存储
1. 对比MVP的需求，外置存储需要保存的内容：汇编程序的**源代码**，汇编编译器编译出来的**可执行程序**
2. 因此，外置存储需要支持保存**多个文件**，衍生出来的问题：**如何组织多个文件**，可行的解决方案如下
    - **文件系统**，种类很多，但有**统一的抽象**
        - 文件系统是一棵树；节点要么是目录，要么是文件；文件必然是叶子节点；根目录是目录，目录可以有子节点
    - **键值存储系统**，可以做**统一的抽象**
        - 每个文件都有名字（Key），通过名字可以唯一定位到该文件，从而对文件内容进行读写
        - 支持对文件名做模糊查询（通配符）
        - 允许每个文件设定额外的元数据，通过元数据可以检索到相关的文件

## BIOS + 外置存储的程序
1. BIOS是刻在计算机主板**ROM**上的启动程序，变更BIOS是非常麻烦的，所以BIOS只做**稳定不变**的事情
2. 只要键盘、显示器和外置存储没有太大的演进，驱动程序是不需要变动的，比较**稳定**，可以交给BIOS负责
3. 汇编程序编辑器，编辑器的需求是模糊的，包含很多额外的交互细节，因此这应该交给外置存储的程序负责
4. 汇编程序编译器，依然有很大的不确定性，也应该交给外置存储的程序负责
    - CPU的指令会增加，汇编指令也会相应地增加，汇编语言及其编译器需要完整地呈现CPU的能力，因此需要及时跟进
    - 虽然汇编指令与机器指令基本是一一对应，但汇编指令是面向程序员的生产力工具，还是会演进出一些高阶的语法
    - 因此汇编语言并不稳定，也会迭代变化，这意味着汇编程序的编译器也会相应地迭代变化
5. 执行机器代码的程序，这也需要进一步细化，例如执行程序的方式是基于外置存储的物理地址还是基于文件系统的文件
    - 前者只需要依赖外置存储的驱动程序即可，后者还需要理解文件系统的格式

### BIOS交接控制权
1. CPU加电启动时，会从存储的一个**固定地址**（指向**BIOS**）开始执行指令
2. BIOS也会从一个**外置存储的固定地址**（**引导区**）来加载程序并执行，而**无需关心磁盘的数据格式**
3. _**引导区是BIOS和操作系统的边界**_
4. 对于BIOS来说，为了把控制权交给外置存储的程序，必须要能够执行外置存储的程序
    - BIOS只需要执行**引导区**的程序，该程序不会很长，可以直接读入到内存中，然后再执行

### 引导程序取得控制权后的需求
1. 支持识别外置存储的数据格式，提供统一的功能给其它程序使用，无论是文件系统还是键值存储系统
2. 提供管理外置存储的基础能力，例如查询外置存储里有什么文件，这可以实现为一个独立的程序_**ls**_
3. 支持执行外置存储上的可执行程序，这可以实现为一个独立的程序_**sh**_
4. 汇编程序编辑器，该程序与汇编语言没什么关系，是一个纯文本编辑器，这可以实现为一个独立的程序_**vi**_
5. 汇编程序编译器，这可以实现为一个独立的程序_**asm**_
6. 引导程序取得控制权后，最终需要把控制权交给sh程序，**sh程序是自我迭代的计算机扩展性的体现**

### BIOS的职责
1. 驱动程序：**键盘**和**显示器**
2. 驱动程序：**外置存储**
3. 支持跳转到外置存储的固定地址，把控制权交给该地址上的**引导程序**

### 外置存储程序的职责
1. 汇编程序编辑器（vi）
2. 汇编程序编译器（asm）
3. 执行机器代码的程序（sh）

## 需求变化点小结
1. 外置存储的**数据格式**，通过设计文件系统或键值存储系统来解决，另外提供ls等程序**管理外置存储中的文件**
2. 执行**引导程序**后会迭代出怎样的能力：设计了sh程序，支持执行外置存储上的任何程序
3. **编辑器的交互范式**，对此设计了vi程序，让它迭代编辑器的能力
4. **汇编语言的使用范式**，对此设计了asm程序，让它响应CPU指令集和汇编语言的迭代

<!-- indicate-the-source -->
