# Resource 资源/热更新 · Facade 精简仿写

> 目标：复刻不可能"全量"——本模块是底座集成者。这里聚焦**最能体现集成思想的两条主线**：① 三方比对的热更新决策；② 加载 = 对象池复用 + 依赖递归 + 引用计数卸载。约 180 行，复用前文 MiniObjectPool/MiniTaskPool/MiniDownload 的概念。

---

## 1. 设计映射表（哪些复刻、哪些指针引用）

| 原框架 | 精简版 | 处理 |
|--------|--------|------|
| ResourceMode 三模式 | 仅 Updatable | 聚焦 |
| ResourceChecker 三方比对 | `CheckResources` | **复刻**（热更核心） |
| ResourceUpdater + Download | `UpdateResources` 调 MiniDownload | 引用 Download 仿写 |
| ResourceLoader + ObjectPool | `LoadAsset` 调对象池 | **复刻**（加载核心） |
| 依赖递归加载 | `LoadWithDependencies` | **复刻** |
| 引用计数卸载 | Unspawn 回池 | **复刻** |
| 版本列表序列化/CRC/FileSystem 归档 | 简化为内存 VersionEntry | 砍 |
| 7 子系统 partial 切分 | 3 个聚焦类 | 简化 |

---

## 2. 核心代码（聚焦比对 + 加载两条主线）

```csharp
using System;
using System.Collections.Generic;

namespace MiniResource
{
    // ---- 版本信息（简化版本列表条目）----
    public sealed class VersionEntry
    {
        public string Name; public int Version; public int Length;
        public string[] Dependencies = Array.Empty<string>();
    }

    public enum CheckResult { UpToDate, UseReadOnly, NeedDownload, Disuse }

    // ============ 主线①：三方比对（热更新决策核心）============
    public sealed class ResourceChecker
    {
        // 三方存储区
        public Dictionary<string, VersionEntry> Remote = new();    // 远端清单(期望)
        public Dictionary<string, VersionEntry> ReadOnly = new();  // 只读区(随包)
        public Dictionary<string, VersionEntry> ReadWrite = new(); // 读写区(已下载)

        public Dictionary<string, CheckResult> CheckResources()
        {
            var result = new Dictionary<string, CheckResult>();
            // 1. 远端每个资源：读写>只读>下载 优先级
            foreach (var kv in Remote)
            {
                var name = kv.Key; var remote = kv.Value;
                if (ReadWrite.TryGetValue(name, out var rw) && rw.Version == remote.Version)
                    result[name] = CheckResult.UpToDate;        // 读写区已最新
                else if (ReadOnly.TryGetValue(name, out var ro) && ro.Version == remote.Version)
                    result[name] = CheckResult.UseReadOnly;     // 用包内的(免下载!)
                else
                    result[name] = CheckResult.NeedDownload;    // 三方不匹配才下载
            }
            // 2. 读写区有但远端已无 → 可移除
            foreach (var name in ReadWrite.Keys)
                if (!Remote.ContainsKey(name)) result[name] = CheckResult.Disuse;
            return result;
        }
    }

    // ============ 更新：把 NeedDownload 的交给 Download（引用 MiniDownload）============
    public sealed class ResourceUpdater
    {
        private readonly ResourceChecker m_Checker;
        public Action<string> OnResourceUpdated;   // 下完一个写入读写区 + 更新本地版本列表

        public ResourceUpdater(ResourceChecker checker) => m_Checker = checker;

        public List<string> CollectNeedDownload()
        {
            var list = new List<string>();
            foreach (var kv in m_Checker.CheckResources())
                if (kv.Value == CheckResult.NeedDownload) list.Add(kv.Key);
            return list;
            // 真实项目: 逐个 DownloadManager.AddDownload → OnDownloadSuccess 校验CRC
            //           → 写读写区(可入 FileSystem 归档) → 更新 LocalVersionList → 广播事件
        }

        // 下载成功回调：把远端条目"提升"为读写区条目
        public void OnDownloaded(string name)
        {
            m_Checker.ReadWrite[name] = m_Checker.Remote[name];
            OnResourceUpdated?.Invoke(name);
        }
    }

    // ============ 主线②：加载 = 对象池复用 + 依赖递归 + 引用计数 ============
    public sealed class ResourceLoader
    {
        private readonly Dictionary<string, VersionEntry> m_Available;  // 已就绪资源(只读∪读写)
        // 简化的"资源对象池"：name → (asset, 引用计数)
        private readonly Dictionary<string, (object asset, int refCount)> m_Pool = new();
        private readonly Func<string, object> m_RawLoader;   // 模拟从 AB 加载原始 asset

        public ResourceLoader(Dictionary<string, VersionEntry> available, Func<string, object> rawLoader)
        { m_Available = available; m_RawLoader = rawLoader; }

        public object LoadAsset(string name)
        {
            if (!m_Available.ContainsKey(name))
                throw new InvalidOperationException($"Asset '{name}' not ready (need download/update).");

            return LoadWithDependencies(name, new HashSet<string>());
        }

        // 依赖递归 + 对象池命中复用 + 引用计数
        private object LoadWithDependencies(string name, HashSet<string> visiting)
        {
            if (!visiting.Add(name))
                throw new InvalidOperationException($"Circular dependency at '{name}'.");

            var entry = m_Available[name];
            foreach (var dep in entry.Dependencies)            // 先加载依赖(深度优先)
                LoadWithDependencies(dep, visiting);

            if (m_Pool.TryGetValue(name, out var cached))      // 对象池命中 → 复用 + 计数++
            {
                m_Pool[name] = (cached.asset, cached.refCount + 1);   // Spawn 语义
                visiting.Remove(name);
                return cached.asset;
            }

            var asset = m_RawLoader(name);                     // 未命中 → 真正加载 AB/asset
            m_Pool[name] = (asset, 1);                          // 入池, refCount=1
            visiting.Remove(name);
            return asset;
        }

        // 卸载 = Unspawn(计数--)，不立即销毁；归零才可被池回收
        public void UnloadAsset(string name)
        {
            if (!m_Pool.TryGetValue(name, out var c)) return;
            int rc = c.refCount - 1;
            if (rc <= 0)
                m_Pool.Remove(name);   // 真实项目: Unspawn 回 ObjectPool, 由容量/过期策略决定何时 Release AB
            else
                m_Pool[name] = (c.asset, rc);
        }

        public int RefCountOf(string name) => m_Pool.TryGetValue(name, out var c) ? c.refCount : 0;
    }
}
```

---

## 3. 使用示例

```csharp
// 构造三方存储
var checker = new ResourceChecker();
checker.Remote["hero.ab"]   = new VersionEntry { Name = "hero.ab",   Version = 2, Dependencies = new[] { "shared.ab" } };
checker.Remote["shared.ab"] = new VersionEntry { Name = "shared.ab", Version = 1 };
checker.ReadOnly["shared.ab"] = new VersionEntry { Name = "shared.ab", Version = 1 };  // 包内已有
checker.ReadWrite["hero.ab"]  = new VersionEntry { Name = "hero.ab",  Version = 1 };   // 旧版本

// 比对
foreach (var kv in checker.CheckResources())
    Console.WriteLine($"{kv.Key}: {kv.Value}");
// hero.ab: NeedDownload  (读写区是v1, 远端v2)
// shared.ab: UseReadOnly (包内v1==远端v1, 免下载!)

// 更新
var updater = new ResourceUpdater(checker);
foreach (var n in updater.CollectNeedDownload()) updater.OnDownloaded(n);  // 模拟下完

// 加载（依赖递归 + 引用计数）
var available = new Dictionary<string, VersionEntry>(checker.ReadWrite);
foreach (var kv in checker.ReadOnly) available[kv.Key] = kv.Value;
var loader = new ResourceLoader(available, name => $"[Asset:{name}]");

var hero = loader.LoadAsset("hero.ab");   // 先加载依赖 shared.ab，再 hero.ab
Console.WriteLine(loader.RefCountOf("shared.ab"));  // 1
loader.LoadAsset("hero.ab");                          // 再次加载 → 命中复用
Console.WriteLine(loader.RefCountOf("hero.ab"));     // 2
loader.UnloadAsset("hero.ab");                        // 计数-- → 1（不销毁）
```

---

## 4. 取舍自检

- ✅ 保留：三方比对（读写>只读>下载优先级 + Disuse 标记）、需下载收集、依赖递归加载、对象池命中复用、引用计数卸载、循环依赖检测。
- ❌ 砍掉：版本列表序列化/CRC 校验/FileSystem 归档、Download 真实续传、LoadType 解密、变体(variant)、资源分组、场景/二进制加载、7 子系统完整切分。
- ⚠️ 提醒：这是**集成模块**，仿写的价值在于看清"它如何编排底座"而非复刻细节。三方比对漏掉只读区会浪费流量；卸载用 Destroy 而非引用计数会导致共享依赖被误删（shared.ab 被多个资源依赖时）。

---

> 配套考题见 `03_Resource_考题.md`。这是检验"是否真懂整个框架"的总考点。
