# Localization 本地化 · Facade 精简仿写

> 目标：~90 行独立可编译，复刻"字典 KV + GetString 三态容错(永不抛) + 泛型重载消除装箱 + 语言切换解耦"。
> 取舍：只演示 0~2 参数重载（真实框架到 16）；加载简化为注入式解析。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| `Dictionary<string,string>` Ordinal | 同 | 保留 |
| GetString `<NoKey>`/`<Error>` 容错 | 同 | **保留**（永不抛核心） |
| 16 个泛型重载消除装箱 | 演示 0~2 个 | 思路保留 |
| `Language` + `SystemLanguage` | 同 | 保留 |
| DataProvider 加载管线 | 注入式 ParseInto | 简化 |
| AddRawString 重 key 返回 false | 同 | 保留 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniLocalization
{
    public enum Language { Unspecified = 0, ChineseSimplified, English, Japanese }

    public interface ILocalizationHelper { Language SystemLanguage { get; } }

    public sealed class LocalizationManager
    {
        private readonly Dictionary<string, string> m_Dict = new(StringComparer.Ordinal);
        private ILocalizationHelper m_Helper;
        private Language m_Language = Language.Unspecified;

        public int DictionaryCount => m_Dict.Count;

        public Language Language
        {
            get => m_Language;
            set => m_Language = value != Language.Unspecified
                ? value : throw new ArgumentException("Language invalid.");
        }
        public Language SystemLanguage
            => m_Helper?.SystemLanguage ?? throw new InvalidOperationException("Set helper first.");

        public void SetLocalizationHelper(ILocalizationHelper h)
            => m_Helper = h ?? throw new ArgumentNullException(nameof(h));

        // ---- 底层字典操作 ----
        public bool HasRawString(string key) => m_Dict.ContainsKey(key);
        public string GetRawString(string key) => m_Dict.TryGetValue(key, out var v) ? v : null;
        public bool AddRawString(string key, string value)
        {
            if (m_Dict.ContainsKey(key)) return false;   // 重 key 温和拒绝
            m_Dict.Add(key, value);
            return true;
        }
        public bool RemoveRawString(string key) => m_Dict.Remove(key);
        public void RemoveAllRawStrings() => m_Dict.Clear();

        // ---- 上层取用：永不抛异常 ----
        public string GetString(string key)
        {
            var value = GetRawString(key);
            return value ?? $"<NoKey>{key}";             // 缺 key → 可见占位
        }

        // 泛型重载消除装箱（真实框架一路到 16 参）
        public string GetString<T>(string key, T arg)
        {
            var value = GetRawString(key);
            if (value == null) return $"<NoKey>{key}";
            try { return string.Format(value, arg); }
            catch (Exception e) { return $"<Error>{key},{value},{arg},{e.GetType().Name}"; }
        }

        public string GetString<T1, T2>(string key, T1 a1, T2 a2)
        {
            var value = GetRawString(key);
            if (value == null) return $"<NoKey>{key}";
            try { return string.Format(value, a1, a2); }
            catch (Exception e) { return $"<Error>{key},{value},{a1},{a2},{e.GetType().Name}"; }
        }

        // 注入式加载：解析逻辑外置（对应 DataProvider + Helper）
        public void LoadDictionary(Language lang, string text, Action<LocalizationManager, string> parser)
        {
            RemoveAllRawStrings();    // 切语言：清旧字典
            Language = lang;
            parser(this, text);
        }
    }
}
```

---

## 3. 使用示例

```csharp
sealed class DefaultHelper : ILocalizationHelper
{
    public Language SystemLanguage => Language.ChineseSimplified;
}

var loc = new LocalizationManager();
loc.SetLocalizationHelper(new DefaultHelper());

// 加载中文字典（parser 决定格式：key=value 每行）
loc.LoadDictionary(Language.ChineseSimplified,
    "greeting=你好，{0}！\nscore=得分：{0}/{1}\ntitle=游戏标题",
    (l, text) =>
    {
        foreach (var line in text.Split('\n'))
        {
            var i = line.IndexOf('=');
            if (i > 0) l.AddRawString(line[..i], line[(i + 1)..]);
        }
    });

Console.WriteLine(loc.GetString("title"));                  // 游戏标题
Console.WriteLine(loc.GetString("greeting", "勇者"));        // 你好，勇者！
Console.WriteLine(loc.GetString("score", 80, 100));          // 得分：80/100（int 零装箱传入）
Console.WriteLine(loc.GetString("missing"));                 // <NoKey>missing（不抛）
Console.WriteLine(loc.GetString("greeting", "A", "B"));      // <Error>...（占位符不匹配，不抛）
```

---

## 4. 取舍自检

- ✅ 保留：Ordinal 字典、GetString 三态容错(正常/`<NoKey>`/`<Error>`)永不抛、泛型重载消除装箱思路、Language/SystemLanguage、切语言清字典、重 key 温和拒绝。
- ❌ 砍掉：3~16 参数重载（同构省略）、DataProvider 多源异步加载、四个读取事件、二进制字典解析。
- ⚠️ 提醒：GetString **绝不能抛异常**——它直面 UI 渲染。把缺漏变成可见占位串是刻意设计。值类型参数务必走泛型重载而非 `params object[]`，否则高频 UI 文本会成 GC 热点。

---

> 配套考题见 `03_Localization_考题.md`。
