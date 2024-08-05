---
title: AnimNotifyState的GAS适配改造
categories: 
- UE
- GAS
tag:
- program
- ue
date: 2024-08-05 00:00:00
updated: 2024-08-05 20:00:00

---

# 前言

这是个人动作游戏DEMO的一些计数实现笔记，主要在于AnimNotifyState和GAS系统适配改造。

<!-- more -->

该游戏主要灵感来源为流星蝴蝶剑.net。

具备搓招操作和连招设计。这篇文章主要讨论如何实现的技能连招。主要围绕**AnimNotifyState**展开。

动作游戏不可避免地需要讨论各种各样的动作。

# AnimNotifyState

**AnimNofityState**是UE中，为动画或蒙太奇播放过程中触发特定逻辑的一种机制。

相较于**AnimNotify**只会在固定时间轴上触发的离散机制，**AnimNotifyState**提供了完整的开始、结束、Tick功能，因此在游戏开发中，更容易用**AnimNotifyState**来作为动画相关逻辑机制的载体。

## AnimNotifyState实例绑定于动画

在开发中可以发现，在一个动画中，添加多个相同的**ANS**时，会产生不同的实例。

但是对于一个动画，由不同的Actor进行播放时，会发现实例的地址都是相同的，即是多个不同的Actor会触发同样的实例。

这就导致了使用**ANS**时，不应该在内部存储状态。

> 这里个人采取的做法时将相关数据直接绑定到通过**MeshComponent**获得的**Actor**上。**ANS**只进行逻辑调整，不存储信息。
>
> 听着很像ECS的System。

## AnimNotifyState的触发逻辑 UAnimInstance::TriggerAnimNotifies

```C++
// AnimInstance.cpp:1473
	USkeletalMeshComponent* SkelMeshComp = GetSkelMeshComponent();

	// 定义新激活的ANS列表
	TArray<FAnimNotifyEvent> NewActiveAnimNotifyState;
	NewActiveAnimNotifyState.Reserve(NotifyQueue.AnimNotifies.Num());

	TArray<FAnimNotifyEventReference> NewActiveAnimNotifyEventReference;
	NewActiveAnimNotifyEventReference.Reserve(NotifyQueue.AnimNotifies.Num());

	
	// 定义要触发开始事件的ANE
	TArray<const FAnimNotifyEvent *> NotifyStateBeginEvent;
	TArray<const FAnimNotifyEventReference *> NotifyStateBeginEventReference;

	// 遍历所有NotifyQueue.AnimNotifies这个是什么还不确定
	for (int32 Index=0; Index<NotifyQueue.AnimNotifies.Num(); Index++)
	{
		if(const FAnimNotifyEvent* AnimNotifyEvent = NotifyQueue.AnimNotifies[Index].GetNotify())
		{
			// 如果AnimNoftiyEvent对应的是AnimNotifyState类型
			if (AnimNotifyEvent->NotifyStateClass)
			{
				int32 ExistingItemIndex = INDEX_NONE;

				if (ActiveAnimNotifyState.Find(*AnimNotifyEvent, ExistingItemIndex))
				{
                    //如果在已经激活的列表内，将其移除
					check(ActiveAnimNotifyState.Num() == ActiveAnimNotifyEventReference.Num());
					ActiveAnimNotifyState.RemoveAtSwap(ExistingItemIndex, 1, false); 
					ActiveAnimNotifyEventReference.RemoveAtSwap(ExistingItemIndex, 1, false);
				}
				else
				{
                    //未激活的ANS添加到BeginEvent列表中，后续对这个列表触发开始事件
					NotifyStateBeginEvent.Add(AnimNotifyEvent);
					NotifyStateBeginEventReference.Add(&NotifyQueue.AnimNotifies[Index]);
				}
				// 这时还存在的所有ANS单独保存一个成一个列表
				NewActiveAnimNotifyState.Add(*AnimNotifyEvent);
				FAnimNotifyEventReference& EventRef = NewActiveAnimNotifyEventReference.Add_GetRef(NotifyQueue.AnimNotifies[Index]);
				EventRef.SetNotify(&NewActiveAnimNotifyState.Top());
				continue;
			}

			// AnimNotify直接触发
			TriggerSingleAnimNotify(NotifyQueue.AnimNotifies[Index]);
		}
	}
```

- 遍历目前的所有**ANE**，对于**AN**类型直接触发事件
- 对于**ANS**类型，根据现有的已激活**ANS**，分成新激活和已激活的两类

```c++
// 遍历之前激活的动画列表，上一步中，已经将所有还存在的ANS移除了，因此实质上这里筛选的是所有已经结束了的ANS
	for (int32 Index = 0; Index < ActiveAnimNotifyState.Num(); ++Index)
	{
		const FAnimNotifyEvent& AnimNotifyEvent = ActiveAnimNotifyState[Index];
		const FAnimNotifyEventReference& EventReference = ActiveAnimNotifyEventReference[Index];
		if (AnimNotifyEvent.NotifyStateClass && ShouldTriggerAnimNotifyState(AnimNotifyEvent.NotifyStateClass))
		{
			{
				//触发结束事件
				AnimNotifyEvent.NotifyStateClass->NotifyEnd(SkelMeshComp, Cast<UAnimSequenceBase>(AnimNotifyEvent.NotifyStateClass->GetOuter()), EventReference);
			}
		}
		// The NotifyEnd callback above may have triggered actor destruction and the tear down
		// of this instance via UninitializeAnimation which empties ActiveAnimNotifyState.
		// If that happened, we should stop iterating the ActiveAnimNotifyState array
        // 一个容错处理，没太看懂，保留原文注释
		if (ActiveAnimNotifyState.IsValidIndex(Index) == false)
		{
			ensureMsgf(false, TEXT("UAnimInstance::ActiveAnimNotifyState has been invalidated by NotifyEnd. AnimInstance: %s, Owning Component: %s, Owning Actor: %s "), *GetNameSafe(this), *GetNameSafe(GetOwningComponent()), *GetNameSafe(GetOwningActor()));
			return;
		}
	}
```

对于所有已经不在的**ANS**调用结束事件

```c++
// 遍历之前筛选出的新ANS，触发开始事件
for (int32 Index = 0; Index < NotifyStateBeginEvent.Num(); Index++)
	{
		const FAnimNotifyEvent* AnimNotifyEvent = NotifyStateBeginEvent[Index];
		const FAnimNotifyEventReference * AnimNotifyEventReference = NotifyStateBeginEventReference[Index];
		if (ShouldTriggerAnimNotifyState(AnimNotifyEvent->NotifyStateClass))
		{
			{
				AnimNotifyEvent->NotifyStateClass->NotifyBegin(SkelMeshComp, Cast<UAnimSequenceBase>(AnimNotifyEvent->NotifyStateClass->GetOuter()), AnimNotifyEvent->GetDuration(), *AnimNotifyEventReference);
			}
		}
	}
```

将新出现的的**ANS**调用开始事件。

```c++

// 将临时存储转正
ActiveAnimNotifyState = MoveTemp(NewActiveAnimNotifyState);
ActiveAnimNotifyEventReference = MoveTemp(NewActiveAnimNotifyEventReference);
// 遍历存在的ANS，触发TICK事件
for (int32 Index = 0; Index < ActiveAnimNotifyState.Num(); Index++)
{
    const FAnimNotifyEvent& AnimNotifyEvent = ActiveAnimNotifyState[Index];
    const FAnimNotifyEventReference& EventReference = ActiveAnimNotifyEventReference[Index];
    if (ShouldTriggerAnimNotifyState(AnimNotifyEvent.NotifyStateClass))
    {
        {
            AnimNotifyEvent.NotifyStateClass->NotifyTick(SkelMeshComp, Cast<UAnimSequenceBase>(AnimNotifyEvent.NotifyStateClass->GetOuter()), DeltaSeconds, EventReference);
        }
    }
}
```

对现有的所有**ANS**触发TICK事件，并且将激活情况保存

### 总结

- 对于某一个时间点，事件类型的触发顺序是确定的

1. AN触发
2. ANS结束事件
3. ANS开始事件
4. ANS帧事件

- 根据部分资料上判断，ANS和AN的顺序不保证触发顺序一致，因此在相关功能开发中需要注意。

## 参考资料

[UE4/UE5 动画通知AnimNotify AnimNotifyState源码解析](UE4/UE5 动画通知AnimNotify AnimNotifyState源码解析)