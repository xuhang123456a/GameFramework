# TaskPool 任务池 · Facade 精简仿写

> 目标：~160 行独立可编译，复刻"任务/代理分离 + 优先级队列 + 双阶段轮询 + StartTaskStatus 三维决策 + 跨帧续传"。
> 取舍：任务复用与 ReferencePool 化简为普通对象；完整保留 4×3 决策矩阵与双阶段调度这两大精髓。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `Stack<ITaskAgent<T>>` 空闲栈 | 同 | 保留（LIFO 复用） |
| `LinkedList` 工作链表 + 等待队列 | 同 | 保留 |
| `AddTask` 优先级有序插入 | 同 | **保留** |
| `StartTaskStatus` 四值 | 同 | **保留**（决策命脉） |
| 双阶段 ProcessRunning/Waiting | 同 | **保留** |
| `TaskBase : IReference` 复用 | 普通 `TaskBase` | 简化 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniTaskPool
{
    public enum StartTaskStatus : byte { Done = 0, CanResume, HasToWait, UnknownError }

    public abstract class TaskBase
    {
        public int SerialId { get; internal set; }
        public string Tag { get; internal set; }
        public int Priority { get; internal set; }
        public object UserData { get; internal set; }
        public bool Done { get; set; }
        public virtual void Clear() { Done = false; }
    }

    public interface ITaskAgent<T> where T : TaskBase
    {
        T Task { get; }
        void Initialize();
        StartTaskStatus Start(T task);          // 只判断"能否开工"
        void Update(float dt);                  // 持续推进续传任务，完成时置 Task.Done=true
        void Reset();                           // 解除任务绑定，归还前清理
        void Shutdown();
    }

    public sealed class TaskPool<T> where T : TaskBase
    {
        private readonly Stack<ITaskAgent<T>> m_Free = new();
        private readonly LinkedList<ITaskAgent<T>> m_Working = new();
        private readonly LinkedList<T> m_Waiting = new();
        public bool Paused { get; set; }
        public int FreeAgentCount => m_Free.Count;

        public void AddAgent(ITaskAgent<T> agent)
        {
            if (agent == null) throw new ArgumentNullException(nameof(agent));
            agent.Initialize();
            m_Free.Push(agent);
        }

        // 优先级降序、同级 FIFO：从队尾向前找第一个不低于自己的，插其后
        public void AddTask(T task)
        {
            var current = m_Waiting.Last;
            while (current != null)
            {
                if (task.Priority <= current.Value.Priority) break;
                current = current.Previous;
            }
            if (current != null) m_Waiting.AddAfter(current, task);
            else m_Waiting.AddFirst(task);
        }

        public void Update(float dt)
        {
            if (Paused) return;
            ProcessRunning(dt);
            ProcessWaiting(dt);
        }

        private void ProcessRunning(float dt)
        {
            var current = m_Working.First;
            while (current != null)
            {
                var task = current.Value.Task;
                if (!task.Done)
                {
                    current.Value.Update(dt);     // 续传推进
                    current = current.Next;
                    continue;
                }
                var next = current.Next;           // 完成：回收代理 + 释放任务
                current.Value.Reset();
                m_Free.Push(current.Value);
                m_Working.Remove(current);
                // ReferencePool.Release(task)  → 精简版交给 GC
                current = next;
            }
        }

        private void ProcessWaiting(float dt)
        {
            var current = m_Waiting.First;
            while (current != null && FreeAgentCount > 0)
            {
                var agent = m_Free.Pop();
                var agentNode = m_Working.AddLast(agent);
                var task = current.Value;
                var next = current.Next;
                var status = agent.Start(task);

                // 维度1 代理归还：非 CanResume
                if (status is StartTaskStatus.Done or StartTaskStatus.HasToWait or StartTaskStatus.UnknownError)
                {
                    agent.Reset();
                    m_Free.Push(agent);
                    m_Working.Remove(agentNode);
                }
                // 维度2 任务出队：非 HasToWait
                if (status is StartTaskStatus.Done or StartTaskStatus.CanResume or StartTaskStatus.UnknownError)
                    m_Waiting.Remove(current);
                // 维度3 任务释放：已彻底结束（精简版省略，交 GC）
                // if (status is Done or UnknownError) ReferencePool.Release(task);

                current = next;
            }
        }

        public bool RemoveTask(int serialId)
        {
            foreach (var t in m_Waiting)
                if (t.SerialId == serialId) { m_Waiting.Remove(t); return true; }

            var cur = m_Working.First;
            while (cur != null)
            {
                var next = cur.Next;
                if (cur.Value.Task.SerialId == serialId)
                {
                    cur.Value.Reset();
                    m_Free.Push(cur.Value);
                    m_Working.Remove(cur);
                    return true;
                }
                cur = next;
            }
            return false;
        }

        public void Shutdown()
        {
            m_Waiting.Clear();
            foreach (var a in m_Working) { a.Reset(); m_Free.Push(a); }
            m_Working.Clear();
            while (m_Free.Count > 0) m_Free.Pop().Shutdown();
        }
    }
}
```

---

## 3. 使用示例（模拟下载代理）

```csharp
sealed class DownloadTask : TaskBase
{
    public string Url;
    public int TotalBytes, ReceivedBytes;
}

sealed class DownloadAgent : ITaskAgent<DownloadTask>
{
    public DownloadTask Task { get; private set; }
    public void Initialize() { }
    public StartTaskStatus Start(DownloadTask task)
    {
        Task = task;
        if (string.IsNullOrEmpty(task.Url)) return StartTaskStatus.UnknownError;
        return StartTaskStatus.CanResume;        // 开始下载，转入续传
    }
    public void Update(float dt)
    {
        Task.ReceivedBytes += 1024;              // 模拟每帧收 1KB
        if (Task.ReceivedBytes >= Task.TotalBytes) Task.Done = true;
    }
    public void Reset() => Task = null;
    public void Shutdown() => Task = null;
}

var pool = new TaskPool<DownloadTask>();
pool.AddAgent(new DownloadAgent());              // 1 个并发下载器
pool.AddAgent(new DownloadAgent());              // 2 个

pool.AddTask(new DownloadTask { SerialId = 1, Url = "a.zip", TotalBytes = 4096, Priority = 0 });
pool.AddTask(new DownloadTask { SerialId = 2, Url = "b.zip", TotalBytes = 2048, Priority = 5 }); // 高优先先派

for (int frame = 0; frame < 10; frame++) pool.Update(0.016f);  // 分帧推进直到完成
```

---

## 4. 取舍自检

- ✅ 保留：任务/代理分离、空闲栈 LIFO、优先级有序插入、双阶段轮询、4×3 决策矩阵、跨帧续传、Paused、RemoveTask 双扫。
- ❌ 砍掉：TaskBase 的 ReferencePool 复用（交 GC）、TaskInfo 快照导出、按 Tag 批量移除、详细计数。
- ⚠️ 提醒：`CanResume` 任务**不可释放**（代理仍持有它，下帧继续推进）；`HasToWait` 任务**不可出队**（留队列重试）。把这两条搞反是最常见的崩溃来源。

---

> 配套考题见 `03_TaskPool_考题.md`。
