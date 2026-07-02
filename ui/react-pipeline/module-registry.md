# Module Registry

Gameface resolves modules through the `ModRegistrar` exported from your UI entry point:

```ts
import { ModRegistrar } from "cs2/modding";
import { registerVicePanel } from "./vice-panel";

const registrar: ModRegistrar = (registry) => {
  registry.append("Game.UI.GameMenu", registerVicePanel);
};

export default registrar;
```

- Use `registry.append` to add content, `registry.extend` to wrap existing components, and `registry.override` to replace them entirely.
- Inspect the registry with `registry.find(/pattern/)` or via the Gameface inspector to avoid overriding unexpected modules.
- Log registrations during init so you can confirm hooks executed.
