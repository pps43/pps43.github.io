---
title: "ECS 架构精髓"
date: 2026-09-05
update:
hideSummary: false
tags: ["游戏开发"]
---

经过业界这些年的实践，ECS 虽然因为种种原因没能成为主流，但还是在部份场景发挥了应有的价值。并且其设计思想依然值得借鉴。今天得空翻到一个 ECS 的 C# 实现 [Friflo.Engine.ECS](https://github.com/friflo/Friflo.Engine.ECS)，借此机会把 ECS 中的精髓提炼一下，温故知新。

# 一些概念

ECS 三大概念
- `Entity`：只是一个带有 `ID` 的逻辑容器，由一组`Component`定义其状态。
- `Component`：只有数据没有逻辑（纯状态）。
- `System`： 理想情况下，只有逻辑没有数据（无状态）。一个`System`通过访问自己关注的数据，进行逻辑处理即可。

ECS 真正的精髓
- `Archetype`：由于不同`Entity`可能包含不同的`Component`集合，每种集合归为一类，叫做`Archetype`。实际代码实现中，每个`Archetype`对象记录了“属于该类”的实体ID集合、以及若干组件数组。
- `Query`：是一个查询条件，它定义了需要查找哪些组件。内部会高效的从所有符合条件的`Archetype`中获取实体ID集合等数据，并返回给`System`进行遍历。
- `Tag`：用于标识一个`Entity`的某种特性/临时状态。有人将其视为一种特殊的`Component`，它没有数据，只有标识。因此同一个`Archetype`中，所有`Entity`都共享这一组`Tag`。

# 内存布局
只凭概念还是很难理解 ECS 如何解决缓存友好性，以及会带来什么额外的问题。下面以一个简单的例子来说明。

假设世界里有以下5个实体：

| Entity | 组件                        |
| ------ | -------------------------- |
| E1     | Position, Velocity         |
| E2     | Position, Velocity, Health |
| E3     | Position                   |
| E4     | Position, Velocity         |
| E5     | Position, Velocity, Health |

Entity 只保存了一个EntityStore引用、一个ID、一个版本号。在64位系统下，占用16字节。其中Id占用4字节，即最多可支持同时存在42亿多个实体。

```cs
public readonly struct Entity
{
    public EntityStore EntityStore; // 8B
    public int Id; // 4B
    public short Revision; // 2B
    // 2B padding
}
```

框架会建立三个 Archetype：

```
A_PV   = { Position, Velocity }
A_PVH  = { Position, Velocity, Health }
A_P    = { Position }
```

实体被放进组件集合完全相同的 Archetype：

```
A_PV    -> E1, E4
A_PVH   -> E2, E5
A_P     -> E3
```

以 A_PV 为例，其 `Archetype` 对象的内存结构如下。注：同一个数组下标 i，始终属于同一个实体。这就叫 **Structure of Arrays (SOA)**：

```
Archetype A_PV                    托管堆上的 class 对象
│
├── entityCount = 2
├── capacity    = 512
├── componentTypes = Position | Velocity
├── tags
├── heapMap ──────► 用于根据组件ID快速定位到该组件在该 ArcheType 中的数组对象
│
├── entityIds ───────────────► int[512]
│                              ┌────┬────┬────┬─────
│                              │ 1  │ 4  │ 0  │ ...
│                              └────┴────┴────┴─────
│                                0    1
│
├── StructHeap<Position> ─────► Position[512]
│                              ┌────┬────┬──────────
│                              │ P1 │ P4 │ unused...
│                              └────┴────┴──────────
│                                0    1
│
└── StructHeap<Velocity> ─────► Velocity[512]
                               ┌────┬────┬──────────
                               │ V1 │ V4 │ unused...
                               └────┴────┴──────────
                                 0    1
```

其中 `heapMap` 类型为 `StructHeap[]`，下标是组件的全局ID，值是该组件在该 ArcheType 中的数组对象。例如 Position 的组件Index假设为 `POS_IDX`，则 `heapMap[POS_IDX]` 就是 Position 在该 ArcheType 中的数组对象。概念上相当于 `Dictionary<Type, StructHeap>`。

### 一点小优化

实体必须是轻量的，这不仅应当时逻辑上的设计，还应在内存和GC上精打细算。

上面提到的 `Entity` 结构体已经很简单，但在真正存储实体的管理类中，并不需要每个实体都保存 `EntityStore` 引用，只需保存一个引用即可。
这样不仅省去了一半的内存（1M实体省去8MB内存），更因为剩下的字段都不是引用类型，减轻了GC扫描和标记的负担。这在框架中定义为 `RawEntity`：

```cs
[StructLayout(LayoutKind.Explicit)]
public readonly struct RawEntity
{
    [FieldOffset(0)] public readonly int   Id;        // 4B
    [FieldOffset(4)] public readonly short Revision;  // 2B
    // 2B padding

    [FieldOffset(0)] internal readonly long value;    // 8B
}
```

以上还用了个C#的小技巧，实现类似 union 的效果，目的是比较两个 `RawEntity` 是否相等时，直接比较 `value` 指令更精简。


# 管理Entity

上面已经说过，所有实体的数据 `EntityNode` 保存在 `EntityStore` 的数组中，预分配一个足够大的连续空间，下标就是实体的ID。

新建实体时，主要工作是：1、获取 `Archetype`并写入组件数据、2. `EntityStore` 创建节点、3. 触发创建事件。
1. 如果 `Archetype` 不存在，则创建一个，会产生堆内存分配。另外如果组件数组需要扩容，也会触发堆内存分配。
2. `EntityStore` 创建节点，也可能因扩容触发堆内存分配。至于`EntityID`，优先从回收队列中获取，没有则分配一个新ID（**递增**）。
3. 返回的 `Entity` 是结构体，通常存在于寄存器和栈中，框架层不会进行堆内存分配。


删除实体时，主要工作是：1.触发删除事件、2.`EntityStore` 删除节点、3.`Archetype`删除节点。
1. 必须先触发，此时仍能访问实体和组件，但不宜做复杂操作。
2. `EntityStore` 并不是真正删除 `EntityNode`，而是将`EntityID`放入回收队列、`EntityNode.Revision`加1。另外的细节如处理父子节点等不再赘述。业务逻辑通过比较 `EntityNode.Revision` 和自身可能持有的 `Entity.Revision` 比较来判断实体是否失效。这一步在删除实体时没有GC和遍历，非常高效。
3. 根据 `EntityNode.Archetype` 直接找到`ArcheType` 对象，再根据 `EntityNode.compIndex` 得到该实体在`ArcheType`中的下标，将数组最后一个元素的值赋值给该位置，然后最后一个位置的元素值设置为默认值。这一步也没有GC和遍历，非常高效。

### 查询Entity的属性
为了方便，`Entity` 的关联信息保存在一份单独的 `EntityNode` 结构中，所有实体的关联信息保存在 `EntityStore` 中。查询某个实体是否包含某个组件，主要通过查询对应的 `EntityNode.Archetype` 来实现。

```
Entity
   │
   ▼
EntityStore.nodes[Entity.Id]
   │
EntityNode
   │
   ├── archetype  ──────► 实体当前属于哪个 Archetype
   ├── compIndex  ──────► 实体在该 Archetype 数组的下标（所有组件数组该下标属于同一实体）
   └── revision   ──────► 判断旧 Entity 句柄是否已经失效
```

如果要获取一次该实体的 Position 组件，路径大致为：

```cs
var position = EntityStore.nodes[entity.Id].archetype.heapMap[POS_ID].data[entity.compIndex];
```

为了简便，实际封装成 `entity.GetComponent<Position>()` 的形式。

这比传统的面向对象的方式要复杂得多，虽然没有遍历和Hash，但有多次指针跳转，并不见得比直接访问对象的成员变量快。另外，在对 `Query.Entities`进行遍历时，只要实体的ArcheType和compIndex与之前的不同，也会破坏缓存局部性。更高效的方式是通过 `Query.Chunks` 进行遍历。

`Query.Chunks` 每次返回的 tuple 来自于一个 `ArcheType` 的对应数组对象。而不是把所有 `ArcheType`中的数组拼接起来。因此并不需要临时内存分配或拷贝。批量处理同一个 `ArcheType` 的实体时，缓存局部性非常好，只在跳到下一个 `ArcheType` 时，会有一次缓存丢失。

实际使用时，尽量使用`Query.Chunks` 进行遍历，但`entity.GetComponent`也有其使用场景。


### 序列化实体

序列化时不能直接使用上述的 `EntityID`，主要是因为全局不唯一、会ID复用和回绕，因此在多人编辑场景时会冲突。因此需要设计一个可持久化的实体ID，即`PID`。框架应当维护`PID`和`EntityID`的双向映射关系。

序列化实体会先将实体和相关的组件数据保存在一个新的结构`DataEntity`中，再将其序列化。

```cs
public sealed class DataEntity
{
    public long         pid;         // 永久 ID
    public List<long>   children;    // 有序子实体 PID
    public JsonValue    components;  // 组件数据
    public List<string> tags;
}
```

注意如果组件数据中持有`Entity`引用，也需要序列化时转化为其`PID`。

总之，序列化和反序列化是重度操作，有大概率是加载游戏时的性能热点。具体优化方式可能也和业务需求有关，这里先不展开。

# 管理和调度System

System之间往往有执行顺序的要求和依赖关系（比如一个系统依赖若干系统执行完毕）。
这个框架的System是用树状结构管理的（见`SystemRoot`, `SystemGroup`），而不是 DAG。在单线程下基本够用。

(未完待续)