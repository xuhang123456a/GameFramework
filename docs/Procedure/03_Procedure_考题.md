# Procedure 流程 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. `ProcedureBase` 继承自哪个类？这说明"流程"本质是什么？
2. 整个游戏有几台流程 FSM？与 FsmManager 管多台 FSM 有何不同？
3. `IProcedureManager` 在 FSM 里扮演什么角色？
4. 流程间如何切换？复用了 Fsm 的什么机制？
5. `CurrentProcedure` 实际取自 FSM 的什么？

---

## 二、机制题 🟡

6. Procedure 概念与 Fsm 概念如何一一对应（流程/持有者/容器/切换/启动）？
7. `Initialize` 做了什么？流程是如何成为 FSM 状态的？
8. `StartProcedure<T>` 对应 Fsm 的什么调用？
9. Procedure 模块自己重写了哪些生命周期逻辑？
10. `Shutdown` 时流程 FSM 发生什么？

---

## 三、架构 / 陷阱题 🔴

11. 为什么 ProcedureManager 的 Priority 是 -2（最低）？这保证了什么时序？
12. 为什么"游戏宏观流程"适合用状态机建模？用一堆 bool flag 管理会有什么问题？
13. Procedure 几乎不写代码（薄封装），这种"术语映射"封装的价值是什么？反面（在薄封装塞重复逻辑）有何害处？
14. 热更新流程（CheckVersion→Update→Preload）如何映射到 Procedure？结合 Resource 文档说明。
15. 流程切换若不经 FsmState.ChangeState 而直接改当前流程引用，会破坏什么？
16. Procedure 作为"总指挥"如何连接 Resource/Scene/UI 等模块？它在什么时机调用它们？

---

## 四、实操题 ✍️

17. 设计一个带"超时跳转"的流程：进入后 5 秒若未满足条件则强制 ChangeState 到错误流程（用 CurrentProcedureTime）。
18. 写一个三流程链：A→B→C，A 在 OnEnter 直接转 B，B 在 OnUpdate 满足条件转 C。
19. 解释为什么流程对象通常是单例长生命周期，不需要对象池复用。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：`FsmState<IProcedureManager>`；流程本质是状态机的一个状态。
- **2**：一台（单台流程 FSM）；FsmManager 可管多台不同 Owner 的 FSM，Procedure 只用其中一台。
- **3**：FSM 的 Owner(持有者 T)。
- **4**：`ChangeState<NextProcedure>`；复用 Fsm 的封闭式状态切换。
- **5**：`m_ProcedureFsm.CurrentState`。
- **6**：流程=State、持有者=Owner、容器=IFsm、切换=ChangeState、启动=Start。
- **7**：用 FsmManager.CreateFsm(this, procedures) 把所有流程作为状态注册建一台 FSM，每个流程 OnInit。
- **9**：几乎没有——全部复用 Fsm。
- **11**：最低优先级→最后轮询(等其他模块就绪)、最先关闭(它依赖的模块还在)；保证总指挥的启动/关闭时序正确。
- **12**：流程有明确状态与迁移条件；bool flag 组合爆炸、状态隐式、迁移分散，迅速失控。
- **13**：价值=给通用机制一个领域专用接口，让业务用"流程"思考、降心智负担、复用全部 FSM 能力；塞重复逻辑则破坏复用、引入不一致 bug。
- **14**：每个热更阶段是一个 ProcedureBase，OnUpdate 里判断完成条件 ChangeState 到下一阶段，进度经 EventPool 监听。
- **15**：破坏 OnLeave/OnEnter 成对调用，导致旧流程未清理、新流程未初始化。
- **16**：流程在 OnEnter/OnUpdate 里按时机调用——如 UpdateResources 流程调 Resource 下载、ChangeScene 流程调 Scene 加载、Menu 流程调 UI 打开。
- **19**：流程数量固定且整局常驻，创建一次用到底，复用无收益。

</details>
