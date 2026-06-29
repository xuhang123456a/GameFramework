# DataNode 数据结点 · Facade 精简仿写

> 目标：~120 行独立可编译，复刻"路径树 + 懒分配子列表 + 递归回收 + Get/Set 容错不对称"。
> 取舍：节点复用与 Variable 的 IReference 化简为普通对象 + IDisposable 风格递归清理；保留路径解析与递归释放精髓。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `DataNode : IReference` 复用 | `DataNode`(普通) + 递归 Clear | 简化（不接池，但保留递归释放语义） |
| 路径分隔 `. / \` | 同 | 保留 |
| `m_Childs` 懒分配 | 同 | **保留** |
| `SetData` 覆盖即 Release | 覆盖即递归 Dispose 旧值 | 保留语义 |
| Get 抛异常 / Set 建链 | 同 | **保留**（难点②） |
| `Variable` | `object` | 简化 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniDataNode
{
    public sealed class DataNode
    {
        private static readonly char[] Separators = { '.', '/', '\\' };

        public string Name { get; private set; }
        public DataNode Parent { get; private set; }
        public object Data { get; private set; }
        private List<DataNode> m_Childs;   // 懒分配
        public int ChildCount => m_Childs?.Count ?? 0;

        public string FullName => Parent == null ? Name : $"{Parent.FullName}.{Name}";

        private DataNode(string name, DataNode parent)
        {
            if (!IsValidName(name)) throw new ArgumentException($"Invalid node name '{name}'.");
            Name = name; Parent = parent;
        }

        public static DataNode CreateRoot() => new DataNode("<Root>", null);

        public DataNode GetChild(string name)
        {
            if (m_Childs == null) return null;
            foreach (var c in m_Childs) if (c.Name == name) return c;
            return null;
        }

        public DataNode GetOrAddChild(string name)
        {
            var node = GetChild(name);
            if (node != null) return node;
            node = new DataNode(name, this);
            (m_Childs ??= new List<DataNode>()).Add(node);
            return node;
        }

        public void SetData(object data)
        {
            if (Data is IDisposable d) d.Dispose();   // 覆盖旧值：释放（对应 ReferencePool.Release）
            Data = data;
        }

        public void RemoveChild(string name)
        {
            var node = GetChild(name);
            if (node == null) return;
            m_Childs.Remove(node);
            node.ReleaseRecursively();                // 递归回收整棵子树
        }

        // 深度优先：先清子树+数据，再清自身（对应 IReference.Clear 递归）
        public void ReleaseRecursively()
        {
            if (Data is IDisposable d) d.Dispose();
            Data = null;
            if (m_Childs != null)
            {
                foreach (var c in m_Childs) c.ReleaseRecursively();
                m_Childs.Clear();
            }
        }

        public override string ToString() => $"{FullName}: {(Data == null ? "<Null>" : $"[{Data.GetType().Name}] {Data}")}";

        private static bool IsValidName(string name)
        {
            if (string.IsNullOrEmpty(name)) return false;
            return name.IndexOfAny(Separators) < 0;
        }
    }

    public sealed class DataNodeManager
    {
        private static readonly char[] Separators = { '.', '/', '\\' };
        public DataNode Root { get; } = DataNode.CreateRoot();

        private static string[] Split(string path)
            => string.IsNullOrEmpty(path) ? Array.Empty<string>()
               : path.Split(Separators, StringSplitOptions.RemoveEmptyEntries);

        public DataNode GetNode(string path, DataNode start = null)
        {
            var cur = start ?? Root;
            foreach (var seg in Split(path))
            {
                cur = cur.GetChild(seg);
                if (cur == null) return null;   // 只读：缺失返回 null
            }
            return cur;
        }

        public DataNode GetOrAddNode(string path, DataNode start = null)
        {
            var cur = start ?? Root;
            foreach (var seg in Split(path)) cur = cur.GetOrAddChild(seg);  // 缺失即建链
            return cur;
        }

        // Set 容错：自动建链
        public void SetData(string path, object data) => GetOrAddNode(path).SetData(data);

        // Get 严格：缺失抛异常
        public T GetData<T>(string path)
        {
            var node = GetNode(path) ?? throw new InvalidOperationException($"Data node not exist: '{path}'.");
            return (T)node.Data;
        }

        public void RemoveNode(string path, DataNode start = null)
        {
            var cur = start ?? Root; DataNode parent = cur.Parent;
            foreach (var seg in Split(path)) { parent = cur; cur = cur.GetChild(seg); if (cur == null) return; }
            parent?.RemoveChild(cur.Name);
        }

        public void Clear() => Root.ReleaseRecursively();
    }
}
```

---

## 3. 使用示例

```csharp
var mgr = new DataNodeManager();

mgr.SetData("Player.Stats.Level", 12);        // 自动建 Player→Stats→Level 三级
mgr.SetData("Player.Stats.Exp", 3400);

int level = mgr.GetData<int>("Player.Stats.Level");   // 12
Console.WriteLine(mgr.GetNode("Player.Stats"));        // Stats: <Null>（中间节点无数据）

mgr.RemoveNode("Player");   // 整棵 Player 子树连数据递归回收

// mgr.GetData<int>("Player.Stats.Level");  // 抛 InvalidOperationException（Get 严格）
```

---

## 4. 取舍自检

- ✅ 保留：路径树、三分隔符解析、懒分配子列表、递归释放整树、SetData 覆盖即释放、Get 严格/Set 建链、名称约束、FullName 递归。
- ❌ 砍掉：DataNode 的 ReferencePool 复用（用普通对象 + IDisposable 模拟释放语义）、Variable 类型系统、按索引访问子节点。
- ⚠️ 提醒：`ReleaseRecursively` 必须深度优先——先递归子节点再清自身；颠倒会丢失子节点引用导致泄漏。节点名禁含分隔符这条约束必须与解析配套，否则"存得进取不出"。

---

> 配套考题见 `03_DataNode_考题.md`。
