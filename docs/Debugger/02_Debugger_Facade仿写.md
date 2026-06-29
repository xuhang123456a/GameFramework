# Debugger 调试器 · Facade 精简仿写

> 目标：~90 行，复刻"组合模式（组=窗口）+ 路径注册 + 选中即进入 + 激活才轮询 + 递归转发"。
> 取舍：OnDraw 用 Console 输出模拟 GUI；聚焦组合模式与路径树。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `IDebuggerWindowGroup : IDebuggerWindow` | 同 | **保留**（组合模式核心） |
| 路径注册(GetOrAdd 子组) | 同 | **保留** |
| 五钩子 Init/Enter/Leave/Update/Draw | 同 | 保留 |
| SelectedWindow 转发 | 同 | **保留**（递归转发） |
| ActiveWindow 零开销开关 | 同 | **保留** |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniDebugger
{
    public interface IDebuggerWindow
    {
        void Initialize(params object[] args);
        void OnEnter();
        void OnLeave();
        void OnUpdate(float dt);
        void OnDraw();
    }

    // 组合模式：窗口组本身是窗口 → 可嵌套
    public sealed class DebuggerWindowGroup : IDebuggerWindow
    {
        private readonly List<string> m_Names = new();
        private readonly List<IDebuggerWindow> m_Windows = new();
        public int SelectedIndex { get; set; }
        public IDebuggerWindow SelectedWindow
            => (SelectedIndex >= 0 && SelectedIndex < m_Windows.Count) ? m_Windows[SelectedIndex] : null;

        public void Initialize(params object[] args) { }
        public void OnEnter() => SelectedWindow?.OnEnter();
        public void OnLeave() => SelectedWindow?.OnLeave();
        public void OnUpdate(float dt) => SelectedWindow?.OnUpdate(dt);   // 递归转发当前选中
        public void OnDraw()
        {
            Console.WriteLine($"[标签栏: {string.Join(" | ", m_Names)}]");
            SelectedWindow?.OnDraw();                                      // 转发给选中窗口(可能又是组)
        }

        // 路径注册：逐级 GetOrAdd 子组，末级挂窗口（类似 DataNode 路径树）
        public void RegisterDebuggerWindow(string path, IDebuggerWindow window)
        {
            int slash = path.IndexOf('/');
            if (slash < 0)                              // 末级：直接挂
            {
                m_Names.Add(path); m_Windows.Add(window);
                return;
            }
            string groupName = path[..slash];
            string rest = path[(slash + 1)..];
            var group = GetOrAddGroup(groupName);
            group.RegisterDebuggerWindow(rest, window); // 递归下钻
        }

        public IDebuggerWindow GetDebuggerWindow(string path)
        {
            int slash = path.IndexOf('/');
            if (slash < 0)
            {
                int i = m_Names.IndexOf(path);
                return i >= 0 ? m_Windows[i] : null;
            }
            var group = Find(path[..slash]) as DebuggerWindowGroup;
            return group?.GetDebuggerWindow(path[(slash + 1)..]);
        }

        private DebuggerWindowGroup GetOrAddGroup(string name)
        {
            if (Find(name) is DebuggerWindowGroup g) return g;
            g = new DebuggerWindowGroup();
            m_Names.Add(name); m_Windows.Add(g);
            return g;
        }
        private IDebuggerWindow Find(string name)
        {
            int i = m_Names.IndexOf(name);
            return i >= 0 ? m_Windows[i] : null;
        }
        public string[] GetDebuggerWindowNames() => m_Names.ToArray();
    }

    public sealed class DebuggerManager
    {
        private readonly DebuggerWindowGroup m_Root = new();
        public bool ActiveWindow { get; set; }
        public IDebuggerWindow DebuggerWindowRoot => m_Root;

        public void RegisterDebuggerWindow(string path, IDebuggerWindow window, params object[] args)
        {
            m_Root.RegisterDebuggerWindow(path, window);
            window.Initialize(args);                    // 注册时一次性初始化
        }

        public void Update(float dt)
        {
            if (!ActiveWindow) return;                  // 关闭时零开销
            m_Root.OnUpdate(dt);
        }
        public void Draw() { if (ActiveWindow) m_Root.OnDraw(); }
        public void Shutdown() { ActiveWindow = false; }
    }
}
```

---

## 3. 使用示例

```csharp
sealed class FpsWindow : IDebuggerWindow
{
    public void Initialize(params object[] args) { }
    public void OnEnter() => Console.WriteLine("进入 FPS 窗口");
    public void OnLeave() { }
    public void OnUpdate(float dt) { }
    public void OnDraw() => Console.WriteLine("  FPS: 60");
}
sealed class PoolWindow : IDebuggerWindow
{
    public void Initialize(params object[] args) { }
    public void OnEnter() { } public void OnLeave() { } public void OnUpdate(float dt) { }
    public void OnDraw() => Console.WriteLine("  ObjectPool: 12 pools");   // 读 GetAllObjectInfos 快照
}

var dbg = new DebuggerManager();
dbg.RegisterDebuggerWindow("Information/FPS", new FpsWindow());
dbg.RegisterDebuggerWindow("Profiler/ObjectPool", new PoolWindow());   // 自动建 Profiler 组

dbg.ActiveWindow = true;
dbg.Update(0.016f);
dbg.Draw();   // 画标签栏 + 当前选中窗口（递归转发）
```

---

## 4. 取舍自检

- ✅ 保留：组合模式(组实现窗口接口、可嵌套)、路径注册建组树、SelectedWindow 递归转发、Initialize 一次性、ActiveWindow 零开销开关。
- ❌ 砍掉：OnGUI/IMGUI 渲染、内置窗口实现(FPS/内存/各模块)、OnEnter/OnLeave 的选中切换完整处理、Unregister。
- ⚠️ 提醒：组合模式让管理器无需区分"窗口 vs 组"——统一当 IDebuggerWindow 递归处理。调试设施必须有 ActiveWindow 总开关短路，发布版零成本。Debugger 消费各模块的快照 DTO——数据与呈现分离。

---

> 配套考题见 `03_Debugger_考题.md`。
