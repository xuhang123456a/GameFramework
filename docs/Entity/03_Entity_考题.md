# Entity 实体 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. 一个实体组（EntityGroup）内部持有什么来复用实例？
2. `EntityInstanceObject` 继承自谁？它的 `Release` 做什么？
3. "实例（GameObject）"与"逻辑实体（IEntity）"为什么要分离？
4. 列出 IEntity 的核心生命周期钩子（至少 5 个）。
5. `OnInit` 的 `isNewInstance` 参数区分什么？

---

## 二、机制题 🟡

6. ShowEntity 的两条路径（池命中 vs 未命中）分别做什么？在哪里汇合？
7. HideEntity 时实体实例被销毁了吗？发生了什么？
8. 实例真正被销毁（Destroy GameObject）由什么决定？
9. `EntityGroup.Update` 如何保证 OnUpdate 中 HideEntity 不崩？
10. 父子实体挂接时有哪些双向回调？
11. 隐藏一个有子实体的父实体，子实体会怎样？

---

## 三、架构 / 陷阱题 🔴

12. Entity 如何复用 ObjectPool？实体组容量/过期参数最终作用在哪？
13. 同一种怪物反复 Show/Hide 100 次，理想情况实例化（Instantiate）了几次？为什么？
14. ShowEntity 池未命中时要异步加载资源，期间实体处于什么中间状态？加载失败如何回滚？
15. 为什么"实例池化、逻辑对象轻量重建"是合理的取舍？反过来（逻辑池化、实例重建）会怎样？
16. Entity 与 UI 模块结构几乎同构，它们共享什么模式？
17. 隐藏级联不完整（父隐藏但没处理子）会产生什么问题？
18. EntityInstanceObject 的 Release 委托 `EntityHelper.ReleaseEntity`，为什么核心层不直接 Destroy？

---

## 四、实操题 ✍️

19. 给精简版补 `GetEntity(int id)` 和 `GetEntities(string assetName)`。
20. 实现父子挂接 `AttachEntity(child, parent)` + 父隐藏时级联 detach 子实体。
21. 分析：实例池容量设为 5，同时 Show 8 个同种怪物再全 Hide，池里会留几个实例？多余的何时销毁？

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：`IObjectPool<EntityInstanceObject>`（实例对象池）。
- **2**：`ObjectBase`；Release 调 `EntityHelper.ReleaseEntity` 真正销毁 GameObject。
- **3**：实例(GameObject)创建/销毁贵需池化复用；逻辑对象轻量，分离后可独立管理。
- **5**：区分实例是新建的还是从池复用的（影响初始化逻辑）。
- **6**：命中→Spawn 复用跳过加载；未命中→LoadAsset 加载+实例化+Register。汇合于 OnInit/AddEntity/OnShow。
- **7**：未销毁；OnHide → 出组链表 → Unspawn 回实例池。
- **8**：ObjectPool 的容量超限/过期策略（EntityInstanceObject.Release）。
- **9**：用 m_CachedNode 缓存 Next，RemoveEntity 时若删的是缓存节点则前移游标。
- **12**：EntityGroup 用 CreateSingleSpawnObjectPool 建池，容量/过期/优先级直接转发给 IObjectPool。
- **13**：理想 1 次——首次 Show 实例化，之后 Hide 回池、Show 命中复用，不再实例化（前提池容量够、未过期）。
- **14**：WillInit/加载中状态；失败广播 ShowEntityFailure 并回滚(不入组)。
- **15**：实例化/销毁是性能热点，池化收益大；逻辑对象轻，重建成本低。反过来省小钱花大钱，得不偿失。
- **16**：组 + 对象池 + 异步加载 + 生命周期钩子 + 迭代安全轮询。
- **17**：子实体悬空成孤儿（父没了仍显示/更新），内存与显示泄漏。
- **18**：核心层纯 C# 不能引用 UnityEngine.Object.Destroy；委托 helper 保持分层。
- **21**：池容量 5 → 全 Hide 后保留约 5 个待复用，超出的 3 个在 PoolUpdate 触发容量释放时 Release 销毁（或过期时）。

</details>
