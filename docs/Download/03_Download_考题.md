# Download 下载 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. Download 模块的"任务"和"代理"分别对应 TaskPool 的哪两个抽象？
2. 真正发起网络请求的是谁（DownloadManager / DownloadAgent / IDownloadAgentHelper）？
3. 下载并发数由什么决定？
4. `DownloadTaskStatus` 有哪四个状态？
5. `IDownloadAgentHelper` 通过哪四个事件向 Agent 回吐信息？

---

## 二、机制题 🟡

6. 下载时文件先写到什么名字？完成后做了什么操作变成正式文件？
7. 断点续传时，续传的起始字节偏移从哪里得到？
8. `Agent.Start` 返回的是哪个 `StartTaskStatus`？为什么不是 `Done`？
9. FlushSize 控制什么？为什么不是每次收到字节就 Flush？
10. 超时是怎么检测的？为什么每次 UpdateBytes/UpdateLength 要把 `m_WaitTime` 置 0？
11. `OnComplete` 里 `m_SavedLength != CurrentLength` 校验的是什么？

---

## 三、架构 / 陷阱题 🔴

12. Download 几乎不含调度代码，调度逻辑在哪？这体现了什么"组合底座"思想？
13. 为什么"完成后才 rename"如此重要？如果直接写正式文件名会有什么后果？
14. 三层下沉（TaskPool / DownloadAgent / IDownloadAgentHelper）各负责什么？为什么网络 API 要放最底层 helper？
15. FlushSize 设得很大 vs 很小，分别有什么利弊（吞吐 vs 崩溃安全）？
16. Timeout 检测的是"总下载时长"还是"无响应时长"？两者实现上的关键区别是什么？
17. DownloadCounter 为什么用"滑动窗口"测速而不是"总字节/总时间"？滑动窗口节点如何复用？
18. 下载中断（断网）后重启下载，临时文件与 Range 请求如何配合实现续传？

---

## 四、实操题 ✍️

19. 给精简版 Agent 增加"已下载百分比"计算（需要总长度，思考从哪个事件拿）。
20. 实现一个"下载失败自动重试 3 次"的包装（提示：Error 时重新 AddDownload，计数）。
21. 描述 DownloadCounter 在"刚开始下载""稳定下载""下载停止"三个阶段 CurrentSpeed 的变化。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：DownloadTask=TaskBase；DownloadAgent=ITaskAgent。
- **2**：IDownloadAgentHelper（Manager 调度、Agent 管文件流与状态、Helper 干网络）。
- **3**：AddDownloadAgentHelper 加入的代理数量。
- **6**：先写 `{path}.download`；完成后删旧正式文件 + `File.Move` rename。
- **7**：续传时打开已存在的 `.download`，`m_StartLength = 文件当前长度`，作为 Range 请求的 fromPosition。
- **8**：`CanResume`——下载是跨多帧的续传任务，转入工作中由 TaskPool 逐帧 Update 推进，不是一帧完成。
- **9**：控制累积多少字节才刷盘一次；避免频繁磁盘 IO，提升吞吐。
- **10**：每帧累加 m_WaitTime，超 Timeout 报错；收到数据即归零→检测"卡死"而非"总时长"。
- **11**：写入磁盘的累计字节(SavedLength)与下载报告的当前长度(CurrentLength=Start+Downloaded)是否一致，防数据缺失。
- **12**：在 TaskPool（并发/优先级/续传推进）；Download 只实现 Agent 的 Start/Update。组合底座=通用调度复用、业务执行自写。
- **13**：保证正式文件要么不存在要么完整；直接写正式名会让中断的半截文件冒充完整，引发后续解压/加载错误。
- **16**：无响应时长；区别在于"收到数据就重置计时器"，所以测的是停顿而非累计耗时。
- **17**：平滑瞬时速度避免单帧抖动；过期节点(超 RecordInterval)从队首移除并 ReferencePool.Release 复用。
- **18**：临时文件保留已下字节→重启时 Seek 到末尾取 StartLength→Helper 发带 Range(fromPosition) 的请求从断点续传。

</details>
