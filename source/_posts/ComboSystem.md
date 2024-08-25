---
title: 基于AnimNotifyState、EnhancedInput、GAS的搓招系统实现
categories: 
- UE
tag:
- program
- ue
- 
date: 2024-08-05 00:00:00
updated: 2024-08-25 20:00:00

---

# 前言

这是个人动作游戏DEMO的搓招系统实现一些实现笔记，主要在于AnimNotifyState、GAS、EnhencedInput的适配改造。

该DEMO主要灵感来源为流星蝴蝶剑.net。具备搓招操作和连招设计。这篇文章主要讨论如何实现的搓招连招系统。

<!-- more -->

# 基础知识

## AnimNotifyState

**AnimNofityState**是UE中，为动画或蒙太奇播放过程中触发特定逻辑的一种机制。

相较于**AnimNotify**只会在固定时间轴上触发的离散机制，**AnimNotifyState**提供了完整的开始、结束、Tick功能，因此在游戏开发中，更容易用**AnimNotifyState**来作为动画相关逻辑机制的载体。

### AnimNotifyState实例绑定于动画

在开发中可以发现，在一个动画中，添加多个相同的**ANS**时，会产生不同的实例。

但是对于一个动画，由不同的Actor进行播放时，会发现实例的地址都是相同的，即是多个不同的Actor会触发同样的实例。

这就导致了使用**ANS**时，不应该在内部存储状态。

> 这里个人采取的做法时将相关数据直接绑定到通过**MeshComponent**获得的**Actor**上。**ANS**只进行逻辑调整，不存储信息。
>
> 听着很像ECS的System。

#### AnimNotifyState的触发逻辑 UAnimInstance::TriggerAnimNotifies

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

#### 总结

- 对于某一个时间点，事件类型的触发顺序是确定的

1. AN触发
2. ANS结束事件
3. ANS开始事件
4. ANS帧事件

- 根据部分资料上判断，ANS和AN、同类事件的触发顺序不保证一致。在后续涉及到网络同步时需要谨慎处理
- **AnimNotify**相关触发流程在**AnimInstance**上，**AnimIntance**是一个对应于**Actor**的动画控制系统。因此不必太关心动画提前结束、打断等是否会导致结束事件不能正确触发的相关问题

#### 参考资料

[UE4/UE5 动画通知AnimNotify AnimNotifyState源码解析](UE4/UE5 动画通知AnimNotify AnimNotifyState源码解析)

## 格斗游戏基本概念

这是一些格斗游戏系统中存在的基本概念，在本DEMO中也存在对应的类似概念

### 招式

一段连续的动作，在这个动作中，会存在对敌人造成伤害等效果。往往由自己输入而开始一段招式，但是也有可能由其他情况进入一段招式。例如受击可以视作为一种由别人而开始特殊招式。

### 硬直

在招式中。大部分输入都不可用，这些输入不能激活其他招式。

### 取消 & 连招

在招式中的某些特定时间点，可以用某些特定输入直接激活其他招式

# 连招系统实现

由于EnhancedInput已经提供了combo系统，可以满足初步的搓招需求。因此这部分暂且不考虑连招输入的相关设计，直接使用EnhancedInput的相关功能实现连招的输入。

这里主要在介绍对于输入的技能激活逻辑。

## 基本思路

- 每个搓招输入会对应一个**InputAction**
- **IA**对应的要激活**GA**不固定
- 动作游戏，因此基本上每个个**GA**都包括一个或多个**Montage**的动作表现。这里将这种**GA**称之为**ActionGA**，即格斗游戏的招式
- 使用**ActionGA**相当于进入一段动作状态，在这个情况下，不可以激活常规的**ActionGA**，即格斗游戏的硬直
- 但是可以在**ActionGA**中可以通过某些特定的**IA**打断当前的**ActionGA**触发一个新的**ActionGA**，即格斗游戏的取消和连招
- 取消时，**IA**所激活的**ActionGA**不一定和通常状态下同一**IA**激活的**ActionGA**一致
- 因此在的**Montage**的时间轴用**AnimNotifyState**进行标记，用于**IA**和对应**ActionGA**之间的关系

## 输入和激活ActionGA的关系主要分成三种：

1. 连招：在一个**ActionGA**中只能激活造成特定的连招**ActionGA**
2. 基本：不在**ActionGA**中，可以激活当前的基本**ActionGA**
3. 始终：始终会尝试激活的**ActionGA**

## IA激活ActionGA

输入方面，对于所有连招的**InputAction**绑定到一个通用的函数上。

```c++
void ADemoPlayerGASCharacterBase::BindActionInputAction(TObjectPtr<UInputAction> IA)
{
    // 将激活绑定的一个专门函数上
	if(!BindedInputActions.Contains(IA) && BindEnhancedInputComponent)
	{
		BindEnhancedInputComponent->BindAction(IA, ETriggerEvent::Triggered, this, &ADemoPlayerGASCharacterBase::UseActionGameplayAbility);
		
		BindedInputActions.Add(IA);
	}
}

// 函数参数直接获得InputAction
void ADemoPlayerGASCharacterBase::UseActionGameplayAbility(const FInputActionInstance& InputInstance)
{
	const UInputAction* Action = InputInstance.GetSourceAction();

	if (AbilitySystemComponent.IsValid())
	{
        // 判断现在是否在一个使用ActionGA的阶段，这里用一个GameplayTag来进行判断
		if(AbilitySystemComponent->HasMatchingGameplayTag(OnActionTag))
		{
            // 如果已经处于释放ActionGA中的话，尝试激活这个IA对应的连招ActionGA，实现连招
			FGameplayAbilitySpecHandle * comboSpec = InputActionToComboAbilityMap.Find(Action);
			if (comboSpec)
			{
				AbilitySystemComponent->TryActivateAbility(*comboSpec, true);
			}
		}
		else
		{
            // 如果不在释放ActionGA中的话，尝试激活这个IA对应的基本ActionGA，实现基本招式
			FGameplayAbilitySpecHandle * basicSpec = InputActionToBasicAbilityMap.Find(Action);
			if (basicSpec)
			{
				AbilitySystemComponent->TryActivateAbility(*basicSpec, true);
			}
		}
		// 无论是否在连招中都会尝试激活的ActionGA，主要用于特定的通用技能，比如假定存在一个所有时候都可以使用的爆气击退别人招式
		if (InputActionToAlwaysAbilityMap.Contains(Action))
		{
			FGameplayAbilitySpecHandle * alwaysSpec = InputActionToAlwaysAbilityMap.Find(Action);
			if (alwaysSpec)
			{
				AbilitySystemComponent->TryActivateAbility(*alwaysSpec, true);
			}
		}
	}
}
```

## EnhancedInput绑定ActionGA

在ActionGA中，增添对应IA的配置项来配置三种输入。

存储三种情况下**IA**和**ActionGA**的关系

```c++
bool ADemoPlayerGASCharacterBase::onAddActionGameplayAbility(TSubclassOf<UActionGameplayAbility> Ability, FGameplayAbilitySpecHandle AbilitySpecHandle)
{
	if (Ability && GetLocalRole() == ROLE_Authority && AbilitySystemComponent.IsValid())
	{
	
		ActionAbilityToSpec.Add(Ability, AbilitySpecHandle);

		TArray<TObjectPtr<UInputAction>> AbilityInputActions;
		
        // 获取对应GA的可能存在的基本输入
		for(UInputAction * IA: Ability->GetDefaultObject<UActionGameplayAbility>()->BasicInputActions)
		{
            // 绑定技能的输入到EnhancedInput上
			BindActionInputAction(IA);
			
            // 存储基本输入和技能的关系
			InputActionToBasicAbilityMap.Add(IA, AbilitySpecHandle);
			
			AbilityInputActions.Add(IA);
		}

        // 获取对应GA的可能存在的连招输入
		for (UInputAction * IA: Ability->GetDefaultObject<UActionGameplayAbility>()->ComboInputActions)
		{
            // 绑定技能的输入到EnhancedInput上
			BindActionInputAction(IA);
			AbilityInputActions.Add(IA);
		}

        // 获取对应GA的可能存在的始终
		for (UInputAction * IA: Ability->GetDefaultObject<UActionGameplayAbility>()->AlwaysInputActions)
		{
            // 绑定技能的输入到EnhancedInput上
			BindActionInputAction(IA);
			
            // 存储输入和始终激活的关系
			InputActionToAlwaysAbilityMap.Add(IA, AbilitySpecHandle);
			
			AbilityInputActions.Add(IA);
		}
		
        // 绑定GA可能存在的所有输入
		ActionAbilityToInputActionMap.Add(Ability, AbilityInputActions);

		return true;
	}

	return false;
}
```

改动ASC，让其赋予技能时主动调起绑定关系

```c++
void UCharacterAbilitySystemComponent::OnGiveAbility(FGameplayAbilitySpec& AbilitySpec)
{
	Super::OnGiveAbility(AbilitySpec);

	if (UActionGameplayAbility* Ability = Cast<UActionGameplayAbility>(AbilitySpec.Ability))
	{
		if(ADemoPlayerGASCharacterBase* character = Cast<ADemoPlayerGASCharacterBase>(GetAvatarActor()))
		{
			character->onAddActionGameplayAbility(Ability->GetClass(), AbilitySpec.Handle);
		}
		
	}
}
```

## ANS激活Combo输入

在**AnimNotifyState**中存储**ActionGA**和可以用于激活的**IA**

```c++
void UComboAnimNotifyState::NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, float TotalDuration, const FAnimNotifyEventReference& EventReference)
{
	Super::NotifyBegin(MeshComp, Animation, TotalDuration, EventReference);

	AActor* OwnerActor = MeshComp->GetOwner();
	
	if(ADemoPlayerGASCharacterBase * Character = Cast<ADemoPlayerGASCharacterBase>(OwnerActor))
	{
		Character->ActiveActionGameplayAbilityComboInput(ActionGameplayAbility, ComboInputActions);
	}
}


void UComboAnimNotifyState::NotifyEnd(USkeletalMeshComponent * MeshComp, UAnimSequenceBase * Animation, const FAnimNotifyEventReference& EventReference)
{
	Super::NotifyEnd(MeshComp, Animation, EventReference);

	AActor* OwnerActor = MeshComp->GetOwner();

	if(ADemoPlayerGASCharacterBase * Character = Cast<ADemoPlayerGASCharacterBase>(OwnerActor))
	{
		Character->DeActiveActionGameplayAbilityComboInput(ActionGameplayAbility, ComboInputActions);
	}
}
```

激活**IA**和连招**ActionGA**的关系

```c++
void ADemoPlayerGASCharacterBase::DeActiveActionGameplayAbilityComboInput(TSubclassOf<UActionGameplayAbility> Ability, TArray<UInputAction*> InputActions)
{
	
    FGameplayAbilitySpecHandle AbilitySpecHandle = ActionAbilityToSpec[Ability];

    for(UInputAction * InputAction: InputActions)
    {
        if(InputActionToComboAbilityMap.Find(InputAction))
        {
            InputActionToComboAbilityMap.Remove(InputAction);
        }
    }
}

void ADemoPlayerGASCharacterBase::ActiveActionGameplayAbilityComboInput(TSubclassOf<UActionGameplayAbility>Ability, TArray<UInputAction*> InputActions)
{
	FGameplayAbilitySpecHandle AbilitySpecHandle = ActionAbilityToSpec[Ability];
		
    for(UInputAction * InputAction: InputActions)
    {
        InputActionToComboAbilityMap.Add(InputAction, AbilitySpecHandle);
    }
}
```



## 总结

### Q & A

#### Q：为什么不通过GameplayTag确定激活能力，通过TryActivateAbilitiesByTag批量尝试触发，GA再通过Tag禁止触发？

A：因为这是一个搓招的动作游戏，技能触发会和输入具备密切相关性。例如一个动作希望在不同动作阶段中用不同的输入操作触发，还要避免被不合适的输入触发，这样的功能**GA**并没有相关机制提供。

GA的Tag机制只能一层条件判定，对于复杂情况并不适宜。无论是增添Tag判定机制还是维护一套能表达状态的Tag都会提高复杂性。

而且本身**GA**和**EnhancedInput**也没有什么联动机制，不如根据需求，直接实现一整套。Tag则更关注角色状态和GA触发的关系，而非输入。

