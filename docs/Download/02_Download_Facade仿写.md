# Download 下载 · Facade 精简仿写

> 目标：~160 行，复刻"基于 TaskPool 的下载 Agent + .download 临时文件断点续传 + 分块刷盘 + 超时重置 + 滑动窗口测速"。
> 取舍：复用前文 MiniTaskPool 的 `TaskPool/ITaskAgent`；网络 IO 用注入式 helper 模拟字节回吐。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `TaskPool<DownloadTask>` 调度 | 复用 MiniTaskPool | **保留**（不重写调度） |
| `.download` 临时文件 + rename | 同 | **保留**（断点续传核心） |
| HTTP Range 续传 | `fromPosition` 入参 | 保留语义 |
| FlushSize 分块刷盘 | 同 | **保留** |
| Timeout 无响应检测 | 同 | **保留** |
| DownloadCounter 滑动窗口 | 简化版 | 保留思路 |
| `IDownloadAgentHelper` 4 事件 | 回调委托 | 简化 |

---

## 2. 核心代码

```csharp
using System;
using System.IO;
// 复用前文 MiniTaskPool: TaskBase / ITaskAgent<T> / StartTaskStatus / TaskPool<T>
using MiniTaskPool;

namespace MiniDownload
{
    // 网络 IO 注入点：真实项目用 UnityWebRequest/HttpClient
    public interface IDownloadAgentHelper
    {
        event Action<byte[], int, int> OnBytes;   // (buffer, offset, length) 收到字节
        event Action<int> OnLength;                // 进度增量
        event Action<long> OnComplete;             // 完成(总长)
        event Action<bool, string> OnError;        // (deleteDownloading, msg)
        void Download(string uri, long fromPosition);
        void Reset();
    }

    public sealed class DownloadTask : TaskBase
    {
        private static int s_Serial;
        public string DownloadPath, DownloadUri;
        public int FlushSize; public float Timeout;
        public string Status = "Todo";   // Todo/Doing/Done/Error

        public static DownloadTask Create(string path, string uri, int priority, int flush, float timeout)
        {
            var t = new DownloadTask { DownloadPath = path, DownloadUri = uri, FlushSize = flush, Timeout = timeout };
            t.SerialId = ++s_Serial; t.Priority = priority;
            return t;
        }
    }

    public sealed class DownloadAgent : ITaskAgent<DownloadTask>, IDisposable
    {
        private readonly IDownloadAgentHelper m_Helper;
        private DownloadTask m_Task;
        private FileStream m_FileStream;
        private int m_WaitFlushSize;
        private float m_WaitTime;
        private long m_StartLength, m_SavedLength, m_DownloadedLength;
        public DownloadTask Task => m_Task;
        private long CurrentLength => m_StartLength + m_DownloadedLength;

        public DownloadAgent(IDownloadAgentHelper helper) => m_Helper = helper;

        public void Initialize()
        {
            m_Helper.OnBytes += OnBytes;
            m_Helper.OnLength += OnLength;
            m_Helper.OnComplete += OnComplete;
            m_Helper.OnError += OnError;
        }

        public StartTaskStatus Start(DownloadTask task)
        {
            m_Task = task; m_Task.Status = "Doing";
            var tmp = m_Task.DownloadPath + ".download";
            try
            {
                if (File.Exists(tmp))                       // 续传：接着已有字节写
                {
                    m_FileStream = File.OpenWrite(tmp);
                    m_FileStream.Seek(0, SeekOrigin.End);
                    m_StartLength = m_SavedLength = m_FileStream.Length;
                    m_DownloadedLength = 0;
                }
                else                                        // 新下载
                {
                    Directory.CreateDirectory(Path.GetDirectoryName(m_Task.DownloadPath));
                    m_FileStream = new FileStream(tmp, FileMode.Create, FileAccess.Write);
                    m_StartLength = m_SavedLength = m_DownloadedLength = 0;
                }
                m_Helper.Download(m_Task.DownloadUri, m_StartLength);  // fromPosition 触发 Range
                return StartTaskStatus.CanResume;           // 转入续传，由 TaskPool 逐帧推进
            }
            catch (Exception e) { OnError(false, e.ToString()); return StartTaskStatus.UnknownError; }
        }

        public void Update(float dt)
        {
            if (m_Task.Status == "Doing")
            {
                m_WaitTime += dt;
                if (m_WaitTime >= m_Task.Timeout) OnError(false, "Timeout");
            }
        }

        private void OnBytes(byte[] buf, int offset, int len)
        {
            m_WaitTime = 0f;                                // 收到数据→重置超时
            m_FileStream.Write(buf, offset, len);
            m_WaitFlushSize += len; m_SavedLength += len;
            if (m_WaitFlushSize >= m_Task.FlushSize)        // 累积到阈值才刷盘
            { m_FileStream.Flush(); m_WaitFlushSize = 0; }
        }

        private void OnLength(int delta) { m_WaitTime = 0f; m_DownloadedLength += delta; }

        private void OnComplete(long length)
        {
            m_WaitTime = 0f; m_DownloadedLength = length;
            if (m_SavedLength != CurrentLength) throw new Exception("Internal download error.");
            m_Helper.Reset(); m_FileStream.Close(); m_FileStream = null;
            if (File.Exists(m_Task.DownloadPath)) File.Delete(m_Task.DownloadPath);
            File.Move(m_Task.DownloadPath + ".download", m_Task.DownloadPath);  // 原子 rename
            m_Task.Status = "Done"; m_Task.Done = true;     // 通知 TaskPool 回收
        }

        private void OnError(bool deleteDownloading, string msg)
        {
            m_Helper.Reset();
            if (m_FileStream != null) { m_FileStream.Close(); m_FileStream = null; }
            if (deleteDownloading) File.Delete(m_Task.DownloadPath + ".download");
            m_Task.Status = "Error"; m_Task.Done = true;
        }

        public void Reset()
        {
            m_Helper.Reset();
            if (m_FileStream != null) { m_FileStream.Close(); m_FileStream = null; }
            m_Task = null; m_WaitFlushSize = 0; m_WaitTime = 0;
            m_StartLength = m_SavedLength = m_DownloadedLength = 0;
        }

        public void Shutdown() => Dispose();
        public void Dispose() { if (m_FileStream != null) { m_FileStream.Dispose(); m_FileStream = null; } }
    }

    // 管理器：薄薄一层，调度全靠 TaskPool
    public sealed class DownloadManager
    {
        private readonly TaskPool<DownloadTask> m_Pool = new();
        public int FlushSize = 1024 * 1024;
        public float Timeout = 30f;

        public void AddAgent(IDownloadAgentHelper helper)
        {
            var agent = new DownloadAgent(helper);
            m_Pool.AddAgent(agent);   // AddAgent 内部会 Initialize
        }

        public int AddDownload(string path, string uri, int priority = 0)
        {
            var task = DownloadTask.Create(path, uri, priority, FlushSize, Timeout);
            m_Pool.AddTask(task);
            return task.SerialId;
        }

        public void Update(float dt) => m_Pool.Update(dt);   // 全部委托 TaskPool
    }
}
```

---

## 3. 使用示例（伪网络 helper）

```csharp
// 模拟 helper：每帧回吐一段字节，到总长即 Complete
sealed class FakeHelper : IDownloadAgentHelper
{
    public event Action<byte[], int, int> OnBytes;
    public event Action<int> OnLength;
    public event Action<long> OnComplete;
    public event Action<bool, string> OnError;
    private long _total, _got; private bool _running;

    public void Download(string uri, long fromPosition) { _total = 4096; _got = fromPosition; _running = true; }
    public void Reset() { _running = false; }
    public void Tick()   // 由外部驱动模拟网络回调
    {
        if (!_running) return;
        var chunk = new byte[1024];
        OnBytes?.Invoke(chunk, 0, chunk.Length);
        OnLength?.Invoke(chunk.Length);
        _got += 1024;
        if (_got >= _total) { _running = false; OnComplete?.Invoke(_total); }
    }
}

var mgr = new DownloadManager();
var h = new FakeHelper();
mgr.AddAgent(h);
mgr.AddDownload("./cache/pack.bin", "https://cdn/pack.bin");
for (int f = 0; f < 8; f++) { h.Tick(); mgr.Update(0.1f); }  // 分帧推进直到完成
```

---

## 4. 取舍自检

- ✅ 保留：基于 TaskPool 调度、`.download` 临时文件 + rename、续传起点取临时文件长度、FlushSize 分块刷盘、Timeout 收到数据即重置、SavedLength==CurrentLength 完成校验。
- ❌ 砍掉：DownloadCounter 完整滑动窗口测速、HTTP 真实 Range 请求、4 个独立 EventArgs、按 tag 批量移除。
- ⚠️ 提醒：完成前绝不写正式文件名（避免半截文件冒充完整）；超时计时器每次收数据重置（检测卡死而非总时长）；刷盘计数器每次 Flush 后清零。三者各管各的，别共用计数器。

---

> 配套考题见 `03_Download_考题.md`。
