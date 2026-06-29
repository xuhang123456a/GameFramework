# WebRequest Web 请求 · Facade 精简仿写

> 目标：~90 行，复刻"基于 TaskPool 的请求 Agent + GET/POST 最小分叉 + 整请求超时 + Complete 一次性取响应"。
> 取舍：复用 MiniTaskPool；对照 Download 仿写，刻意展示"砍掉文件流/续传/刷盘后剩下什么"。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `TaskPool<WebRequestTask>` | 复用 MiniTaskPool | **保留** |
| GET/POST = 有无 postData | 同 | **保留**（最小分叉） |
| 整请求超时(Start 重置) | 同 | **保留** |
| Complete 一次性取 byte[] | 同 | 保留 |
| `IWebRequestAgentHelper` 2 事件 | 回调委托 | 简化 |
| 无文件流/续传/刷盘/测速 | 无 | **刻意对照 Download** |

---

## 2. 核心代码

```csharp
using System;
using MiniTaskPool;   // 复用 TaskBase / ITaskAgent<T> / StartTaskStatus / TaskPool<T>

namespace MiniWebRequest
{
    public interface IWebRequestAgentHelper
    {
        event Action<byte[]> OnComplete;      // 一次性拿整个响应体
        event Action<string> OnError;
        void Request(string uri, byte[] postData);   // postData==null 即 GET
        void Reset();
    }

    public sealed class WebRequestTask : TaskBase
    {
        private static int s_Serial;
        public string Uri; public byte[] PostData; public float Timeout;
        public string Status = "Todo";

        public static WebRequestTask Create(string uri, byte[] postData, int priority, float timeout)
        {
            var t = new WebRequestTask { Uri = uri, PostData = postData, Timeout = timeout };
            t.SerialId = ++s_Serial; t.Priority = priority;
            return t;
        }
    }

    public sealed class WebRequestAgent : ITaskAgent<WebRequestTask>
    {
        private readonly IWebRequestAgentHelper m_Helper;
        private WebRequestTask m_Task;
        private float m_WaitTime;
        public WebRequestTask Task => m_Task;
        public Action<byte[]> OnSuccess; public Action<string> OnFailure;

        public WebRequestAgent(IWebRequestAgentHelper helper) => m_Helper = helper;

        public void Initialize()
        {
            m_Helper.OnComplete += OnComplete;
            m_Helper.OnError += OnError;
        }

        public StartTaskStatus Start(WebRequestTask task)
        {
            m_Task = task; m_Task.Status = "Doing";
            m_Helper.Request(task.Uri, task.PostData);   // 唯一分叉：postData 有无 = POST/GET
            m_WaitTime = 0f;                              // 整请求超时：仅此处重置
            return StartTaskStatus.CanResume;
        }

        public void Update(float dt)
        {
            if (m_Task.Status == "Doing")
            {
                m_WaitTime += dt;                         // 无流式回调可重置 → 累加到整请求超时
                if (m_WaitTime >= m_Task.Timeout) OnError("Timeout");
            }
        }

        private void OnComplete(byte[] response)
        {
            m_Helper.Reset();
            m_Task.Status = "Done";
            OnSuccess?.Invoke(response);                  // 整包响应上抛
            m_Task.Done = true;
        }

        private void OnError(string msg)
        {
            m_Helper.Reset();
            m_Task.Status = "Error";
            OnFailure?.Invoke(msg);
            m_Task.Done = true;
        }

        public void Reset() { m_Helper.Reset(); m_Task = null; m_WaitTime = 0f; }
        public void Shutdown() => Reset();
    }

    public sealed class WebRequestManager
    {
        private readonly TaskPool<WebRequestTask> m_Pool = new();
        public float Timeout = 30f;
        public Action<byte[]> WebRequestSuccess; public Action<string> WebRequestFailure;

        public void AddAgent(IWebRequestAgentHelper helper)
        {
            var agent = new WebRequestAgent(helper)
            { OnSuccess = b => WebRequestSuccess?.Invoke(b), OnFailure = e => WebRequestFailure?.Invoke(e) };
            m_Pool.AddAgent(agent);
        }

        public int AddWebRequest(string uri, byte[] postData = null, int priority = 0)
        {
            var task = WebRequestTask.Create(uri, postData, priority, Timeout);
            m_Pool.AddTask(task);
            return task.SerialId;
        }

        public void Update(float dt) => m_Pool.Update(dt);
    }
}
```

---

## 3. 使用示例

```csharp
sealed class FakeWebHelper : IWebRequestAgentHelper
{
    public event Action<byte[]> OnComplete;
    public event Action<string> OnError;
    private int _ticks; private bool _running;
    public void Request(string uri, byte[] postData) { _running = true; _ticks = 0; }
    public void Reset() => _running = false;
    public void Tick()   // 模拟 2 帧后返回响应
    {
        if (!_running) return;
        if (++_ticks >= 2) { _running = false; OnComplete?.Invoke(new byte[] { 1, 2, 3 }); }
    }
}

var mgr = new WebRequestManager();
mgr.WebRequestSuccess = bytes => Console.WriteLine($"响应 {bytes.Length} 字节");
var helper = new FakeWebHelper();
mgr.AddAgent(helper);

mgr.AddWebRequest("https://api/login", postData: new byte[] { 9 });  // POST（带 postData）
mgr.AddWebRequest("https://api/rank");                                // GET（无 postData）

for (int f = 0; f < 4; f++) { helper.Tick(); mgr.Update(0.1f); }
```

---

## 4. 取舍自检

- ✅ 保留：基于 TaskPool、GET/POST 单字段分叉、整请求超时(Start 重置)、Complete 一次性取响应、CanResume 续传语义。
- ❌ 砍掉：文件流/断点续传/分块刷盘/测速（这些是 Download 独有，WebRequest 本就不需要）、按 tag 批量操作。
- ⚠️ 提醒：超时只在 Start 重置（整请求耗时），与 Download 的"每次收数据重置"语义不同——取决于有无流式回调。别照搬 Download 的超时重置逻辑到 WebRequest。

---

> 配套考题见 `03_WebRequest_考题.md`。
