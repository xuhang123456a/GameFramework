# ObjectPool 对象池 · 考题（输出倒逼输入）

> 用法：先合上前两份文档，独立作答；再回查验证。难度分三档：🟢概念 🟡机制 🔴架构/陷阱。

---

## 一、概念题 🟢

1. `Spawn` / `Unspawn` / `ReleaseObject` 三个动词中，哪个会真正分配或归还内存？其余两个在做什么？
2. `IsInUse` 由哪个字段决定？写出它的判定表达式。
3. `ObjectPool<T>` 同时继承 `ObjectPoolBase` 又实现 `IObjectPool<T>`，分别是为了满足谁的需求？
4. 业务对象基类 `ObjectBase` 实现了哪个接口？这对它的内存管理意味着什么？
5. `ObjectInfo` 为什么要设计成只读 `struct`？它服务于哪一层？

---

## 二、机制题 🟡

6. 写出 `Release()`（无参）那一行核心代码，并解释 `toReleaseCount` 为什么"只是目标值而非保证值"。
7. 容量检查在源码里有**三个**触发点，分别列出，并说明为什么 `Unspawn` 里要附加 `internalObject.SpawnCount <= 0` 这个条件。
8. `DefaultReleaseObjectFilterCallback` 的两个阶段分别是什么？第一阶段（过期释放）是否受 `toReleaseCount` 限制？
9. 自动释放的节拍是怎么驱动的？`m_AutoReleaseTime`、`m_AutoReleaseInterval`、`realElapseSeconds` 三者关系是什么？
10. `GetCanReleaseObjects` 会把哪三类对象排除在候选集之外？
11. `Shutdown` 时传给 `ObjectBase.Release` 的参数是什么？业务侧为什么需要区分这个布尔值？
12. `AllowMultiSpawn` 为 true 时，同一对象 Spawn 三次后需要 Unspawn 几次才能进入可释放候选集？

---

## 三、架构 / 陷阱题 🔴

13. 池中对象数量**能否长期稳定地大于 Capacity**？给出会发生这种情况的具体场景，并说明 Capacity 是"硬上限"还是"软上限"。
14. `ObjectPool<T>` 内部为什么需要 `m_Objects`(按 name) 与 `m_ObjectMap`(按 target) **两套**索引？如果只保留一套，分别会失去什么能力？
15. `ReleaseObject` 里 `m_ObjectMap.Remove(internalObject.Peek().Target)` 用真身 `Target` 做 key。如果仿写时误用包装器对象做 key 会出现什么 bug？
16. 框架里"池中有池"指什么？`Object<T>` 包装器自身的 GC 由谁兜底？若仿写时省掉这一层，高频 Register/Release 会出现什么性能问题？
17. `InternalCreateObjectPool(Type, ...)`（非泛型版）用 `Activator.CreateInstance(typeof(ObjectPool<>).MakeGenericType(objectType), ...)` 构造池。为什么不能直接 `new`？这暴露了什么设计约束？
18. 管理器用 `Dictionary<TypeNamePair, ObjectPoolBase>` 存所有池。为什么 key 是"类型+名称"复合键而不是单纯 `Type`？
19. `ObjectPoolManager.Priority = 6` 的语义是什么？它如何影响 `Update` 轮询与 `Shutdown` 顺序？
20. 跨层桥接：核心层不引用 UnityEngine，Unity 的 Inspector/Debugger 如何拿到池内对象的实时信息？指出充当 DTO 的类型及其字段。

---

## 四、实操题 ✍️

21. 不看代码，凭记忆补全 `Entry.Unspawn()` 中防止计数为负的校验逻辑。
22. 给 `SimpleObjectPool<T>` 增加一个 `ReleaseAllUnused()`：释放所有可释放对象（忽略容量）。写出实现并说明与 `Release(int)` 的区别。
23. 设计一个**自定义释放策略** `ReleaseFilter<T>`：优先释放 `Priority` 最低的对象，忽略过期与 LRU。写出委托实现。
24. 画出（或文字描述）一个对象从 `Register(spawned:false)` 到最终 `Released` 的完整状态迁移路径，标注每步的 `SpawnCount` 与触发的钩子。

---

## 参考答案要点（自评用，先作答再展开）

<details>
<summary>点开核对</summary>

- **1**：只有 `ReleaseObject` 真正逐出并归还 ReferencePool；Spawn/Unspawn 是零分配的纯状态翻转。
- **2**：`SpawnCount > 0`。
- **3**：`ObjectPoolBase`(非泛型) 满足管理器统一字典存储；`IObjectPool<T>`(泛型) 满足业务方强类型 Spawn。
- **6**：`Release(Count - m_Capacity, m_DefaultReleaseObjectFilterCallback);`。受候选集过滤（InUse/Locked/!CustomFlag）约束，实际释放数可能远小于目标。
- **7**：Register 末尾、Unspawn 末尾、Capacity setter；附加条件确保只有刚变回 Free 的对象才触发瘦身，避免无谓扫描。
- **8**：阶段1 过期全释放(不受 toReleaseCount 限制)；阶段2 选择排序按 Priority↑/LastUseTime↑ 补足配额。
- **13**：能。当超出容量的对象全部 `IsInUse`/`Locked`/`!CustomCanReleaseFlag` 时无法释放 → Capacity 是软上限。
- **15**：`Unspawn`/`ReleaseObject` 按 target 反查会全部 miss，导致对象永远逐不出去 → 内存泄漏（或抛 "Can not find target"）。
- **16**：`Object<T>` 包装器也是 `IReference`，走 `ReferencePool.Acquire/Release`；省掉则包装器本身成为 GC 热点。
- **17**：实现类是 `private` 嵌套泛型，编译期拿不到具体 `T`，只能反射构造 → 暴露"实现全封闭、仅接口可见"的约束。
- **19**：数值越大越先 `Update`、越后 `Shutdown`；保证依赖它的模块先于它关闭。
- **20**：`ObjectInfo[]`（来自 `GetAllObjectInfos()`），含 Name/Locked/CustomCanReleaseFlag/Priority/LastUseTime/SpawnCount/IsInUse。

</details>
