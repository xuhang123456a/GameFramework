# Localization 本地化 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. 本地化字典的存储结构是什么？值的类型固定为什么？
2. `Language` 与 `SystemLanguage` 有何区别？后者从哪来？
3. `GetRawString` 与 `GetString` 的区别是什么？
4. Localization 复用了哪个共享组件来加载字典？还有哪些模块也复用它？
5. `AddRawString` 遇到重复 key 返回什么？

---

## 二、机制题 🟡

6. `GetString` 在"key 缺失"和"格式化异常"两种情况下分别返回什么？
7. 为什么 `GetString` 设计成永不抛异常？这与 DataNode 的 `GetData` 有何相反？
8. 框架为什么提供 0~16 共 17 个 `GetString` 泛型重载，而不用一个 `params object[]` 版本？
9. 切换语言时，字典内存里发生了什么？是同时存多语言还是只存当前语言？
10. `Language` 属性 setter 拒绝什么值？

---

## 三、架构 / 陷阱题 🔴

11. 一段本地化文本 `"得分：{0}/{1}"` 只传了一个参数会怎样？玩家会看到什么？开发者如何定位？
12. 用 `int`/`float` 调 `GetString(key, 80, 100)`，泛型重载与 `params object[]` 在内存分配上有何差异？为什么对 UI 重要？
13. "加载字典(ReadData)"与"设置当前语言(Language)"为什么是解耦的两件事？这带来什么取舍？
14. Localization 与 Config 都基于 `DataProvider` 且都是扁平字典，它们的本质区别是什么？
15. 如果要做"主语言缺 key 时回退到英语"，在现有结构上该如何扩展？会破坏"永不抛"契约吗？
16. 为什么本地化文本不放进 ReferencePool 复用，而读取事件 EventArgs 却要？

---

## 四、实操题 ✍️

17. 给精简版补 `GetString<T1,T2,T3>` 重载，保持三态容错。
18. 写一个支持 `#` 注释与 `key=value` 的字典 parser。
19. 设计一个"语言切换广播"：切语言后通过事件通知所有 UI 刷新（结合 EventPool 思路描述即可）。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：`Dictionary<string, string>`(Ordinal)；值固定为 string(本地化文本)。
- **2**：Language=当前选定语言(可写)；SystemLanguage=设备系统语言(从 helper 读，只读)。
- **3**：GetRawString 直接查字典(缺失返回 null)；GetString 带格式化与容错(缺失返回 `<NoKey>`)。
- **4**：DataProvider；DataTable、Config 也复用。
- **6**：缺 key → `<NoKey>{key}`；格式化异常 → `<Error>{key,value,args,exception}`。两者都是有效字符串。
- **7**：直面 UI 渲染，缺 key 不该崩溃；DataNode.GetData 缺失抛异常(调用方应能承受)。
- **8**：消除值类型装箱与 params 数组分配，UI 高频格式化避免 GC 热点。
- **9**：切语言重新 ReadData 对应字典，只存当前语言，不同时驻留多语言。
- **11**：string.Format 抛 FormatException 被 catch，返回 `<Error>...`；玩家看到 `<Error>` 占位；开发者从占位串里的 key/value/exception 定位。
- **13**：字典内容与语言标记分离 → 只承载当前语言省内存，代价是切换需重新加载。
- **14**：Config=开发期固化只读配置(一项四值，多类型解读)；Localization=多语言文本(值为 string，带占位符格式化与回退占位)。
- **15**：在 GetRawString 返回 null 时查回退字典；只要回退也返回串或 `<NoKey>`，不抛异常即不破坏契约。
- **16**：文本是长生命周期只读数据，复用无益；EventArgs 高频短命，池化省 GC。

</details>
