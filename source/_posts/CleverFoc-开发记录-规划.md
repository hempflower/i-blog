---
title: 'CleverFoc 开发记录: 规划'
date: 2023-04-21 19:27:52
tags: [技术,foc,cleverfoc]
---

## 背景

最近因为被安排给一个无刷电机控制器写控制代码，因此与 FOC 打上了交道。在准备开发一个项目前，调查了一番现有的 FOC 项目，以下是调查的结果：

<!--MORE-->

### SimpleFoc

SimpleFoc 是一个基于 Arduino 的 Foc 库。借助 Arduino 的生态，可以轻易移植到支持 Arduino 平台的 MCU。但是依赖 Arduino 也会带来一个缺点，一旦所使用的 MCU 不支持 Arduino, 移植的难度将直线上升。   

更糟糕的是 SimpleFoc 并非只依赖 Arduino 就实现了 FOC 功能。在 `/src/drivers/hardware_specific/` 处可以看到有大量依赖具体硬件平台的代码。这也就意味着，如果 SimpleFoc 库没有支持目前所使用的单片机，即使有 Arduino 的支持，也需要移植。不支持 Arduino 平台的单片机更是双份痛苦。粗略查阅官方文档后，发现关于移植方面的说明也比较少。。。

### ODrive

ODrive 是一整套 FOC 方案，因此软件的通用性也不好，不符合需求。    

> 还有一些开源的 FOC 项目，但无一例外的高度依赖于某一个平台，故不过多提及。

## 规划

既然没有符合要求的项目，那么就要自己设计一个 FOC 项目了。又是开新坑的时候，~~可是我挖坑不填的~~第一步要制定大体的路线，CleverFoc 的主要特点包含以下几点：

- 高度可移植性。CleverFoc 应该不依赖任何平台，它本身是一个纯 C 库，只要在目标平台实现了所需的接口，就能正常运行。
- 尽可能少的第三方依赖，避免出现第三方依赖的兼容性问题
- 简单的架构。便于进行二次开发
- 丰富的文档。一个项目的文档是关键。好的文档可以吸引其他开发者的兴趣

### 技术路线

- [ ] 设备驱动框架
    - [ ] PWM 设备
    - [ ] 电流检测设备
    - [ ] 编码器/霍尔传感器
- [ ] FOC 核心
    - [ ] 电流环控制
    - [ ] 速度环控制
    - [ ] 位置环控制
- [ ] 平台抽象层

> 以上是初步的规划，随着开发的进行，可能会有所调整。

## 已知难点

FOC 部分功能实现往往依赖硬件平台的特性，例如电流采样的触发方式，有的单片机支持 TRGO 信号采集，有的支持 PWM 中断采集。如果 CleverFoc 抽象程度过高，就会导致底层特性不能很好的应用，因此平衡好抽象程度是项目的一大关键。

CleverFoc 不假定运行在任何一个平台上，因此时间信息没法直接获取，对于一些依赖与时间的逻辑，时间信息是比较关键的，需要设计一个跟时间有关的接口，解决这一问题。

