# ObjectPool 对象池 · Facade 精简仿写

> 目标：脱离 GameFramework，用 ~200 行独立可编译的 C# 复刻对象池的**核心不变量**——引用计数、软容量、过期清理、可插拔释放策略。
> 取舍：砍掉泛型管理器的几十个重载、TypeNamePair 复合键、MultiDictionary 链表区间；保留三层结构与三个动词的精确语义。

---

## 1. 设计映射表（原框架 → 精简版）

| 原框架 | 精简版 | 是否保留 | 说明 |
|--------|--------|----------|------|
| `IObjectPoolManager` (40+ 重载) | `SimplePoolManager` | 简化 | 只留 `CreatePool/GetPool/Update/Shutdown` |
| `IObjectPool<T>` + `ObjectPoolBase` 双契约 | `SimpleObjectPool<T>` 单类 | 合并 | 单进程无需非泛型统一存储时可合并 |
| `ObjectBase`(IReference) | `PooledObject` 抽象类 | 保留 | 仍是业务对象基类，承载 Target |
| `Object<T>` 内部包装器 | `Entry` 内部类 | 保留 | **引用计数的真正载体** |
| `m_Objects`(按 name) + `m_ObjectMap`(按 target) | 仅 `Dictionary<object, Entry>` + `List<Entry>` | 简化 | 不做按名分组，按需可加回 |
| `ReleaseObjectFilterCallback<T>` | `ReleaseFilter` 委托 | 保留 | 策略注入点，核心难点之一 |
| `ReferencePool` 二级复用 | 不复刻 | 砍掉 | 标注为难点③，仿写者需自知 GC 代价 |

---

## 2. 核心代码（独立可编译，无第三方依赖）

```csharp
using System;
using System.Collections.Generic;

namespace MiniPool
{
    /// <summary>业务对象基类：复刻 ObjectBase 的状态字段与三个钩子。</summary>
    public abstract class PooledObject
    {
        public string Name { get; private set; }
        public object Target { get; private set; }
        public bool Locked { get; set; }
        public int Priority { get; set; }
        public DateTime LastUseTime { get; internal set; }
        /// <summary>业务可重写：返回 false 表示当前禁止被释放。</summary>
        public virtual bool CustomCanReleaseFlag => true;

        protected void Initialize(string name, object target)
        {
            if (target == null) throw new ArgumentNullException(nameof(target));
            Name = name ?? string.Empty;
            Target = target;
            Locked = false;
            Priority = 0;
            LastUseTime = DateTime.UtcNow;
        }

        protected internal virtual void OnSpawn() { }
        protected internal virtual void OnUnspawn() { }
        /// <param name="isShutdown">true=池关闭触发，false=正常逐出。</param>
        protected internal abstract void Release(bool isShutdown);
    }

    /// <summary>释放筛选策略：从候选集中挑出要真正逐出的对象。</summary>
    public delegate List<T> ReleaseFilter<T>(List<T> candidates, int toReleaseCount, DateTime expireTime)
        where T : PooledObject;

    public sealed class SimpleObjectPool<T> where T : PooledObject
    {
        // 内部包装器：引用计数的唯一载体（复刻 Object<T>）
        private sealed class Entry
        {
            public T Obj;
            public int SpawnCount;
            public bool IsInUse => SpawnCount > 0;

            public T Spawn()
            {
                SpawnCount++;
                Obj.LastUseTime = DateTime.UtcNow;
                Obj.OnSpawn();
                return Obj;
            }
            public void Unspawn()
            {
                Obj.OnUnspawn();
                Obj.LastUseTime = DateTime.UtcNow;
                if (--SpawnCount < 0)
                    throw new InvalidOperationException($"Object '{Obj.Name}' spawn count < 0.");
            }
        }

        private readonly Dictionary<object, Entry> m_Map = new Dictionary<object, Entry>();
        private readonly List<Entry> m_All = new List<Entry>();
        private readonly List<T> m_Candidates = new List<T>();
        private readonly List<T> m_ToRelease = new List<T>();
        private readonly bool m_AllowMultiSpawn;
        private readonly ReleaseFilter<T> m_Filter;

        public int Count => m_Map.Count;
        public int Capacity { get; set; }
        public float ExpireTime { get; set; }            // 秒；float.MaxValue = 永不过期
        public float AutoReleaseInterval { get; set; }
        private float m_AutoReleaseTimer;

        public SimpleObjectPool(bool allowMultiSpawn, int capacity, float expireTime,
                                float autoReleaseInterval, ReleaseFilter<T> filter = null)
        {
            m_AllowMultiSpawn = allowMultiSpawn;
            Capacity = capacity;
            ExpireTime = expireTime;
            AutoReleaseInterval = autoReleaseInterval;
            m_Filter = filter ?? DefaultFilter;
        }

        // ---- 三个动词 ----
        public void Register(T obj, bool spawned)
        {
            if (obj == null) throw new ArgumentNullException(nameof(obj));
            var e = new Entry { Obj = obj, SpawnCount = spawned ? 1 : 0 };
            if (spawned) obj.OnSpawn();
            m_Map.Add(obj.Target, e);
            m_All.Add(e);
            if (Count > Capacity) Release();          // 入池即尝试瘦身
        }

        public T Spawn()
        {
            foreach (var e in m_All)
                if (m_AllowMultiSpawn || !e.IsInUse)
                    return e.Spawn();
            return null;                               // 取不到由调用方决定是否新建并 Register
        }

        public void Unspawn(object target)
        {
            if (!m_Map.TryGetValue(target, out var e))
                throw new InvalidOperationException("Target not in pool.");
            e.Unspawn();
            if (Count > Capacity && e.SpawnCount <= 0) Release();
        }

        public bool ReleaseObject(T obj)
        {
            if (obj == null || !m_Map.TryGetValue(obj.Target, out var e)) return false;
            if (e.IsInUse || e.Obj.Locked || !e.Obj.CustomCanReleaseFlag) return false;
            m_Map.Remove(obj.Target);
            m_All.Remove(e);
            e.Obj.Release(false);                      // 真正销毁；这里没有二级 ReferencePool
            return true;
        }

        // ---- 容量/过期驱动的批量释放 ----
        public void Release() => Release(Count - Capacity);

        public void Release(int toReleaseCount)
        {
            if (toReleaseCount < 0) toReleaseCount = 0;
            DateTime expire = ExpireTime < float.MaxValue
                ? DateTime.UtcNow.AddSeconds(-ExpireTime) : DateTime.MinValue;

            m_AutoReleaseTimer = 0f;
            CollectCandidates();
            var picks = m_Filter(m_Candidates, toReleaseCount, expire);
            if (picks == null) return;
            foreach (var o in picks) ReleaseObject(o);
        }

        public void Update(float realElapseSeconds)
        {
            m_AutoReleaseTimer += realElapseSeconds;
            if (m_AutoReleaseTimer < AutoReleaseInterval) return;
            Release();
        }

        public void Shutdown()
        {
            foreach (var e in m_All) e.Obj.Release(true);   // isShutdown=true
            m_Map.Clear(); m_All.Clear();
        }

        // 候选集 = 非使用中 & 未加锁 & 自定义允许
        private void CollectCandidates()
        {
            m_Candidates.Clear();
            foreach (var e in m_All)
            {
                if (e.IsInUse || e.Obj.Locked || !e.Obj.CustomCanReleaseFlag) continue;
                m_Candidates.Add(e.Obj);
            }
        }

        // 默认策略：过期优先，其次按 Priority/LastUseTime 升序补足配额
        private List<T> DefaultFilter(List<T> candidates, int toReleaseCount, DateTime expireTime)
        {
            m_ToRelease.Clear();
            if (expireTime > DateTime.MinValue)
            {
                for (int i = candidates.Count - 1; i >= 0; i--)
                    if (candidates[i].LastUseTime <= expireTime)
                    {
                        m_ToRelease.Add(candidates[i]);
                        candidates.RemoveAt(i);
                    }
                toReleaseCount -= m_ToRelease.Count;
            }
            for (int i = 0; toReleaseCount > 0 && i < candidates.Count; i++)
            {
                for (int j = i + 1; j < candidates.Count; j++)
                    if (candidates[i].Priority > candidates[j].Priority ||
                        (candidates[i].Priority == candidates[j].Priority &&
                         candidates[i].LastUseTime > candidates[j].LastUseTime))
                        (candidates[i], candidates[j]) = (candidates[j], candidates[i]);
                m_ToRelease.Add(candidates[i]);
                toReleaseCount--;
            }
            return m_ToRelease;
        }
    }
}
```

---

## 3. 使用示例

```csharp
// 业务对象
sealed class Bullet : PooledObject
{
    public static Bullet Create(string name, object real)
    {
        var b = new Bullet();
        b.Initialize(name, real);
        return b;
    }
    protected internal override void OnSpawn()   => Console.WriteLine("子弹激活");
    protected internal override void OnUnspawn() => Console.WriteLine("子弹回收");
    protected internal override void Release(bool isShutdown)
        => Console.WriteLine($"子弹销毁(shutdown={isShutdown})");
}

// 容量2、永不过期、每5秒自动释放
var pool = new SimpleObjectPool<Bullet>(allowMultiSpawn: false, capacity: 2,
                                        expireTime: float.MaxValue, autoReleaseInterval: 5f);

var go = new object();
pool.Register(Bullet.Create("普通子弹", go), spawned: false);

var b = pool.Spawn();        // SpawnCount 0->1, OnSpawn
pool.Unspawn(b.Target);      // SpawnCount 1->0, OnUnspawn
pool.Release();              // 超容量时按策略逐出 + Release(false)
```

---

## 4. 刻意保留 vs 刻意砍掉（自检清单）

- ✅ 保留：`SpawnCount` 引用计数、`IsInUse` 不变量、软容量、过期+优先级双阶段筛选、`isShutdown` 区分、`Locked`/`CustomCanReleaseFlag` 否决权。
- ❌ 砍掉：按名分组的 MultiDictionary（改全表扫描 Spawn，O(n)）、非泛型 `ObjectPoolBase` 统一存储、`ReferencePool` 二级复用、`TypeNamePair` 复合键、`Activator.CreateInstance` 的运行时构造。
- ⚠️ 代价提醒：`m_All.Remove(e)` 是 O(n)；高频释放场景需换成交换删除或链表。这正是原框架用 MultiDictionary + 双索引的原因。

---

> 配套考题见 `03_ObjectPool_考题.md`——先合上本文，凭状态机图默写三个动词的计数变化与释放筛选两阶段。
