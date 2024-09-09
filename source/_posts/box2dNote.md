---
title: 碰撞和Box2D的一些笔记
categories: 
- gameplay
tag:
- program
- gameplay
- 
date: 2024-09-09 00:00:00
updated: 2024-09-09 20:00:00

---

# 前言

一点对于box2D的使用的学习笔记。

<!-- more -->

# 碰撞管理

## 脚本响应物理事件 b2PhysicsContactListener

**b2PhysicsContactListener**是一个box2d本身提供出来用于处理物理事件的类。它提供了 `BeginContact`、`EndContact`、`PreSolve`、`PostSolve`四个方法处理事件。

项目中使用这个接口进行对于物理碰撞的脚本控制。根据目前代码判断，主要使用的即是`BeginContact`、和`EndContact`两个方法。

## 重定碰撞类型

通过box2d的一些学习可以了解到，能在begin和presolve两个阶段中` contact:setEnable(false)`来关闭碰撞。

为了能够项目中更可控地处理碰撞关系，在脚本层我们定义了所有的碰撞关系，分为四类：

- 不产生任何碰撞效果，关闭contact
- 不产生任何碰撞效果，关闭contact，但是会产生一个碰撞事件允许脚本进行处理
- 产生碰撞但是不允许穿透、会发生碰撞事件
- 产生碰撞，只有在从下往上以一定速度时发生穿透

目前有大约200多种不同的物理碰撞TAG。通过`beginContact`中读取到两个**fixture**的对应Tag后，根据脚本中的碰撞表进行事件派发和处理。

## 平台

在2D平台跳跃游戏中，必然会存在平台机制，可以从下往上进行穿透。平台作为一种特殊的物理存在有着许多特殊的规则。

对于平台这个机制的处理相关规则如下：

1. 对于产生碰撞的，对碰撞双方进行分析：如果一个为static、一个为kinematic则判断static为平台。否则根据相对速度进行判断。
2. 对于碰撞为下往上穿透的平台，只有一定速度后才会关闭contact
3. 对于平台额外发出平台的落地事件以供响应，并记录下平台的接触信息供AI等其他功能使用
