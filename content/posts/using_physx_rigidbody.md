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

Let's start with some key concepts:
- Both `Kinematic` and `Dynamic` rigidbodies are represented as `PxRigidDynamic` in PhysX. You can switch between them at runtime using `PxRigidBody::setRigidBodyFlag(PxRigidBodyFlag::eKINEMATIC, true)`.
- `Kinematic` and `Static` actors remain stationary unless explicitly moved in code.
- Moving `Static` actors can result in incorrect collision behavior with dynamic actors.
- When moving `Kinematic` actors, always use `PxRigidDynamic::setKinematicTarget` each frame instead of `PxRigidActor::setGlobalPose` to ensure correct collision detection with dynamic actors.

This post focuses on dynamic rigidbody movement, covering topics such as **force and torque, gravity, sleeping**, and more.

Below are the essential physical concepts with their mathematical formulas:

|Translation|formular|Rotation|formular|
|-|-|-|-|
|Position|\\(\vec{x}\\)|Orientation (3x3 matrix)|\\(\mathbf{R}\\)|
|Linear Velocity|\\(\vec{v}=\frac{d\vec{x}}{dt}\\)|Angular Velocity|\\(\vec{\omega}=\frac{\vec{v}\times\vec{r}}{\lVert{\vec{r}}\rVert^2}\\)|
|Linear Acceleration|\\(\vec{a}=\frac{d\vec{v}}{dt}\\)|Angular Acceleration|\\(\vec{\alpha}=\frac{d\vec{\omega}}{dt}\\)|
|Mass|\\(M=\sum{m_i}\\)|Intertia tensor|\\(\mathbf{I}=\mathbf{R}\mathbf{I}_0\mathbf{R}^T\\)|
|Linear momententum|\\(\vec{p}=M\vec{v}\\)|Angular momententum|\\(\vec{L}=\mathbf{I}\vec{\omega}\\)|
|Force|\\(\vec{F}=\frac{d\vec{p}}{dt}=m\vec{a}\\)|Torque|\\(\vec{\tau}=\frac{d\vec{L}}{dt}\\)|



# Setup Rigidbody
A dynamic actor has 3 mass-related properties: **mass**, **center of mass**, and **inertia tensor**.

> ~~Setting **mass** to 0 means it cannot move.~~ Although the documentation suggests 0 is acceptable for mass, it actually causes random crashes. See [Golden Tips](#golden-tips) for details.
> 
> **Center of mass** is the point where applied forces generate translation without rotation. It's defined in local space as a Vector3, with a default value of (0,0,0).
> 
> **Moment of inertia** is a scalar value describing the resistance to rotation about a specific axis, while the **inertia tensor** is a 3x3 matrix describing resistance to rotation about any axis. Setting the **inertia tensor** to (0,0,0) prevents rotation around any axis.

The simplest way to calculate mass properties is to use `PxRigidBodyExt::updateMassAndInertia`. You don't need `setMassAndUpdateInertia`.

In the official `North Pole` demo (`PhysX_3.4\Samples\SampleNorthPole\SampleNorthPoleDynamics.cpp`), all objects have a low center of mass but use different configurations to achieve varied physical behaviors.

Here's my code snippet for initializing a dynamic rigidbody:
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

# Fix Large Mass
When a rigidbody with large mass (> 100kg) collides with other objects (e.g., static ground), the simulation can quickly become unrealistic or even collapse.

> The image below shows 5 rigidbodies falling to the ground with masses of 1kg, 10kg, 100kg, 1000kg, and 10,000kg (from left to right). 
> ![](/using_physx_rigidbody/large_mass_simulation_pvd.png)

There are several parameters you can tune to stabilize the simulation. The configuration below can boost the maximum supported mass from ~400kg to ~3800kg:

- **Scene-level**
  - `sceneDesc.flags |= PxSceneFlag::eENABLE_PCM` - PCM (Persistent Contact Manifold) makes objects more stable when at rest.
  - `sceneDesc.flags |= PxSceneFlag::eADAPTIVE_FORCE` - Adaptive force makes stacked objects more stable.

- **Actor-level**
  - `PxRigidDynamic::setSolverIterationCounts(32,8)` - This is the most significant change for stability, though it comes at a performance cost.
  - `PxRigidbody::setMaxDepenetrationVelocity(10)` - This prevents excessive velocity during collision resolution.

- **Shape-level**
  - `PxShape::setContactOffset(0.02)` - PhysX will resolve collisions earlier, improving stability.

# Add Force & Torque

- `addForce` causes translation
- `addTorque` causes rotation
- `addForceAtPos` causes both translation and rotation if the position is not at the center of mass

Both force and torque support 4 modes:
|PxForceMode|Physics Equivalent|
|-|-|
|eFORCE|\\(ma\\)|
|eIMPULSE|\\(mat\\)|
|eVELOCITY_CHANGE|\\(at\\)|
|eACCELERATION|\\(a\\)|

Here's my code snippet for implementing addForce and addTorque:
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

Gravity is scene-wide for all dynamic rigidbodies and is set using `PxScene::setGravity()`. 

You can exclude specific dynamic actors from scene-wide gravity by calling `PxActor::setActorFlag(PxActorFlag::eDISABLE_GRAVITY, true)`. Then manually apply forces each frame using `addForce` to implement custom gravity for that actor.

# Sleep & Awake

## Why Sleep Matters

Sleeping rigidbodies have virtually no performance cost. Putting less important rigidbodies to sleep can significantly reduce CPU usage, especially when dealing with thousands of objects.

## Sleep Mechanism

An actor goes to sleep when its mass-normalized kinetic energy (i.e., \\(\frac{1}{2}v^2\\)) remains **below a threshold for a certain duration**. Internally, PhysX uses a wake-counter; when it reaches 0, the actor becomes a candidate for sleeping.

The default threshold is \\(5∗10^{−5}∗v^2\\), where \\(v\\) is `PxTolerancesScale.velocity`. This means actors are allowed to sleep when their velocity drops below 1% of `PxTolerancesScale.velocity`. You can customize this threshold using `PxRigidDynamic::setSleepThreshold`.

You can also directly control the wake-counter using `PxRigidDynamic::setWakeCounter`.

Common APIs for sleeping (only for dynamic actors):
|Name|Notes|
|-|-|
|PxRigidDynamic::setSleepThreshold|Set the energy threshold for sleeping|
|PxRigidDynamic::setWakeCounter|Calling on a sleeping rigidbody will automatically wake it up|
|PxRigidDynamic::isSleeping()|Check if the actor is currently sleeping|
|PxRigidDynamic::wakeUp()|Force the actor to wake up|
|PxRigidDynamic::putToSleep()|Force the actor to sleep|
|PxSimulationEventCallback::onWake/onSleep|To receive these events, set the flag: `PxActorFlag::eSEND_SLEEP_NOTIFIES`|


## Awake Mechanism

The following actions will wake an actor up:
- Calling `PxRigidDynamic::setKinematicTarget()` on a kinematic actor
- Calling `PxRigidActor::setGlobalPose()` with the autowake parameter set to true (default)
- Raising the `PxActorFlag::eDISABLE_SIMULATION` flag
- Calling `PxScene::resetFiltering()`
- Calling `PxShape::setSimulationFilterData()` that results in a different filtering outcome
- Contact with an actor that is awake
- A touching rigid actor is removed from the scene
- Loss of contact with a static rigid actor
- Loss of contact with a dynamic rigid actor (wakes up in the next simulation step)
- Being hit by a two-way interaction particle


# Golden Tips
- When calling `PxRigidBodyExt::setMassAndUpdateInertia(actor, mass)`, ensure `mass` is **above zero** for dynamic rigidbodies, otherwise random crashes will occur during collisions.
- When calling `PxRigidDynamic::setAngularDamping(value)`, ensure `value` is **above zero**, otherwise rotation becomes unstable.
- Changing the scene-wide gravity value will **NOT** automatically wake sleeping rigidbodies. Call `PxRigidDynamic::wakeUp()` manually if needed.