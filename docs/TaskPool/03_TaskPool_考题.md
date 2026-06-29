# TaskPool 任务池 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. TaskPool 中"任务"(`TaskBase`) 与"代理"(`ITaskAgent`) 各自扮演什么角色？为什么要分离？
2. 三大容器 `m_FreeAgents`/`m_WorkingAgents`/`m_WaitingTasks` 分别用什么数据结构？空闲代理为什么用 `Stack`？
3. `StartTaskStatus` 的四个值分别是什么含义？
4. `TaskStatus`（Todo/Doing/Done）与 `StartTaskStatus` 是同一个枚举吗？各用在哪？
5. `TaskInfo.IsValid` 为 false 代表什么？什么时候返回 default(TaskInfo)？

---

## 二、机制题 🟡

6. `AddTask` 的插入算法保证了什么排序？同优先级的两个任务谁先被执行？
7. `Update` 的两个阶段分别叫什么，各做什么？`Paused` 时会怎样？
8. 阶段1 `ProcessRunningTasks` 如何判断一个任务完成？完成后做了哪四件事？
9. 阶段2 `ProcessWaitingTasks` 的循环终止条件是什么（两个）？
10. 默认（normal）情况下代理在哪两个容器之间往返？代理会被销毁吗？何时销毁？

---

## 三、架构 / 陷阱题 🔴（核心）

11. **画出 `StartTaskStatus` 的 4×3 决策矩阵**：四种状态分别对"代理归还/任务出队/任务释放"三个动作如何取舍？
12. `CanResume` 与 `HasToWait` 最容易混淆。两者对"任务是否出队、代理是否归还"的处理正好如何不同？分别对应下载的什么真实场景？
13. 如果把 `CanResume` 的任务也 `ReferencePool.Release` 掉，会发生什么灾难？
14. 为什么"代理归还"判断是 `Done || HasToWait || UnknownError`，而"任务出队"判断是 `Done || CanResume || UnknownError`？这两个集合的差集说明了什么？
15. 一个下载几分钟的任务，是如何用"单帧返回的 Start" + "逐帧推进的 Update"协同完成的？
16. 代理用 `Stack` 复用、任务用 `ReferencePool` 复用，两者生命周期有何本质不同？为什么不能把任务也做成常驻缓存？
17. `RemoveTask` 移除一个"工作中"的任务时，对代理做了什么？为什么不能直接把代理也丢弃？
18. TaskPool 不是 `GameFrameworkModule`，它如何获得帧驱动？这与 ObjectPool/Fsm 的 Module 身份有何不同？

---

## 四、实操题 ✍️

19. 给精简版补 `GetTaskInfo(int serialId)`：先扫工作中（Doing/Done），再扫等待（Todo），查无返回 `IsValid=false`。
20. 实现一个 `Start` 返回 `HasToWait` 的代理场景：同一 host 已有 2 个下载在跑时让新任务等待。说明任务会怎样。
21. 写出阶段2 中三个 if 判断的状态集合（用 enum 列举），并解释为什么是三个独立判断而非 switch 一刀切。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **2**：Free=`Stack`(LIFO，复用最近归还的，缓存友好)，Working=`LinkedList`，Waiting=`LinkedList`(优先级有序)。
- **6**：优先级降序、同级 FIFO；先加入的同级任务先执行。
- **8**：`task.Done==true` → agent.Reset → 压回 FreeAgents → 移出 Working → Release(task)。
- **9**：`current != null && FreeAgentCount > 0`。
- **11**：见解析文档 2.3 矩阵：Done(✅✅✅)、CanResume(❌✅❌)、HasToWait(✅❌❌)、UnknownError(✅✅✅)。
- **12**：CanResume=出队+代理不归还(任务转工作中)；HasToWait=不出队+代理归还(原样退回下帧重试)。CanResume↔开始下载续传；HasToWait↔并发上限需等待。
- **13**：代理仍持有该 task 引用并逐帧 Update，task 已被回收/复用 → 数据串改、空引用或崩溃。
- **14**：差集——"代理归还但任务不出队"=HasToWait；"任务出队但代理不归还"=CanResume。正交分解的体现。
- **16**：代理与池同寿(Reset 复用，Shutdown 才销毁)；任务与单次请求同寿(Done 即 Release)。任务是一次性数据，常驻会串用。
- **18**：被 Download/Resource 等 Manager(Module) 内嵌持有，在其 Update 里转调 TaskPool.Update。
- **21**：归还=`{Done, HasToWait, UnknownError}`；出队=`{Done, CanResume, UnknownError}`；释放=`{Done, UnknownError}`。三动作正交，switch 一刀切无法表达"同时满足不同子集"。

</details>
