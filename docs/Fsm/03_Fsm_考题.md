# Fsm 有限状态机 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. `Fsm<T>` 的泛型参数 `T` 代表什么？`where T : class` 约束的意义是什么？
2. 状态在 FSM 内以什么作为唯一键存储？能否同时存在两个同类型状态实例？
3. `IsRunning` 与 `IsDestroyed` 分别由什么条件决定？
4. 列出 `FsmState<T>` 的五个生命周期钩子及各自触发时机。
5. `FsmBase` 为什么要把 `CurrentStateName`、`CurrentStateTime` 等提到非泛型层？

---

## 二、机制题 🟡

6. `Start<TState>` 做了哪三步？若 FSM 已在运行会怎样？
7. `ChangeState` 内部的四步操作及其固定顺序是什么？
8. `Shutdown()` 只调了一行 `ReferencePool.Release(this)`，真正的析构逻辑写在哪？包含哪些动作？
9. `OnLeave` 的 `isShutdown` 参数 true / false 分别对应什么场景？
10. `SetData` 覆盖一个已存在的 key 时，对旧 `Variable` 做了什么？为什么？
11. `FsmManager.Update` 为什么要先把 `m_Fsms` 拷进 `m_TempFsms` 再遍历？

---

## 三、架构 / 陷阱题 🔴

12. 为什么 `ChangeState` 被设计成 `internal`，业务状态只能经 `FsmState<T>.ChangeState`（protected）触发？如果开放成 public 会破坏什么不变量？
13. `OnInit` 与 `OnEnter` 在调用次数上有何本质区别？把一次性初始化误写进 `OnEnter` 会导致什么 bug？
14. 整台 `Fsm<T>` 是 `IReference`。在 `Clear()` 里如果忘记释放黑板数据 `m_Datas` 里的 Variable，会发生什么？
15. `Procedure`（流程）模块如何复用 FSM？它的"状态"对应什么？
16. `Fsm<T>` 同时实现 `FsmBase`、`IFsm<T>`、`IReference` 三套契约，各服务于哪个使用者？
17. 在 `OnUpdate` 里调用 `DestroyFsm` 销毁自己所在的 FSM，为什么管理器的快照轮询能避免崩溃？销毁后当帧还会继续 Update 吗？
18. `m_Datas` 为什么用懒初始化（首次 SetData 才 new）而不是构造时就建？这体现了什么取舍？

---

## 四、实操题 ✍️

19. 写一个带"超时自动切换"的状态：进入后 3 秒自动 ChangeState 到另一个状态（利用 `CurrentStateTime` 或自累加）。
20. 给精简版补 `HasState<TState>()` 与 `GetAllStates()`。
21. 画出一台 FSM 从 Create 到 Destroyed 的状态迁移，标注每个迁移触发的钩子与 `IsRunning`/`IsDestroyed` 取值。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **2**：以状态的 `Type` 为键（`Dictionary<Type, FsmState<T>>`）；同类型只能一个实例。
- **3**：`IsRunning = m_CurrentState != null`；`IsDestroyed` 由 Create 置 false、Clear 置 true。
- **6**：检查未运行 → 取状态 → 计时清零、设当前状态、`OnEnter`；已运行则抛异常。
- **7**：`OnLeave(false)` → 计时清零 → 换 `m_CurrentState` → `OnEnter`。顺序不可乱。
- **8**：写在 `Clear()`（IReference 契约，Release 触发）；动作：当前状态 `OnLeave(true)` → 所有状态 `OnDestroy` → 释放所有 Variable → 置 IsDestroyed=true。
- **11**：状态 OnUpdate 中可能 Create/Destroy FSM，直接遍历原字典会抛"集合被修改"。
- **12**：保证"迁移必由当前状态发起"、OnLeave/OnEnter 成对；public 化后外部可乱切，破坏成对不变量。
- **13**：OnInit 每状态仅一次，OnEnter 每次进入都调；一次性初始化写进 OnEnter 会重复执行（重复订阅/重复分配）。
- **14**：Variable 不归还 ReferencePool → 内存泄漏。
- **15**：Procedure 的每个 `ProcedureBase` 就是 `FsmState<IProcedureManager>`，游戏流程即状态迁移。
- **17**：快照副本遍历 + `if (fsm.IsDestroyed) continue`；销毁后 IsDestroyed=true，当帧后续不再 Update 它。

</details>
