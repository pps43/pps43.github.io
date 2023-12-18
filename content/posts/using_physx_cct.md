---
title: "PhysX物理引擎（4）Character Controller"
date: 2023-12-04
hideSummary: false
draft: false
tags: ["Game Dev", "PhysX"]
---

> 本文主要介绍PhysX角色控制器相关的内部机制和使用方法。
> 
> [PhysX物理引擎系列]({{< tagref "PhysX" >}})记录了在实际项目中使用Nvdia PhysX 3.4物理引擎（[Code](https://github.com/pps43/PhysX-3.4), [Doc](https://github.com/pps43/PhysX-3.4/raw/master/PhysX_3.4/Documentation/PhysXGuide.chm)）的一些经验，有不少对官方资料的补充。

# Warm-up

Character Controller (a.k.a CCT) is a special physical object handling player movement. In PhysX, CCT is not a Rigidbody, which means it does not integrate seamlessly in collision system. However, there is a kinematic actor underlying in CCT, and you can attach custom data via `PxController::getActor()->userData`.

Generally, CCT can be kinematic or dynamic, but according to PhysX document, kinematic controller has below advantages:
- Direct control. For dynamic rigidbody, Adding force/impulse/velocity to move player to final position is impossible.
- Built-in CCD.
- No **jitter** in a corner.
- No **friction** to finetune. Standing on a slope requires +inf friction, but walking on a slope without slowing down requires 0 friction. It's difficult to tune.
- No **restitution**. For dynamic rigidbody, even with 0 restitution, it will bounce a bit, due to imperferct nature of linear solver, and the way of recovering from penetration.
- Easy to stick to ground.
- Easy to keep standup and never rotate. For dynamic rigidbody, it's difficult to really constrain that way. Joints are often used, but with less robustness and speed.

> Above advantages apply to all kinematic CCT, not exclusive to PhysX's CCT implementation. You can just use a real kinematic rigidbody to implement your own CCT (Ground detection, Collide-and-Slide algorithm), which can be even more flexibile than PhysX's CCT.

# CCT Setup

There are two shapes of CCT, AABB and Capsule. Take Capsule as example.

![](/using_physx_cct/capsule_shape.png)

> ⚠ In Unity, the height in inspector equals to `CCT.height + 2 * CCT.radius`.

To change height, use `setHeight` or `resize`.
![](/using_physx_cct/resize.png)

## Skin Width
To avoid numerical issue, there is a skin around character, defined by `PxControllerDesc::contactOffset`. E.g. 0.08, or 10% of the radius. Also called "Contact Offset".

## Foot Position
![](/using_physx_cct/footposition.png)

Foot position is helpful in some cases, access via `PxController::getFootPosition/setFootPosition`

If need to keep foot position when change height, use `PxController::resize`.

## Up Direction

![](/using_physx_cct/upvector.png)

Up direction can be arbitrary, defined by `PxController::setUpDirection()`.

> ⚠ Unity locks up direction to `(0,1,0)`, which makes no sense and they have not changed that for years. [Unity Forum](https://forum.unity.com/threads/request-unity-to-add-support-for-arbitrary-up-axis-for-charactercontroller.480962/)

## Example

Below is the creation process of a CCT, using the layer defined in ["PhysX物理引擎（2）Collision"]({{< ref "/posts/using_physx_collision.md">}}).

```cpp
bool PhysXManager::AddCCT(ActorWrapper &actor, float radius, float height, float skinWidth, float stepOffset, float slopeLimit, int layer, int layerAgainst)
{
	if (!m_pxControllerManager)
	{
		m_pxControllerManager = PxCreateControllerManager(*m_pxScene, true);
		//m_pxControllerManager->setOverlapRecoveryModule(true);
		m_pxControllerManager->setPreciseSweeps(false);
	}
	PxCapsuleControllerDesc desc;
	desc.scaleCoeff = 1;
	desc.position = actor.GetPosition();
	desc.contactOffset = skinWidth;
	desc.stepOffset = stepOffset > height ? height : stepOffset; // [0, height]
	desc.slopeLimit = slopeLimit > 0? slopeLimit : 0; // cos(theta), [0, 1]
	desc.radius = radius; // [0, ]
	float h = height - 2 * radius;
	desc.height = h < 0 ? 0 : h; // [0,]
	desc.upDirection = PxVec3(0, 1, 0);
	desc.material = m_pxMaterial;
	desc.climbingMode = PxCapsuleClimbingMode::eCONSTRAINED;
	desc.reportCallback = actor.GetCCTHitReportHandler();
	//desc.behaviorCallback = NULL; currently we don't need this

	//create controller
	PxController* controller = m_pxControllerManager->createController(desc);
	controller->getActor()->userData = &actor;
	actor.SetPxActor(controller);
	actor.SetActorType(EActorType_CCT);

	//set query info
	PxShape* shape = NULL;
	controller->getActor()->getShapes(&shape, 1);
	if (shape)
	{
		PxFilterData filterData;
		filterData.word0 = layer;
		shape->setQueryFilterData(filterData);

		PxFilterData simFilterData;
		simFilterData.word0 = layer;
		simFilterData.word1 = layerAgainst;
		shape->setSimulationFilterData(simFilterData);
	}
	return false;
}
```

# CCT Move

Rather than `PxController::setPosition`, `PxController::move` uses a "collide-and-slide" algorithm. Internally, it use sweep tests in required direction. If found obstacle, CCT will slide smoothly against it.

```cpp
flags = PxController::move(disp, minDist, elapsedTime, filters, obstacles=NULL);
```

|name|type|description|
|-|-|-|
|flags|PxControllerCollisionFlags|combination of `eCOLLISION_SIDES`, `eCOLLISION_UP`, `eCOLLISION_DOWN`|
|disp|PxVec3|displacement, or delta. You need to apply gravity yourself.|
|minDist|PxF32|epsilon for moving. you would better keep it 0.|
|elapsedTime|PxF32|how much time passed since last call to move.|
|filters|PxControllerFilters|customize how CCT collides against the world, and with other CCT.|
|obstacle|PxObstacleContext|user-defined obstacles (can be moving), only for CCT, without shape object in scene.|


## Collide-And-Slide

Here are basic ideas on implementing "collide and slide" algorithm yourself.

1. Call a Sweep from the current position of the CCT shape to its goal position.
2. If no initial overlap is detected, move the CCT shape to the position of the first hit, and adjust the trajectory of the CCT by removing the motion relative to the contact normal of the hit.
3. Repeat Steps 1 and 2 until the goal is reached, or until an Sweep in Step 1 detects an initial overlap.
4. If a Sweep in Step 1 detects an initial overlap, use the Penetration Depth computation function to generate a direction for depenetration. Move the CCT shape out of penetration and begin again with Step 1.

## Custom Gravity

There is no internal Gravity applied on CCT. Calculate yourself and add it when calling `cct->move`. You may want to modify up-direction at the same time.

# CCT Interaction

## Hit Callback

⚠**This is called when the CCT moves and hits a shape. This will not be called when a moving shape hits a non-moving CCT**.

Here is how to add hit callback.

1. Inherit class `PxUserControllerHitReport` and override its `onShapeHit`, `onControllerHit`, `onObstacleHit` to record hitinfo. E.g., in `onShapeHit`, you can apply forces to other rigidbody, play sounds, etc. In Unity, it records this and throw `OnControllerColliderHit` event.

```cpp
class MyControllerHitReport : public PxUserControllerHitReport
{
public:
	virtual void onShapeHit(const PxControllerShapeHit& hit);
	virtual void onControllerHit(const PxControllersHit& hit);
	virtual void onObstacleHit(const PxControllerObstacleHit& hit) {} // don't need this

	void Clear();
	bool GetResult(PhysXCCTHitReportList& out);
private:
	PhysXCCTHitReportList m_HitRecords;
};

void MyControllerHitReport::onShapeHit(const PxControllerShapeHit& hit)
{
	PhysXCCTHitReport result;
	PxRigidActor* controllerActor = hit.controller->getActor();
	PxRigidActor* otherActor = hit.shape->getActor();

	result.Controller = controllerActor == NULL? NULL : (ActorWrapper*)controllerActor->userData;
	result.Other = otherActor == NULL? NULL : (ActorWrapper*)otherActor->userData;
	result.HitNormal = hit.worldNormal;
	result.HitPosition = physx::toVec3(hit.worldPos);
	result.Dir = hit.dir;
	result.Distance = hit.length;

	m_HitRecords.push_back(result);
}

void MyControllerHitReport::onControllerHit(const PxControllersHit& hit)
{
    //similiar to onShapeHit
}

//other functions
```

2. When creating CharacterController, create a object of above class and assign to `PxCapsuleControllerDesc.reportCallback`


## Behaviour Callback

This is called after CCT hit different objects to define three different behaviours: 

|flag|meaning|
|-|-|
|`eCCT_CAN_RIDE_ON_OBJECT`|Travel horizontally with the object it is standing on.|
|`eCCT_SLIDE`|Slide when standing on the object. It can be used to make CCT fall off a platform's edge if you think it's not enough to climb on it.|
|`eCCT_USER_DEFINED_RIDE`|Disable all built-in logic|

Here is how to add a behaviour callback.

1. Inherit class PxControllerBehaviorCallback and overide its "getBehaviorFlags". Notice there are 3 different functions with same name, used for against shapes, CCTs and internal obstacles. Return above flag.
2. When creating CharacterController, create a object of above class and assign to `PxCapsuleControllerDesc.behaviorCallback`.

## CCT vs Rigidbody

As for CCT pushes Rigidbody, it's difficult to control by applying force at contact points. You should use `onShapeHit` just mentioned.

As for Rigidbody pushes CCT, as mentioned above, when a CCT is not moving, hitcallback won't even called. There is no official way to solve this, see [next section](#golden-tips).

## CCT vs CCT

1. Inherit class `PxControllerFilterCallback` and override its "`filter(const PxController& a, const PxController& b)`" to determin if two CCTs can interact.

2. If you simply want two CCTs just overlap each other, return `false`. Return `true` means they will collide-and-slide. You can add custom logic using their shapes' `PxFilterData`.

```cpp
class ControllerFilterCallback : PxControllerFilterCallback
{
public:
	virtual bool filter(const PxController& a, const PxController& b)
	{
		PxShape* shapeA = NULL;
		a.getActor()->getShapes(&shapeA, 1);
					
		PxShape* shapeB = NULL;
		b.getActor()->getShapes(&shapeB, 1);
					
		const physx::PxFilterData filterData0 = shapeA->getQueryFilterData();
		const physx::PxFilterData filterData1 = shapeB->getQueryFilterData();
		
		if ((0 == (filterData0.word0 & filterData1.word1)) || (0 == (filterData1.word0 & filterData0.word1)))
		{
			return false;
		}
        return true;
	}
}

```

3. In each CCT.Move, create a object of above class and pass as filter.

# Golden Tips

- CCT's `slopeLimit` must be `[0, 1]` (`[0, 90]` in degree). Otherwise PhysX crashes.
- CCT's `stepOffset` must be `[0, height]`. Otherwise PhysX crashes.
- Unity does not draw `skinWidth` in scene view, but it exists as part of volumn.
- Unity locks upvector to `(0,1,0)` so you can't rotate controller.
- Unity's `IsGrounded` is NOT reliable. It's just implemented as `collisionFlags & eCOLLISION_DOWN::eCOLLISION_DOWN`, but very often when running uphill, the flag is `eCOLLISION_SIDES` so you get wrong result that `IsGrounded == false`. You should use `capsule raycast` or anything else to implement your own Ground-Detection algorithm instead.
![](/using_physx_cct/Unity_IsGround.png)

- When CCT is not moving, it loses the ability to collide with non-dynamic physical objects. That is, it will penetrate into a kinematic moving platform. **There is NO official way to solve it** since it's as designed by PhysX. To solve this, add a minor movement every frame on your CCT, e.g., `0.001 * Forward * deltaTime`. [See more discussion](https://forum.unity.com/threads/proper-collision-detection-with-charactercontroller.292598/).
