# Fsm 有限状态机 · Facade 精简仿写

> 目标：~150 行独立可编译，复刻"泛型持有者 + 五钩子 + 类型寻址切换 + 黑板数据 + 封闭式 ChangeState"。
> 取舍：FSM 复用与 Variable 的 IReference 化简化为普通对象；保留五钩子配对与切换封闭性这两条不变量。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `Fsm<T> : FsmBase, IReference, IFsm<T>` 三契约 | `Fsm<T>` 单类 | 简化（不接 ReferencePool） |
| `m_States : Dictionary<Type, FsmState<T>>` | 同 | 保留（类型寻址核心） |
| 五钩子 OnInit/Enter/Update/Leave/Destroy | 同 | **保留** |
| `ChangeState` internal + 状态内 protected 触发 | 同 | **保留**（难点①） |
| 黑板 `Dictionary<string, Variable>` + Release | `Dictionary<string, object>` | 简化 |
| `m_TempFsms` 快照轮询 | 同 | 保留（难点防并发） |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniFsm
{
    public abstract class FsmState<T> where T : class
    {
        protected internal virtual void OnInit(Fsm<T> fsm) { }
        protected internal virtual void OnEnter(Fsm<T> fsm) { }
        protected internal virtual void OnUpdate(Fsm<T> fsm, float dt) { }
        protected internal virtual void OnLeave(Fsm<T> fsm, bool isShutdown) { }
        protected internal virtual void OnDestroy(Fsm<T> fsm) { }

        // 切换权封闭：业务状态只能经此 protected 入口触发，内部转回具体 Fsm
        protected void ChangeState<TState>(Fsm<T> fsm) where TState : FsmState<T>
        {
            if (fsm == null) throw new ArgumentNullException(nameof(fsm));
            fsm.ChangeState<TState>();
        }
    }

    public sealed class Fsm<T> where T : class
    {
        public string Name { get; private set; }
        public T Owner { get; private set; }
        public FsmState<T> CurrentState { get; private set; }
        public float CurrentStateTime { get; private set; }
        public bool IsRunning => CurrentState != null;
        public bool IsDestroyed { get; private set; } = true;

        private readonly Dictionary<Type, FsmState<T>> m_States = new();
        private Dictionary<string, object> m_Datas;

        public static Fsm<T> Create(string name, T owner, params FsmState<T>[] states)
        {
            if (owner == null) throw new ArgumentNullException(nameof(owner));
            if (states == null || states.Length < 1) throw new ArgumentException("states empty");
            var fsm = new Fsm<T> { Name = name, Owner = owner, IsDestroyed = false };
            foreach (var s in states)
            {
                var t = s.GetType();
                if (fsm.m_States.ContainsKey(t)) throw new InvalidOperationException($"State '{t}' dup.");
                fsm.m_States.Add(t, s);
                s.OnInit(fsm);                       // 每状态一次性初始化
            }
            return fsm;
        }

        public void Start<TState>() where TState : FsmState<T>
        {
            if (IsRunning) throw new InvalidOperationException("FSM already running.");
            if (!m_States.TryGetValue(typeof(TState), out var s))
                throw new InvalidOperationException($"State '{typeof(TState)}' not exist.");
            CurrentStateTime = 0f;
            CurrentState = s;
            CurrentState.OnEnter(this);
        }

        // internal-等价：切换核心，成对 OnLeave→OnEnter，顺序固定
        internal void ChangeState<TState>() where TState : FsmState<T>
        {
            if (CurrentState == null) throw new InvalidOperationException("No current state.");
            if (!m_States.TryGetValue(typeof(TState), out var s))
                throw new InvalidOperationException($"State '{typeof(TState)}' not exist.");
            CurrentState.OnLeave(this, false);       // 1 旧状态离开
            CurrentStateTime = 0f;                   // 2 计时清零
            CurrentState = s;                        // 3 换引用
            CurrentState.OnEnter(this);              // 4 新状态进入
        }

        public void Update(float dt)
        {
            if (CurrentState == null) return;
            CurrentStateTime += dt;
            CurrentState.OnUpdate(this, dt);
        }

        public TState GetState<TState>() where TState : FsmState<T>
            => m_States.TryGetValue(typeof(TState), out var s) ? (TState)s : null;

        // ---- 黑板数据 ----
        public void SetData(string name, object data)
        {
            if (string.IsNullOrEmpty(name)) throw new ArgumentException(nameof(name));
            (m_Datas ??= new(StringComparer.Ordinal))[name] = data;
        }
        public TData GetData<TData>(string name)
            => m_Datas != null && m_Datas.TryGetValue(name, out var v) ? (TData)v : default;
        public bool RemoveData(string name) => m_Datas?.Remove(name) ?? false;

        public void Shutdown()
        {
            if (CurrentState != null) CurrentState.OnLeave(this, true);  // 关闭路径 isShutdown=true
            foreach (var kv in m_States) kv.Value.OnDestroy(this);
            m_States.Clear();
            m_Datas?.Clear();
            CurrentState = null;
            CurrentStateTime = 0f;
            IsDestroyed = true;
        }
    }

    // 管理器：快照轮询防止 OnUpdate 中增删 FSM 导致集合修改异常
    public sealed class FsmManager
    {
        private readonly Dictionary<(Type, string), Fsm<object>> m_Fsms = new();
        private readonly List<Fsm<object>> m_Temp = new();

        public void Update(float dt)
        {
            m_Temp.Clear();
            foreach (var kv in m_Fsms) m_Temp.Add(kv.Value);
            foreach (var fsm in m_Temp)
            {
                if (fsm.IsDestroyed) continue;
                fsm.Update(dt);
            }
        }
    }
}
```

---

## 3. 使用示例

```csharp
sealed class Monster { public int Hp = 100; }

sealed class IdleState : FsmState<Monster>
{
    protected internal override void OnEnter(Fsm<Monster> fsm) => Console.WriteLine("待机");
    protected internal override void OnUpdate(Fsm<Monster> fsm, float dt)
    {
        if (fsm.Owner.Hp < 50) ChangeState<FleeState>(fsm);   // 经封闭入口切换
    }
}
sealed class FleeState : FsmState<Monster>
{
    protected internal override void OnEnter(Fsm<Monster> fsm) => Console.WriteLine("逃跑");
}

var m = new Monster();
var fsm = Fsm<Monster>.Create("AI", m, new IdleState(), new FleeState());
fsm.Start<IdleState>();
m.Hp = 30;
fsm.Update(0.016f);   // Idle.OnUpdate → ChangeState<FleeState> → Idle.OnLeave + Flee.OnEnter
```

---

## 4. 取舍自检

- ✅ 保留：泛型 Owner、五钩子、类型寻址、`Start` 防重入、切换四步固定顺序、`isShutdown` 区分、快照轮询、黑板数据。
- ❌ 砍掉：FSM 自身的 ReferencePool 复用（直接 new/Shutdown）、Variable 的 IReference 级联释放（用 object 替代）、TypeNamePair 复合键。
- ⚠️ 提醒：`ChangeState` 保持 internal/受控；若开放为 public 会破坏"迁移必由当前状态发起"的不变量。OnInit 写一次性逻辑、OnEnter 写每次进入逻辑，切勿混。

---

> 配套考题见 `03_Fsm_考题.md`。
