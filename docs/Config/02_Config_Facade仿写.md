# Config 全局配置 · Facade 精简仿写

> 目标：~90 行独立可编译，复刻"一项四值预解析 + 严格/容错双模读取 + 可空结构体表达缺失"。
> 取舍：复用上一篇 DataTable 仿写里的 DataProvider/Helper 思路，这里聚焦存储与读取语义。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `ConfigData` 四值结构体 | 同 | **保留**（一项四值核心） |
| `Dictionary<string, ConfigData>` Ordinal | 同 | 保留 |
| `ConfigData?` 表达缺失 | 同 | **保留** |
| `Get*(name)` 严格 / `Get*(name,default)` 容错 | 同 | **保留**（双模） |
| `AddConfig(name,string)` TryParse 三型 | 同 | 保留 |
| DataProvider 多源加载 | 注入式 `LoadAndParse(text)` | 简化 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniConfig
{
    public readonly struct ConfigData
    {
        public bool BoolValue { get; }
        public int IntValue { get; }
        public float FloatValue { get; }
        public string StringValue { get; }
        public ConfigData(bool b, int i, float f, string s)
        { BoolValue = b; IntValue = i; FloatValue = f; StringValue = s; }
    }

    public sealed class ConfigManager
    {
        private readonly Dictionary<string, ConfigData> m_Configs = new(StringComparer.Ordinal);
        public int Count => m_Configs.Count;

        // 写入：一次性把字符串 parse 成四种类型固化
        public bool AddConfig(string name, string value)
        {
            bool.TryParse(value, out var b);
            int.TryParse(value, out var i);
            float.TryParse(value, out var f);
            return AddConfig(name, b, i, f, value);
        }

        public bool AddConfig(string name, bool b, int i, float f, string s)
        {
            if (m_Configs.ContainsKey(name)) return false;   // 重名温和拒绝(不抛/不覆盖)
            m_Configs.Add(name, new ConfigData(b, i, f, s));
            return true;
        }

        public bool RemoveConfig(string name) => m_Configs.Remove(name);
        public void RemoveAllConfigs() => m_Configs.Clear();
        public bool HasConfig(string name) => GetConfigData(name).HasValue;

        // 严格读取：缺失抛异常
        public int GetInt(string name)
            => GetConfigData(name)?.IntValue ?? throw NotExist(name);
        public bool GetBool(string name)
            => GetConfigData(name)?.BoolValue ?? throw NotExist(name);
        public float GetFloat(string name)
            => GetConfigData(name)?.FloatValue ?? throw NotExist(name);
        public string GetString(string name)
            => GetConfigData(name) is { } d ? d.StringValue : throw NotExist(name);

        // 容错读取：缺失返回默认值
        public int GetInt(string name, int def)
            => GetConfigData(name)?.IntValue ?? def;
        public bool GetBool(string name, bool def)
            => GetConfigData(name)?.BoolValue ?? def;
        public float GetFloat(string name, float def)
            => GetConfigData(name)?.FloatValue ?? def;
        public string GetString(string name, string def)
            => GetConfigData(name) is { } d ? d.StringValue : def;

        // 注入式加载：解析逻辑外置（对应 DataProvider + Helper）
        public void LoadAndParse(string text, Action<ConfigManager, string> helper)
            => helper(this, text);

        private ConfigData? GetConfigData(string name)
        {
            if (string.IsNullOrEmpty(name)) throw new ArgumentException("name invalid");
            return m_Configs.TryGetValue(name, out var d) ? d : (ConfigData?)null;
        }

        private static Exception NotExist(string name)
            => new InvalidOperationException($"Config '{name}' not exist.");
    }
}
```

---

## 3. 使用示例

```csharp
var cfg = new ConfigManager();

// Helper 决定解析格式：每行 name=value
cfg.LoadAndParse(
    "IsDebug=true\nMaxRetry=3\nVolume=0.8\nServerUrl=https://api.x.com",
    (c, text) =>
    {
        foreach (var line in text.Split('\n'))
        {
            var kv = line.Split('=');
            if (kv.Length == 2) c.AddConfig(kv[0].Trim(), kv[1].Trim());
        }
    });

bool debug = cfg.GetBool("IsDebug");              // true（同一份字符串解读为 bool）
int retry = cfg.GetInt("MaxRetry");                // 3
float vol = cfg.GetFloat("Volume", 1.0f);          // 0.8
string lang = cfg.GetString("Language", "zh-CN");  // 缺失 → 默认 zh-CN（容错模式）
// cfg.GetInt("NotExist");                          // 抛异常（严格模式）
```

---

## 4. 取舍自检

- ✅ 保留：一项四值预解析、Ordinal 字典、`ConfigData?` 表达缺失、严格/容错双模 Get、重名温和拒绝。
- ❌ 砍掉：DataProvider 多源异步加载、静态字节缓存、四个读取事件、IConfigHelper 空接口。
- ⚠️ 提醒：严格版用于"必须存在"的关键配置（缺失即报错暴露 bug），容错版用于"可选"配置（走默认值）。不要图省事全用容错版——会让缺失的关键配置静默走默认值，埋下隐患。

---

> 配套考题见 `03_Config_考题.md`。
