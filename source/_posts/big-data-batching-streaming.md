---
title: 大数据 -- 批处理 + 流处理
mathjax: false
date: 2019-06-28 10:14:20
categories:
    - Big Data
tags:
    - Big Data
---

## 无边界数据 + 有边界数据
1. **无边界数据**（Unbounded Data）：一种不断增长、无限的数据集，也叫**流数据**（Streaming Data）
2. **有边界数据**（Bounded Data）：一种有限的数据集
3. 把无边界数据按照时间窗口提取一部分，将成为有边界数据，有边界数据可以看作无边界数据的一个**子集**

<!-- more -->

## 事件时间 + 处理时间
1. 事件时间（Event Time）：数据**实际产生**的时间点
2. 处理时间（Processing Time）：处理数据的系统架构**实际接收**到这个数据的时间点

## 批处理
1. 批处理：一系列相关联的任务按顺序（或并行）一个接一个地执行
2. 批处理的输入是在一段时间内已经收集保存好的数据，每次批处理所产生的输出也可以作为下一次批处理的输入
3. 绝大部分情况下，批处理的输入数据都是**有边界数据**，输出结果也一样是有边界数据，批处理更多关注的是**事件时间**
4. 在许多情况下，批处理任务会被安排，以**预先定义好的时间间隔**来运行，例如信用卡消费账单
5. 批处理架构的应用场景：日志分析、计费应用程序、数据仓库
6. 开源项目（由Google MapReduce衍生）：Apache **Hadoop**、Apache **Spark**
7. 批处理任务具有**高延迟性**

## 流处理
1. 流处理：系统需要接收并处理一系列连续不断变化的数据
2. 流处理的输入数据基本上都是**无边界数据**，而流处理系统将依据具体的应用场景来关注数据的事件时间还是处理时间
3. 流处理的特点：**高吞吐**、**低延迟**，流处理所需的响应时间应该以**毫秒**（或**微秒**）来进行计算
4. 流处理速度快的原因：在数据**到达磁盘之前**就已经对其进行了分析
5. 实时处理 + 准实时处理
    - **实时处理**：系统架构拥有在**一定**时间间隔（**毫秒**）内产生逻辑上正确的结果
    - **准实时处理**：系统架构可以接受以**分钟**为单位的处理**延时**
6. 流处理架构的应用场景：实时监控、实时商业智能（如智能汽车）、实时分析
7. 开源项目：Apache **Kafka**、Apache **Flink**、Apache **Storm**、Apache **Samza**