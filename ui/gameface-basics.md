# Gameface Runtime Basics

Coherent Gameface (and its Cohtml engine) powers the Cities: Skylines II UI layer. When you build React modules for Vice & Order, you are ultimately running inside a Chromium-derived runtime with a subset of the web platform. This section outlines how to work with that environment and highlights the pitfalls we have already hit.

## Rendering Pipeline
- The game mounts our bundles through Gameface. React components render into Gameface views just like they would in a browser, but the engine only exposes a curated list of HTML elements, attributes, CSS properties, and JavaScript APIs.
- Launch the game with `--developerMode --uiDeveloperMode` to expose the in-game UI debugger at `https://localhost:9444`. The inspector lets you inspect DOM nodes, tweak styles, and monitor console output in real time.
- All assets are served from `coui://` URLs. Webpack should emit CSS, images, and JS into the mod deploy directory so Gameface can load them.

## Supported Layout Primitives
- **Flexbox works** and is the preferred layout model.
- **CSS Grid is not supported.** Any `display: grid` rules compile but are ignored at runtime. Replace grids with flex layouts or manually calculated columns.
- Avoid shorthand like `grid-template-columns: repeat(auto-fit, minmax(260px, 1fr))`; Gameface logs a syntax error when it encounters the `1fr` token.

## Functions and Sizing Helpers
- `min()` / `max()` / `clamp()` are not parsed reliably. Use breakpoint-specific widths and percentages instead of relying on math helpers.
- `backdrop-filter` is ignored. Layer semi-transparent gradients or texture overlays to simulate blur or frosted glass.
- Shorthand `background: currentColor` fails to parse—supply explicit RGBA values (or variables that resolve to RGBA) when drawing pseudo elements or icon fallbacks.

## Unsupported CSS & Selectors
- Composite layout helpers such as `gap` and `inline-flex` are dropped during parsing. Build spacing with utility classes or `> * + *` patterns, and stick to `display: flex`.
- Asset helpers like `object-fit`, `background: currentColor`, and logical shorthands (`inset`, `list-style`) do not resolve. Use absolute positioning, explicit edges, or custom pseudo-elements instead.
- Bullet lists should be rendered as custom structures (`role="list"` / `role="listitem"`) with manual markers; rely on CSS pseudo-elements instead of `list-style`.

## Pseudo Classes
- Stick to widely supported selectors (`:hover`, `:focus`, `:active`) and provide visual focus states manually.
- Advanced selectors (`:focus-visible`, `:disabled`, `:not()`, `:is()`) are rejected, so toggle helper classes from React instead of leaning on pseudo logic.

## JavaScript Availability
- Gameface ships a pared-down JS runtime. Many modern browser APIs exist, but some pieces are missing.
- `Intl.DateTimeFormat` and related internationalisation helpers are stripped. Format dates/times manually or use preformatted strings provided by C#.
- Optional chaining (`obj?.prop`) and other ES2020 language features transpile away, so they are safe to use when compiling through TypeScript.
- The runtime does not include the Fetch API while running within the sandbox. If you must call into C# or native code, rely on the provided `cs2/*` bindings instead of making network calls.

## Logging and Diagnostics
- The UI log file (`UI.log` under `%LocalLow%/Colossal Order/Cities Skylines II/Logs`) surfaces CSS parser warnings and JS errors. Use it to spot unsupported declarations—Gameface prints explicit "Unsupported CSS" entries whenever it discards a rule.
- Keep `SetShowsErrorsInUI(false)` on our `ILog` instances to avoid spamming players, but mirror anything critical to the UI console through the debugger.
- After verifying a build, snapshot the log into `.logs/<timestamp>/` so other contributors can audit warnings without reproducing the run.

## Authoring Guidelines
- Design for flex-first layouts; write helper utilities that abstract column gaps and alignment so we can reuse them across modules.
- Prefer CSS modules for encapsulation. Because Gameface doesn't support every CSS selector, keep modules self-contained and avoid global overrides.
- Use absolutely positioned `<img>` elements when you need `alt` text or runtime swapping, and reserve CSS `background-image` for decorative-only layers.

## References
- Official feature tables: Coherent Gameface documentation - `https://docs.coherent-labs.com/unity-gameface/content_development/supported_features_tables/`
- Cities: Skylines II wiki notes on UI modding - `docs/research/wiki/ui_modding_reference.md`
- Localisation caveats (missing Intl APIs) - `docs/research/wiki/localize_your_mod.md`

Use this page as the living checklist for Gameface quirks. Update it whenever we discover a new limitation or workaround so future contributors and agents can avoid rediscovering the same issues.
