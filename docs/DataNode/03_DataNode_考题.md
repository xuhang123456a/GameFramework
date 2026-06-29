# DataNode 数据结点 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. DataNode 模块的数据组织形态是什么（字典/树/链表）？根节点叫什么名字？
2. 路径支持哪三种分隔符？连续分隔符（如 `a..b`）会怎样处理？
3. 节点上挂的数据是什么类型？一个节点能挂几份数据？
4. `m_Childs` 为什么默认是 null 而不是空 List？
5. 节点名称有什么硬性约束？为什么？

---

## 二、机制题 🟡

6. `GetNode` 与 `GetOrAddNode` 在路径缺失时的行为分别是什么？
7. `SetData(path)` 与 `GetData(path)` 在路径不存在时的容错为何不对称？分别如何处理？
8. 节点 `SetData` 覆盖旧数据时，对旧 `Variable` 做了什么？
9. `FullName` 是怎么计算出来的？递归的终止条件是什么？
10. `RemoveNode` 如何定位"要删的节点"和"执行删除的父节点"？
11. `DataNodeManager.Update` 是空实现，为什么还要把它做成 Module？

---

## 三、架构 / 陷阱题 🔴

12. `DataNode.IReference.Clear()` 如何实现"删一个节点就回收整棵子树"？描述递归释放的顺序。
13. 如果 `Clear()` 里只 Release 自己不递归 Release 子节点，会发生什么？
14. 节点既是 `IDataNode` 又是 `IReference`，这两个契约分别服务于谁？
15. 为什么"写入宽容（建链）、读取严格（抛异常）"是更好的 API 设计？反过来会有什么隐患？
16. DataNode 的递归回收与 FSM 黑板数据释放、ObjectPool 包装器回收，在"容器负责释放其持有的可复用子对象"这条铁律上是否一致？举例说明。
17. 如果节点名允许包含 `.`，会破坏什么？给一个具体的"存得进取不出"的例子。
18. DataNode 与 Config/Setting 模块在职责上有何区分（运行期 vs 持久化）？

---

## 四、实操题 ✍️

19. 给精简版补 `GetAllChild(List<DataNode> results)`，注意 `m_Childs` 为 null 的处理。
20. 写一段代码：建立 `World.Region1.MonsterCount=10`，然后整体移除 `World`，并解释回收了哪些对象。
21. 实现 `ToDataString()` 风格输出：节点无数据显示 `<Null>`，有数据显示 `[类型名] 值`。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：树；根节点名 `<Root>`。
- **2**：`.` `/` `\`；连续分隔符因 `RemoveEmptyEntries` 被忽略空段。
- **4**：稀疏树叶子多，懒分配避免给无子节点的叶子浪费 List 内存。
- **6**：GetNode 缺失返回 null；GetOrAddNode 缺失即创建建链。
- **7**：Set 走 GetOrAddNode 自动建链（宽容）；Get 走 GetNode，null 时抛异常（严格）。
- **8**：`ReferencePool.Release(m_Data)` 归还旧 Variable。
- **9**：`Parent==null ? Name : Parent.FullName + "." + Name`；终止于根（Parent==null）。
- **10**：下钻时维护 parent 游标，结束后 current=目标、parent=其父，由 parent.RemoveChild 删除。
- **12**：`IReference.Clear`→`Clear()`：先 Release m_Data，再对每个 child `ReferencePool.Release`（触发其 Clear 递归），深度优先整树归还。
- **13**：子孙节点与其数据永久脱离引用池 → 内存泄漏。
- **16**：一致。FSM.Clear 释放所有 Variable；ObjectPool.Shutdown 释放包装器与真身；DataNode.Clear 递归释放子树——均为"容器释放其持有的 IReference 子对象"。
- **17**：名为 `a.b` 的节点，GetNode("a.b") 会被切成 a→b 两级，找不到该单节点 → 取不出。

</details>
