---
title: BLE 数据交换：基于 ESP 官方代码的实践
date: 2026-06-24 20:30:00 +0800
categories: [Embedded, BLE]
tags: [ble, esp32-c3, esp-idf]
---

> 本文以 ESP-IDF 官方示例 Bluedroid_GATT_Server 为主线，梳理 BLE 数据交换的完整流程。
官方代码仓库地址：<https://github.com/espressif/esp-idf/tree/master/examples/bluetooth/bluedroid/gatt_server>
{: .prompt-info }

## 问题背景

- BLE 数据交换是 BLE 协议栈中最核心的功能之一，涉及到服务、特性、描述符的定义与操作，以及客户端与服务端之间的数据传输。理解 BLE 数据交换的基本模型和操作方式，对于开发 BLE 应用至关重要。
- 相信大家学习 BLE 协议栈事件时，都会遇到一些困惑，比如：
  - Attribute、Service、Characteristic、Descriptor 的关系是什么？
  - GATT 层次结构是如何组织的？
  - Read、Write、Notify、Indicate 四种操作的区别和使用场景是什么？
  - ESP 官方代码中，服务与特性的定义、读写回调的实现、Notify/Indicate 的触发是如何实现的？
  - MTU 协商对单包数据量的影响，以及 Data Length Extension 的作用是什么？
  - 别着急，Aries 会把自己在学习过程中的疑惑和官方代码分析整理成一篇文章，帮助大家更好地理解 BLE 数据交换的完整流程。有问题的同学可以联系邮箱：<k75439607@gmail.com>，或者在评论区留言，Aries 会和你一起讨论解决。

## 数据交换的基本模型

<!-- Server / Client 角色,Attribute、Service、Characteristic、Descriptor 的关系 -->
- 在 BLE 数据交换中，通常有两个角色：服务端（Server）和客户端（Client）。服务端是拿着数据的一方，例如温湿度传感器；客户端是请求数据的一方，例如手机 App/电脑
- Aries 在大一学习数据结构时非常迷茫，像栈、堆、链表、队列这些还好，它们的实现不太难理解。这里拿链表举例，一个最小的链表中包含 Value 和 Next 即可，但是 Aries 学到树、图这些数据结构的关系和操作方式时，我是真懵了。为什么要提一嘴数据结构哩？因为在 BLE 当中也有一个最小的数据单元 -> attribute，我们在 BLE 中所有的 GATT 操作的访问最小单元就是 attribute，这个在我们往后的学习当中会经常见到。至于 attribute 的结构和作用，Aries 会在下面展开讲解
- 既然 attribute 是最小的数据单元，那大家就会想，在 attribute 之上，可以构成哪些更大的数据结构呢？答案是：Characteristic，Service。下图简单展示了三者之间的关系，其中最小的数据单元就是 attribute
![BLE GATT 层次结构手写图](/assets/img/posts/ble-data-exchange/whiteboard_exported_image.png){: width="400" height="700" .center-img }

_图 1：BLE GATT 层次结构手写图_

上图简单展示了 attribute，Characteristic，Service 三者之间的关系，一个 Characteristic 通常由 2 个或者 3 个 attribute 构成，Service 是由一堆 Characteristic 加一个 Service Declaration 构成。
我们在了解完 attribute，Characteristic，Service 三者之间的关系后，接着来分析 attribute 的结构组成。最权威的参考资料一定是 SIG 的 Core Specification[^1]，不过，在初学阶段直接看 Core Specification 太枯燥了！这里 Aries 也整理好了 attribute 的组成

| 字段  | Type   | Handle    | Value | Permission   |
|:---:|:------:| --------- | ----- | ------------ |
| 含义  | 属性类型   | 属性句柄      | 属性值   | 属性权限         |
| 长度  | 16 bit | 16/128 bit | 不定长   | 未定义，常见为 16 bit |

`TYPE` 段描述属性的类型，大小是 128 bit，其值是一个 UUID。UUID 是通用唯一识别码，这里用来表示属性的类型。大家如果想了解有哪些 UUID，[那么客官请点击查阅](https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/Assigned_Numbers/out/en/Assigned_Numbers.pdf)。回头看我们的[图一](/assets/img/posts/ble-data-exchange/whiteboard_exported_image.png)，是不是很像 Excel 表格（不像的话，你就想象一下吧），`Handle` 段就相当于 Excel 的行号，这里表示该属性在属性表的哪一行，你可以把它当作一个定位符。`Value` 段存储的是用户数据和描述性元数据。`Permission` 段指示了属性的加密/授权所需的安全级别和读写权限

  接着来看在属性之上实现的特征和服务。首先我们先看特征，这里请出我们的常住嘉宾[图一](/assets/img/posts/ble-data-exchange/whiteboard_exported_image.png)。图中可以看出，一个特征由 2 个必需属性（特征描述符属性、特征值属性）和一个可选的特征配置符属性组成。在 C 语言中，如果我们要定义一个宏，首先得写 `#define`，那么 C 编译器在读到源文件的 `#define` 这个 token 时，就知道后面跟的是一个宏，宏后面是替换体。替换体每次都必须要写吗？不必然。我们来做一个类比：特征描述符属性就是 `#define`，特征值属性就是宏，特征描述配置符属性就是替换体。这个野路子理解起来是不是很清晰[旺柴]，下面我们来对这三个属性进行剖析

- 特征声明属性：该属性的 UUID 是 `0x2803`，表明这个属性是特征声明属性。Permission 是 `Read Only`（别问为啥是只读，你家单元楼门牌号物业也不会闲得没事给你天天换），Value 段内细分为三个组成单元：**`属性`** 规定了这个特征所允许的 GATT 操作，**`UUID`** 是所声明特征的 UUID，可不是该属性的 UUID，**`Handle`** 是特征值属性的 handle 值

### Attribute 与 GATT 层次结构

<!-- Handle、UUID、Service → Characteristic → Descriptor -->

## 数据交换的四种操作

### Read（读取）

<!-- 问题背景 / 官方依据 / ESP 官方代码 / 实验验证 -->

### Write（写入）

<!-- Write with Response vs Write without Response -->

### Notify（通知）

<!-- 服务端主动推送,无需 ACK;CCCD 配置 -->

### Indicate（指示）

<!-- 服务端主动推送,需要 ACK;与 Notify 的区别 -->

## ESP 官方代码分析

<!-- 选取的官方示例:路径、关键 API、回调结构 -->

### 服务与特性的定义

<!-- gatt_svc_def / 特性表的注册 -->

### 读写回调的实现

<!-- access callback 中如何处理 read / write -->

### Notify / Indicate 的触发

<!-- 发送通知的 API 调用与时机 -->

## MTU 与数据长度

<!-- MTU 协商对单包数据量的影响,Data Length Extension -->

## 实验验证

<!-- 抓包、串口日志、寄存器或调试现象 -->

## 小结

<!-- 关键结论与后续待研究的问题 -->

[^1]: <https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/Core-54/out/en/host/attribute-protocol--att-.html>
