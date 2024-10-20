---
title: BUFF框架的基本理解
categories: 
- gameprogram
tag:
- program
- gameplay
- blog
hidden: true
date: 2024-10-19 00:00:00
updated: 2024-10-19 20:00:00

---

# 前言

在游戏开发中，（buff）框架是一种用于管理角色能力增强的系统。它能够处理各种增益效果的应用、持续时间、叠加和取消等，通常涉及属性修改和状态变化等功能。

<!-- more -->

## BUFF表字段

BUFF表主要包括以下几个字段：

1. **基本效果**：包括BUFF的持续时间、最大叠加层数、叠加后升级的BUFF、标记划分及免疫关系、BUFF移除后生成的BUFF。
  
2. **表现关系**：包括特效表中的特效字段和动画字段。比如，对于眩晕效果，我们可能需要不同的动画表现，如角色被眩晕或以其他形式表现。

3. **逻辑字段**：如伤害字段和生命值，因我们的需求主要集中在伤害效果上，生命值作为单独列出。还包括环境交互的黑白名单功能，例如开灯、开门等交互效果的配置。

4. **效果标记**：用于标记BUFF的具体效果。例如，effect=1表示带有眩晕功能，effect=7表示护盾效果，阻挡其他BUFF的添加。这些效果也可能涉及反向、冲锋和隐身等状态。

5. **属性列**：包括加伤、减防、移动速度等基本属性，允许BUFF在持续时间内影响这些属性。

6. **额外组件效果**：定义BUFF期间所具有的额外逻辑或功能，例如定时添加BUFF、技能效果或改变角色动画等。

## BUFF添加流程

BUFF添加的基本流程包括：

1. **基本检测**：检查BUFF的添加者和被添加者是否有特殊组件或技能效果阻止添加。

2. **交互检测**：某些BUFF可能会被标记为护盾，从而消耗护盾层数。需检查角色身上的BUFF是否存在免疫字段以阻止添加。

3. **伤害计算**：使用BUFF上的伤害字段计算生命值的变化。

4. **二次检测**：检查是否存在免疫效果但不免疫伤害的情况，若有则结束BUFF添加流程。

5. **BUFF实例化**：实际添加的BUFF将作为实例挂在角色身上。如果已存在相同BUFF，则修改添加者并重新进行添加流程，处理叠层及衍生BUFF。

6. **效果结算**：根据BUFF的effect值调整角色状态机并播放相应动画。

## 常用功能

### 属性修改和效果触发

BUFF的基本功能就是属性修改

### 标记情况

通过标记来传递信息，通过存在BUFF调整逻辑

### 确定逻辑

通过BUFF添加之类的结果，在技能中判断逻辑分支

## 常见问题

### 多来源化问题

由于BUFF系统是独立实例，重新添加可能导致添加者标记转移。为解决此问题，新版本的法术场记录进入的角色，并在每帧检查其BUFF状态，确保持续性BUFF正常工作。

### 工具链不完善

之前缺乏实时记录BUFF状态的功能，使得QA无法清楚理解战斗中发生的事件。因此，我们完善了统计面板，通过消息机制捕获游戏内事件，包括技能使用、准备动作运行、BUFF的添加来源逻辑、免疫原因和伤害计算结果等。这一工具显著提升了调试效率，使问题排查变得更加高效。

### 缺乏持续调整能力

缺乏BUFF在持续期间动态修改角色属性，比如逐步增加或减少角色攻击力、体型等。但是这方面可能更倾向于通过BUFF组件和额外BUFF实现逻辑。持续性的修改可能会有其他问题。

如GAS可以周期性修改一个属性，但是这会导致属性修改的结果无法回退。只能反方向增加。

### 免疫关系直接

通过BUFF直接造成效果，但是会导致BUFF期间难以临时性接触。如减速期间可以直接减速眩晕，但是不可以让一段时间内暂时减速效果无效。