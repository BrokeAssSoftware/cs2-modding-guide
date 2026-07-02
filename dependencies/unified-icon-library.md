# Unified Icon Library

Reference mod: `UnifiedIconLibrary`.

Unified Icon Library (UIL) provides a curated catalogue of SVG assets that match the Cities: Skylines II visual language. Using UIL keeps Vice & Order overlays consistent, avoids duplicate art bundles, and gives contributors (human or AI) a predictable way to find icons. This guide covers dependency setup, runtime guards, URI usage, recoloring, and how to supply custom artwork when UIL does not have what we need.

## Quick Start Checklist
- Install UIL locally and declare it as a required dependency in every module that renders icons (`PublishConfiguration.xml`, `mod.json`, README).
- During `OnLoad`, confirm the UIL assembly is present. If it is missing, fall back to text or neutral icons and log a warning.
- Reference library icons with the URI pattern `coui://uil/<Style>/<IconName>.svg`.
- Use the style set (`Standard`, `Dark`, `Colored`) that matches your UI palette; rely on CSS variables for tinting.
- Only ship custom icons when UIL cannot cover the use case; register them under your own COUI host to prevent collisions.

## Wiring the Dependency

### Declare UIL Everywhere
- **Publish configuration**
  ```xml
  <Publish>
    ...
    <Dependency Id="74417" DisplayName="Unified Icon Library" />
  </Publish>
  ```
- **UI module `mod.json`**
  ```json
  {
    "id": "VNO.UI",
    "dependencies": ["algernon.UnifiedIconLibrary"]
  }
  ```
  Verify the dependency ID against the current UIL release (case-sensitive).
- **Documentation** - mark UIL as "Required" in module READMEs and release notes so players understand the dependency chain.

### Runtime Guard
Metadata alone does not guarantee UIL loads before our code. Add a quick check:

```csharp
using System;
using System.Linq;
using Colossal.Logging;

private static readonly ILog Log = LogManager.GetLogger("VNO.UI").SetShowsErrorsInUI(false);

internal static bool IsUilAvailable()
{
    var assemblyPresent = AppDomain.CurrentDomain
        .GetAssemblies()
        .Any(a => a.GetName().Name.Equals("UnifiedIconLibrary", StringComparison.OrdinalIgnoreCase));

    if (!assemblyPresent)
    {
        Log.Warn("Unified Icon Library missing. Falling back to text labels.");
    }

    return assemblyPresent;
}
```

Wrap any icon-dependent feature behind `IsUilAvailable()` and substitute text or placeholder art when it returns `false`.

## Referencing Library Icons

UIL pre-registers every SVG with the Gameface runtime. Use the URI format:

```
coui://uil/<Style>/<IconName>.svg
```

Available styles:
- `Standard` - matches vanilla UI chrome.
- `Dark` - higher-contrast dark theme.
- `Colored` - multi-colour variants with baked gradients.

### Gameface / HTML Example
```html
<uil-icon
  class="toolbar-button__icon"
  src="coui://uil/Standard/ArrowLeft.svg"
  aria-hidden="true" />
```

### React Example
```tsx
export const SquadBadge: React.FC = () => (
  <img
    className="squad-badge__icon"
    src="coui://uil/Colored/StarFilled.svg"
    alt=""
  />
);
```

### C# Const Example
```csharp
internal static class IconRefs
{
    internal const string Dispatch = "coui://uil/Colored/PoliceBadge.svg";
}
```

Use constants when pushing icon URIs through ECS notifications or DTOs so both the native and UI layers stay in sync.

## Recoloring and Theme Alignment
- **Tint coloured sets** - UIL exposes CSS custom properties (`--uil-red`, `--uil-blue`, etc.). Override them for contextual colouring:
  ```css
  .warning-icon { color: var(--uil-red); }
  ```
- **Single-tone icons** - apply CSS `filter` or `fill` overrides to `Standard`/`Dark` icons. Keep colours within the tone palette defined in `docs/vision/index.md`.
- **Accessibility** - maintain a minimum 4.5:1 contrast ratio. UIL defaults meet this threshold; re-validate when you apply custom tints.

## Discovering Icons
- Browse the repository previews (https://github.com/algernon-A/UnifiedIconLibrary/tree/master/Properties/Previews).
- Programmatically list icons to drive validation scripts:
  ```powershell
  Invoke-RestMethod `
    -Uri 'https://api.github.com/repos/algernon-A/UnifiedIconLibrary/contents/Icons/Standard' |
    Select-Object -ExpandProperty name
  ```
- Cache the list in CI and fail builds when a referenced icon does not exist.

## Shipping Custom Icons
Only add bespoke SVGs when UIL lacks a suitable asset.

1. **Place files**
   ```
   vno-ui/
     Icons/
       Standard/ViceLedger.svg
       Dark/ViceLedger.svg
   ```
2. **Register a COUI host**
   ```csharp
   using Game.SceneFlow;
   using Game.UI;
   using Colossal.Logging;

   internal static class VnoIconHost
   {
       private const string HostKey = "vno-icons";
       private static readonly ILog Log = LogManager.GetLogger("VNO.UI");

       internal static void Mount()
       {
           if (!GameManager.instance.modManager.TryGetExecutableAsset(typeof(VnoIconHost).Assembly, out var asset))
           {
               Log.Error("Unable to locate module path for icons.");
               return;
           }

           var iconPath = Path.Combine(Path.GetDirectoryName(asset.path)!, "Icons");
           UIManager.defaultUISystem.AddHostLocation(HostKey, iconPath, shouldWatch: true);
       }

       internal static void Unmount()
       {
           UIManager.defaultUISystem.RemoveHostLocation(HostKey);
       }
   }
   ```
   Call `VnoIconHost.Mount()` during `OnLoad` and `VnoIconHost.Unmount()` during `OnDispose`.
3. **Reference the icons**
   ```
   coui://vno-icons/Standard/ViceLedger.svg
   ```
4. **Bundle via MSBuild**
   ```xml
   <ItemGroup>
     <Content Include="Icons\**\*.svg">
       <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
     </Content>
   </ItemGroup>
   ```
5. **Promote shared art** - if multiple modules rely on the same custom icon, migrate it to a shared dependency (for example ExtraLib) instead of duplicating files.

## QA and Troubleshooting
- **Broken icon** - double-check the URI, style, and dependency presence. Temporarily enable `SetShowsErrorsInUI(true)` to surface missing asset warnings in-game.
- **Dependency warning in UI** - ensure `PublishConfiguration.xml` and Paradox Mods metadata list UIL. The launcher only auto-installs declared dependencies.
- **Colour mismatch** - confirm you are overriding a valid token (`--uil-red`, `--uil-grey`, etc.). Typos silently revert to defaults.
- **COUI host conflicts** - use a unique host key (`vno-icons`) when registering custom bundles to avoid clashing with other mods.
- **Large SVGs** - keep icons under roughly 50 KB. Run `svgo` or similar optimisers in CI to strip metadata and shrink paths.

## References
- Unified Icon Library repo: <https://github.com/algernon-A/UnifiedIconLibrary>
- ExtraLib icon helpers: [Shared Library: ExtraLib](shared-library-extra.md)
- Gameface and React pipeline: [React Pipeline](../ui/react-pipeline/overview.md)
- Tone and accessibility guidance: `docs/vision/index.md`

Following these steps ensures both humans and automation can source icons quickly, maintain visual cohesion, and extend the library safely when our scenarios demand new art.

