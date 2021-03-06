---
title: JVM基础 -- 运行过程+运行效率
date: 2018-12-14 20:00:03
categories:
    - Java
    - JVM
    - Baisc
tags:
    - Java
    - JVM
---

## 虚拟机视角
<img src="https://jvm-1253868755.cos.ap-guangzhou.myqcloud.com/basic/jvm-basic-jvm-vision.png" />

1. 将class文件加载到JVM中，加载后的Java类会被存放在**方法区**，实际运行时，虚拟机会执行方法区内的代码
2. JVM同样会将内存划分出**堆**和**栈**来存储运行时数据，栈会细分**本地方法栈**和**Java方法栈**
3. PC寄存器：用于记录**各个线程的执行位置**
4. 在运行过程中，每当调用进入一个**Java方法**，JVM会在**当前线程的Java方法栈**中生成一个**栈帧**
    - 栈帧用于存放**局部变量表**和**操作数**
    - 栈帧的大小是**提前计算**好的，并且JVM**不要求**栈帧在内存空间里**连续分布**
5. 当退出当前执行的方法时，不管是**正常返回**还是**异常返回**，JVM都会**弹出并舍弃当前线程的当前栈帧**

<!-- more -->

## 硬件视角
<img src="https://jvm-1253868755.cos.ap-guangzhou.myqcloud.com/basic/jvm-basic-hardware-vision.png" />

1. Java字节码无法直接执行，需要JVM将字节码翻译成机器码，有两种形式：**解析执行**+**即时编译**
    - 解释执行：逐条将字节码翻译成机器码并执行，**无需等待编译**
    - 即时编译（JIT）：将**一个方法中包含的所有字节码**编译成机器码后再执行，**实际运行速度更快**
2. HotSpot默认采用**混合模式**，**先解析执行** 字节码，然后将其反复执行的热点代码，**以方法为单位** 进行**即时编译**
    - 即时编译建立在**2-8定律**的假设之上
    - 对于占据大部分的不常用代码，无需耗费时间将其编译成机器码，而是采用解释执行的方式
    - 对于仅占小部分的热点代码，我们可以将其编译成机器码，以达到理想的运行速度

## JVM的运行效率
1. 理论上讲，即时编译后的Java程序的执行效率，是有可能超过C++程序的，这是因为与静态编译相比，即时编译拥有程序的**运行时信息**，并且能够根据这个信息作出**相应的优化**
2. 为了满足不同用户场景的需要，HotSpot内置了多个即时编译器：**C1**、**C2**和**Graal**（Java 10引入，实验性）
    - 引入多个即时编译器，是为了在**编译时间**和**生成代码的执行效率**之间进行取舍
    - C1又叫做**Client编译器**，面向对**启动性能**有要求的GUI程序，采用的优化手段相对简单，因此编译时间较短
    - C2又叫做**Server编译器**，面向对是对**峰值性能**有要求的服务端程序，采用的优化手段相对复杂，因此编译时间较长，但生成代码执行效率较高
3. 从Java 7开始，HotSpot默认采用**分层编译**的方式，**热点方法首先会被C1编译，而后热点方法中的热点会进一步被C2编译**
4. 为了不干扰应用的正常运行，HotSpot的即时编译是放在**额外的编译线程**中进行的，HotSpot会根据CPU的数量设置编译线程的数目，默认按**1:2**的比例配置给C1和C2编译器
5. 在计算资源充足的情况下，字节码的**解释执行和即时编译可同时进行**。解析完成后的机器码会在**下一次调用**该方法时启用，以替换原本的解释执行

<!-- indicate-the-source -->
