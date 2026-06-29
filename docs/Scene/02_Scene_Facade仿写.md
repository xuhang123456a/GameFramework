# Scene 场景 · Facade 精简仿写

> 目标：~90 行，复刻"三态名单 + 防重入校验 + 委托加载 + 回调迁移名单"。
> 取舍：Resource 加载用注入式异步回调模拟；聚焦状态追踪层。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| 三态名单(Loaded/Loading/Unloading) | 同 | **保留**（核心） |
| LoadScene 四重防重入校验 | 同 | **保留** |
| 委托 ResourceManager | 注入加载回调 | 保留语义 |
| 回调里迁移名单 | 同 | **保留** |
| 6 事件 | 精简回调 | 简化 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniScene
{
    // 模拟 Resource 的异步场景加载/卸载
    public interface ISceneLoader
    {
        void LoadScene(string name, Action<string> onSuccess, Action<string, string> onFailure);
        void UnloadScene(string name, Action<string> onSuccess, Action<string> onFailure);
    }

    public sealed class SceneManager
    {
        private readonly List<string> m_Loaded = new();
        private readonly List<string> m_Loading = new();
        private readonly List<string> m_Unloading = new();
        private readonly ISceneLoader m_Loader;

        public event Action<string> LoadSceneSuccess;
        public event Action<string, string> LoadSceneFailure;
        public event Action<string> UnloadSceneSuccess;

        public SceneManager(ISceneLoader loader) => m_Loader = loader;

        public bool SceneIsLoaded(string n) => m_Loaded.Contains(n);
        public bool SceneIsLoading(string n) => m_Loading.Contains(n);
        public bool SceneIsUnloading(string n) => m_Unloading.Contains(n);
        public string[] GetLoadedSceneAssetNames() => m_Loaded.ToArray();

        public void LoadScene(string name)
        {
            if (string.IsNullOrEmpty(name)) throw new ArgumentException(nameof(name));
            // 四重防重入校验（Scene 模块存在的核心意义）
            if (SceneIsUnloading(name)) throw new InvalidOperationException($"'{name}' is unloading.");
            if (SceneIsLoading(name))   throw new InvalidOperationException($"'{name}' is loading.");
            if (SceneIsLoaded(name))    throw new InvalidOperationException($"'{name}' already loaded.");

            m_Loading.Add(name);                       // 进"加载中"名单
            m_Loader.LoadScene(name, OnLoadSuccess, OnLoadFailure);
        }

        public void UnloadScene(string name)
        {
            if (SceneIsUnloading(name)) throw new InvalidOperationException($"'{name}' is unloading.");
            if (SceneIsLoading(name))   throw new InvalidOperationException($"'{name}' is loading.");
            if (!SceneIsLoaded(name))   throw new InvalidOperationException($"'{name}' not loaded.");

            m_Unloading.Add(name);                     // 进"卸载中"名单
            m_Loader.UnloadScene(name, OnUnloadSuccess, OnUnloadFailure);
        }

        // ---- 回调里迁移名单（状态转移）----
        private void OnLoadSuccess(string name)
        {
            m_Loading.Remove(name);                    // 加载中 → 已加载
            m_Loaded.Add(name);
            LoadSceneSuccess?.Invoke(name);
        }
        private void OnLoadFailure(string name, string error)
        {
            m_Loading.Remove(name);                    // 失败必须移出加载中(否则永久卡住)
            LoadSceneFailure?.Invoke(name, error);
        }
        private void OnUnloadSuccess(string name)
        {
            m_Unloading.Remove(name);
            m_Loaded.Remove(name);                     // 卸载中 + 已加载 双移除
            UnloadSceneSuccess?.Invoke(name);
        }
        private void OnUnloadFailure(string name)
        {
            m_Unloading.Remove(name);                  // 卸载失败：移出卸载中，仍已加载
        }

        public void Shutdown()
        {
            foreach (var n in m_Loaded.ToArray())
                if (!SceneIsUnloading(n)) UnloadScene(n);   // 跳过正在卸载的
            m_Loaded.Clear(); m_Loading.Clear(); m_Unloading.Clear();
        }
    }
}
```

---

## 3. 使用示例

```csharp
sealed class FakeLoader : ISceneLoader
{
    public Action pendingLoad, pendingUnload;
    public void LoadScene(string n, Action<string> s, Action<string, string> f)
        => pendingLoad = () => s(n);     // 延迟到外部触发模拟异步
    public void UnloadScene(string n, Action<string> s, Action<string> f)
        => pendingUnload = () => s(n);
}

var loader = new FakeLoader();
var sm = new SceneManager(loader);
sm.LoadSceneSuccess += n => Console.WriteLine($"{n} 加载完成");

sm.LoadScene("Battle");
Console.WriteLine(sm.SceneIsLoading("Battle"));   // True（加载中）
// sm.LoadScene("Battle");                          // 抛异常：正在加载，防重入
loader.pendingLoad();                               // 模拟异步完成
Console.WriteLine(sm.SceneIsLoaded("Battle"));     // True（已迁移到已加载）
```

---

## 4. 取舍自检

- ✅ 保留：三态互斥名单、四重防重入校验、委托加载、回调迁移名单、失败必移出加载中、Shutdown 跳过卸载中。
- ❌ 砍掉：真实 Resource/Unity Additive 加载、进度/依赖事件、优先级、userData。
- ⚠️ 提醒：失败回调**必须**从"加载中"移除，否则场景永久卡在加载态再也无法加载。任何异步资源操作都需要状态追踪层防并发竞态——别把防重入塞进通用加载器。

---

> 配套考题见 `03_Scene_考题.md`。
