---
title: "PhysX零值Crash"
date: 2023-06-13
draft: false
tags: ["Game Dev", "PhysX"]
---

> 本文是[PhysX物理引擎系列]({{< tagref "PhysX" >}})的特别篇，记录了影响近一周的物理引擎底层概率性Crash的定位过程和修复方法，具有**很高**的实践参考价值。*“有多高？”“三四层楼那么高啦！”*

# 发现问题

运维同事发现体验服和某区在新版本上线一小段时间后，会出现概率不高但持续出现的进程Crash。这里先简单说明一下：我们会在一台机器上部署多个GameServer实例，每个GameServer实例进程同时进行着多场不同的Match，如果某一场Match出现了业务层Crash，并不会影响其他Match。但如果是C++物理库内出现Crash，则会同时中止其他正常运行的Match，对玩家的影响较大。

虽然看不到完整的堆栈，但从中还是发现和物理引擎的`SceneQuery`有关。
![](/using_physx_solving_zero_value_crash/crash_log.png)

其实还有另一个很相似的堆栈（忘记保存截图了）在`prefilter`上方还有一行`NpShape vtable...`。由于`prefilter`实现逻辑中确实调用了`shape.getFlags()`，所以怀疑是跟`shape`相关逻辑中出现空指针引用。但由于shape是引擎内部维护的，所以上述怀疑并不能提供明确的修复方法。
```cpp
// prefilter implementaion
PxQueryHitType::Enum PhysxQueryFilterCallback::preFilter(const PxFilterData& filterData, const PxShape* shape, const PxRigidActor* actor, PxHitFlags& queryFlags)
{
	bool isTrigger = shape->getFlags() & physx::PxShapeFlag::eTRIGGER_SHAPE;

	if (isTrigger && !m_IncludeTrigger) {
		return PxQueryHitType::eNONE;
	}

	PxFilterData shapefilterData = shape->getQueryFilterData();
	if (shapefilterData.word0 & filterData.word0)
	{
        // m_HitType should be PxQueryHitType::eBLOCK or PxQueryHitType::eTOUCH
		return m_HitType;
	}
	return PxQueryHitType::eNONE;
}

// shape->getFlags() implementation (fron physx source code)
PxShapeFlags NpShape::getFlags() const
{
	NP_READ_CHECK(getOwnerScene());
	return mShape.getFlags();
}
```

由于同期发布的还有其他物理相关功能，先通过开关这些功能来对Crash做初步判定。有一个延迟删除Actor的优化较为可疑，关闭该功能后，Crash确实减少了一小部分。

另外也尝试通过分析日志对这些Crash发生的情境做猜测，便于复现。前面提到过，物理层一旦Crash会影响到该进程下其他正常进行的比赛。所以只好抓取了Crash发生前200ms的所有相关比赛的日志，大致能看出最后几秒内玩家进行了哪些操作：商店购买、切换武器、伤害扣血、扔手雷。这里只有扔手雷和物理层有关联，进一步找到该场比赛的模式信息，告诉QA同学尝试复现。


# 第一次尝试修复

测试环境中一直未能复现，所以先尝试了一些防御性判空修复。比如在每帧驱动物理更新时，对Scene的判空；又比如在`OnContact`时，增加了对`actor`和`shape`失效状态的判断，并且获取`actor`的方式由`pairs[i].shapes[0]->getActor()`改为`pairHeader.actors[0]`。这些修复确实能加强程序的健壮性，但可惜对本次的Crash并没有直接帮助。

```cpp
void PhysxSimulationEventCallback::onContact(const PxContactPairHeader& pairHeader, const PxContactPair* pairs, PxU32 nbPairs)
{
	// add actor validation
	if (pairHeader.flags & PxContactPairHeaderFlag::eREMOVED_ACTOR_0 ||
		pairHeader.flags & PxContactPairHeaderFlag::eREMOVED_ACTOR_1)
	{
		return;
	}
	PxRigidActor* actorA = (PxRigidActor*)pairHeader.actors[0];
	PxRigidActor* actorB = (PxRigidActor*)pairHeader.actors[1];

	for (PxU32 i = 0; i < nbPairs; i++)
	{
		const PxContactPair& curPair = pairs[i];

		// add shape validation
		if (curPair.flags & (PxContactPairFlag::eREMOVED_SHAPE_0 | PxContactPairFlag::eREMOVED_SHAPE_1))
		{
			continue;
		}

		if (curPair.events & PxPairFlag::eNOTIFY_TOUCH_PERSISTS)
		{
			// do nothing when contact persists
		}
		else
		{
			PhysXContactResult result;
			result.Lost = curPair.events & PxPairFlag::eNOTIFY_TOUCH_LOST;
			result.ColliderA = actorA == NULL ? NULL : (PhysXActor*) actorA->userData;
			result.ColliderB = actorB == NULL ? NULL : (PhysXActor*) actorB->userData;

			if (result.ColliderA != NULL && result.ColliderB != NULL)
			{
				m_ContactRecords.push_back(result);
			}
		}
	}
}
```


# 第二次尝试修复

为了获取更多信息，决定费一些周章，在一台线上机器部署Debug版物理库。事实证明，这个努力是值得的、立竿见影的。

平时服务器程序打Release包时，链接到Release版的物理库工程（底层使用了`physX`）：
```go
package physxgo

/*
#cgo CPPFLAGS: -Wno-attributes -I ./include -O3 -DNDEBUG
#cgo LDFLAGS:-L ./lib -lPhysXWrapper_x64 -O3
*/
import "C
```

现改为链接到Debug版物理库工程：
```go
package physxgo

/*
#cgo CPPFLAGS: -Wno-attributes -I ./include -O1 -DNDEBUG
#cgo LDFLAGS:-L ./lib -lPhysXWrapperDEBUG_x64 -O1
*/
import "C"
```

在本地Windows上测试通过后，部署到Linux却失败了：发现依然链接到Release物理库。
> 这里稍加说明：服务器是Linux的，但为了平时能在Windows上开发调试物理库，我们搭建了两套构建流程。在Linux上使用makefile编译出`.so`，在Windows上构建出`.dll`供`cgo`调用。注意`cgo`使用`gcc`，由于name mangling方式和`MSVC`不同，在使用`MSVC`编译时需要在`.def`文件中指定每个导出的函数编译后的名字，形如`x=y`，这样`MSVC`会把自己编出的`y`翻译成`x`，以便和`gcc`兼容。
>
> 举例：`_ZN10PhysXActor11SetPositionEfff=?SetPosition@PhysXActor@@QEAAXMMM@Z`


原因是`go build`是强制使用了配置的环境变量以及缓存。使用`go build -d`清空缓存即可。

成功部署Debug包到一台服务器上后，过15min就Crash了，在最后一刻传递出了新的信息：

![](/using_physx_solving_zero_value_crash/crash_log_2.png)

创建shape时参数不合法，首先怀疑传入了0或NaN。进一步的排查发现，这个版本使用了新版的创建函数，确实相比老的创建流程少了一个参数合法性校验。当发现某个维度出现0值时，虽然不会立即Crash，但在后续被Raycast等操作访问到时，会概率性Crash。解决方法很简单，在业务层已有代码中，发现传入0时给出警告并return；在底层代码中创建shape时，强制将值为0的维度改为一个微小的值，如下`ForceNonZero`。

```cpp
void ForceNonZero()
{
	//EPS = 1.192092896e-07F (machine epsilon for float)
	if (physx::PxAbs(x) < EPS) { x = x >= 0 ? EPS : -EPS; }
	if (physx::PxAbs(y) < EPS) { y = y >= 0 ? EPS : -EPS; }
	if (physx::PxAbs(z) < EPS) { z = z >= 0 ? EPS : -EPS; }
}
```

部署该修复后，Crash归零。

![](/using_physx_solving_zero_value_crash/crash_over.png)

并且通过Debug版给出的警告，我们还对数值类参数做了NaN或Inf校验`physx::PxIsFinite`，并且对方向类参数做了强制归一化，进一步提高了物理系统稳定性。值此，修复完成。

# 总结
- 对于传入`physX`的参数必须严格保证有效性。
    - 基本要求：数值类不可以是NaN, Inf
    - 尺寸类不可以是0
    - 方向类矢量必须模长为1

- 平时开发中多关注Debug版的输出，对于警告和错误都要高度重视。

- 问题发生时，如果传统的定位、复现问遇到困难，不妨考虑部署Debug版到线上环境，获取更多信息。