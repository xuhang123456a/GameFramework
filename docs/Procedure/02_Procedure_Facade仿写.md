# Procedure 流程 · Facade 精简仿写

> 目标：~60 行，复刻"流程 = FsmState 的薄封装 + 单台流程 FSM + 启动入口"。
> 取舍：复用前文 MiniFsm 的 `Fsm<T>/FsmState<T>`；本文展示"如何用术语映射把 FSM 包装成 Procedure"。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `ProcedureBase : FsmState<IProcedureManager>` | 同 | **保留**（核心：流程=状态） |
| 单台流程 FSM | 同 | **保留** |
| Initialize/StartProcedure | 同 | 保留 |
| Priority=-2 时序 | 注释说明 | 概念保留 |
| 复用 Fsm 全部机制 | 复用 MiniFsm | **保留** |

---

## 2. 核心代码

```csharp
using MiniFsm;   // 复用 Fsm<T> / FsmState<T>

namespace MiniProcedure
{
    public interface IProcedureManager { }   // 流程持有者标记

    // 流程基类 = FsmState 的术语封装（几乎零新代码）
    public abstract class ProcedureBase : FsmState<IProcedureManager>
    {
        // 全部钩子继承自 FsmState：OnInit/OnEnter/OnUpdate/OnLeave/OnDestroy
        // 流程切换复用 FsmState.ChangeState
    }

    public sealed class ProcedureManager : IProcedureManager
    {
        private Fsm<IProcedureManager> m_Fsm;

        public ProcedureBase CurrentProcedure
            => (ProcedureBase)(m_Fsm?.CurrentState
               ?? throw new System.InvalidOperationException("Initialize first."));
        public float CurrentProcedureTime => m_Fsm.CurrentStateTime;

        // 用所有流程建一台 FSM（流程即状态）
        public void Initialize(params ProcedureBase[] procedures)
            => m_Fsm = Fsm<IProcedureManager>.Create("Procedure", this, procedures);

        public void StartProcedure<T>() where T : ProcedureBase => m_Fsm.Start<T>();
        public T GetProcedure<T>() where T : ProcedureBase => m_Fsm.GetState<T>();
        public bool HasProcedure<T>() where T : ProcedureBase => m_Fsm.GetState<T>() != null;

        public void Update(float dt) => m_Fsm.Update(dt);   // 驱动当前流程 OnUpdate
        public void Shutdown() => m_Fsm?.Shutdown();
    }
}
```

---

## 3. 使用示例（游戏流程链）

```csharp
sealed class LaunchProcedure : ProcedureBase
{
    protected internal override void OnEnter(Fsm<IProcedureManager> fsm)
    {
        System.Console.WriteLine("启动：初始化配置/语言");
        ChangeState<MenuProcedure>(fsm);   // 初始化完直接进菜单（封闭式切换）
    }
}

sealed class MenuProcedure : ProcedureBase
{
    protected internal override void OnEnter(Fsm<IProcedureManager> fsm)
        => System.Console.WriteLine("进入主菜单");
    protected internal override void OnUpdate(Fsm<IProcedureManager> fsm, float dt)
    {
        // if (玩家点开始) ChangeState<GameProcedure>(fsm);
    }
}

sealed class GameProcedure : ProcedureBase
{
    protected internal override void OnEnter(Fsm<IProcedureManager> fsm)
        => System.Console.WriteLine("进入游戏");
}

var pm = new ProcedureManager();
pm.Initialize(new LaunchProcedure(), new MenuProcedure(), new GameProcedure());
pm.StartProcedure<LaunchProcedure>();   // Launch.OnEnter → 自动 ChangeState 到 Menu
// 帧循环: while(true) pm.Update(0.016f);
```

---

## 4. 取舍自检

- ✅ 保留：流程=FsmState（术语映射）、单台流程 FSM、Initialize/Start、复用 Fsm 全部生命周期与封闭切换。
- ❌ 砍掉：借用外部 FsmManager（直接 Create）、Priority 时序（单进程示例无需）、反射收集流程列表。
- ⚠️ 提醒：Procedure 的价值在"术语映射"——别在这层塞重复逻辑。流程切换必须经 FsmState.ChangeState（封闭式），保证 OnLeave/OnEnter 成对。游戏宏观流程该用 FSM 建模，而非散落的 bool flag。

---

> 配套考题见 `03_Procedure_考题.md`。
