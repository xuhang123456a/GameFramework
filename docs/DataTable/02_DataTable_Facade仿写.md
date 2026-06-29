# DataTable 数据表 · Facade 精简仿写

> 目标：~150 行独立可编译，复刻"行自解析 + id 字典存储 + Min/Max 增量维护 + 管线/策略/存储三层解耦"。
> 取舍：Resource 异步加载简化为同步"读文本"；保留 `IDataProviderHelper` 策略注入与三层职责分离这一核心架构。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `IDataRow.ParseDataRow` | 同 | **保留**（行自解析） |
| `DataProvider<T>` 多源加载 | `DataProvider<T>` 仅"读文本" | 简化（去异步/Resource） |
| `IDataProviderHelper<T>` 策略 | 同 | **保留**（解析注入点） |
| `Dictionary<int,T>` + Min/Max | 同 | **保留** |
| 四个 ReadData 事件 + ReferencePool | 单个成功/失败回调 | 简化 |
| `IEnumerable<T>` | 同 | 保留 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections;
using System.Collections.Generic;

namespace MiniDataTable
{
    public interface IDataRow
    {
        int Id { get; }
        bool ParseDataRow(string line);     // 行自解析（自描述反序列化）
    }

    // 解析策略：把"整表文本"拆成行并喂给表（依赖注入点）
    public interface IDataProviderHelper<T>
    {
        bool ParseData(IDataTable<T> table, string text);
    }

    public interface IDataTable<T> : IEnumerable<T> where T : class, IDataRow, new()
    {
        int Count { get; }
        T this[int id] { get; }
        T MinIdDataRow { get; }
        T MaxIdDataRow { get; }
        bool HasDataRow(int id);
        T GetDataRow(int id);
        bool AddDataRow(string line);
        bool RemoveDataRow(int id);
    }

    // 加载管线：只负责"从哪读"，解析委托给 Helper（这里简化为同步读文本）
    public sealed class DataProvider<T> where T : class, IDataRow, new()
    {
        private readonly IDataTable<T> m_Table;
        private IDataProviderHelper<T> m_Helper;
        public event Action<string> OnReadSuccess;
        public event Action<string, string> OnReadFailure;   // (name, error)

        public DataProvider(IDataTable<T> table) => m_Table = table;
        public void SetHelper(IDataProviderHelper<T> helper) => m_Helper = helper;

        public void ReadData(string name, Func<string, string> loader)
        {
            if (m_Helper == null) throw new InvalidOperationException("Set helper first.");
            try
            {
                string text = loader(name);                  // 模拟 Resource 加载
                if (!m_Helper.ParseData(m_Table, text))
                    throw new InvalidOperationException("Parse failed.");
                OnReadSuccess?.Invoke(name);
            }
            catch (Exception e)
            {
                if (OnReadFailure != null) OnReadFailure(name, e.ToString());
                else throw;
            }
        }
    }

    public sealed class DataTable<T> : IDataTable<T> where T : class, IDataRow, new()
    {
        private readonly Dictionary<int, T> m_Set = new();
        private readonly DataProvider<T> m_Provider;
        public T MinIdDataRow { get; private set; }
        public T MaxIdDataRow { get; private set; }
        public int Count => m_Set.Count;
        public T this[int id] => GetDataRow(id);

        public DataTable() => m_Provider = new DataProvider<T>(this);
        public void SetHelper(IDataProviderHelper<T> h) => m_Provider.SetHelper(h);
        public void ReadData(string name, Func<string, string> loader) => m_Provider.ReadData(name, loader);

        public bool HasDataRow(int id) => m_Set.ContainsKey(id);
        public T GetDataRow(int id) => m_Set.TryGetValue(id, out var r) ? r : null;

        public bool AddDataRow(string line)
        {
            var row = new T();                  // 表只负责 new + 调解析，逻辑下沉到行
            if (!row.ParseDataRow(line)) return false;
            if (m_Set.ContainsKey(row.Id))
                throw new InvalidOperationException($"Duplicate id {row.Id}.");
            m_Set.Add(row.Id, row);
            if (MinIdDataRow == null || MinIdDataRow.Id > row.Id) MinIdDataRow = row;  // O(1) 增量
            if (MaxIdDataRow == null || MaxIdDataRow.Id < row.Id) MaxIdDataRow = row;
            return true;
        }

        public bool RemoveDataRow(int id)
        {
            if (!m_Set.Remove(id)) return false;
            // 删到边界行才全表重算（O(n)），常态删非边界行 O(1)
            if ((MinIdDataRow != null && MinIdDataRow.Id == id) ||
                (MaxIdDataRow != null && MaxIdDataRow.Id == id))
            {
                MinIdDataRow = MaxIdDataRow = null;
                foreach (var kv in m_Set)
                {
                    if (MinIdDataRow == null || MinIdDataRow.Id > kv.Key) MinIdDataRow = kv.Value;
                    if (MaxIdDataRow == null || MaxIdDataRow.Id < kv.Key) MaxIdDataRow = kv.Value;
                }
            }
            return true;
        }

        public IEnumerator<T> GetEnumerator() => m_Set.Values.GetEnumerator();
        IEnumerator IEnumerable.GetEnumerator() => m_Set.Values.GetEnumerator();
    }
}
```

---

## 3. 使用示例

```csharp
// 业务行类（实际项目由 Excel 代码生成）
sealed class HeroRow : IDataRow
{
    public int Id { get; private set; }
    public string Name; public int Atk;
    public bool ParseDataRow(string line)
    {
        var f = line.Split('\t');
        Id = int.Parse(f[0]); Name = f[1]; Atk = int.Parse(f[2]);
        return true;
    }
}

// 解析策略：按行切分喂给表
sealed class TsvHelper<T> : IDataProviderHelper<T> where T : class, IDataRow, new()
{
    public bool ParseData(IDataTable<T> table, string text)
    {
        foreach (var line in text.Split('\n'))
            if (!string.IsNullOrWhiteSpace(line)) table.AddDataRow(line.Trim());
        return true;
    }
}

var table = new DataTable<HeroRow>();
table.SetHelper(new TsvHelper<HeroRow>());
table.ReadData("Hero", name => "1\t战士\t10\n2\t法师\t8\n3\t射手\t12");  // 模拟加载

Console.WriteLine(table[2].Name);                 // 法师
Console.WriteLine(table.MaxIdDataRow.Name);        // 射手(id=3)
foreach (var h in table) Console.WriteLine(h.Name);
```

---

## 4. 取舍自检

- ✅ 保留：行自解析、id 字典、Min/Max 增量维护 + 删边界重算、管线/策略/存储三层解耦、成功/失败回调、IEnumerable。
- ❌ 砍掉：Resource 多源异步加载（HasAssetResult 四分支）、静态字节缓存、进度/依赖事件、ReferencePool 化的 EventArgs、二进制解析路径。
- ⚠️ 提醒：行对象**不要做对象池复用**——配置是只读常驻数据，复用无收益且易串改。Helper 决定解析格式，换格式只换 Helper 不动表，这是分层的回报。

---

> 配套考题见 `03_DataTable_考题.md`。
