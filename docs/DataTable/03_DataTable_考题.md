# DataTable 数据表 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. 一张数据表内部用什么结构存行？查询单行的复杂度是多少？
2. `IDataRow` 要求行实现哪两类成员？`ParseDataRow` 体现了什么设计（谁负责反序列化）？
3. `DataProvider<T>`、`IDataProviderHelper<T>`、`DataTable<T>` 三者职责如何划分？
4. `DataTableBase` 与 `DataProvider` 是继承关系还是组合关系？
5. `MinIdDataRow` / `MaxIdDataRow` 分别是什么？

---

## 二、机制题 🟡

6. `DataProvider.ReadData` 根据 `HasAssetResult` 走哪几条不同路径？哪条是同步的？
7. 同步路径（BinaryOnFileSystem）用什么承接字节数据？这块缓存如何分配大小？
8. `AddDataRow` 的完整步骤是什么（从字符串到入表）？
9. `InternalAddDataRow` 如何 O(1) 维护 Min/Max？
10. `RemoveDataRow` 在什么情况下需要全表扫描？为什么这个权衡是合理的？
11. 读表成功/失败时，`ReadDataSuccessEventArgs` 等是如何创建和回收的？

---

## 三、架构 / 陷阱题 🔴

12. 为什么行对象（`new T()`）不走 ReferencePool，而读表事件 EventArgs 却走 ReferencePool？背后的判断标准是什么？
13. 同一张表要支持"从磁盘 TextAsset 读"和"从二进制文件系统读"两种来源，三层解耦如何让这成为可能？
14. 如果把"解析格式"硬编码进 `DataTable`（而非交给 Helper），会损失什么扩展性？
15. `s_CachedBytes` 是静态字段，多张表共用。这在什么前提下是安全的？什么场景会出问题？
16. 加载可能同步也可能异步，框架如何把两种路径统一到同一套成功/失败处理？（提示：try/catch + 事件 + finally）
17. `LoadAssetSuccessCallback` 的 `finally` 里调了 `ReleaseDataAsset`，为什么解析成功或失败都要释放原始资源？
18. DataTable 与 DataNode 都存"数据"，职责定位有何不同？DataTable 与 Config 又有何不同？

---

## 四、实操题 ✍️

19. 给精简版补 `GetDataRows(Predicate<T> condition, Comparison<T> comparison)`：先筛选再排序。
20. 实现一个二进制解析的行类骨架：`ParseDataRow(byte[] bytes, int start, int len)`，说明与字符串解析的差异。
21. 写一个 Helper，支持"#" 开头的注释行被跳过，其余行喂给表。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：`Dictionary<int, T>`；按 id 查询 O(1)。
- **2**：`Id` 属性 + `ParseDataRow`；反序列化下沉到行自身（自描述）。
- **3**：DataProvider=加载管线(从哪读)，Helper=解析策略(字节→行)，DataTable=存储查询(行存哪/怎么查)。
- **4**：组合（DataTableBase 持有 DataProvider 实例并转发）。
- **6**：AssetOnDisk/OnFileSystem(异步 LoadAsset)、BinaryOnDisk(异步 LoadBinary)、BinaryOnFileSystem(同步读 s_CachedBytes)。同步=最后一条。
- **9**：加行时若 id < Min 则更新 Min，若 id > Max 则更新 Max，O(1)。
- **10**：删的恰是 Min 或 Max 行时全表重算；配置表加载时批量加、运行期几乎不删，罕见重算可接受。
- **12**：标准=对象生命周期与访问频率。行是只读常驻、长生命周期 → 复用无益；EventArgs 高频短命 → 复用省 GC。
- **13**：管线层判 HasAssetResult 走不同加载，Helper 层统一把结果解析成行，表层不变 → 换来源不动表。
- **15**：前提是单线程读表；并发读表会因共用静态缓冲互相覆盖。
- **17**：原始资源（TextAsset 等）已被 Helper 解析成行后不再需要，无论成败都应释放，避免资源泄漏。
- **18**：DataTable=只读批量配置(id 索引)；DataNode=运行期可变黑板树(路径索引)；Config=全局单值 kv 配置。

</details>
