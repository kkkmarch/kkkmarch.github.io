---
title: BLE 数据交换：基于 ESP 官方代码的实践
date: 2026-06-24 20:30:00 +0800
categories: [Embedded, BLE]
tags: [ble, esp32-c3, esp-idf]
---

> 本文以 ESP-IDF官方示例代码Bluedroid_GATT_Server实例代码为主线,梳理 BLE 数据交换的完整流程。
官方代码仓库地址：<https://github.com/espressif/esp-idf/tree/master/examples/bluetooth/bluedroid/gatt_server>
{: .prompt-info }

## 问题背景

- BLE 数据交换是 BLE 协议栈中最核心的功能之一，涉及到服务、特性、描述符的定义与操作，以及客户端与服务端之间的数据传输。理解 BLE 数据交换的基本模型和操作方式，对于开发 BLE 应用至关重要。
- 相信大家学习BLE协议栈事件时，都会遇到一些困惑，比如：
  - Attribute、Service、Characteristic、Descriptor 的关系是什么？
  - GATT 层次结构是如何组织的？
  - Read、Write、Notify、Indicate 四种操作的区别和使用场景是什么？
  - ESP 官方代码中，服务与特性的定义、读写回调的实现、Notify/Indicate 的触发是如何实现的？
  - MTU 协商对单包数据量的影响，以及 Data Length Extension 的作用是什么？
  - 别着急 Aries将把自己在学习过程中的疑惑和官方代码分析整理成一篇文章，帮助大家更好地理解 BLE 数据交换的完整流程。有问题的同学可以联系邮箱：<k75439607@gmail.com>或者在评论区留言，Aries会和你一起讨论解决。

## 数据交换的基本模型

<!-- Server / Client 角色,Attribute、Service、Characteristic、Descriptor 的关系 -->
- 在BLE数据交换中，通常有两个角色：服务端（Server）和客户端（Client）。服务端是拿着数据的一方，例如温湿度传感器；客户端是请求数据的一方，例如手机APP/电脑
- Aries在大一时学习数据结构是非常迷茫的，像栈，堆，链表，队列...这些还好，他们的实现不太难理解，这里拿链表举例，一个最小的链表中包含Value，Next即可，但是Aries学到树、图等数据结构的关系和操作方式，我是真懵了。为什么要提一嘴数据结构哩 因为在BLE当中也有一个最小的数据单元->attribute，我们在BLE中所有的GATT操作的访问最小单元就是attribute，这个在我们往后的学习当中会经常见到，至于attribute的结构和作用，Aries会在下面展开讲解
- 既然attribute是最小的数据单元，那大家就会想，在attribute之上，可以构成哪些更大的数据结构哪？答案是：Characteristic，Service
他们之间的关系
![BLE GATT 层次结构手写图](/assets/img/posts/ble-data-exchange/whiteboard_exported_image.png)
_图 1：BLE GATT 层次结构手写图_

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
