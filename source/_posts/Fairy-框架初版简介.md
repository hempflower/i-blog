---
title: Fairy 框架初版简介
date: 2023-05-01 20:23:49
tags: ['技术','智能车','fairy']
---

目前版本实现了大部分的基础功能，可以应用到车上了。

代码地址(铁人内部仓库)：[https://code.ironspirit.cn/hempflower/fairy](https://code.ironspirit.cn/hempflower/fairy)

<!--more-->
## 功能

### 设备驱动框架

设备驱动框架提供了对智能车常用外设的一致抽象，降低上层控制逻辑与驱动程序的耦合程度。在设备驱动框架中，每一个设备都被视为一个对象。它们拥有自己的名称，类别，和操作接口。
    
使用设备时只需要通过 `air_device_find` 函数寻找对应的设备对象，并通过操作函数实现设备操作。

```c
air_device_t motor_dev = air_device_find("motor_left");
air_device_motor_set_speed(motor_dev ,0.3);
```

把现有的设备注册到设备框架也同样简单。如果设备类型是框架所支持的，可遵循框架定义的接口注册设备(请参考`include/builtin_devs`)。   

例如，注册一个电机设备

```c
struct air_device motor_dev = {0};
struct air_device_motor_ops ops = {
    .set_speed = you_set_speed, // 你的电机速度控制函数
    .set_brake = you_set_brake, // 你的电机刹车控制函数
};

air_device_init(&motor_dev, "left_motor", FAIRY_DEVICE_CLASS_MOTOR, &ops , FAIRY_NULL);
air_device_register(&motor_dev);

```

目前支持的设备:

- 电机
- 编码器
- 舵机
- IMU
- GPS

### 底盘抽象层

对于底盘，框架也提供了抽象的控制接口，目前版本中提供了速度设置，转向设置以及刹车功能，后续会根据实际使用情况调整抽象 API，底盘抽象 API 的定义请见 `fairy_chassis.h` 。   

设置底盘实现
```c
struct air_chassis my_chassis = {
    .set_speed = your_set_speed,
    .set_dir = you_set_dir,
};

air_chassis_set(&my_chassis);
```

控制底盘
```
air_chassis_set_speed(0.3);
air_chassis_set_dir(-0.3);
```

### 日志系统

日志系统可以协作定位问题，目前版本的日志系统支持多日志级别输出，通过 `printf` 输出日志，未来版本计划加入多后端输出支持。

```c
air_log_debug("Task1 passed!");
air_log_error("IMU init failed");
```

### 链表

虽然是一个主要被内部使用的功能库，但事实上独立于框架。由于是一个通用库，在此不多做介绍。

### 任务调度器

由于比赛的任务流程复杂多变，框架提供了较为灵活的调度机制，并称之为任务调度器。   

#### 什么是任务?

整个比赛通常由很多种阶段组成，例如环岛，坡道或者弯道。对于这些元素，往往需要针对它们单独编写处理逻辑，而任务正是对这些处理逻辑的称呼。   

每个任务有 3 个钩子函数，分别是 `enter`, `tick`, `leave`。

- `enter` 进入任务时触发
- `tick` 每次逻辑循环均会触发
- `leave` 离开任务时触发

调度器会定期检查任务是否满足进入或离开的条件，这一过程通过调用任务对象的 `task_should_enter` 和 `task_should_leave` 实现。



任务调度器的主要功能是根据当前情况执行预设的任务，分为两种模式：队列模式和事件模式。


#### 队列模式

队列模式将按照预先设计的顺序执行任务，这种模式下不会检查任务的触发条件，只检查当前任务是否满足离开条件，并自动切换到下一个任务。   

队列模式适合顺序固定的比赛任务，例如18 届越野组。

#### 事件模式

事件模式的任务切换完全依赖于任务触发条件和结束条件。任务调度器将第一个任务视为常驻任务，常驻任务是在没有其他任务触发执行的情况下默认执行的任务，一般用来实现最基础的循迹。

调度器定期检查 `task_should_enter`, `task_should_leave` 来决定任务的切换。为避免混乱，任务触发不可嵌套，最多有一个任务被触发。

以下是一个简单的事件模式的任务调度器示例：

```c
struct air_task backTask = {0};
struct air_task_hooks backTaskHooks = {
    .task_enter = your_enter_fn, // 任务进入时执行一次此函数
    .task_tick = your_tick_fn, // 任务 tick ，每次逻辑循环调用一次，频率取决于 air_dispatcher_on_tick 调用的频率
    .task_leave = your_leave_fn, // 任务离开时执行
    .task_should_leave = your_should_leave, // 任务是否应该结束
// 常驻任务不需要触发检查
};

// 任务添加
air_dispatcher_task_insert(&backTask,&backTaskHooks,"back",-1, FAIRY_NULL);

// 开始调度
air_dispatcher_start();

// 不要忘记在你的定时器或循环中调用这个函数, 否则任务调度器无法工作
// air_dispatcher_on_tick();
```
更多说明将在之后补充

### 虚拟文件系统

虚拟文件系统概念借鉴于 Linux , 针对单片机环境设计了一个精简版 VFS 系统，主要提供了文件系统挂载的能力。

#### 挂载文件系统

常见的场景是对单片机的 Flash 格式化为指定文件系统使用，通过 `air_vfs_mountfs` 函数即可实现文件系统的挂载。具体的代码示例如下：

```c
struct air_vfs_fs_ops fops = {
    .read = my_read,
    .write = my_write,
    // .... 具体请见 fairy_vfs.h 中的结构体定义
};
uint8_t buf[16];

// 将文件系统挂载到 /flash 下
air_vfs_mountfs("/flash", &fops);

// 实际将读取目标文件系统的 hello 文件
air_vfs_read("/flash/hello", buf, 16);

```

#### 挂载内存文件

vfs 支持将一段内存挂载为一个虚拟的文件，并支持基础的读写功能。~~你问我有啥用？我不到啊，后面万一有用呢~~

```c
uint8_t myfile[16];

air_vfs_mount_memfile("/myfile",myfile,sizeof(myfile));

```

## 还未实现

### 驱动

- 摄像头设备的支持
- 电磁设备的支持
- Flash 设备支持

### 内置调试功能

- 内置 NDB Server


嗯，该去享受假期了，后面的更新等假期结束再说吧。
