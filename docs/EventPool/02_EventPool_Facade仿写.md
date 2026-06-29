# EventPool 事件池 · Facade 精简仿写

> 目标：~160 行独立可编译，复刻"延迟队列 + 立即分发 + 迭代安全增删 + 模式校验 + 参数复用"。
> 取舍：用 `LinkedList` 替代框架的 MultiDictionary 链表区间；保留 `m_CachedNodes` 重入安全游标这一精髓。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `GameFrameworkMultiDictionary<int, EventHandler<T>>` | `Dictionary<int, LinkedList<Handler>>` | 等价替换 |
| `Fire`(线程安全延迟) / `FireNow`(立即) | 同名两法 | 保留（核心双模） |
| `m_CachedNodes` 重入游标 | 同 | **保留**（难点①） |
| `Event` 结点 IReference 复用 | 简化为直接入队 tuple | 简化 |
| EventArgs 走 ReferencePool | 复用接口 `IPoolableArgs.Clear` | 保留语义 |
| `EventPoolMode` Flags | 同 | 保留 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniEvent
{
    [Flags]
    public enum EventPoolMode : byte
    {
        Default = 0, AllowNoHandler = 1, AllowMultiHandler = 2, AllowDuplicateHandler = 4
    }

    public abstract class BaseEventArgs : EventArgs
    {
        public abstract int Id { get; }
        public virtual void Clear() { }   // 复用前清理
    }

    public sealed class EventPool<T> where T : BaseEventArgs
    {
        public delegate void Handler(object sender, T e);

        private readonly Dictionary<int, LinkedList<Handler>> m_Handlers = new();
        private readonly Queue<(object sender, T args)> m_Events = new();
        // 以事件实例为 key 的重入游标：记录"下一个待回调节点"
        private readonly Dictionary<T, LinkedListNode<Handler>> m_Cached = new();
        private readonly EventPoolMode m_Mode;
        private Handler m_DefaultHandler;

        public EventPool(EventPoolMode mode) => m_Mode = mode;

        private bool Has(EventPoolMode f) => (m_Mode & f) == f;

        public void Subscribe(int id, Handler handler)
        {
            if (handler == null) throw new ArgumentNullException(nameof(handler));
            if (!m_Handlers.TryGetValue(id, out var list))
            {
                list = new LinkedList<Handler>();
                m_Handlers[id] = list;
                list.AddLast(handler);
                return;
            }
            if (list.Count > 0 && !Has(EventPoolMode.AllowMultiHandler))
                throw new InvalidOperationException($"Event '{id}' not allow multi handler.");
            if (!Has(EventPoolMode.AllowDuplicateHandler) && list.Contains(handler))
                throw new InvalidOperationException($"Event '{id}' not allow duplicate handler.");
            list.AddLast(handler);
        }

        public void Unsubscribe(int id, Handler handler)
        {
            if (!m_Handlers.TryGetValue(id, out var list)) return;
            var node = list.Find(handler);
            if (node == null) return;

            // 若该节点正是某个重入游标，前移游标到 Next，避免迭代失效
            foreach (var key in new List<T>(m_Cached.Keys))
                if (m_Cached[key] == node)
                    m_Cached[key] = node.Next;

            list.Remove(node);
        }

        public void SetDefaultHandler(Handler handler) => m_DefaultHandler = handler;

        /// <summary>延迟：线程安全入队，Update 时主线程分发。</summary>
        public void Fire(object sender, T e)
        {
            if (e == null) throw new ArgumentNullException(nameof(e));
            lock (m_Events) m_Events.Enqueue((sender, e));
        }

        /// <summary>立即：当场同步分发，非线程安全。</summary>
        public void FireNow(object sender, T e)
        {
            if (e == null) throw new ArgumentNullException(nameof(e));
            HandleEvent(sender, e);
        }

        public void Update()
        {
            lock (m_Events)
            {
                while (m_Events.Count > 0)
                {
                    var (sender, args) = m_Events.Dequeue();
                    HandleEvent(sender, args);
                }
            }
        }

        private void HandleEvent(object sender, T e)
        {
            bool noHandler = false;
            if (m_Handlers.TryGetValue(e.Id, out var list) && list.Count > 0)
            {
                var current = list.First;
                while (current != null)
                {
                    m_Cached[e] = current.Next;          // 先存 Next（重入/删改安全）
                    current.Value(sender, e);            // 回调，可能改链表
                    current = m_Cached[e];               // 从游标读下一个
                }
                m_Cached.Remove(e);
            }
            else if (m_DefaultHandler != null)
            {
                m_DefaultHandler(sender, e);
            }
            else if (!Has(EventPoolMode.AllowNoHandler))
            {
                noHandler = true;
            }

            e.Clear();                                   // 归还前清理（此处简化为直接清，不接二级池）
            if (noHandler)
                throw new InvalidOperationException($"Event '{e.Id}' not allow no handler.");
        }

        public void Clear() { lock (m_Events) m_Events.Clear(); }
        public void Shutdown() { Clear(); m_Handlers.Clear(); m_Cached.Clear(); m_DefaultHandler = null; }
    }
}
```

---

## 3. 使用示例

```csharp
sealed class HpChangedArgs : BaseEventArgs
{
    public const int EventId = 1001;
    public override int Id => EventId;
    public int NewHp;
    public override void Clear() { NewHp = 0; }
}

var pool = new EventPool<BaseEventArgs>(EventPoolMode.AllowMultiHandler | EventPoolMode.AllowNoHandler);

void OnHp(object s, BaseEventArgs e) => Console.WriteLine($"HP={((HpChangedArgs)e).NewHp}");
pool.Subscribe(HpChangedArgs.EventId, OnHp);

// 跨线程抛
var args = new HpChangedArgs { NewHp = 80 };
pool.Fire(this, args);   // 入队，下一帧分发

// 主线程帧循环
pool.Update();           // 此时 OnHp 被回调；Fire 后不应再持有 args
```

---

## 4. 取舍自检

- ✅ 保留：延迟/立即双模、线程安全入队、`m_Cached` 重入游标、Unsubscribe 游标修正、模式 Flags 校验、Clear 复用语义。
- ❌ 砍掉：`Event` 结点的 ReferencePool 二级复用（直接用 tuple 入队，会有装箱/分配）、MultiDictionary 链表区间结构、详细计数。
- ⚠️ 提醒：`Fire` 之后业务**禁止再读写 args**（已移交，正式实现里会被回收复用）。`Unsubscribe` 里遍历 `m_Cached.Keys` 我复制了一份避免迭代中改字典——这是必须的。

---

> 配套考题见 `03_EventPool_考题.md`。
