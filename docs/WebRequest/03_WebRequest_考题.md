# WebRequest Web 请求 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱（重点对照 Download）

---

## 一、概念题 🟢

1. WebRequest 与 Download 共享哪个底座？各自的 Task/Agent 如何对应到该底座的抽象？
2. `IWebRequestAgentHelper` 有几个事件？比 `IDownloadAgentHelper` 少了哪两个？为什么？
3. GET 与 POST 在代码里靠什么区分？
4. 响应数据是分块流式返回还是一次性返回？通过哪个事件？
5. `WebRequestTaskStatus` 有哪些状态？

---

## 二、机制题 🟡

6. `Agent.Start` 返回什么状态？这与 Download 一致吗？
7. WebRequest 的超时计时器在何时重置？只重置一次还是多次？
8. `OnComplete` 里如何拿到响应体？它通过哪个回调上抛给业务？
9. WebRequestTask 是否走 ReferencePool？postData 和响应 byte[] 呢？
10. 没有 postData 时调用 helper 的哪个 Request 重载？

---

## 三、架构 / 陷阱题 🔴（对照 Download）

11. **核心题**：Download 与 WebRequest 用同一个 TaskPool，却派生出完全不同的业务。请说明"调度复用、业务自写"这条边界各自落在哪里。
12. Download 的超时是"无响应停顿"，WebRequest 的超时是"整请求耗时"。同一个 `m_WaitTime` 字段为何语义不同？根因是什么？
13. 如果把 Download 的"每次 UpdateBytes 重置超时"逻辑照搬到 WebRequest，会发生什么？
14. WebRequest 为什么不需要 `.download` 临时文件和断点续传？
15. 为什么 GET/POST 不各建一个类，而用"有无 postData"一个字段表达？
16. 响应体 `byte[]` 不走 ReferencePool，频繁大请求会有什么隐患？如何缓解？
17. 若要支持"请求重试"，应该在哪一层加（Manager / Agent / Helper / 业务包装）？为什么？

---

## 四、实操题 ✍️

18. 给精简版加"请求成功后自动 JSON 反序列化为对象"的便捷包装。
19. 实现一个"同一 URI 的重复请求去重"（短时间内相同 URI 只发一次）。
20. 对照画出 Download 与 WebRequest 的 Agent 状态机，标出关键差异（流式回调 vs 一次性、超时重置点）。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：TaskPool；WebRequestTask=TaskBase，WebRequestAgent=ITaskAgent（与 Download 同构）。
- **2**：2 个(Complete/Error)；少了 UpdateBytes/UpdateLength——短请求一次性返回，无流式分块与进度。
- **3**：Task 是否携带 postData（`GetPostData()==null` 即 GET）。
- **4**：一次性；通过 Complete 事件的 `GetWebResponseBytes()`。
- **6**：CanResume；与 Download 一致（都是跨帧等待的任务）。
- **7**：仅在 Start 时重置一次，之后单调累加直到结束。
- **9**：WebRequestTask 走 ReferencePool；postData 与响应 byte[] 不走，交 GC。
- **11**：调度（并发/优先级/续传推进/代理复用）复用 TaskPool；业务（Download=文件流+续传+刷盘，WebRequest=发请求+等整包响应）各写 Agent/Task。
- **12**：根因是有无流式回调。Download 每次 UpdateBytes 把 m_WaitTime 归零→测停顿；WebRequest 无中间回调→只在 Start 归零→测整请求耗时。
- **13**：WebRequest 没有 UpdateBytes 回调，照搬的重置代码永不触发，等价于现状；但若误以为有进度回调而依赖它重置，会导致超时逻辑失效。
- **14**：短请求一问一答、响应小、无需中断恢复；续传是为大文件长下载设计的。
- **15**：GET/POST 99% 相同，仅差请求体；用可空字段表达可选差异避免类型爆炸。
- **16**：大响应体频繁分配增加 GC 压力；缓解=复用缓冲、限制并发、及时释放引用。
- **17**：业务包装层（Error 回调里重新 AddWebRequest 计数）；底层 TaskPool/Agent 不该耦合重试策略，保持单一职责。

</details>
