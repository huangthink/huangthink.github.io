---
layout: post
title: "使用MAT(Memory Analyzer Tool)分析内存泄漏"
date: 2015-07-08 18:21:16 +0800
comments: true
categories: blog
tags: [能工巧匠]
---
在工作中，有时会遇到OutOfMemoryError，我们知道遇到Error一般表明程序存在着严重问题，可能是灾难性的。所以找出是什么原因造成OutOfMemoryError非常重要。Memory Analyzer tool(MAT)来化解我们遇到的难题。

<!-- more -->

从可用内存和请求数量的变化情况判断是突发性的内存泄露还是不断积累的结果换句话说，首先定位内存泄露的性质：

突发内存泄露：

- 表现：可用内存直线下降，短时间内消耗殆尽。
- 原因：定时任务执行、用户集中访问等，频繁调用有内存泄露的代码块。
- 排查：这种情况相对比较容易定位问题，查看可用内存骤降的时间点，从日志中查找对应时间点系统的行为（前提是日志必须很规范，有据可循）。

隐式内存泄露：

- 表现：可用内存持续下降，随着系统运行不断减少。
- 原因：偶发性调用到有内存泄露的代码块。
- 排查：这种情况需要借助dump文件和内存分析工具来排查。常用方式：在系统启动后获取dump文件，运行一段时间后（可用内存明显减少），再次获取dump文件（如果触发了JVM的OOM，可以直接拿到dump），对比前后两个时间点的内存占用情况，找出占用内存大幅增长的类。如果足够明显，可以直接从自动生成的dump文件中发现占用内存过多的类。

分析不同服务器的内存使用情况可以进一步缩小排查范围

![](http://pic.yupoo.com/hunng/ER0GLkC4/z6DV7.jpg)

![](http://pic.yupoo.com/hunng/ER0GLZIg/SIDUg.jpg)

### 使用jmap工具生成dump文件

jmap是什么？简单来说，jmap是JDK自带的一种用于生成内存镜像文件的工具，通过该工具，开发人员可以快速生成dump文件。



> 进入目录：/usr/java/jdk1.6.0_33/bin
> 
> 找到服务的进程ID：`ps –ef | grep search`
> 
> 执行命令：`jmap -dump:format=b,file=<path<PID>`
> 
> 下载dump文件至本地

### MAT工具的下载和安装

MemoryAnalyzer.ini	可以设置最大/最小堆内存，与eclipse.ini配置类似

### 使用MAT工具进行内存泄露分析

当成功启动MAT后，通过菜单选项“File->Open heap dump...”打开指定的dump文件后，将会生成Overview选项，在概况中可以初步查看占用内存最多的几个类以及对应的一些属性、引用层次和统计信息

![](http://pic.yupoo.com/hunng/ER0GMiDK/WVSID.png)

在Overview选项中，以饼状图的形式列举出了程序内存消耗的一些基本信息，其中每一种不同颜色的饼块都代表了不同比例的内存消耗情况。如果说需要定位内存泄露的代码点，我们可以通过Dominator Tree菜单选项来进行排查(MAT工具仅仅只是一个辅助，分析OutofMemory并不存在一个固定的方式和准则，因此仔细观察和分析才能够找到问题所在)

![](http://pic.yupoo.com/hunng/ER0GMwlh/VM8YV.png)

Histogram图表中主要统计了消耗占比较高的类的实例数量及占用空间，Shallow Size: 对象自身占用的内存大小，不包括它引用的对象Retained Size: 当前对象大小 + 当前对象直接或间接引用的对象大小的总和 - 被GC Roots直接或间接引用的对象大小的总和

![](http://pic.yupoo.com/hunng/ER0GLXOt/jq1E4.png)

Top Consumers图表可以更直观地看到内存消耗占比最多的类，由于很多类都囊括在ClassLoader中，因此还需要进一步查看引用的层级。如果一次观察的结果不够明显，可以对比不同服务器（不同组）、不同时间点的内存使用情况，进一步缩小排查范围。

![](http://pic.yupoo.com/hunng/ER0GMkFO/WspKs.png)

List objects with outgoing/incoming references 根据引用层级查看问题根源

![](http://pic.yupoo.com/hunng/ER0GMuF4/pSdwz.png)

Leak suspects，MAT可以自动生成可疑泄露点的报告，能够从以下几点全面分析问题：

- Shortest Paths To the Accumulation Point
- Accumulated Objects in Dominator Tree
- Accumulated Objects by Class in Dominator Tree
- All Accumulated Objects by Class

![](http://pic.yupoo.com/hunng/ER0GNebl/kV2eP.png)
