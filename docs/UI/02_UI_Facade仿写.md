# UI 界面 · Facade 精简仿写

> 目标：~150 行，复刻"界面组栈 + 深度刷新 + 暂停/遮挡联动 + Refocus 置顶 + ObjectPool 复用"。
> 取舍：复用 Entity 仿写的"组+池+生命周期"骨架，本文聚焦 UI 特有的栈深度与状态联动。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| UIGroup 栈 + 深度 | 同 | **保留**（UI 核心） |
| Refresh 重算暂停/遮挡 | 同 | **保留** |
| PauseCoveredUIForm | 同 | **保留** |
| Refocus 重复打开置顶 | 同 | **保留** |
| ObjectPool 实例复用 | 引用 Entity 骨架 | 复用 |
| 11 钩子 | 聚焦 Open/Close/Pause/Resume/Cover/Reveal/Refocus | 聚焦 |

---

## 2. 核心代码（聚焦栈深度与状态联动）

```csharp
using System;
using System.Collections.Generic;

namespace MiniUI
{
    public interface IUIForm
    {
        int SerialId { get; }
        string AssetName { get; }
        bool PauseCoveredUIForm { get; }   // 打开我时是否暂停下层
        void OnInit(int id, string asset, bool pauseCovered, bool isNew);
        void OnOpen(object userData);
        void OnClose(bool isShutdown);
        void OnPause();   void OnResume();   // 暂停/恢复(停逻辑)
        void OnCover();   void OnReveal();   // 遮挡/露出(停渲染)
        void OnRefocus(object userData);     // 重复打开置顶激活
        void OnDepthChanged(int groupDepth, int depthInGroup);
    }

    public sealed class UIGroup
    {
        public string Name { get; }
        public int Depth { get; set; }
        // 栈：第一个是栈顶(最新打开,深度最高)
        private readonly LinkedList<IUIForm> m_Forms = new();
        public int UIFormCount => m_Forms.Count;
        public IUIForm CurrentUIForm => m_Forms.First?.Value;

        public UIGroup(string name, int depth) { Name = name; Depth = depth; }

        public IUIForm Get(string assetName)
        {
            foreach (var f in m_Forms) if (f.AssetName == assetName) return f;
            return null;
        }

        public void AddToTop(IUIForm form) => m_Forms.AddFirst(form);   // 新界面置顶
        public void Remove(IUIForm form) => m_Forms.Remove(form);

        public void Refocus(IUIForm form)   // 重复打开：移到栈顶
        {
            m_Forms.Remove(form);
            m_Forms.AddFirst(form);
        }

        // 每次栈变化后全量重算深度 + 暂停/遮挡状态（UI 核心算法）
        public void Refresh()
        {
            int depth = m_Forms.Count;
            bool pause = false;       // 上方是否已有"暂停下层"的界面
            bool cover = false;       // 上方是否有任意界面(造成遮挡)
            var node = m_Forms.First;
            while (node != null)
            {
                var form = node.Value;
                form.OnDepthChanged(Depth, depth--);   // 从高到低分配组内深度

                if (pause)
                {
                    form.OnPause();                     // 被上层暂停
                }
                else
                {
                    if (cover) form.OnCover();          // 被遮挡(不暂停)
                    else form.OnReveal();               // 栈顶/未被遮挡
                    if (form.PauseCoveredUIForm) pause = true;  // 此界面会暂停其下所有
                }
                cover = true;                           // 再往下都被当前界面遮挡
                node = node.Next;
            }
        }
    }

    public sealed class UIManager
    {
        private readonly Dictionary<string, UIGroup> m_Groups = new();
        private readonly Dictionary<int, IUIForm> m_Forms = new();
        private readonly Func<string, IUIForm> m_FormFactory;   // 模拟加载+实例化+创建逻辑
        private int m_Serial;

        public UIManager(Func<string, IUIForm> formFactory) => m_FormFactory = formFactory;
        public void AddGroup(string name, int depth) => m_Groups[name] = new UIGroup(name, depth);

        public IUIForm OpenUIForm(string assetName, string groupName, bool pauseCovered, bool allowMultiInstance = false, object userData = null)
        {
            var group = m_Groups[groupName];

            if (!allowMultiInstance)
            {
                var existing = group.Get(assetName);
                if (existing != null)                   // 已打开 → Refocus 置顶, 不新建
                {
                    group.Refocus(existing);
                    existing.OnRefocus(userData);
                    group.Refresh();
                    return existing;
                }
            }

            var form = m_FormFactory(assetName);        // (实际: 命中池复用 or 异步加载实例化)
            int id = ++m_Serial;
            form.OnInit(id, assetName, pauseCovered, isNew: true);
            group.AddToTop(form);
            m_Forms[id] = form;
            form.OnOpen(userData);
            group.Refresh();                            // 重算栈状态(暂停/遮挡下层)
            return form;
        }

        public void CloseUIForm(int id, bool isShutdown = false)
        {
            if (!m_Forms.TryGetValue(id, out var form)) return;
            var group = FindGroup(form);
            form.OnClose(isShutdown);
            group?.Remove(form);                        // (实际: Unspawn 回 ObjectPool)
            m_Forms.Remove(id);
            group?.Refresh();                           // 下层 OnResume/OnReveal
        }

        private UIGroup FindGroup(IUIForm form)
        {
            foreach (var g in m_Groups.Values)
                if (g.Get(form.AssetName) == form) return g;
            // 简化: 遍历找(真实项目 form 持组引用)
            foreach (var g in m_Groups.Values) return g;
            return null;
        }
    }
}
```

---

## 3. 使用示例

```csharp
sealed class Form : IUIForm
{
    public int SerialId { get; private set; }
    public string AssetName { get; private set; }
    public bool PauseCoveredUIForm { get; private set; }
    public void OnInit(int id, string a, bool p, bool isNew)
    { SerialId = id; AssetName = a; PauseCoveredUIForm = p; }
    public void OnOpen(object u) => Console.WriteLine($"打开 {AssetName}");
    public void OnClose(bool s) => Console.WriteLine($"关闭 {AssetName}");
    public void OnPause() => Console.WriteLine($"  {AssetName} 暂停");
    public void OnResume() => Console.WriteLine($"  {AssetName} 恢复");
    public void OnCover() { } public void OnReveal() { }
    public void OnRefocus(object u) => Console.WriteLine($"{AssetName} 置顶");
    public void OnDepthChanged(int gd, int d) { }
}

var ui = new UIManager(name => new Form());
ui.AddGroup("Normal", depth: 0);

ui.OpenUIForm("Main.prefab", "Normal", pauseCovered: false);
ui.OpenUIForm("Dialog.prefab", "Normal", pauseCovered: true);  // 全屏弹窗 → Main 被暂停
// 输出: 打开 Main / 打开 Dialog / Main 暂停
ui.OpenUIForm("Main.prefab", "Normal", false);                 // 重复 → Main 置顶 Refocus
```

---

## 4. 取舍自检

- ✅ 保留：界面组栈、Refresh 全量重算深度、PauseCoveredUIForm 暂停下层、遮挡 Cover/Reveal、Refocus 置顶、单例/多开判断、OnDepthChanged。
- ❌ 砍掉：ObjectPool 实例复用（引用 Entity 骨架）、异步加载、OpenUIFormInfo 中转、完整 11 钩子。
- ⚠️ 提醒：**每次栈增删必须 Refresh 全量重算**，否则暂停/遮挡状态不对称（暂停了没恢复）。Refocus 与新建要按"是否单例"区分。UI 复用 Entity 的池化骨架，只新增栈层级——别重写一遍对象池集成。

---

> 配套考题见 `03_UI_考题.md`。
