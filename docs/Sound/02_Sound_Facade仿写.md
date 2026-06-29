# Sound 声音 · Facade 精简仿写

> 目标：~130 行，复刻"组 + 固定声道代理 + 优先级抢占分配 + 异步加载过时取消 + 两级音量"。
> 取舍：AudioSource 用接口模拟；淡变简化为瞬时；聚焦抢占分配与过时取消两个难点。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| 组 + 固定数 SoundAgent | 同 | **保留** |
| 优先级抢占 + 同级保护 | 同 | **保留**（核心） |
| 异步加载 + SerialId 取消 | 同 | **保留**（过时取消） |
| 组音量 × 组内音量 | 同 | **保留** |
| 淡入淡出 | 注释 | 简化 |
| PlaySoundParams IReference | 普通参数 | 简化 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniSound
{
    // 声道实现注入点（真实项目包 AudioSource）
    public interface ISoundAgentHelper
    {
        void SetSound(object clip);
        void Play(float volume);
        void Stop();
        bool IsPlaying { get; }
    }

    public sealed class SoundAgent
    {
        private readonly ISoundAgentHelper m_Helper;
        public int SerialId { get; private set; }
        public int Priority { get; private set; }
        public float VolumeInSoundGroup { get; set; } = 1f;
        public bool IsPlaying => m_Helper.IsPlaying;

        public SoundAgent(ISoundAgentHelper helper) => m_Helper = helper;

        public void Play(int serialId, int priority, object clip, float groupVolume)
        {
            SerialId = serialId; Priority = priority;
            m_Helper.SetSound(clip);
            m_Helper.Play(groupVolume * VolumeInSoundGroup);   // 两级音量
        }
        public void Stop() { m_Helper.Stop(); SerialId = 0; }
    }

    public sealed class SoundGroup
    {
        public string Name { get; }
        public float Volume { get; set; } = 1f;
        public bool Mute { get; set; }
        public bool AvoidBeingReplacedBySamePriority { get; set; }
        private readonly List<SoundAgent> m_Agents = new();

        public SoundGroup(string name) => Name = name;
        public void AddAgent(SoundAgent a) => m_Agents.Add(a);

        // 声道分配：空闲优先，否则优先级抢占
        public SoundAgent PickAgent(int priority)
        {
            foreach (var a in m_Agents)             // 1. 找空闲
                if (!a.IsPlaying) return a;

            SoundAgent candidate = null;            // 2. 找可抢占的最低优先级
            foreach (var a in m_Agents)
            {
                if (a.Priority > priority) continue;                 // 比新声音高，不能抢
                if (a.Priority == priority && AvoidBeingReplacedBySamePriority) continue;  // 同级受保护
                if (candidate == null || a.Priority < candidate.Priority) candidate = a;
            }
            return candidate;   // null = 无可用声道，播放失败
        }

        public SoundAgent FindBySerialId(int serialId)
        {
            foreach (var a in m_Agents) if (a.SerialId == serialId && a.IsPlaying) return a;
            return null;
        }
    }

    public sealed class SoundManager
    {
        private readonly Dictionary<string, SoundGroup> m_Groups = new();
        private readonly Func<string, Action<object>, Action> m_AsyncLoader;  // (asset, onLoaded) → 返回(无)
        private readonly HashSet<int> m_LoadingSerialIds = new();   // 加载中追踪
        private int m_Serial;

        public SoundManager(Func<string, Action<object>, Action> asyncLoader) => m_AsyncLoader = asyncLoader;
        public void AddGroup(string name) => m_Groups[name] = new SoundGroup(name);
        public void AddAgent(string group, ISoundAgentHelper helper) => m_Groups[group].AddAgent(new SoundAgent(helper));
        public bool IsLoadingSound(int serialId) => m_LoadingSerialIds.Contains(serialId);

        public int PlaySound(string asset, string groupName, int priority = 0)
        {
            var group = m_Groups[groupName];
            int serialId = ++m_Serial;
            m_LoadingSerialIds.Add(serialId);          // 进加载中追踪

            m_AsyncLoader(asset, clip =>               // 异步加载回调
            {
                if (!m_LoadingSerialIds.Remove(serialId))
                    return;                            // 过时取消：已被 Stop → 不播放
                var agent = group.PickAgent(priority); // 分配声道(空闲/抢占)
                if (agent == null) { Console.WriteLine("无可用声道"); return; }
                if (agent.IsPlaying) agent.Stop();     // 抢占：停掉旧声音
                agent.Play(serialId, priority, clip, group.Mute ? 0f : group.Volume);
            });
            return serialId;
        }

        public bool StopSound(int serialId)
        {
            if (m_LoadingSerialIds.Remove(serialId)) return true;   // 还在加载 → 标记取消
            foreach (var g in m_Groups.Values)
            {
                var a = g.FindBySerialId(serialId);
                if (a != null) { a.Stop(); return true; }
            }
            return false;
        }
    }
}
```

---

## 3. 使用示例

```csharp
sealed class FakeAudio : ISoundAgentHelper
{
    public bool IsPlaying { get; private set; }
    public void SetSound(object clip) { }
    public void Play(float v) { IsPlaying = true; Console.WriteLine($"播放 音量{v}"); }
    public void Stop() { IsPlaying = false; }
}

// 模拟异步加载：立即回调（真实项目走 Resource）
var mgr = new SoundManager((asset, onLoaded) => { onLoaded(new object()); return () => { }; });
mgr.AddGroup("SFX");
mgr.AddAgent("SFX", new FakeAudio());
mgr.AddAgent("SFX", new FakeAudio());   // 2 个声道

int id1 = mgr.PlaySound("hit.wav", "SFX", priority: 0);
int id2 = mgr.PlaySound("explode.wav", "SFX", priority: 5);
int id3 = mgr.PlaySound("crit.wav", "SFX", priority: 10);   // 声道满 → 抢占 priority=0 的 hit
mgr.StopSound(id2);
```

---

## 4. 取舍自检

- ✅ 保留：组 + 固定声道、空闲优先/优先级抢占/同级保护、异步加载 + SerialId 过时取消、组音量×组内音量两级、IsLoadingSound 追踪。
- ❌ 砍掉：淡入淡出插值、3D 空间音频参数、PlaySoundParams 池化、PauseSound/ResumeSound、AudioMixer 总线。
- ⚠️ 提醒：异步加载回调里**必须检查 SerialId 是否已被取消**（过时取消），否则切场景后过时 BGM 才加载完仍会播放。声道满时的策略（抢占/拒绝）要明确。两级音量别合并成一级。

---

> 配套考题见 `03_Sound_考题.md`。
