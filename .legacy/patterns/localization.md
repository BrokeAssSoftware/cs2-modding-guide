# Localisation Helper Pattern

```csharp
using Colossal.Localization;

public sealed class LocaleEN : IDictionarySource
{
    private readonly Setting _setting;

    public LocaleEN(Setting setting) => _setting = setting;

    public string name => "MyModule Locale (EN)";

    public IEnumerable<KeyValuePair<string, string>> Pairs
    {
        get
        {
            yield return Map(_setting.GetOptionLabelLocaleID(nameof(Setting.Modifier)), "Simulation modifier");
            yield return Map(_setting.GetOptionDescLocaleID(nameof(Setting.Modifier)), "Scales all simulated values driven by this module.");
        }
    }

    public void Unload() { }

    private static KeyValuePair<string, string> Map(string key, string value) => new(key, value);
}
```

**Highlights**
- Localisation keys derive from the settings class, ensuring labels and descriptions stay in sync.
- Additional locales inherit from the same pattern and override only the translated strings.
- `Unload()` can remain empty when dictionaries are static.
