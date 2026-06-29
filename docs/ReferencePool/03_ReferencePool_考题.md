# ReferencePool 引用池 · 考题（输出倒逼输入）

> 先合上文档独立作答，再回查。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. `IReference` 接口只有一个方法，是什么？它为什么是复用正确性的根本？
2. ReferencePool 是不是 `GameFrameworkModule`？它的生命周期是怎样的？
3. 引用池如何隔离不同类型的对象？用的什么数据结构做"桶"？
4. `Acquire<T>()` 与 `Acquire(Type)` 两条路径在"构造新对象"时分别用什么手段？各自适用什么场景？
5. `UnusedReferenceCount` / `UsingReferenceCount` 分别对应内部什么状态？

---

## 二、机制题 🟡

6. 写出 `Release` 内部的三步关键操作（顺序很重要）。
7. 池空时 Acquire 会怎样？`AddReferenceCount` 在什么时候递增？
8. `UsingReferenceCount` 与 `AcquireReferenceCount`、`ReleaseReferenceCount` 之间的等式关系是什么？
9. 开启 `EnableStrictCheck` 后，`Release` 多了哪一项检查？它能拦截哪个最致命的 bug？
10. 缓存容器用的是 `Queue` 还是 `Stack`？这决定了复用顺序是 FIFO 还是 LIFO？
11. 哪些操作加了 `lock`？计数字段的自增是否在锁内？这带来什么后果？

---

## 三、架构 / 陷阱题 🔴

12. ReferencePool 与 ObjectPool 都叫"池"，在"复用对象类型、是否有容量/过期、生命周期"三个维度上各有何不同？
13. ObjectPool 的 `Object<T>` 包装器为什么要存进 ReferencePool？这构成了怎样的两级缓存？
14. 如果业务 `Acquire` 后忘记 `Release`，统计上会看到什么现象？长期运行会发生什么？
15. 如果同一个引用被 `Release` 两次（且未开严格检查），后续会发生什么数据层面的灾难？
16. `Clear()` 实现里对一个 `List<int>` 字段，应该 `field = null` 还是 `field.Clear()`？为什么？从复用角度解释。
17. 为什么计数字段放在锁外自增是"可接受的不精确"？如果要求精确统计应怎么改，代价是什么？
18. 非泛型 `Acquire(Type)` 用 `Activator.CreateInstance`，相比泛型 `new T()` 有什么性能差异？框架为什么仍保留它？

---

## 四、实操题 ✍️

19. 给精简版补一个 `Remove<T>(int count)`：从队列丢弃 count 个空闲引用交给 GC。注意 count 超过队列长度的边界处理。
20. 设计一个会"自我清理脏数据"的 `IReference` 实现（含一个 `Dictionary` 字段和一个 `string` 字段），写出正确的 `Clear()`。
21. 写一段测试：预热 4 个、Acquire 6 个、Release 6 个，最终断言 `Unused == 6`、`Using == 0`，并解释为什么 Unused 是 6 而不是 4。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：`Clear()`；归还前重置自身字段，避免脏数据被下一个使用者拿到。
- **2**：不是 Module，纯静态、进程级常驻，无 Update/Shutdown。
- **6**：`reference.Clear()` → (严格模式)重复归还检测 → `m_References.Enqueue`，同时 `ReleaseCount++/UsingCount--`。
- **8**：`UsingReferenceCount = AcquireReferenceCount - ReleaseReferenceCount`。
- **9**：检查队列是否已含该引用 → 拦截"重复归还导致一实例多持有"。
- **10**：`Queue`，FIFO。
- **12**：见解析文档 2.3 对比表（复用类型/容量过期/生命周期三维）。
- **13**：包装器是 `IReference`，走 ReferencePool 复用 → ObjectPool 复用业务真身、ReferencePool 复用包装器，两级。
- **14**：`Using` 单调增、`Unused` 始终≈0，Acquire 退化为持续 new → 内存泄漏式增长。
- **15**：同一实例进队列两次，会被两个不同 Acquire 取出 → 两处业务共享同一对象，状态互相踩踏。
- **16**：`field.Clear()`，保留容器实例避免反复分配，符合复用初衷。
- **21**：预热 4，Acquire 6（前 4 复用、后 2 新建），Release 6 全部入队 → Unused=6，Using=0。

</details>
