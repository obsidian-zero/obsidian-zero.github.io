---
title: UE中动画蒙太奇相关研究笔记
categories: 
- UE
tag:
- program
- ue
- anim
- gas
date: 2024-09-18 00:00:00
updated: 2024-09-18 20:00:00
---
# 前言

一些对于AnimMontage的相关研究和分析。
<!-- more -->

# 基本架构

可以将动画蒙太奇相关设计的架构大体上如此表示

![image-20240918234744876](./UE_AnimMontage/image-20240918234744876.png)

大体上可以总结出如下四点：

1. **AnimInstance**是和**SkeletalMeshComponent**一一绑定的，用来控制对应**SkeletalMeshComponent**的动画情况
2. **AnimMontage**是编辑器中蓝图资产的基类，用来编辑相关动画的
3. **AnimMontageInstance**是实际播放蒙太奇时由**AnimInstance**创建的实例，它会绑定到对应的资产**AnimMontage**，从而获得在里面编辑出的数据
4. **AnimNotify**和**AnimNotifyState**都是对应**AnimMontage**，同一个**Montage**中会多个**Notify**是不同的实例，但是播放时不同**MontageInstance**的实例是同一个，因此不能在**Notify**中储存和**Actor**相关的状态

# 播放蒙太奇

每次在尝试播放蒙太奇时，都会调用到`UAnimInstance::Montage_PlayInternal()` 这个函数作为最终的入口。

```c++
//AnimInstance::2127
float UAnimInstance::Montage_PlayInternal(UAnimMontage* MontageToPlay, const FMontageBlendSettings& BlendInSettings, float InPlayRate /*= 1.f*/, EMontagePlayReturnType ReturnValueType /*= EMontagePlayReturnType::MontageLength*/, float InTimeToStartMontageAt /*= 0.f*/, bool bStopAllMontages /*= true*/)
{
	//只播放具备长度的蒙太奇
	if (MontageToPlay && (MontageToPlay->GetPlayLength() > 0.f) && MontageToPlay->HasValidSlotSetup())
	{
        //验证是否都有骨骼网格体信息
		if (CurrentSkeleton && MontageToPlay->GetSkeleton())
		{
            // 停止所有同组的动画蒙太奇
			const FName NewMontageGroupName = MontageToPlay->GetGroupName();
			if (bStopAllMontages)
			{
				StopAllMontagesByGroupName(NewMontageGroupName, BlendInSettings);
			}

			// 如果动画蒙太奇具备根运动的话，准备停止现在的根运动蒙太奇，避免
			if (MontageToPlay->bEnableRootMotionTranslation || MontageToPlay->bEnableRootMotionRotation)
			{
				FAnimMontageInstance* ActiveRootMotionMontageInstance = GetRootMotionMontageInstance();
				if (ActiveRootMotionMontageInstance)
				{
					ActiveRootMotionMontageInstance->Stop(BlendInSettings);
				}
			}
			// 创建一个空白的蒙太奇实例，并将对应资产数据绑定并调整具体情况
			FAnimMontageInstance* NewInstance = new FAnimMontageInstance(this);

			const float MontageLength = MontageToPlay->GetPlayLength();
            
			NewInstance->Initialize(MontageToPlay);
			NewInstance->Play(InPlayRate, BlendInSettings);
			NewInstance->SetPosition(FMath::Clamp(InTimeToStartMontageAt, 0.f, MontageLength));
			MontageInstances.Add(NewInstance);
			ActiveMontagesMap.Add(MontageToPlay, NewInstance);

			// 如果根运动的话，将其设为唯一的根运动蒙太奇，避免有多个根运动蒙太奇导致的冲突
			if (MontageToPlay->HasRootMotion())
			{
				RootMotionMontageInstance = NewInstance;
			}

            // 广播蒙太奇开始播放的委托
			OnMontageStarted.Broadcast(MontageToPlay);

            // 返回值可以选蒙太奇的总长或者蒙太奇的具体播放时长
			return (ReturnValueType == EMontagePlayReturnType::MontageLength) ? MontageLength : (MontageLength / (InPlayRate*MontageToPlay->RateScale));
		}
	}

	return 0.f;
}
```

- 蒙太奇播放的业务相关逻辑

# 结束蒙太奇

蒙太奇的结束存在两个重要的概念节点，**BlendOut**和**Ended**，**BlendOut**代表着蒙太奇开始混出，**Ended**则代表着蒙太奇实例的彻底销毁。

## BlendOut

## Ended

# GAS联动

GAS和蒙太奇的自带的主要联动就是**AbilityTask_playMontageAndWait**这个**AT**。

# 参考材料

[UE4/UE5 动画蒙太奇Animation Montage 源码解析](https://zhuanlan.zhihu.com/p/664971350)

[【UE5】对GAS的一点小改进——任意Mesh任意数量蒙太奇播放](https://zhuanlan.zhihu.com/p/718524837)