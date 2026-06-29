# Setting 持久化设置 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. Setting 与 Config 在"可写性、是否存盘、随包还是用户产生"三方面有何相反？
2. `SettingManager` 自己持有设置数据吗？数据真正存在哪里？
3. `ISettingManager` 与 `ISettingHelper` 的接口关系是什么（独立/镜像/继承）？
4. Setting 比 Config 多了哪两类能力？
5. `SettingManager.Count` 是怎么得到的？

---

## 二、机制题 🟡

6. `SettingManager` 每个公开方法的统一骨架是什么（两步）？
7. 未调用 `SetSettingHelper` 就使用任何方法会怎样？
8. `Shutdown()` 做了什么？为什么这么设计？
9. `GetObject<T>/SetObject<T>` 与基础类型 Get/Set 在开销上有何不同？
10. Load/Save 各自把数据在"内存"与"存储介质"之间往哪个方向搬？

---

## 三、架构 / 陷阱题 🔴

11. Setting 是"纯 Facade 模式"的范本。哪些逻辑刻意留在 Manager？哪些刻意外置给 Helper？这个边界的判断依据是什么？
12. Manager 与 Helper 接口几乎完全镜像，看似冗余。这层镜像换来了什么收益？什么情况下它属于过度设计？
13. "内存写、显式/关闭时存"模型下，什么情况会丢失用户更改？如何缓解？
14. 为什么核心层（纯 C#）不直接用 PlayerPrefs，而要经 `ISettingHelper`？这与"GameFramework 严格分层"有何关系？
15. 如果要把存储从 PlayerPrefs 换成"加密文件"，需要改动哪些代码？上层与 Manager 是否受影响？
16. Setting、Config、DataNode 三者职责边界如何划分？各举一个典型用途。

---

## 四、实操题 ✍️

17. 给精简版 Helper 增加"脏标记 + 自动节流保存"：Set 时标脏，每隔 N 秒若脏则自动 Save。
18. 写一个基于"真实 JSON 序列化"的 `GetObject<T>/SetObject<T>`（用你熟悉的 JSON 库），说明与占位实现的差异。
19. 设计一个 mock Helper 用于单元测试 SettingManager 的"helper 为空时抛异常"逻辑，说明镜像接口为何让 mock 变得容易。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：Config 只读/不存盘/随包发布；Setting 可写/存盘/用户运行期产生。
- **2**：不持有；数据全在 helper 内（如 PlayerPrefs/文件）。
- **3**：镜像（方法集几乎一一对应）。
- **4**：写入（Set*）与对象序列化存取（Get/SetObject）。
- **5**：转发给 `m_SettingHelper.Count`。
- **6**：① 校验(helper 非空 + name 非空) ② 转发给 helper。
- **7**：抛 GameFrameworkException("Setting helper is invalid.")。
- **8**：调 `Save()` 落盘；关闭时兜底持久化防丢失。
- **9**：对象需序列化/反序列化(JSON/二进制)，比基础类型重，有分配开销。
- **11**：留内=参数校验、契约稳定性；外置=平台相关的存储实现。依据：平台相关且可替换的踢给 helper，通用校验与契约留 manager。
- **12**：收益=上层只依赖稳定的 ISettingManager，helper 可任意替换(PlayerPrefs→文件→云)且上层无感、易 mock 测试。过度设计=确定永不换存储时。
- **13**：崩溃跳过 Shutdown 且未主动 Save 时丢失；缓解=关键设置 Set 即 Save 或节流保存。
- **14**：PlayerPrefs 是 UnityEngine API，核心层不可引用 UnityEngine；经 helper 注入保持纯 C# 层无引擎依赖，符合严格分层。
- **15**：只改/换一个 ISettingHelper 实现；上层与 Manager 不受影响。
- **16**：Config=只读全局(如服务器地址)；Setting=用户设置(如音量)；DataNode=运行期黑板(如临时战斗状态树)。

</details>
