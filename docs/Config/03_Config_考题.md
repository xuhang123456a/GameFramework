# Config 全局配置 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. 一个配置项 `ConfigData` 里同时存了哪几种类型的值？
2. Config 的存储结构与 DataTable 有何不同（树/字典/有无 id）？
3. `IConfigManager` 继承了哪个泛型接口？这说明 Config 与 DataTable 共享了什么？
4. `GetConfigData` 返回的是什么类型？为什么不直接返回 `ConfigData`？
5. 字典为什么用 `StringComparer.Ordinal`？

---

## 二、机制题 🟡

6. `AddConfig(name, string)` 是如何把一个字符串变成四种类型值的？parse 失败会怎样？
7. 每个 `Get*` 的两个重载在配置缺失时分别如何处理？
8. `AddConfig` 遇到重名配置项是抛异常、覆盖、还是温和拒绝？与 DataTable 重 id 的处理有何不同？
9. 配置值的解析发生在"写入时"还是"读取时"？这是什么优化思路？
10. Config 的 `Update` 和 `Shutdown` 实现内容是什么？为什么？

---

## 三、架构 / 陷阱题 🔴

11. Config 与 DataTable "同管线异存储"，具体共享了哪一层、各自不同在哪一层？
12. "一项四值预解析"在什么场景下是划算的？什么场景下反而浪费？
13. 用 `ConfigData?` 表达缺失 与 `TaskInfo` 用 `IsValid` 表达缺失，是两种什么范式？各有何优劣？
14. 如果所有 Get 都只提供"容错版（带默认值）"，会埋下什么隐患？举一个具体后果。
15. DataNode、Config、Setting 三者都管数据，职责边界如何划分？
16. Config 加载复用了 DataProvider 的多源分支（磁盘/二进制/文件系统）。这对"Config 的解析格式可替换"意味着什么？

---

## 四、实操题 ✍️

17. 给精简版补 `TryGetInt(string name, out int value)`：存在返回 true 并输出值，不存在返回 false。
18. 写一个支持 `#` 注释与空行跳过的配置解析 helper。
19. 解释为什么 `GetString` 的实现用 `is { } d` 模式而其他 Get 用 `?.Value ??`（提示：string 是引用类型，可能本身为 null）。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：bool / int / float / string 四种。
- **2**：Config=扁平 `Dictionary<string, ConfigData>` 无 id 无树；DataTable=`Dictionary<int, T>` 按 id。
- **3**：`IDataProvider<IConfigManager>`；与 DataTable 共享 DataProvider 读取管线。
- **4**：`ConfigData?`（Nullable struct）；用 HasValue 区分"存在/不存在"，避免给 struct 设魔法无效值。
- **6**：bool/int/float 各 TryParse，失败取默认(false/0/0f)，原串存 stringValue。
- **7**：无 default 版缺失抛异常；带 default 版缺失返回 default。
- **8**：温和拒绝(返回 false，不覆盖)；DataTable 重 id 抛异常。
- **9**：写入时预 parse 固化，读取零解析；空间换时间。
- **11**：共享 DataProvider 加载层；不同在存储/查询层（扁平 KV vs id 字典）。
- **12**：配置项少、读取频繁、类型不定 → 划算；配置项极多且每项只读单一类型 → 预 parse 浪费内存。
- **13**：Nullable<struct> vs 内嵌 IsValid 字段；前者语义清晰无需改结构体，后者省一次装箱但污染结构体语义。
- **14**：缺失的关键配置静默走默认值，bug 被掩盖。例：ServerUrl 拼错 key→GetString 返回默认空串→连不上服务器却无报错。
- **15**：DataNode=运行期可变树(路径)；Config=全局只读 KV(启动加载)；Setting=可持久化用户设置(存盘)。

</details>
