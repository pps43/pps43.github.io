---
title: "实现Quaternion.FromToRotation的细节"
date: 2025-08-26
hideSummary: false
draft: false
tags: ["Game Dev", "数学"]
---

> 这篇记录实际项目中对`Quaternion.FromToRotation(fromVector, toVector)` 的实现思路，重点讨论当 `fromVector` 与 `toVector` 夹角为 180°（反向）时的稳健处理。主要参考 Stan Melax 的“最短弧四元数”（Shortest Arc Quaternion）思路。

### 为什么写这篇

项目里经常需要“把一个方向 A 旋到方向 B”。`FromToRotation` 正是把两个方向向量之间的旋转表示为一个四元数。但当两向量反向时，旋转轴不唯一（无穷多条），稍有不慎就会出现数值不稳、抖动或 NaN。本篇给出一个实际可用的方案。

### 数学基础（最短弧）

- 设单位向量 \\(u = \\frac{from}{\\|from\\|}\\), \\(v = \\frac{to}{\\|to\\|}\\)；点积 \\(d = u\\cdot v\\in[-1,1]\\)。
- 若 \\(d\\) 接近 1（方向相同），返回单位四元数即可（无需旋转）。
- 若 \\(d\\) 接近 -1（方向相反，180°），最短弧旋转角为 \\(\\pi\\)，但轴任意，只需选一条与 \\(u\\) 不共线的轴。
- 否则，可用“最短弧四元数”构造式：
  - 轴方向近似为 \\(u\\times v\\)
  - 或直接使用稳健公式构造四元数：
    - \\(s = \\sqrt{2(1+d)}\\)
    - 四元数 \\(q = [\\frac{u\\times v}{s},\\ \\frac{s}{2}]\\)（向量在前，标量在后，或按你引擎的约定）。

上述构造规避了直接求角度带来的精度与分支问题，是工业界常用写法。

### 特例：两向量夹角180°，如何选轴

- 当 \\(d \\approx -1\\) 时，\\(u\\times v\\) 退化为零向量，轴不再可用。此时选一个与 \\(u\\) 不近似平行的参考向量 \\(a\\)（如 `right(1,0,0)` 或 `up(0,1,0)`），令 \\(axis = \\mathrm{normalize}(u\\times a)\\)。
- 若恰好 \\(u\\) 与 `right` 共线，再换 `up`（或 `forward`）。
- 最终四元数可由 \\(\\theta=\\pi\\) 与该 `axis` 用 `AngleAxis(180°, axis)` 构造。


### 参考实现（C#，Unity 风格）

```csharp
Quaternion FromToRotation(Vector3 from, Vector3 to)
{
    Vector3 u = from.normalized;
    Vector3 v = to.normalized;
    float dot = Vector3.Dot(u, v);

    if (dot > 1.0f - 1e-6f)
        return Quaternion.identity;

    if (dot < -1.0f + 1e-6f)
    {
        Vector3 refAxis = Mathf.Abs(u.x) < 0.9f ? Vector3.right : Vector3.up;
        Vector3 axis = Vector3.Normalize(Vector3.Cross(u, refAxis));
        return Quaternion.AngleAxis(180.0f, axis);
    }

    Vector3 c = Vector3.Cross(u, v);
    float s = Mathf.Sqrt((1.0f + dot) * 2.0f);
    float invS = 1.0f / s;
    return new Quaternion(c.x * invS, c.y * invS, c.z * invS, s * 0.5f);
}
```