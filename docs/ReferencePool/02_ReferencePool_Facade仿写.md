# ReferencePool 引用池 · Facade 精简仿写

> 目标：~120 行独立可编译，复刻"按类型分桶 + Clear 复用 + 配平统计 + 重复归还检测"。
> 取舍：保留双构造路径与严格检查；砍掉详细六维统计中冗余项，保留定位泄漏的关键计数。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `IReference.Clear()` | `IReference.Clear()` | 保留（核心契约） |
| `Dictionary<Type, ReferenceCollection>` | 同 | 保留（分桶本质） |
| 泛型 `Acquire<T>` + 反射 `Acquire(Type)` | 两条路径都留 | 保留（难点③） |
| `EnableStrictCheck` 重复归还检测 | `StrictCheck` | 保留（难点②防护） |
| 六维计数 | Using/Acquire/Release 三维 | 简化 |
| `lock` 双层 | 同 | 保留（线程安全本质） |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniRefPool
{
    public interface IReference
    {
        void Clear();   // 归还前自清理，复用正确性的根
    }

    public static class ReferencePool
    {
        public static bool StrictCheck { get; set; } = false;

        private static readonly Dictionary<Type, Bucket> s_Buckets = new Dictionary<Type, Bucket>();

        public static T Acquire<T>() where T : class, IReference, new()
            => GetBucket(typeof(T)).Acquire<T>();

        public static IReference Acquire(Type type)
        {
            CheckType(type);
            return GetBucket(type).Acquire();
        }

        public static void Release(IReference reference)
        {
            if (reference == null) throw new ArgumentNullException(nameof(reference));
            var type = reference.GetType();
            CheckType(type);
            GetBucket(type).Release(reference);
        }

        public static void Add<T>(int count) where T : class, IReference, new()
            => GetBucket(typeof(T)).Add<T>(count);

        public static void RemoveAll<T>() where T : class, IReference
            => GetBucket(typeof(T)).RemoveAll();

        public static (int unused, int using_, int acquire) Stat<T>()
        {
            var b = GetBucket(typeof(T));
            return (b.Unused, b.Using, b.Acquire);
        }

        private static Bucket GetBucket(Type type)
        {
            if (type == null) throw new ArgumentNullException(nameof(type));
            lock (s_Buckets)
            {
                if (!s_Buckets.TryGetValue(type, out var b))
                {
                    b = new Bucket(type);
                    s_Buckets.Add(type, b);
                }
                return b;
            }
        }

        private static void CheckType(Type type)
        {
            if (!StrictCheck) return;
            if (type == null) throw new ArgumentNullException(nameof(type));
            if (!type.IsClass || type.IsAbstract)
                throw new InvalidOperationException("Reference type must be non-abstract class.");
            if (!typeof(IReference).IsAssignableFrom(type))
                throw new InvalidOperationException($"'{type.FullName}' is not IReference.");
        }

        private sealed class Bucket
        {
            private readonly Queue<IReference> m_Queue = new Queue<IReference>();
            private readonly Type m_Type;
            public int Using { get; private set; }
            public int Acquire { get; private set; }
            public int Unused => m_Queue.Count;

            public Bucket(Type type) => m_Type = type;

            public T Acquire<T>() where T : class, IReference, new()
            {
                if (typeof(T) != m_Type) throw new InvalidOperationException("Type mismatch.");
                Using++; Acquire++;
                lock (m_Queue)
                    if (m_Queue.Count > 0) return (T)m_Queue.Dequeue();
                return new T();                                   // 池空：编译期路径，零反射
            }

            public IReference Acquire()
            {
                Using++; Acquire++;
                lock (m_Queue)
                    if (m_Queue.Count > 0) return m_Queue.Dequeue();
                return (IReference)Activator.CreateInstance(m_Type);  // 运行时路径，反射
            }

            public void Release(IReference r)
            {
                r.Clear();                                        // 先清理
                lock (m_Queue)
                {
                    if (StrictCheck && m_Queue.Contains(r))       // 重复归还检测
                        throw new InvalidOperationException("Reference already released.");
                    m_Queue.Enqueue(r);
                }
                Using--;
            }

            public void Add<T>(int count) where T : class, IReference, new()
            {
                lock (m_Queue)
                    while (count-- > 0) m_Queue.Enqueue(new T());
            }

            public void RemoveAll()
            {
                lock (m_Queue) m_Queue.Clear();
            }
        }
    }
}
```

---

## 3. 使用示例

```csharp
sealed class DamageEvent : IReference
{
    public int Attacker;
    public int Damage;
    public void Clear() { Attacker = 0; Damage = 0; }  // 必须清干净
}

// 预热 16 个
ReferencePool.Add<DamageEvent>(16);

var e = ReferencePool.Acquire<DamageEvent>();
e.Attacker = 1001; e.Damage = 50;
// ... 派发使用 ...
ReferencePool.Release(e);          // 自动 Clear 后回队列

var (unused, inUse, acquire) = ReferencePool.Stat<DamageEvent>();
// inUse 应为 0；acquire 累计取用次数；unused = 队列里可复用数量
```

---

## 4. 取舍自检

- ✅ 保留：分桶、双构造路径、Clear 复用、重复归还检测、锁、配平统计。
- ❌ 砍掉：六维全量计数（Add/Remove 计数）、`Remove(count)` 精确缩容、`ClearAll`/`GetAllReferencePoolInfos` 全局导出。
- ⚠️ 提醒：统计字段在锁外自增 → 高并发下非精确，仅供观测；不要拿它做业务判断。Acquire 与 Release 必须 1:1 配平，否则 `Using` 单调增即泄漏信号。

---

> 配套考题见 `03_ReferencePool_考题.md`。
