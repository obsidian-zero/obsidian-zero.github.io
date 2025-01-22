---
title: UE摄像机基础
categories: 
- Gameplay
tag:
- program
- gameplay
- ue
- 
date: 2024-12-3 00:00:00
updated: 2024-12-3 20:00:00

---

# 前言

对于UE的摄像机系统做一个基础的研究。主要分析基本的时序和概念，为后续学习Lyra和ALS的镜头系统做拓展。

<!-- more -->

# 基本概念

简单介绍一下UE摄像机中需要使用且频繁出现的基本概念。

## PlayerController

代表游戏世界内的真实玩家。一般情况下，客户端只会存在对应这个客户端玩家的PlayerController。（服务器会存在所有参与的玩家，但是服务器不存在摄像机需求）。

## PlayerCameraManager

一个用于管理的中间层，和PlayerController一一对应。主要用于管理对应的摄像机和具体表现。一般来说，无论多个摄像机的混合、摄像机的转换和附加后续镜头效果都是在其中处理。

摄像机拓展的主要内容基本都在这个模块中。

## POV（struct FMinimalViewInfo）

POV是用于决定显示的摄像机数据。包括位置、旋转、FOV等影响渲染的数据。一个POV相当于在游戏世界中的一个眼睛的数据

## ViewTarget / PendingViewTarget (struct FTViewTarget)

**ViewTarget**相当于最终使用的POV数据和这个POV数据的来源对象。。主要结构仅POV数据和Target数据。

- **FTViewTarget.POV**：一个计算的POV
- **FTViewTarget.Target**：一个Actor，代表此时用于观察的Actor对象。

一般来说**ViewTarget**中，是最终应该使用的摄像机数据 **POV**。

**PendingViewTarget** 则是用于摄像机切换时的另一套**POV**数据，当存在**PendingViewTarget**的有效数据时，说明此刻在摄像机切换中。需要结合两个数据来计算最后的。

## CameraActor / CameraComponent

UE中，用于代表摄像机的对象。主要提供基础的摄像数据如位置等。可以摆放于关卡中代表一个视角，即提供一个POV。

CameraComponent则是最终提供数据的来源，只是它不是单独的Actor可以摆放的场景中，而是一个依附在其他角色（Pawn/CameraActor）上的组件。

## SprintArmComponent

一个弹簧臂，主要和CameraComponent结合使用。主要是提供3D场景下基于碰撞的回缩、拖拽等效果。和变化的效果。

# 更新流程

摄像机更新的流程和内部详细解说

## 总体流程

1. **UWorld::Tick** 世界更新，摄像机的更新都是直接由游戏世界的更新直接驱动， 在Actor更新之后，声音系统更新之前
2. **APlayerController::UpdateCameraManager** 无逻辑效果
3. **APlayerCameraManager::UpdateCamera** 
   - **APlayerCameraManager::DoUpdateCamera** 主要的计算ViewTarget，下面详解
   - 网络同步代码，主要是检测判断是否达到位置差距阈值或者时间阈值，如果达到后将数据上传网络



## 相机更新流程 APlayerCameraManager::DoUpdateCamera

### 颜色空间混合

关于颜色的混合，一般不太会用到，略过

### 更新 ViewTarget

主要调用了 **APlayerCameraManager::UpdateViewTarget** 来进行**ViewTarget** 的**POV** 计算。这里会计算出此刻摄像机该使用的基本数据。

### 处理摄像机过度 PendingViewTarget

这里就是处理摄像机混合的内容，包括对**PendingViewTarget**进行计算以获取要转换的**POV**，并和**ViewTarget**的**POV**进行混合，得到实际的效果。

> 这里需要注意，这一步和前面的更新ViewTarget会是一个互斥的关系。
>
> 即需要过度时，只更新计算PendingViewTarget的POV，原本的ViewTarget只使用上一次更新后得到的缓存，在整体的过渡流程中会进行缓存。

### 计算动画和音频的淡入淡出

对于动画和音频的一些淡入淡出，不太会用，略过。

### 处理摄像模式

一个摄像模式的相关代码，和正常游戏的摄像机更新不太一样，略过。

### 保存数据

保存本次更新摄像机的一些数据，用于后续更新和其他地方的引用。这里会缓存一次更新的**POV**信息。



## 计算POV APlayerCameraManager::UpdateViewTarget 

1. 初始化基础的**POV**数据
2. 计算实际的**POV**数据
3. 应用**CameraModifiers**对于**POV**进行修改  （震屏、晃动等）
4. 将**PlayerCameraManager**本身根据POV设置位置和旋转
5. 应用镜头效果 **CameraLensEffects** （镜头上使用的粒子特效）

### 计算POV的几种情况：

1. **CameraActor** ：如果**ViewTarget**上对应的目标是一个**CameraActor**的话，直接调用**CameraActor**上的**CameraComponent**中的 **GetCameraView** 函数。

2. **CameraStyle**：**PlayerCameraManager**本身提供一些默认的摄像机风格和状态

   - **Fixed**：固定视角，直接使用之前的**POV**，
   - **ThirdPerson / FreeCam / FreeCam_Default**：直接使用角色或者控制器的位置和旋转，加上一定的位置和旋转偏移并应用碰撞检测。
   - **FirstPerson**：使用角色的眼睛视角（但是对应接口返回角色本身的位置和旋转）

   它们都会关掉后续使用**CameraModifier**的步骤

3. **Actor**：对于一个普通的**Actor**来说，直接使用角色身上第一个**Active**的**CameraComponent**的**GetCameraView**，如果一个都没有的话，使用角色的眼睛视角。

4. **BlueprintUpdateCamera**：提供给蓝图的默认重载接口，在蓝图判断使用后直接更新角色的POV。

实际上一般对于**PlayerCameraManager**上的修改都相当于蓝图重载 **BlueprintUpdateCamera** 这个部分。在这里通过方式手动计算对应的POV。

### POV

基本定义

```c++
USTRUCT(BlueprintType)
struct FMinimalViewInfo
{
    GENERATED_USTRUCT_BODY()

    /** 位置 */
    FVector Location;

    /** 旋转 */
    FRotator Rotation;

    /** 透视模式下的水平视场角（以度为单位）（在正交模式下忽略）。 */
    float FOV;

    /** 在考虑不同宽高比之前，最初期望的水平视场角 */
    float DesiredFOV;

    /** 正交视图的期望宽度（以世界单位为单位）（在透视模式下忽略） */
    float OrthoWidth;

    /** 正交视图的近裁剪平面距离（以世界单位为单位） */
    float OrthoNearClipPlane;

    /** 正交视图的远裁剪平面距离（以世界单位为单位） */
    float OrthoFarClipPlane;

    /** 透视视图的近裁剪平面距离（以世界单位为单位）。设置为负值以使用默认的全局值 GNearClippingPlane */
    float PerspectiveNearClipPlane;

    /** 宽高比（宽度/高度） */
    float AspectRatio;

    /** 宽高比轴约束覆盖 */
    TOptional<EAspectRatioAxisConstraint> AspectRatioAxisConstraint;

    /** 如果为真，当目标视图的宽高比与此相机请求的不同时，将添加黑边。 */
    uint32 bConstrainAspectRatio:1; 

    /** 如果为真，在计算网格体的细节级别时考虑视场角。 */
    uint32 bUseFieldOfViewForLOD:1;

    /** 相机类型 */
    TEnumAsByte<ECameraProjectionMode::Type> ProjectionMode;

    /** 指示是否应用 PostProcessSettings。 */
    float PostProcessBlendWeight;

    /** 当 PostProcessBlendWeight 非零时使用的后处理设置。 */
    struct FPostProcessSettings PostProcessSettings;

    /** 离轴/离中心投影偏移，作为屏幕尺寸的比例 */
    FVector2D OffCenterProjectionOffset;

    /** 可选的变换，被视为此视图的前一个变换 */
    TOptional<FTransform> PreviousViewTransform;
};
```

大部分变量都和图形学相关，但是一般来说管理摄像机也并不需要手动更改这些变量。

结合 **UCameraComponent::GetCameraView** 可以看出。在游戏中往往更改的也就是**Location**和**Rotation**两个基本变量。

> 注意，常用的摄像头使用角色Controller旋转也是在这里直接获得PlayerController的旋转并设置到CameraComponent上的。


