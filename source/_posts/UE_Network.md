---
title: UE网络与同步的个人笔记
categories: 
- UE
- Network
tag:
- program
- ue
date: 2024-10-18 00:00:00
updated: 2024-10-18 00:00:00
---

# 前言

个人对UE的网络部分的学习笔记。基本都是网络上已经有的教程说明，只是整理后更符合个人的理解路径。

<!-- more -->

# 基本概念

## DS服务器

标准的C-S结构下的服务器，包括玩家本地的客户端和具有权威性的服务端。

DS服务器本身和客户端代码一样的，通过权限控制来明确部分逻辑具体运行的端。

UE是标准的状态同步思路。

> ListenServer较为少见，其即为一个局域网联机的主机端，暂不考虑

## 通信方式

客户端和服务器有两种通信方式：

- 属性复制：对于表明需要复制的属性，服务端会将其同步到客户端上，这是服务器到客户端单向的
- RPC调用：在调用一些代码时，其实际执行的可能在其他端，这是双向皆可的

## 类的网络存在

UE中的类会有四种不同的情况：

| 存在位置                 | 典型例子                          |
| ------------------------ | --------------------------------- |
| 服务器                   | GameMode                          |
| 服务器和所有客户端       | GameState、PlayerState、Character |
| 服务器和仅拥有者的客户端 | PlayerController                  |
| 客户端                   | HUD、UI                           |

## 网络模式 ENetMode

网络模式分成四种：

- **NM_Standalone**：标准的单机，没有服务器
- **NM_DedicatedServer**：DS模式下的服务器
- **NM_ListenServer**：LS模式，局域网联机下的服务器
- **NM_Client**：在有服务器模式下的客户端

## 网络角色 ENetRole

一个Actor在网络复制后、在客户端和远端分别担任一些网络角色

1. **ROLE_None（不存在）**：不存在角色、即没有复制的Actor的远程网络角色为不存在
2. **ROLE_SimulatedProxy（模拟）**：客户端上的角色、即实际控制权不在本客户端上
3. **ROLE_AutonomousProxy（自治）**：客户端上的角色、且实际控制权在本客户端上
4. **ROLE_Authority（权威）**：服务器上的角色一律是权威的

- 复制、服务端就是权威的、客户端是模拟或者自治的
- 不复制、本端就是权威的，其他端的网络角色都是不存在的。即说明，就算某些Actor确认**HasAuthority**，也不代表它是一个服务端的权威。

> 以上均是主要以本端查看Actor的本地网络角色的角度，完整见下图

### GetLocalRole()

| 视角\实际归属权 | A客户端控制 | 服务器控制 | B客户端控制 |
| --------------- | ----------- | ---------- | ----------- |
| A客户端         | 自治        | 模拟       | 模拟        |
| B客户端         | 模拟        | 模拟       | 自治        |
| 服务器          | 权威        | 权威       | 权威        |

### GetRemoteRole()

| 视角\实际归属权 | A客户端控制 | 服务器控制 | B客户端控制 |
| --------------- | ----------- | ---------- | ----------- |
| A客户端         | 权威        | 权威       | 权威        |
| B客户端         | 权威        | 权威       | 权威        |
| 服务器          | 自治        | 模拟       | 自治        |


## 本地控制 isLocalController

用来判断这个是否是本地控制的，主要描述以下几个情况：

### controller 通用于AI

1. 单机时、必然为本地
2. 本地是客户端、且本地网络角色为自治、即客户端对自己控制的Actor
3. 本地网络角色为权威、且远程角色不为自治的、即服务器对不归属于客户端的AI、或者客户端对自己本地生成没有复制的

### playerController 仅玩家控制器

1. 本地网络模式为**NM_DedicatedServer** 服务器端、则肯定不是本地的。DS端没有本地玩家。
2. 本地网络模式为**NM_Standalone**或者**NM_Client**，则肯定是本地的，单机没有其他端、客户端则PlayerController只存在本地客户端
3. 本地网络模式为**NM_ListenServer**的，则返回设置好的，因为LS模式下、其他客户端的playerController也会在扮演服务器的客户端存在，因此会由其他地方设计

## 玩家控制器 playerController 和网络链接 NetConnection

### 联网流程

在联网的过程中，每当客户端尝试连接服务器时，都有如下流程：

1. 客户端发送连接请求
2. 服务端本地通过 **GameMode:PreLogin** 验证是否要接受连接
3. 接受连接时，服务器发送当前地图供客户端加载
4. 客户端加载成功后，发送 **Join** 信息到服务器
5. 服务器接受连接后，客户端创建一个真实的客户端对应的 **PlayerController**， 并将其复制到对应客户端
6. 一切顺利的话，服务器调用到 **GameMode:PostLogin**，此时**RPC**调用才可以正常进行

在第五步创建具备真实意义的PlayerController时，这个PlayerController才能够正常的拥有**网络连接**，这是一个对应关系

## Actor的Owner

Actor的Owner是不断向上层追溯，寻找**Owner**的**Owner**，直到找到一个拥有网络连接的**PlayerController**。这说明属于

如果不存在时，说明在这个端上是没有网络链接的。

## 网络相关性 Relevancy

UE支持的地图很大，足以让有可能一局游戏都没法遇见另一个玩家，这种情况下的话，自然很有可能完全不需要关心对方的更新。

### 相关性判断

大部分Actor都统一使用一套相关性判断，

```c++
// ActorReplication.cpp:322
bool AActor::IsNetRelevantFor(const AActor* RealViewer, const AActor* ViewTarget, const FVector& SrcLocation) const
{
    // 如果这个Actor是被标记了始终相关的，或者它的拥有者是视图目标，或者真实查看者是它的拥有者，或者这个Actor本身就是视图目标，或者视图目标是这个Actor的发起者， 则直接相关
    if (bAlwaysRelevant || IsOwnedBy(ViewTarget) || IsOwnedBy(RealViewer) || this == ViewTarget || ViewTarget == GetInstigator())
    {
        
        return true;
    }
    // 如果启用了拥有者相关性并且有拥有者，则调用拥有者的IsNetRelevantFor方法，判断拥有者是否相关
    else if (bNetUseOwnerRelevancy && Owner)
    {
        return Owner->IsNetRelevantFor(RealViewer, ViewTarget, SrcLocation);
    }
    // 如果标记了仅拥有者相关，则不相关
    else if (bOnlyRelevantToOwner)
    {
        return false;
    }
    // 如果有根组件，并且根组件的父组件存在，且父组件的拥有者存在，则调用父组件拥有者的IsNetRelevantFor方法，判断父组件拥有者是否相关
    else if (RootComponent && RootComponent->GetAttachParent() && RootComponent->GetAttachParent()->GetOwner() && 
            (Cast<USkeletalMeshComponent>(RootComponent->GetAttachParent()) || (RootComponent->GetAttachParent()->GetOwner() == Owner)))
    {
        return RootComponent->GetAttachParent()->GetOwner()->IsNetRelevantFor(RealViewer, ViewTarget, SrcLocation);
    }
    // 如果这个Actor是隐藏的，并且根组件不存在或根组件的碰撞未启用，则不相关
    else if(IsHidden() && (!RootComponent || !RootComponent->IsCollisionEnabled()))
    {
        return false;
    }

    // 如果没有根组件，则不相关
    if (!RootComponent)
    {
        return false;
    }

    // 返回是否启用了距离基础相关性，或者是否在网络相关性距离内
    return !GetDefault<AGameNetworkManager>()->bUseDistanceBasedRelevancy ||
            IsWithinNetRelevancyDistance(SrcLocation);
}
```



## 网络优先级 Prioritization 

网络优先级则是衡量它在网络复制的权重，权重越高，更新越频繁

网络权重重要的是比率，无法通过提升数值来增强网络优先级

# 属性复制

将服务器的数据复制到客户端，包括Actor、ActorComponent、各种变量等。UObject也可以复制，但是必须要依托于一个Actor。

## 复制回调

- **蓝图**：蓝图通过**RepNotify**来执行属复制后的回调事件。 
- **C++**：C++通过**ReplicateUsing**宏来标注一个变量在同步后调用的函数。

## 复制条件

可以调整属性复制的条件、减少无端的流量消耗。

| Condition 条件          | 说明                                                     | 实例                                               |
| ----------------------- | -------------------------------------------------------- | -------------------------------------------------- |
| COND_InitialOnly        | 该属性仅在初始数据组尝试发送                             |                                                    |
| COND_OwnerOnly          | 该属性只会发送给Actor的所有者（owner）                   | 一些可能只有玩家自己关心的数据、例如自己的CD倒计时 |
| COND_SkipOwner          | 此属性会发送至除 Owner 之外的所有连接                    |                                                    |
| COND_SimulatedOnly      | 此属性只会发送到模拟Actor（Simulated Actor）             |                                                    |
| COND_AutonomousOnly     | 该属性只会发送给自治Actor(autonomous Actor)              |                                                    |
| COND_SimulatedOrPhysics | 该属性将发送到simulated 或 bRepPhysics Actor。           |                                                    |
| COND_InitialOrOwner     | 该属性将发送初始数据组，或发送给 Actor 的所有者          |                                                    |
| COND_Custom             | 该属性没有特定条件，但需要通过 SetCustomIsActiveOverride |                                                    |

## 使用思路

1. 持久性数据都至少应该保留一个需要复制的对应数据和回调，这样能避免重连的情况下、**RPC**调用不会再次发生导致的数据丢失
2. 不要调用其他对象上需要复制的数据，其他对象可能并未完成复制
3. 只能关注一个接近最新的属性、它的变化过程很有可能丢失
4. **TArray**在属性同步时很容易出现问题，在需要属性同步的应该一律使用**TFastArray**来代替

## 常见坑点：

1. 蓝图的回调会在客户端和服务器上调用、而C++的只会在客户端调用，如果需要在服务器做变更后逻辑，得在服务器赋值后手动调用

2. C++的回调只会在调用后发现属性值不一致，需要覆盖时调用。如果客户端已经修改成相同值，则不会调用。

   但是这个可以通过修改宏**DOREPLIFETIME_WITH_PARAMS_FAST**和修改 **RepNotifyCondition** 为 **REPNOTIFY_Always** 来保证每次服务器复制都会调用修改。

3. 由于同步需要时间和同步顺序不确定、有可能同步时对象的属性还未能同步，所以最好不要使用其他对象上需要同步的数据

4. **TMap**和**TSet**都不支持网络同步，只有**TArray**能够进行网络同步。但是对于中间的**Remove**导致的数组频繁删除，会有性能问题

5. **TArray**对于普通类型能够直接判断值变化，但是复杂类型如**UObject** 指针，**TArray**是基于内存变化来判断是否发生改变的，单纯的修改不能触发网络复制

# RPC

即调用函数，但是函数可能在远程其他端执行就是RPC

## 调用和类型

### 类型

RPC存在三种类型：

- **server**：希望只在服务器上调用
- **client**：希望只在对应客户端上调用
- **NetMuticast**：希望在所有端上调用

### 从服务器调用的 RPC

| Actor Ownership | 未复制       | NetMulticast        | Server | Client                      |
| --------------- | ------------ | ------------------- | ------ | --------------------------- |
| 被客户端拥有    | 服务器上运行 | 服务器 + 所有客户端 | 服务器 | 在 actor 的所属客户端上运行 |
| 被服务器拥有    | 服务器       | 服务器 + 所有客户端 | 服务器 | 服务器                      |
| 未被拥有        | 服务器       | 服务器 + 所有客户端 | 服务器 | 服务器                      |

### 从客户端调用的RPC

| Actor Ownership        | 未复制                   | NetMulticast     | Server   | Client           |
| ---------------------- | ------------------------ | ---------------- | -------- | ---------------- |
| 被执行调用的客户端拥有 | 在执行调用的客户端上运行 | 执行调用的客户端 | 服务器上 | 执行调用的客户端 |
| 被不同客户端拥有       | 执行调用的客户端         | 执行调用的客户端 | 丢弃     | 执行调用的客户端 |
| 被服务器拥有           | 执行调用的客户端         | 执行调用的客户端 | 丢弃     | 执行调用的客户端 |
| 未被拥有               | 执行调用的客户端         | 执行调用的客户端 | 丢弃     | 执行调用的客户端 |

可以简单理解为，只有客户端调用具备所有权的Actor的ServerRPC，才能逻辑正确的使用，其他情况都是本地调用

## 可靠性

RPC默认是不可靠的，使用的是UDP，即有可能丢失调用。

只有标记了Reliable宏的RPC才会使用TCP，但也不能保证一定调用到，如果判断网络相关性不足的话，有可能也不会被调用到。

```c++
// NetDriver.cpp:6830
// Only send or queue multicasts if the actor is relevant to the connection
FNetViewer Viewer(Connection, 0.f);

if (Connection->GetUChildConnection() != nullptr)
{
    Connection = ((UChildConnection*)Connection)->Parent;
}

// It's possible that an actor is not relevant to a specific connection, but the channel is still alive (due to hysteresis).
// However, it's also possible that the Actor could become relevant again before the channel ever closed, and in that case we
// don't want to lose Reliable RPCs.
if (Actor->IsNetRelevantFor(Viewer.InViewer, Viewer.ViewTarget, Viewer.ViewLocation) ||
    ((Function->FunctionFlags & FUNC_NetReliable) && !!CVarAllowReliableMulticastToNonRelevantChannels.GetValueOnGameThread() && Connection->FindActorChannelRef(Actor)))
{
    // We don't want to call this unless necessary, and it will internally handle being called multiple times before a clear
    // Builds any shared serialization state for this rpc
    RepLayout->BuildSharedSerializationForRPC(Parameters);

    InternalProcessRemoteFunctionPrivate(Actor, SubObject, Connection, Function, Parameters, OutParms, Stack, bIsServer, RemoteFunctionFlags);
}
```



### 安全验证

**WithValidation**宏指定一个RPC验证函数，在RPC执行前需要验证，只有验证通过才可以执行。

**WithValidation**实际上可以用于Client，Server，NetMulticast的RPC函数，但一般来说还是用在Server的最多，因为一般是Server的数据最权威可以进行数据合法性校验。

## 使用思路

1. RPC 主要作用是执行那些不可靠的暂时性/修饰性游戏事件（不要用RPC来发送持久性状态， 而应该选择使用属性复制）。 这其中包括播放声音、生成粒子或产生其他临时效果之类的事件，它们对于 Actor 的正常运作并不重要。
2. RPC不存在返回值、如果希望有后续结果、需要使用反方向的一个新的RPC调用

## 常见坑点

1. 在遇到“游戏状态恢复”的场景，比如网络游戏中的断线重连。然后你就可能会遇到一些对象在重连后状态不对，因为变化时使用的RPC是一次性的。

   当重连后，RPC不会再执行一次，所以客户端重连的状态与服务器其实是不同的。

   这时候需要使用属性同步来解决问题，但是属性回调在断线重连的时候也并不一定想执行，所以要重新审视一下回调函数里面的内容。

2. 不要大量使用可靠RPC或者Tick中使用rpc，这会导致调用过多堵塞网络

3. 在RPC调用时，如果传参带有某些依赖属性同步的变量，有可能会导致RPC执行时使用的变量是未复制的，可以通过 **DelayUnmappedRPCs** 来延迟RPC调用，但是还是最好避免这些戏法

4. RPC时有可能直接被舍弃，例如一个很远的Actor上调用的RPC有可能不会在某个客户端上被调用，哪怕它是一个**多播可靠RPC**

5. beginplay在客户端服务器都会执行，如果在beginplay执行另外一个actor的生成。可能会触发客户端和服务器都生成一遍自己的actor，结果客户端存在了两个Actor（一个自己生成的，一个服务器生产的）。

   之后在调用RPC的时候很可能会出现RPC执行失败，因为没有复制，本地生成的Actor没有任何connection信息。

6. 不要把随时可能被**destroyed**的对象传进RPC的参数里面，RPC参数里面没有判断对象是否是合法的。如果传递的过程中对象被**destroy**掉，后续可能触发序列化找不到**NETGUID**的相关崩溃

#  参考资料

[UE网络精粹](https://www.gamejianghu.cn/post/62793.html#5-RPC-%E8%BF%9C%E7%A8%8B%E8%BF%87%E7%A8%8B%E8%B0%83%E7%94%A8)

[UE网络同步](https://zhuanlan.zhihu.com/p/533754693)

[《Exploring in UE4》关于网络同步的理解与思考[概念理解]](https://zhuanlan.zhihu.com/p/34721113)

[UE4 网络同步框架介绍及使用](https://zhuanlan.zhihu.com/p/640632055)
