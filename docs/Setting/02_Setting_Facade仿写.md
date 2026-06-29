# Setting 持久化设置 · Facade 精简仿写

> 目标：~110 行独立可编译，复刻"空壳 Manager 转发 + 可注入 Helper + Load/Save + 对象序列化 + Shutdown 兜底存盘"。
> 取舍：提供一个基于内存字典 + JSON 的参考 Helper，演示"存储实现可替换"。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `ISettingManager`/`ISettingHelper` 镜像 | 同 | **保留**（解耦核心） |
| `SettingManager` 校验+转发 | 同 | **保留** |
| `SetSettingHelper` 注入 | 同 | 保留 |
| `Shutdown` 自动 Save | 同 | **保留**（崩溃兜底） |
| Get/Set 五类型 + Object | 同 | 保留 |
| PlayerPrefs Helper | 内存字典 + JSON 风格 Helper | 等价替换 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniSetting
{
    public interface ISettingHelper
    {
        int Count { get; }
        bool Load();
        bool Save();
        bool HasSetting(string name);
        bool RemoveSetting(string name);
        void RemoveAllSettings();
        bool GetBool(string name, bool def);   void SetBool(string name, bool v);
        int GetInt(string name, int def);      void SetInt(string name, int v);
        float GetFloat(string name, float def);void SetFloat(string name, float v);
        string GetString(string name, string def); void SetString(string name, string v);
        T GetObject<T>(string name, T def);    void SetObject<T>(string name, T obj);
    }

    // 空壳管理器：校验 + 转发，零业务逻辑
    public sealed class SettingManager
    {
        private ISettingHelper m_Helper;

        public int Count => Ensure().Count;
        public void SetSettingHelper(ISettingHelper helper)
            => m_Helper = helper ?? throw new ArgumentNullException(nameof(helper));

        public bool Load() => Ensure().Load();
        public bool Save() => Ensure().Save();
        public void Shutdown() => Save();      // 关闭即落盘

        public bool HasSetting(string name) => Ensure().HasSetting(Check(name));
        public bool RemoveSetting(string name) => Ensure().RemoveSetting(Check(name));
        public void RemoveAllSettings() => Ensure().RemoveAllSettings();

        public bool GetBool(string name, bool def = false) => Ensure().GetBool(Check(name), def);
        public void SetBool(string name, bool v) => Ensure().SetBool(Check(name), v);
        public int GetInt(string name, int def = 0) => Ensure().GetInt(Check(name), def);
        public void SetInt(string name, int v) => Ensure().SetInt(Check(name), v);
        public float GetFloat(string name, float def = 0f) => Ensure().GetFloat(Check(name), def);
        public void SetFloat(string name, float v) => Ensure().SetFloat(Check(name), v);
        public string GetString(string name, string def = null) => Ensure().GetString(Check(name), def);
        public void SetString(string name, string v) => Ensure().SetString(Check(name), v);
        public T GetObject<T>(string name, T def = default) => Ensure().GetObject(Check(name), def);
        public void SetObject<T>(string name, T obj) => Ensure().SetObject(Check(name), obj);

        private ISettingHelper Ensure()
            => m_Helper ?? throw new InvalidOperationException("Setting helper is invalid.");
        private static string Check(string name)
            => string.IsNullOrEmpty(name) ? throw new ArgumentException("name invalid") : name;
    }

    // 参考 Helper：内存字典 + 可替换的持久化（这里用回调模拟读写存储介质）
    public sealed class MemorySettingHelper : ISettingHelper
    {
        private Dictionary<string, string> m_Data = new(StringComparer.Ordinal);
        private readonly Func<string> m_LoadRaw;        // 从介质读原始串
        private readonly Action<string> m_SaveRaw;      // 写原始串到介质

        public MemorySettingHelper(Func<string> loadRaw, Action<string> saveRaw)
        { m_LoadRaw = loadRaw; m_SaveRaw = saveRaw; }

        public int Count => m_Data.Count;

        public bool Load()
        {
            var raw = m_LoadRaw?.Invoke();
            m_Data = string.IsNullOrEmpty(raw) ? new(StringComparer.Ordinal) : Deserialize(raw);
            return true;
        }
        public bool Save() { m_SaveRaw?.Invoke(Serialize(m_Data)); return true; }

        public bool HasSetting(string name) => m_Data.ContainsKey(name);
        public bool RemoveSetting(string name) => m_Data.Remove(name);
        public void RemoveAllSettings() => m_Data.Clear();

        public bool GetBool(string n, bool def) => m_Data.TryGetValue(n, out var v) && bool.TryParse(v, out var b) ? b : def;
        public void SetBool(string n, bool v) => m_Data[n] = v.ToString();
        public int GetInt(string n, int def) => m_Data.TryGetValue(n, out var v) && int.TryParse(v, out var i) ? i : def;
        public void SetInt(string n, int v) => m_Data[n] = v.ToString();
        public float GetFloat(string n, float def) => m_Data.TryGetValue(n, out var v) && float.TryParse(v, out var f) ? f : def;
        public void SetFloat(string n, float v) => m_Data[n] = v.ToString();
        public string GetString(string n, string def) => m_Data.TryGetValue(n, out var v) ? v : def;
        public void SetString(string n, string v) => m_Data[n] = v;

        public T GetObject<T>(string n, T def)
            => m_Data.TryGetValue(n, out var v) ? JsonLike.FromJson<T>(v) : def;   // 真实项目用 Utility.Json
        public void SetObject<T>(string n, T obj) => m_Data[n] = JsonLike.ToJson(obj);

        // 极简"格式"：key=value 每行一条（演示用，真实项目用 JSON/二进制）
        private static string Serialize(Dictionary<string, string> d)
        {
            var sb = new System.Text.StringBuilder();
            foreach (var kv in d) sb.Append(kv.Key).Append('=').Append(kv.Value).Append('\n');
            return sb.ToString();
        }
        private static Dictionary<string, string> Deserialize(string raw)
        {
            var d = new Dictionary<string, string>(StringComparer.Ordinal);
            foreach (var line in raw.Split('\n'))
            { var i = line.IndexOf('='); if (i > 0) d[line[..i]] = line[(i + 1)..]; }
            return d;
        }
    }

    internal static class JsonLike   // 占位：真实项目替换为框架 Utility.Json
    {
        public static string ToJson<T>(T o) => o?.ToString() ?? "";
        public static T FromJson<T>(string s) => default;
    }
}
```

---

## 3. 使用示例

```csharp
string disk = "";   // 模拟存储介质

var setting = new SettingManager();
setting.SetSettingHelper(new MemorySettingHelper(
    loadRaw: () => disk,
    saveRaw: raw => disk = raw));

setting.Load();
setting.SetFloat("Volume", 0.8f);
setting.SetString("Language", "zh-CN");
setting.SetBool("Fullscreen", true);

float vol = setting.GetFloat("Volume", 1.0f);      // 0.8
string lang = setting.GetString("Language", "en"); // zh-CN
int q = setting.GetInt("Quality", 2);              // 未设过 → 默认 2

setting.Save();          // 落盘到 disk
setting.Shutdown();      // 关闭再兜底 Save 一次
```

---

## 4. 取舍自检

- ✅ 保留：Manager/Helper 镜像解耦、校验+转发、Load/Save、Shutdown 兜底存盘、五类型 + 对象、Helper 可替换。
- ❌ 砍掉：真正的 JSON/二进制序列化（占位）、PlayerPrefs 平台实现、严格版 Get（精简版只保留带默认值的容错版）。
- ⚠️ 提醒：未 Save 的更改在崩溃跳过 Shutdown 时会丢失。关键设置应"Set 即 Save"，非关键可批量延迟。Helper 是替换点——换存储介质只换 Helper 不动 Manager 与上层。

---

> 配套考题见 `03_Setting_考题.md`。
