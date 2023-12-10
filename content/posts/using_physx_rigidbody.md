---
title: "PhysX物理引擎（3）Rigidbody Dynamics"
date: 2023-09-04
hideSummary: false
draft: false
tags: ["Game Dev", "PhysX"]
---

> 本文主要介绍PhysX刚体动力学相关的内部机制和使用方法。
> 
> [PhysX物理引擎系列]({{< tagref "PhysX" >}})记录了在实际项目中使用Nvdia PhysX 3.4物理引擎（[Code](https://github.com/pps43/PhysX-3.4), [Doc](https://github.com/pps43/PhysX-3.4/raw/master/PhysX_3.4/Documentation/PhysXGuide.chm)）的一些经验，有不少对官方资料的补充。

# Warm-up

We already know that
- `Kinematic` and `Dynamic` rigidbody are both `PxRigidDynamic` in PhysX. Use `PxRigidBody::setRigidBodyFlag(PxRigidBodyFlag::eKINEMATIC, true)` to turn a dynamic actor into kinematic at runtime, and vice versa. 
- `Kinematic` and `Static` actors always stay in the same location unless you move them in your code.
- When moving `Static` actors, their collisions with dynamic actors can be wrong.
- When moving `Kinematic` actors, you should always use `PxRigidDynamic::setKinematicTarget` in each frame rather than `PxRigidActor::setGlobalPose` to achieve correct collisions with other dynamic actors.

In this post we focus on dynamic rigidbody movement, e.g., **force and torque, gravity, sleeping** and so on.

Here are necessary physical concepts with math formulas.

|Translation|formular|Rotation|formular|
|-|-|-|-|
|Position|\\(\vec{x}\\)|Orientation (3x3 matrix)|\\(\mathbf{R}\\)|
|Linear Velocity|\\(\vec{v}=\frac{d\vec{x}}{dt}\\)|Angular Velocity|\\(\vec{\omega}=\frac{\vec{v}\times\vec{r}}{\lVert{\vec{r}}\rVert^2}\\)|
|Linear Acceleration|\\(\vec{a}=\frac{d\vec{v}}{dt}\\)|Angular Acceleration|\\(\vec{\alpha}=\frac{d\vec{\omega}}{dt}\\)|
|Mass|\\(M=\sum{m_i}\\)|Intertia tensor|\\(\mathbf{I}=\mathbf{R}\mathbf{I}_0\mathbf{R}^T\\)|
|Linear momententum|\\(\vec{p}=M\vec{v}\\)|Angular momententum|\\(\vec{L}=\mathbf{I}\vec{\omega}\\)|
|Force|\\(\vec{F}=\frac{d\vec{p}}{dt}=m\vec{a}\\)|Torque|\\(\vec{\tau}=\frac{d\vec{L}}{dt}\\)|



# Setup Rigidbody
Dynamic actor has 3 mass related properties: **mass**, **center of mass**, **inertia tensor**.

> ~~Let **mass** be 0 means it cannot move.~~ Although the doc says 0 is ok for mass, but actually it will cause random crash. See [Golden Tips](#golden-tips).
> 
> **Center of mass** is the position where force applys upon to generate a translation without rotating. It's defined in local space as Vector3, default value is (0,0,0).
> 
> **Moment of inertia** is a single number that describes how hard it is to rotate an object about a particular axis, while the **inertia tensor** is a 3x3 matrix that describes how hard it is to rotate an object about any axis. Let **inertia tensor** be (0,0,0) means it cannot rotate by any axis.

The easiest way to calculate mass properties is to always use the `PxRigidBodyExt::updateMassAndInertia`. You don't need  `setMassAndUpdateInertia`.

In Official demo `North Pole` (`PhysX_3.4\Samples\SampleNorthPole\SampleNorthPoleDynamics.cpp`), all of them are low center of mass, but with different config to achieve different feeling.

Here are my code snippet to initialize a dynamic rigidbody.
```cpp
void ActorWrapper::InitRigidbody(bool useGravity, float mass, const PxVec3& centerOfMass, const PxVec3& interiaTensor, const PxVec3& velocity, float drag, const PxVec3& angularVelocity, float maxAngularVelocity, float angularDrag, int constrainFlags)
{
	PxRigidDynamic* actor = PxActorAs<PxRigidDynamic>();
	if (actor == NULL || actor->getScene() == NULL)
	{
		return;
	}

	actor->setActorFlag(PxActorFlag::eDISABLE_GRAVITY, !useGravity);
	actor->setMaxAngularVelocity(maxAngularVelocity);
	
    //SetConstrains (todo)

	actor->setLinearDamping(drag);
    angularDrag = PxMax(0.01f, angularDrag); // 0 is unstable for angular drag
	actor->setAngularDamping(angularDrag);
	actor->setMass(mass);
	actor->setCMassLocalPose(PxTransform(centerOfMass));
	actor->setMassSpaceInertiaTensor(interiaTensor);

    // not kinematic
    if (!(actor->getRigidBodyFlags() & PxRigidBodyFlag::eKINEMATIC))
    {
        // if has constrain, modify velocity here

        actor->setLinearVelocity(velocity);
	    actor->setAngularVelocity(angularVelocity);
    }
}

void ActorWrapper::SetDensity(float density)
{
    PxRigidDynamic* actor = PxActorAs<PxRigidDynamic>();
	if (actor == NULL || actor->getScene() == NULL)
	{
		return;
	}

    density = PxClamp(density, 0.0f, 1e6f);
    PxRigidBodyExt::updateMassAndInertia(*actor, density);
}
```

# Add Force & Torque

`addForce` causes a translation, `addTorque` causes rotation, and `addForceAtPos` causes linear and rotation if pos is not the center of mass.

Both force and torque support 4 modes:
|PxForceMode|physics equivalent|
|-|-|
|eFORCE|\\(ma\\)|
|eIMPULSE|\\(mat\\)|
|eVELOCITY_CHANGE|\\(at\\)|
|eACCELERATION|\\(a\\)|

Here are my code snippet to implement addForce and addTorque:
```cpp

void ActorWrapper::AddForce(const PxVec3& force, int forceMode)
{
    // validation
    if (force.IsAllZero() || !force.IsFinite())
	{
		return;
	}
    PxRigidDynamic* actor = GetRigidDynamicActor();
	if (actor == NULL || actor->getScene() == NULL || (actor->getRigidBodyFlags() & PxRigidBodyFlag::eKINEMATIC))
	{
		return;
	}

    actor->addForce(force, forceMode);
}

void ActorWrapper::AddRelativeForce(const PxVec3& force, int forceMode)
{
    // same validation as AddForce

    PxVec3 globalForce = actor->getGlobalPose().rotate(localForce);
    actor->addForce(globalForce, forceMode);
}

void ActorWrapper::AddTorque(const PxVec3& torque, int forceMode)
{
    // same validation as AddForce

    actor->addTorque(torque, forceMode);
}

void ActorWrapper::AddRelativeTorque(const PxVec3& torque, int forceMode)
{
    // same validation as AddForce

    PxVec3 globalTorque = actor->getGlobalPose().rotate(torque);
    actor->addForce(globalForce, forceMode);
}

void ActorWrapper::UGCAddForceAtPosition(const PxVec3& force, int forceMode, const PxVec3& position)
{
    // same validation as AddForce

    if (!position.IsFinite())
	{
		return;
	}

    // Only eFORCE and eIMPULSE are supported!
	if (forceMode == PxForceMode::eFORCE || forceMode == PxForceMode::eIMPULSE) {
		PxRigidBodyExt::addForceAtPos(*actor, force, position);
	}
}

```

# Change Gravity

Gravity is scene-wide for dynamic rigidbodies in the scene, by `PxScene::setGravity()`. 

We can let some dynamic actors are not influenced by scene-wide gravity by `PxActor::setActorFlag(PxActorFlag::eDISABLE_GRAVITY,true)`. Then `addForce` each frame manually to make your customized gravity on this actor.

# Sleep & Awake

## Why Sleep Matters

Sleeping rigidbodies almost cost nothing. You can put less important rigidbodies to sleep, which can significantly lower CPU cost when you have thousands of them.

## Sleep Mechanism

An actor goes to sleep when: its mass-normalized kinetic energy (a.k.a. \\(\frac{1}{2}v^2\\)) is **below a given threshold for a certain time** (Internally they use a wake-counter, when counter reaches 0, actor is a candidate to sleep).

Default threshold is \\(5∗10^{−5}∗v^2\\), where \\(v\\) is `PxTolerancesScale.velocity`. Thus the logic of the formular is  that when actor's velocity is below 1% of `PxTolerancesScale.velocity`, they are allowed to go to sleep. Set threshold by `PxRigidDynamic::setSleepThreshold`

You can also set wake-counter value to control sleeping, by `PxRigidDynamic::setWakeCounter`.

Common APIs about sleeping (only for dynamic actors!):
|Name|Notes|
|-|-|
|PxRigidDynamic::setSleepThreshold||
|PxRigidDynamic::setWakeCounter|Calling on a sleeping rigidbody will auto-wakeup|
|PxRigidDynamic::isSleeping()||
|PxRigidDynamic::wakeUp()|Force wakeup.|
|PxRigidDynamic::putToSleep()|Force sleep.|
|PxSimulationEventCallback::onWake/onSleep|To receive these events, set flag on actor: `PxActorFlag::eSEND_SLEEP_NOTIFIES`|


## Awake Mechanism

Overall, these actions wake an actor up:
- `PxRigidDynamic::setKinematicTarget()` for kinematic actor. 
- `PxRigidActor::setGlobalPose()`, if the autowake parameter is set to true (default). 
- Raising `PxActorFlag::eDISABLE_SIMULATION`
- Calling `PxScene::resetFiltering()`. 
- Calling `PxShape::setSimulationFilterData()` and cause a different filtering result.
- Touch with an actor that is awake.
- A touching rigid actor gets removed from the scene.
- Contact with a static rigid actor is lost.
- Contact with a dynamic rigid actor is lost (awake in the next simulation step).
- Actor gets hit by a two-way interaction particle


# Golden Tips
- When calling `PxRigidBodyExt::setMassAndUpdateInertia(actor, mass)`, make sure `mass` is **above zero** for dynamic rigidbody, otherwise it crashes randomly when collision happens.
- When calling `PxRigidDynamic::setAngularDamping(value)`, make sure `value` is **above zero** otherwise it goes unstable for rotating.
- Changing scene-wide gravity value will **NOT** auto-wake sleeping rigidbody. Call `PxRigidDynamic::wakeUp()` manually if required.