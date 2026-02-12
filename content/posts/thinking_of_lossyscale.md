---
title: "深入 LossyScale"
date: 2023-07-19
hideSummary: false
draft: false
tags: ["Game Dev", "PhysX", "数学"]
---

> 本文是[PhysX物理引擎系列]({{< tagref "PhysX" >}})的番外篇，其实要弄明白一个3D数学问题：如何处理父节点带有非均匀缩放、子节点带有旋转时，子节点的最终大小和形态。
> 问题源自笔者在修改物理引擎为其添加`scale`属性时遇到的一个bug。解决后对`WorldScale`为什么叫做`LossyScale`、空间变换和基变换有了更深的理解。

# 背景
- PhysX引擎中，场景内的Actor之间并没有父子层级关系，仅有的层级是Shape可以绑定到Actor作为子节点。
- PhysX引擎中并没有`Scale`的概念，即`PxTransform`只包含`Position`和`Rotation`，而大小只反映在最底层`Shape`的尺寸上（比如球形碰撞盒有半径这个属性）。所以设置缩放比例的实现方式是改变物体尺寸。

# 问题的出现

Actor下可以有若干Shape，这里只讨论有一个Shape的情况。现需要增加属性`Actor.Scale`，修改该属性时要保证`Shape.Dimension`的正确性。
```mermaid
classDiagram
direction LR
class Actor {
    PxVec3 Position
    PxQuat Rotation
    PxVec3 Scale
}
class Shape {
    PxVec3 LocalPosition
    PxQuat LocalRotation
    PxVec3 OriginalDimension
    PxVec3 Dimension
}
Actor o-- Shape
```

经实践，更方便的方式是记录Shape的初始尺寸`OriginalDimension`，然后乘上改变`Actor.Scale`后对应Shape的“缩放系数”。而不是通过`Actor.Scale`改变前后的比例来计算。总之，问题简化为实现`GetShapeScale()`。

```cpp
Shape.Dimension = Shape.OriginalDimension * GetShapeScale();
```

当Shape相对Actor没有旋转，即`Shape.LocalRotation = (0,0,0,1)`时，容易发现：
```cpp
PxVec3 GetShapeScale() {
    return actor.Scale;
}
```
但是当同时存在旋转和非均匀缩放呢？简单来说，当`Actor.Scale=(1,4,1)`，而Shape绕z轴转了90度，那么预期的结果应该是`(4,1,1)`，即Shape相对于自己在横向上扩大到2倍。若绕z轴转了45度，那么预期的结果是`(2,2,1)`。要如何达到这种效果呢？

# 似是而非的算法
一个很自然的想法是，要达到上面的效果，其实是将`Actor.Scale`像方向矢量那样旋转到Shape空间内。

```cpp
PxVec3 GetShapeScale() {
    PxTransform shapeSpace = shape->getLocalPose();
    return shapeSpace->rotate(actor.Scale);
}
```

然而反例是：`actorScale=(1,1,1)`经过旋转后可能不再是`(1,1,1)`，即Shape叠加了一个缩放。这是与事实违背的。针对`(1,1,1)`特殊处理也并不正确，因为对于任意`(X,Y,Z)`，总有一种旋转让其某个分量为`0`。

这种算法的错误之处在最后一节会额外讨论。

# 正确的算法

在[Unity的文档](https://docs.unity3d.com/ScriptReference/Transform-lossyScale.html)中，对于Transform.LossyScale这样说明：
> The global scale of the object (Read Only).
>
> Please note that if you have a parent transform with scale and a child that is arbitrarily rotated, the scale will be skewed. `Thus scale can not be represented correctly in a 3 component vector but only a 3x3 matrix`. Such a representation is quite inconvenient to work with however. `lossyScale is a convenience property that attempts to match the actual world scale as much as it can`. If your objects are not skewed the value will be completely correct and most likely the value will not be very different if it contains skew too.

理解，但不完全理解。直到找来源码分析了一番。去粗取精，根据代码提炼出公式：

$$
R_{world}=R_1R_2...R_N
$$
$$
W_{world}=R_1S_1R_2S_2...R_NS_N
$$
$$
S_{world}=R_{world}^{-1}W_{world}
$$
$$
s = diag(S_{world})
$$

上式中，\\(1...N\\)是根节点到叶子节点的编号，所有矩阵均采用列优先矩阵（因此使用左乘，例如\\(R_{world}\\)含义是先使用\\(R_N\\)进行变换到父空间，然后依次……直到使用\\(R_1\\)变换到世界空间）。\\(R_i\\)是只包含自身旋转信息的3x3旋转矩阵。\\(S_i\\)是只包含自身缩放信息的3x3对角矩阵。

最终结果\\(s\\)是3x1列矢量，取自\\(S_{world}\\)的对角线元素。

> 除了取主对角线元素，还有一种计算法来确定最终的缩放系数：取每一列的模长作为系数。这个做法的优点是精确保留了每个轴缩放后的长度，不过这样需要为负数缩放多一些处理：计算变换的行列式，如果为负数，则将x轴的缩放添加负号。

# 算法推导与解释

为什么是这样呢？这要从`TRS`变换矩阵说起。在3D中间中的姿态、运动和坐标系都可以用矩阵表达。
一般\\(T\\)表示位移，\\(R\\)表示旋转，\\(S\\) 表示缩放。

> 贴心提示：
> - 用欧拉角表示则需要规定旋转轴次序否则有歧义（感兴趣可以搜索万向节死锁）。经过实验，**Unity使用YXZ**，即对于\\((\theta_x,\theta_y,\theta_z)\\)，先按照Y轴转 \\(\theta_y\\)  ，再按照转动后的X轴转\\(\theta_x\\)，再按照转动后的Z轴转\\(\theta_z\\)。
> - 有时使用4x4而不是3x3矩阵只是一个数学上的技巧，为了让所有变换都可以用矩阵乘法串联起来。
> - 有时使用分块矩阵也只是一个数学上的技巧，为了简化公式发现规律。
> - 对于列优先矩阵，将列矢量\\(v_1\\)先按照\\(M_1\\)变换再按照\\(M_2\\)变换到\\(v_2\\)，写作\\(v_2=M_2M_1v_1\\)。
> - 更多资料请参考《[游戏引擎架构](https://www.gameenginebook.com/)》、任何讲解3D游戏开发或图形学的书籍。

$$
T=
\begin{pmatrix}
1 & 0 & 0 & Tx \\\
0 & 1 & 0 & Ty \\\
0 & 0 & 1 & Tz \\\
0 & 0 & 0 & 1
\end{pmatrix}=
\begin{pmatrix}
I & \bar{T}\\\
0 & 0
\end{pmatrix}
$$

$$
R_x=
\begin{pmatrix}
1 & 0 & 0 & 0 \\\
0 & \cos\theta_x & -\sin\theta_x & 0 \\\
0 & \sin\theta_x & \cos\theta_x & 0 \\\
0 & 0 & 0 & 1
\end{pmatrix}
$$

$$
R_y=
\begin{pmatrix}
\cos\theta_y & 0 & \sin\theta_y & 0 \\\
0 & 1 & 0 & 0 \\\
-\sin\theta_y & 0 & \cos\theta_y & 0 \\\
0 & 0 & 0 & 1
\end{pmatrix}
$$

$$
R_z=
\begin{pmatrix}
\cos\theta_z & -\sin\theta_z & 0 & 0 \\\
\sin\theta_z & \cos\theta_z & 0 & 0 \\\
0 & 0 & 1 & 0 \\\
0 & 0 & 0 & 1
\end{pmatrix}
$$

$$
R = R_zR_xR_y=
\begin{pmatrix}
r_{11} & r_{12} & r_{13} & 0 \\\
r_{21} & r_{22} & r_{23} & 0 \\\
r_{31} & r_{32} & r_{33} & 0 \\\
0 & 0 & 0 & 1
\end{pmatrix}=
\begin{pmatrix}
\bar{R} & 0 \\\
0 & 1
\end{pmatrix}
$$

$$
S=
\begin{pmatrix}
s_x & 0 & 0 & 0 \\\
0 & s_y & 0 & 0 \\\
0 & 0 & s_z & 0 \\\
0 & 0 & 0 & 1
\end{pmatrix}=
\begin{pmatrix}
\bar{S} & 0 \\\
0 & 1
\end{pmatrix}
$$

则对于一个节点，其本地坐标系的TRS变换矩阵可以写作：

$$
M=\begin{pmatrix}
r_{11}S_x & r_{12}S_y & r_{13}S_z & T_x \\\
r_{21}S_x & r_{22}S_y & r_{23}S_z & T_y \\\
r_{31}S_x & r_{32}S_y & r_{33}S_z & T_z \\\
0 & 0 & 0 & 1
\end{pmatrix}=
\begin{pmatrix}
\bar{R}\bar{S} & \bar{T} \\\
0 &  1
\end{pmatrix}
$$

对于父子节点\\(M_1\\)下的子节点\\(M_2\\)坐标系，其相对于世界坐标系的变换矩阵可以写作：

$$
M_{world}=M_1M_2=
\begin{pmatrix}
\bar{R_1}\bar{S_1} & \bar{T_1} \\\
0 &  1
\end{pmatrix}
\begin{pmatrix}
\bar{R_2}\bar{S_2} & \bar{T_2} \\\
0 &  1
\end{pmatrix}=
\begin{pmatrix}
\bar{R_1}\bar{S_1}\bar{R_2}\bar{S_2} & \bar{R_1}\bar{S_1}\bar{T_2}+\bar{T_1} \\\
0 & 1
\end{pmatrix}
$$

## 思路1
从另一个角度思考，若要将\\(M_{world}\\)拆分成TRS三个分量，是否就对应世界坐标系下的位移、旋转、缩放（全局缩放正是我们所需要的）呢？容易观察到全局位移是：

$$
T_{world}=
\begin{pmatrix}
 \bar{R_1}\bar{S_1}\bar{T_2}+\bar{T_1}
\end{pmatrix}
$$

而全局旋转由于其物理意义，必然是：

$$
R_{world}= \bar{R_1}\bar{R_2}
$$

则全局缩放只能是：

$$
S_{world}=R_{world}^{-1}=(\bar{R_1}\bar{S_1}\bar{R_2}\bar{S_2})=(\bar{R}_2^{-1}\bar{S_1}\bar{R_2})\bar{S_2}
$$

一般来说，\\(S_{world}\\)是一个非对角矩阵，即主对角线之外也有非零值。从另外一个角度理解，将子空间的标准正交基变换到父空间后，这组基除了模长发生了变换，也不再两两垂直了——这也叫子节点发生了剪切（Shear）。当我们只用一个三维分量表示缩放时，势必要丢失这组基之间的夹角信息，这也是`LossyScale`名字的由来。只有当父节点是均匀缩放（\\(\bar{S_1}=s_1I\\)）、或子节点没有相对旋转（\\(\bar{R_2}=I\\)）时，全局缩放系数可以化简为\\(S_{world}=\bar{S_1}\bar{S_2}=[s_x,s_y,s_z]^T\\)，其是一个对角矩阵，即`LossyScale`包含了完整的缩放信息。

## 思路2
除了上面凑出\\(S_{world}\\)进而得到缩放系数的方法，还有一种思路直接求得结果。

上面已经知道，全局变换矩阵为：
$$
M_{world}=
\begin{pmatrix}
\bar{R_1}\bar{S_1}\bar{R_2}\bar{S_2} & \bar{R_1}\bar{S_1}\bar{T_2}+\bar{T_1} \\\
0 & 1
\end{pmatrix}
$$

**“全局缩放系数”的定义本质上就是：子节点的本地单位轴，经过层层变换后，在世界空间中变成了多长**。因此可以忽略平移部分，只看变换：
$$
L = \bar{R_1}\bar{S_1}\bar{R_2}\bar{S_2}
$$

以子节点本地的x轴基向量\\(e_x = (1,0,0)^T\\)为例，其经过\\(L\\)变换后，模长是：
$$
s_{world, x} = ||L \cdot e_x|| = ||\bar{R_1}\bar{S_1}\bar{R_2}\bar{S_2} \cdot e_x||
$$

最后一次旋转\\(\bar{R_1}\\)不改变模长，直接可得：

$$
s_{world, x} = s_{2,x} \cdot \sqrt{(r_{2,11} \cdot s_{1,x})^2 + (r_{2,21} \cdot s_{1,y})^2 + (r_{2,31} \cdot s_{1,z})^2}
$$

# 问题的终结

回到最初的问题上来，我们还是采用和Unity一样的取主对角线元素法而没有选取模法。对Actor设置缩放时要更新所属Shape的尺寸，为此要实现的`GetShapeScale()`完整版：

```cpp
Shape.Dimension = Shape.OriginalDimension * GetShapeScale();

PhysXVec3 GetShapeScale() {
    PxQuat worldRot = actor.Rotation * shapeTransform.q;
    PxMat33 inverseWorldRotMat = PxMat33(worldRot.getConjugate());
    PxMat33 localRotMat = PxMat33(shapeTransform.q);
    PxMat33 actorRotMat = PxMat33(actor.Rotation);
    PxMat33 actorScaleMat = PxMat33().createDiagonal(actor.Scale);

    PxMat33 worldRotAndScaleMat = actorRotMat * actorScaleMat * localRotMat /* * localScaleMat*/; // localScaleMat is always Identity Matrix, so we omit it
    PxMat33 worldScaleMat = inverseWorldRotMat * worldRotAndScaleMat;
    PhysXVec3 worldscale = PhysXVec3(worldScaleMat[0][0], worldScaleMat[1][1], worldScaleMat[2][2]);

    // can have extra logic to make sure all values in worldscale are non-zero
    return worldscale; 
}
```

# 额外的讨论
已知正确的算法是：

$$
S_{world}=(\bar{R}_2^{-1}\bar{S_1}\bar{R_2})\bar{S_2}
$$

而似是而非的算法其实是：

$$
S_{world}=(\bar{R_2}s_1)\bar{S_2}
$$

本质错误在于：
- 正确的算法是将缩放本身视为一个空间变换\\(S_1\\)，进行**基变换**。
- 错误的算法是将缩放系数视为一个普通矢量\\(s_1\\) ，进行**空间变换**。

这些在大一的课堂上早已学过。往往正向解释很简单，难的是反向思考，即遇到实际问题怎么选择合适的概念去解决。

> 纸上得来终觉浅，绝知此事要躬行。