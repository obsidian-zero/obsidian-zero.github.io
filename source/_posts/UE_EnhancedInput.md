---
title: UE增强输入系统
categories: 
- UE
- EnhancedInput
tag:
- program
- ue
date: 2024-10-22 00:00:00
updated: 2024-10-22 00:00:00
---

# 前言

增强输入系统的相关笔记。

在个人项目中，尝试使用了增强输入的**ComboTrigger**作为增强输入的功能，但是在实际运行中发现一些问题。因此根据问题追溯源码来进行查阅。

<!-- more -->

项目中，采用W，S，A，D，LC（鼠标左键）作为直接的**InputAction**。

然后使用**W-S-A-LC**，**S-A-LC**和**LC**也作为**IA**，绑定到技能触发。在实际操作中，发现**W-S-A-LC**永远触发先于**S-A-LC**，而**LC**则先于他们两者。

由此开始探讨，**EnhancedInputComponent**究竟是如何处理的输入，而**ComboTrigger**又是如何触发的。

# 基本概念

## KEY(输入)

对应实体的输入，包括一个按钮的按下/鼠标的点击/鼠标移动等游戏引擎中真实

## InputAction(IA)

增强输入系统下，会将本身**KEY**最终封装为一个**InputAction**，**InputAction**本身也可以作为其他**InputAction**的组成来源。

**InputAction**是最终和游戏内逻辑绑定的一层。

### InputTrigger

修饰IA的起效条件，将简单的键盘按压上做出多帧按压、松开等

### InputModifier

修改IA的生效值，将IA起效时获得的数值进行更改

> Trigger和Modifier能在IA本身和IMC上配置，先应用IMC的，后应用IA的

## InputMappingContext

用来管理**Key**到**InputAction**的映射关系。

反映了角色在这个时候，能够根据哪些输入产生哪些**InputAction**

## FEnhancedActionKeyMapping

结构体绑定了Key和InputAction的关系

## UEnhancedPlayerInput

增强输入系统（Enhanced Input System）的组成部分。它用于处理玩家输入，并提供更灵活和强大的输入管理功能。以下是关于 `UEnhancedPlayerInput` 的一些关键信息：

- **EnhancedActionMappings**：所有玩家绑定了**Key**的**InputAction**
- **ActionInstanceData**：所有**IA**的实例集合，包括了**IA**的触发值，上次触发时间，目前的触发状态等信息

## ETriggerState

确定触发器的触发状态，包含：

- **None**：无输入
- **Ongoing**：正在监控中
- **Triggered**：已经被处罚

## ETriggerEvent

- **None (0x0)**: 没有发生任何重要的触发状态变化，也没有设备输入。这是默认状态，当没有任何触发器活动时，这个状态会被隐藏（`UMETA(Hidden)`）。
- **Triggered (1 << 0)**: 触发器已成功触发（即从 `None` 或 `Ongoing` 状态转换到 `Triggered`）。这是最常见的事件，用于表示触发器已完成。
- **Started (1 << 1)**: 触发器状态开始评估。例如，当用户开始按下按键时，触发器从 `None` 变为 `Ongoing` 或直接跳到 `Triggered`。这个事件通常会在 `Triggered` 事件之前发生。
- **Ongoing (1 << 2)**: 触发器正在处理中。例如，在“按住键不放”的情况下，触发器会处于 `Ongoing` 状态，直到满足条件。这意味着触发器条件还未完全满足，但正在进行中。
- **Canceled (1 << 3)**: 触发器评估被取消。例如，在“按住键不放”的情况下，用户在满足条件之前松开了按键，导致触发过程被中断。触发器从 `Ongoing` 状态返回到 `None` 状态。
- **Completed (1 << 4)**: 触发器从 `Triggered` 状态转换回 `None`，表示触发器完成并重置。例如，用户松开了键，触发过程结束。

# EnhancedInput处理过程

主要在每一帧调用的 `UEnhancedPlayerInput::EvaluateInputDelegates` 这段代码中。它会将根据按键情况，转换成最后的输入。

## 遍历所有 EnhancedActionMappings（确定触发状态）

1. 遍历所有**IMC**中需要处理的**IA**，查找对应**Key**的状态，按下或松开。
2. 根据之前的缓存情况，判断**key**本次的具体情况，如已经按下/已经松开/按下多帧等具体情况
3. 根据**IMC**中配置的触发器和修改器，确认本次**IA**的触发情况和结果值。

> 按照目前配置的Combo情况，是在这里触发的Trigger检测。
>
> 注意这里虽然Trigger有触发检测，得到触发状态 **ETriggerState** ，但是实际上并没有应用到InputValue的触发事件 **TriggerEvent** 上。

## 处理注入的 InputAction

为了调试方便，可以直接注入到每一帧要处理的输入操作。在这里将注入的**InputAction**直接处理。

## 遍历所有 ActionInstanceData 处理InputAction （确定触发事件）

遍历所有的**IA**实例，对于本帧内有触发的**IA**，再次应用**IA**上本身的触发和修改器。修改本IA的触发情况。

> 只有这一步会修改IA的触发事件 **TriggerEvent** ，也因此，依赖IA的TriggerEvent 的Trigger会在下一帧内才能够检测到

## 触发EnhancedInputComponent上绑定的委托

遍历所有绑定了**InputAction**的委托，检测对应的**InputAction**是否有触发事件。

这一步先不触发，而是将其保存成一个 **TArray**，所有为**ETriggerEvent::Started**的委托会排到前面，优先触发。

```c++
// EnhancedPlayerInput.cpp:545
static TArray<TUniquePtr<FEnhancedInputActionEventBinding>> TriggeredDelegates;
for (const TUniquePtr<FEnhancedInputActionEventBinding>& Binding : IC->GetActionEventBindings())
{
    if (const FInputActionInstance* ActionData = FindActionInstanceData(Binding->GetAction()))
    {
        const ETriggerEvent BoundTriggerEvent = Binding->GetTriggerEvent();
        if (ActionData->TriggerEvent == BoundTriggerEvent ||
            (BoundTriggerEvent == ETriggerEvent::Started && ActionData->TriggerEventInternal == ETriggerEventInternal::StartedAndTriggered))
        {
            
            // started类型的触发事件放在第一个
            if (BoundTriggerEvent == ETriggerEvent::Started)
            {
                TriggeredDelegates.EmplaceAt(0, Binding->Clone());
            }
            else
            {
                TriggeredDelegates.Emplace(Binding->Clone());
            }

           	// 缓存本帧处理的InputAction,在Chord类型要用到
            if (BoundTriggerEvent == ETriggerEvent::Triggered)
            {
                TriggeredActionsThisTick.Add(ActionData->GetSourceAction());
            }
        }
    }
}
```

触发所有绑定的委托，如果对应的**InputAction**带有消耗输入的逻辑，则将涉及到的**Key**消耗掉

```c++
// EnhancedPlayerInput.cpp:577
for (TUniquePtr<FEnhancedInputActionEventBinding>& Delegate : TriggeredDelegates)
{
    TObjectPtr<const UInputAction> DelegateAction = Delegate->GetAction();

    // ...略去一些无关内容
    if (const FInputActionInstance* ActionData = FindActionInstanceData(DelegateAction))
    {

        if (const FKeyConsumptionOptions* ConsumptionData = KeyConsumptionData.Find(ActionData->GetSourceAction()))
        {
            if (static_cast<uint8>(ConsumptionData->EventsToCauseConsumption & Delegate->GetTriggerEvent()) != 0)
            {
                // 判断是否要消耗Key
                for (const FKey& KeyToConsume : ConsumptionData->KeysToConsume)
                {
                    ConsumeKey(KeyToConsume);	
                }
            }
        }
        Delegate->Execute(*ActionData);
    }

}
```

## 保存信息

保存一些本次触发相关的信息，包括时间等

# Combo的触发流程

## Trigger始终检查状态

Combo作为一个特别的Trigger，会在InputAction遍历的时候始终保持Trigger的状态检查。即：

```c++
// InputTrigger.cpp:238
UInputTriggerCombo::UInputTriggerCombo()
{
	bShouldAlwaysTick = true;
}
```

## ComboActions

Combo输入时，每一步的InputAction和触发时间

## 维护Combo IA执行进度

Trigger其本身实例会维护Combo的进度，即 **CurrentComboStepIndex** 和 **CurrentAction**，每一帧都会获取到 **EnhancedInput**上判断**CurrentAction**的触发状态满足，满足则进入下一个等待，不满足则计时等待重置。

如果中间有不按照顺序触发的IA，也会触发重置

> 出于个人喜好，额外增加了总体时间的配置，要求Combo整体触发时间在某个范围内。

## ComboTrigger总结

**ComboTrigger**本质上就是一个独立的Trigger，自行维护自己的触发条件。之间也并没有顺序关系的设置。最开始的问题需要更深一步探讨。

# 总结

回到最开始的问题，很明显，如果Trigger本身没有先后优先级的话，同一帧内决定触发的顺序就是绑定的顺序。

因此对照着发现，绑定**GA**时，**W-S-A-LC**对应的绑定输入在**S-A-LC**之前，他们作为同样的**ComboIA**。本身触发在同一帧内，所以每次触发时，都优先触发**W-S-A-LC**。

而本身**LC**，作为一个初层的**IA**，组成**Combo**的一部分，必然在**Combo**触发前触发。如果想要同一帧判断的话，为**LC**做仅有一层的**ComboIA**即可
