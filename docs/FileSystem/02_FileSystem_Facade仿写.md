# FileSystem 虚拟文件系统 · Facade 精简仿写

> 目标：~160 行，复刻"单大文件归档 + 簇对齐分配 + 空闲块最佳匹配复用 + 元数据/数据分区"。
> 取舍：用内存 + 单 FileStream 演示；砍掉文件名加密与 Marshal 序列化（用简单二进制），保留分配器与碎片复用精髓。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| HeaderData(GFF 魔数/版本/上限) | 简化头 | 保留布局思路 |
| 簇 4KB 对齐分配 | 同 | **保留**（分配粒度核心） |
| `m_FreeBlockIndexes` 按长度最佳匹配 | 同 | **保留**（碎片复用核心） |
| BlockData(stringIndex/cluster/length) | `Block` | 保留 |
| StringData XOR 加密文件名 | 明文 | 简化 |
| 删除标记空闲不收缩 | 同 | **保留**（惰性碎片管理） |
| Marshal 结构体序列化 | BinaryWriter | 简化 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Text;

namespace MiniFileSystem
{
    // 块记录：一个文件占一个块
    public struct Block
    {
        public string Name;          // -1 等价：Name==null 表示空闲块
        public long ClusterOffset;   // 簇数据区内偏移(簇对齐)
        public int Length;           // 实际字节长度
        public int ClusterCount;     // 占用簇数(向上取整)
        public bool Using => Name != null;
    }

    public sealed class MiniFileSystem : IDisposable
    {
        private const int ClusterSize = 4096;             // 4KB 簇
        private static readonly byte[] Magic = { (byte)'G', (byte)'F', (byte)'F' };

        private readonly FileStream m_Stream;
        private readonly Dictionary<string, int> m_FileToBlock = new(StringComparer.Ordinal);
        private readonly List<Block> m_Blocks = new();
        // 空闲块: 簇数 → 空闲块索引列表 (最佳匹配复用)
        private readonly SortedDictionary<int, List<int>> m_FreeBlocks = new();
        private long m_DataAreaStart;                     // 簇数据区起点
        private long m_DataAreaEnd;                        // 已用到的末尾

        public int FileCount => m_FileToBlock.Count;

        public MiniFileSystem(string path, bool create)
        {
            if (create)
            {
                m_Stream = new FileStream(path, FileMode.Create, FileAccess.ReadWrite);
                m_Stream.Write(Magic, 0, Magic.Length);    // 写魔数头
                m_DataAreaStart = m_DataAreaEnd = 4096;     // 头+元数据区预留 4KB(简化)
            }
            else
            {
                m_Stream = new FileStream(path, FileMode.Open, FileAccess.ReadWrite);
                // 真实项目此处读头校验 GFF + 重建 m_Blocks/m_FileToBlock/m_FreeBlocks
                m_DataAreaStart = m_DataAreaEnd = 4096;
            }
        }

        public bool HasFile(string name) => m_FileToBlock.ContainsKey(name);

        public bool WriteFile(string name, byte[] data)
        {
            int needClusters = (data.Length + ClusterSize - 1) / ClusterSize;  // 向上取整到簇

            if (m_FileToBlock.TryGetValue(name, out var oldIdx))   // 已存在：先释放旧块
                FreeBlock(oldIdx);

            int blockIdx = AllocBlock(needClusters);               // 找块(复用空闲 or 新增)
            var b = m_Blocks[blockIdx];
            b.Name = name; b.Length = data.Length;
            m_Blocks[blockIdx] = b;

            m_Stream.Seek(b.ClusterOffset, SeekOrigin.Begin);      // 写簇数据
            m_Stream.Write(data, 0, data.Length);
            m_FileToBlock[name] = blockIdx;
            return true;
        }

        public byte[] ReadFile(string name)
        {
            if (!m_FileToBlock.TryGetValue(name, out var idx)) return null;
            var b = m_Blocks[idx];
            var buf = new byte[b.Length];
            m_Stream.Seek(b.ClusterOffset, SeekOrigin.Begin);
            m_Stream.Read(buf, 0, b.Length);
            return buf;
        }

        // 读片段：演示 Resource 从大归档按偏移取资源包片段
        public byte[] ReadFileSegment(string name, int offset, int length)
        {
            if (!m_FileToBlock.TryGetValue(name, out var idx)) return null;
            var b = m_Blocks[idx];
            var buf = new byte[length];
            m_Stream.Seek(b.ClusterOffset + offset, SeekOrigin.Begin);
            m_Stream.Read(buf, 0, length);
            return buf;
        }

        public bool DeleteFile(string name)
        {
            if (!m_FileToBlock.TryGetValue(name, out var idx)) return false;
            FreeBlock(idx);                                        // 标记空闲不收缩物理文件
            m_FileToBlock.Remove(name);
            return true;
        }

        public bool RenameFile(string oldName, string newName)
        {
            if (!m_FileToBlock.TryGetValue(oldName, out var idx) || m_FileToBlock.ContainsKey(newName))
                return false;
            var b = m_Blocks[idx]; b.Name = newName; m_Blocks[idx] = b;
            m_FileToBlock.Remove(oldName);
            m_FileToBlock[newName] = idx;
            return true;                                           // 只改元数据,簇数据不动
        }

        // ---- 分配器核心 ----
        private int AllocBlock(int needClusters)
        {
            // 最佳匹配：找 >= needClusters 的最小空闲块
            foreach (var kv in m_FreeBlocks)
            {
                if (kv.Key >= needClusters && kv.Value.Count > 0)
                {
                    int idx = kv.Value[^1];
                    kv.Value.RemoveAt(kv.Value.Count - 1);
                    return idx;                                    // 复用空闲块
                }
            }
            // 无合适空闲块：在数据区末尾新增
            var b = new Block
            {
                ClusterOffset = m_DataAreaEnd,
                ClusterCount = needClusters
            };
            m_DataAreaEnd += (long)needClusters * ClusterSize;
            m_Blocks.Add(b);
            return m_Blocks.Count - 1;
        }

        private void FreeBlock(int idx)
        {
            var b = m_Blocks[idx];
            b.Name = null;                                         // 标记空闲
            m_Blocks[idx] = b;
            if (!m_FreeBlocks.TryGetValue(b.ClusterCount, out var list))
                m_FreeBlocks[b.ClusterCount] = list = new List<int>();
            list.Add(idx);                                         // 按簇数入空闲表待复用
        }

        public void Dispose() => m_Stream?.Dispose();
    }
}
```

---

## 3. 使用示例

```csharp
using var fs = new MiniFileSystem("./pack.gff", create: true);

fs.WriteFile("hero.png",  new byte[3000]);   // 1 簇
fs.WriteFile("map.bin",   new byte[9000]);   // 3 簇
Console.WriteLine(fs.FileCount);             // 2

byte[] hero = fs.ReadFile("hero.png");       // 读回
byte[] seg  = fs.ReadFileSegment("map.bin", 4096, 1024);  // 按偏移读片段

fs.DeleteFile("hero.png");                   // 块标记空闲(物理文件不收缩)
fs.WriteFile("icon.png", new byte[2000]);    // 1 簇 → 复用 hero.png 留下的空闲块
fs.RenameFile("map.bin", "level.bin");       // 只改元数据
```

---

## 4. 取舍自检

- ✅ 保留：单大文件归档、簇 4KB 对齐分配、空闲块最佳匹配复用、删除标记空闲不收缩、按偏移读片段、Rename 只改元数据。
- ❌ 砍掉：文件名 XOR 加密、Marshal 结构体序列化、元数据回写物理文件（精简版只在内存维护索引）、StringData/字符串索引复用。
- ⚠️ 提醒：三套索引（文件名→块、块表、空闲表）必须一致维护，任一漏改即数据损坏。删除是"标记空闲进表"而非收缩文件——这是归档分配器与"直接用 OS 文件"的本质区别。簇对齐用空间换分配简单。

---

> 配套考题见 `03_FileSystem_考题.md`。
