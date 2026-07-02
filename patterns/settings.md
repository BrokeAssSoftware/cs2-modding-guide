# Settings Patterns

```csharp
[FileLocation("ModsSettings\\MyModule\\MyModule")]
[SettingsUIGroupOrder(BalanceGroup, DebugGroup)]
[SettingsUIShowGroupName(BalanceGroup)]
public sealed class Setting : ModSetting
{
    public const string BalanceGroup = "Balance";
    public const string DebugGroup = "Debug";

    public Setting(IMod mod) : base(mod)
    {
        if (Modifier <= 0) SetDefaults();
    }

    public override void SetDefaults()
    {
        Modifier = 1.0f;
        EnableDebug = false;
    }

    [SettingsUISlider(0.1f, 3.0f, 0.1f, Unit.kFloatSingleFraction)]
    [SettingsUISection("Gameplay", BalanceGroup)]
    public float Modifier { get; set; }

    [SettingsUICheckbox]
    [SettingsUISection("Developer", DebugGroup)]
    public bool EnableDebug { get; set; }

    [SettingsUIButton]
    [SettingsUISection("Developer", DebugGroup)]
    public bool ResetDefaults
    {
        set
        {
            SetDefaults();
            AssetDatabase.global.SaveSettings(Mod, this);
        }
    }
}
```

**Highlights**
- Group attributes keep related sliders and toggles organised.
- Reset actions persist defaults immediately after confirmation.
- Constructors guard against zero-initialised values when deserialisation fails.
