# Entity 实体 · Facade 精简仿写

> 目标：~150 行，复刻"实体组 + ObjectPool 实例复用 + 七钩子生命周期 + Show/Hide 复用双路径 + 迭代安全轮询"。
> 取舍：复用前文 MiniObjectPool；资源加载用注入式同步加载模拟。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| EntityGroup 持 ObjectPool | 复用 MiniObjectPool | **保留** |
| EntityInstanceObject : ObjectBase | `EntityInstance : PooledObject` | 保留 |
| 七钩子 | 精简到 4 个核心 | 聚焦 |
| Show 命中复用 vs 加载双路径 | 同 | **保留** |
| m_CachedNode 迭代安全 | 同 | **保留** |
| 父子挂接 | 简化 attach | 保留语义 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;
using MiniPool;   // 复用 PooledObject / SimpleObjectPool<T>

namespace MiniEntity
{
    public interface IEntity
    {
        int Id { get; }
        object Handle { get; }
        void OnInit(int id, string assetName, object handle, bool isNew);
        void OnShow(object userData);
        void OnHide(bool isShutdown);
        void OnUpdate(float dt);
    }

    // 实体辅助器：实例化/销毁 GameObject 由它负责（平台相关）
    public interface IEntityHelper
    {
        object InstantiateEntity(object entityAsset);   // 模拟 Instantiate(prefab)
        void ReleaseEntity(object entityAsset, object entityInstance);  // 模拟 Destroy
        IEntity CreateEntity(object entityInstance);
    }

    // 实例包装：池中可复用对象，Release 时真正销毁
    public sealed class EntityInstance : PooledObject
    {
        private object m_Asset; private IEntityHelper m_Helper;
        public static EntityInstance Create(string name, object asset, object instance, IEntityHelper helper)
        {
            var o = new EntityInstance();
            o.Initialize(name, instance);
            o.m_Asset = asset; o.m_Helper = helper;
            return o;
        }
        protected internal override void Release(bool isShutdown)
            => m_Helper.ReleaseEntity(m_Asset, Target);   // 委托 helper 销毁
    }

    public sealed class EntityGroup
    {
        private readonly SimpleObjectPool<EntityInstance> m_Pool;
        private readonly LinkedList<IEntity> m_Entities = new();
        private LinkedListNode<IEntity> m_CachedNode;
        public string Name { get; }

        public EntityGroup(string name, int capacity, float expire)
        {
            Name = name;
            m_Pool = new SimpleObjectPool<EntityInstance>(false, capacity, expire, 60f);
        }

        // 迭代安全轮询（OnUpdate 中可 Hide 自己）
        public void Update(float dt)
        {
            var current = m_Entities.First;
            while (current != null)
            {
                m_CachedNode = current.Next;
                current.Value.OnUpdate(dt);
                current = m_CachedNode;
                m_CachedNode = null;
            }
        }

        public EntityInstance Spawn(string name) => m_Pool.Spawn();   // 命中复用
        public void Register(EntityInstance o, bool spawned) => m_Pool.Register(o, spawned);
        public void Unspawn(object handle) => m_Pool.Unspawn(handle);

        public void AddEntity(IEntity e) => m_Entities.AddLast(e);
        public void RemoveEntity(IEntity e)
        {
            if (m_CachedNode != null && m_CachedNode.Value == e) m_CachedNode = m_CachedNode.Next;
            m_Entities.Remove(e);
        }
        public IEnumerable<IEntity> All => m_Entities;
        public void PoolUpdate(float dt) => m_Pool.Update(dt);   // 驱动池的过期/容量释放
    }

    public sealed class EntityManager
    {
        private readonly Dictionary<string, EntityGroup> m_Groups = new();
        private readonly Dictionary<int, IEntity> m_Entities = new();
        private readonly IEntityHelper m_Helper;
        private readonly Func<string, object> m_AssetLoader;   // 模拟 Resource 加载预制体
        private int m_Serial;

        public EntityManager(IEntityHelper helper, Func<string, object> assetLoader)
        { m_Helper = helper; m_AssetLoader = assetLoader; }

        public void AddGroup(string name, int capacity, float expire)
            => m_Groups[name] = new EntityGroup(name, capacity, expire);

        public IEntity ShowEntity(string assetName, string groupName, object userData = null)
        {
            var group = m_Groups[groupName];
            int id = ++m_Serial;

            EntityInstance inst = group.Spawn(assetName);    // 路径1: 命中池 → 复用
            bool isNew = false;
            if (inst == null)                                 // 路径2: 未命中 → 加载+实例化
            {
                object asset = m_AssetLoader(assetName);      // 模拟 Resource.LoadAsset
                object handle = m_Helper.InstantiateEntity(asset);
                inst = EntityInstance.Create(assetName, asset, handle, m_Helper);
                group.Register(inst, spawned: true);
                isNew = true;
            }

            var entity = m_Helper.CreateEntity(inst.Target);
            entity.OnInit(id, assetName, inst.Target, isNew);
            group.AddEntity(entity);
            m_Entities[id] = entity;
            entity.OnShow(userData);                          // 两路径汇合
            return entity;
        }

        public void HideEntity(int id, bool isShutdown = false)
        {
            if (!m_Entities.TryGetValue(id, out var e)) return;
            e.OnHide(isShutdown);
            var group = FindGroupOf(e);
            group?.RemoveEntity(e);
            group?.Unspawn(e.Handle);                         // 实例回池(不销毁)
            m_Entities.Remove(id);
        }

        public void Update(float dt)
        {
            foreach (var g in m_Groups.Values) { g.Update(dt); g.PoolUpdate(dt); }
        }

        private EntityGroup FindGroupOf(IEntity e)
        {
            foreach (var g in m_Groups.Values)
                foreach (var x in g.All) if (x == e) return g;
            return null;
        }
    }
}
```

---

## 3. 使用示例

```csharp
sealed class Monster : IEntity
{
    public int Id { get; private set; }
    public object Handle { get; private set; }
    public void OnInit(int id, string a, object h, bool isNew)
    { Id = id; Handle = h; Console.WriteLine($"Init {a} (new={isNew})"); }
    public void OnShow(object u) => Console.WriteLine("怪物出现");
    public void OnHide(bool s) => Console.WriteLine("怪物消失");
    public void OnUpdate(float dt) { }
}

sealed class Helper : IEntityHelper
{
    public object InstantiateEntity(object asset) => new object();
    public void ReleaseEntity(object asset, object inst) => Console.WriteLine("销毁实例");
    public IEntity CreateEntity(object inst) => new Monster();
}

var mgr = new EntityManager(new Helper(), name => $"[Prefab:{name}]");
mgr.AddGroup("Monster", capacity: 5, expire: 30f);

var m1 = mgr.ShowEntity("slime.prefab", "Monster");   // 加载+实例化(new=true)
mgr.HideEntity(m1.Id);                                  // 实例回池
var m2 = mgr.ShowEntity("slime.prefab", "Monster");   // 命中复用(跳过实例化, new=false)
mgr.Update(0.016f);
```

---

## 4. 取舍自检

- ✅ 保留：实体组持对象池、实例复用、Show 命中/加载双路径汇合、Hide 回池不销毁、迭代安全轮询、实例销毁委托 helper、PoolUpdate 驱动过期释放。
- ❌ 砍掉：完整七钩子(留 4)、父子实体树挂接、异步加载(用同步模拟)、Show 进度/失败事件、ShowEntityInfo 中转。
- ⚠️ 提醒：实例(贵)池化、逻辑对象(轻)可重建——别一锅端。Hide 是 Unspawn 回池而非 Destroy，真正销毁由池的容量/过期决定（需 PoolUpdate 驱动）。命中复用路径要跳过资源加载，否则失去池化意义。

---

> 配套考题见 `03_Entity_考题.md`。
