# EventPool 事件池 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. 事件分发的路由键是什么（`Type` 还是 `int`）？由谁提供？
2. `Fire` 与 `FireNow` 在线程安全性、分发时机上分别有何区别？
3. `BaseEventArgs` 继承自哪个基类？这让它具备了什么能力（提示：复用）？
4. `EventPoolMode` 的四个值分别是什么含义？`Default=0` 代表什么约束？
5. `Event` 内部结点为什么也实现 `IReference`？

---

## 二、机制题 🟡

6. 延迟事件从 `Fire` 到真正回调，经过哪几个步骤？分发发生在哪个时机/线程？
7. `HandleEvent` 里一次分发结束后，会 `ReferencePool.Release` 哪个对象？`Update` 循环里又 Release 哪个？
8. 当某个 Id 没有任何 handler 时，分发逻辑的三级 fallback 是什么？
9. `Subscribe` 在 `AllowMultiHandler` 与 `AllowDuplicateHandler` 两种模式下的校验差异是什么？
10. `m_CachedNodes` 的 key 为什么是事件实例 `e` 而不是一个全局字段？这支持了什么能力？
11. `Clear()` 与 `Shutdown()` 分别清理了哪些数据？handler 会不会被 `Clear()` 清掉？

---

## 三、架构 / 陷阱题 🔴

12. 在事件回调里调用 `Unsubscribe` 取消**当前正在执行的 handler 自己**，为什么不会让遍历崩溃？`Unsubscribe` 做了什么配合动作？
13. 在事件回调里 `Unsubscribe` 掉**下一个**即将被回调的 handler，迭代游标如何保证不指向已删除节点？
14. 为什么 `Fire` 之后业务代码不能再持有/读写传入的 `EventArgs`？从复用角度说明会发生什么。
15. 跨线程调用 `Fire`，回调最终在哪个线程执行？这套"多生产者单消费者"的边界是靠什么收窄线程安全范围的？
16. 事件回调里又 `FireNow` 触发另一个事件（重入），两层分发的游标会不会互相干扰？为什么？
17. 如果把 `Fire` 实现成"立即分发"（去掉队列），会破坏哪些保证？
18. EventArgs 与 Event 结点构成"双层复用"。如果只回收其一会怎样？

---

## 四、实操题 ✍️

19. 给精简版加一个 `Count(int id)` 返回某事件的 handler 数量，注意 id 不存在时的处理。
20. 写一个会"在回调中取消订阅自己"的 handler，并解释为什么在本框架下安全。
21. 设计一个 `BaseEventArgs` 子类承载一条网络消息（含 `int Cmd` 和 `byte[] Payload`），写出正确的 `Clear()`，并说明 `Payload` 该置 null 还是保留。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：`int Id`，由 `BaseEventArgs.Id` 提供。
- **2**：Fire 线程安全、入队下一帧主线程分发；FireNow 非线程安全、当场同步分发。
- **6**：`Event.Create`(取结点) → lock 入队 → 下一帧 `Update` lock 内 Dequeue → `HandleEvent` 主线程回调。
- **7**：HandleEvent 内 Release `EventArgs e`；Update 循环里 Release `eventNode`(Event 结点)。
- **8**：有 handler→逐个回调；否则有 `m_DefaultHandler`→调它；否则若 `!AllowNoHandler`→抛异常。
- **10**：以 `e` 为 key 支持**重入**——不同事件实例各有独立游标，互不干扰。
- **12**：分发前已把 `current.Next` 存入 `m_CachedNodes[e]`，回调返回后从缓存读下一个；删自己不影响已存的 Next。
- **13**：`Unsubscribe` 遍历缓存节点，若被删节点正是某游标，则把游标前移到其 `Next`（经 m_TempNodes 暂存回写）。
- **14**：args 在 HandleEvent 末尾被 `Release` 回 ReferencePool，可能立刻被复用给别的事件 → 继续持有会读到串改的数据。
- **15**：主线程（Update 所在线程）；安全范围被收窄到 `m_Events` 队列的 lock 进出，回调本身无并发。
- **16**：不干扰，因游标按事件实例 `e` 分键存储。
- **18**：只回收一个 → 另一个泄漏（队列退化为持续 new）。

</details>
