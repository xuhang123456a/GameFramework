# Debugger 调试器 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. `IDebuggerWindowGroup` 继承自哪个接口？这意味着什么？
2. 调试窗口树的根是什么？
3. 调试窗口有哪五个生命周期钩子？
4. 路径注册（如 `Profiler/ObjectPool`）如何组织窗口？
5. `ActiveWindow` 控制什么？

---

## 二、机制题 🟡

6. 窗口组的 OnUpdate/OnDraw 转发给谁？若该窗口又是组会怎样？
7. 路径注册时，中间路径段不存在会怎样？
8. 选中一个窗口（SelectDebuggerWindow）会触发什么钩子？切走呢？
9. `Update` 在调试器关闭时做什么？
10. 窗口的 `Initialize` 在什么时候调用？

---

## 三、架构 / 陷阱题 🔴

11. "组合模式（组也是窗口）"消除了什么样的类型判断分支？
12. 路径注册与 DataNode 的 GetOrAddNode 有何同构之处？
13. 为什么调试设施必须有"激活才轮询"的零开销开关？没有会怎样？
14. Debugger 展示的数据（对象池/引用池信息）从哪来？这体现了什么架构原则？
15. "核心层产出数据 DTO、表现层负责呈现"在 Debugger 上如何体现？为什么不让每个模块自己画 UI？
16. Priority=-1 对 Debugger 的轮询时序有什么影响？为什么靠后轮询合理？

---

## 四、实操题 ✍️

17. 给精简版加 `SelectDebuggerWindow(path)`：选中时对旧窗口 OnLeave、新窗口 OnEnter。
18. 实现一个"对象池信息窗口"，OnDraw 时读取（模拟的）GetAllObjectInfos 快照并逐行打印。
19. 分析：注册 `A/B/C` 和 `A/B/D` 两个窗口后，窗口树结构是怎样的？A、B 是窗口还是组？

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：`IDebuggerWindow`；窗口组本身是窗口 → 可嵌套组、客户端统一处理。
- **2**：`DebuggerWindowRoot`（一个窗口组）。
- **3**：Initialize/OnEnter/OnLeave/OnUpdate/OnDraw。
- **4**：按 '/' 分层，逐级建窗口组，末级挂实际窗口，形成标签页树。
- **5**：调试器是否激活（开关）——关闭时不轮询不绘制。
- **6**：转发给 SelectedWindow（当前选中子窗口）；若它是组则继续向下递归转发。
- **7**：GetOrAdd 自动创建中间窗口组（建链）。
- **8**：选中 → OnEnter；切走 → 旧窗口 OnLeave。
- **9**：`if (!ActiveWindow) return;` 直接返回，零开销。
- **10**：注册（RegisterDebuggerWindow）时一次性调用。
- **11**：消除"if(是组) 递归处理 else 处理叶子"的分支——统一当 IDebuggerWindow 递归。
- **12**：都是路径分层 + 逐级 GetOrAdd 建链（DataNode 建节点、Debugger 建窗口组）。
- **13**：调试工具不能拖累正式运行；没有开关则发布版仍遍历绘制所有窗口，浪费性能。
- **14**：各模块的 GetXxxInfos() 只读快照 DTO（ObjectInfo/ReferencePoolInfo/TaskInfo...）；体现"数据与呈现分离"。
- **15**：Debugger 统一消费各模块快照并渲染；各模块自己画 UI 会让核心层耦合 UnityEngine、可观测性分散难管理。
- **16**：靠后轮询——其他模块先更新好，Debugger 读到的是当帧最新数据；它是观测者，应在被观测者之后跑。
- **19**：A 是组、B 是组（A 的子组）、C 和 D 是 B 组下的两个窗口。

</details>
