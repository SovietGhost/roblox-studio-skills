---
name: roblox-architecture
description: Structuring Roblox experiences — client/server split, single-bootstrap service/controller pattern, module lifecycle (Init/Start), dependency order, cleanup with Trove/Janitor, signals, state management, tags & attributes, and optional Rojo/Wally/Rokit toolchain. Use when starting a new game, adding a major system, or refactoring a messy place.
---

# Roblox game architecture

## Where code runs

| Container | Runs on | Replicated to client? | Use for |
|---|---|---|---|
| ServerScriptService | Server | No | All server logic |
| ServerStorage | — (storage) | No | Server-only assets/modules, secrets-adjacent config |
| ReplicatedStorage | both (require) | Yes | Shared modules, remotes, assets for cloning |
| ReplicatedFirst | Client, first | Yes | Loading screen only |
| StarterPlayerScripts | Client | Yes | Client bootstrap & controllers |
| StarterCharacterScripts | Client (per spawn) | Yes | Per-character effects; prefer controllers instead |
| StarterGui | Client (cloned to PlayerGui) | Yes | UI |

**Anything in ReplicatedStorage/Workspace is readable by exploiters** — including ModuleScript source. Keep
secrets, anti-cheat thresholds you don't want known, and server logic in ServerScriptService/ServerStorage.

## Single-bootstrap pattern

```
ServerScriptService/Server/
  Main (Script)
  Services/ PlayerDataService, CombatService, ShopService ... (ModuleScripts)
StarterPlayerScripts/Client/
  Main (LocalScript)
  Controllers/ InputController, UIController, CameraController ...
ReplicatedStorage/Shared/
  Config/, Types, Util/, Net (remote definitions)
```

Service module contract:

```lua
--!strict
local CombatService = {}

function CombatService.Init()   -- create state, remotes; NO cross-service calls that assume others started
end

function CombatService.Start()  -- connect events, start loops; other services are Init'ed
end

return CombatService
```

Bootstrap:

```lua
--!strict
local services = {}
for _, m in script.Parent.Services:GetChildren() do
	if m:IsA("ModuleScript") then services[m.Name] = require(m) end
end
for name, s in services do
	if s.Init then
		local ok, err = xpcall(s.Init, debug.traceback)
		if not ok then warn(`[Init] {name} failed: {err}`) end
	end
end
for name, s in services do
	if s.Start then task.spawn(s.Start) end
end
```

- Services `require` each other directly (avoid circular requires — split shared state into a third module, or resolve lazily inside functions).
- Knit is no longer maintained by its author; prefer plain modules like above. Don't introduce a framework into a project that doesn't have one without asking.

## Player lifecycle (handle players who joined before your code ran)

```lua
local function onPlayerAdded(player: Player) ... end
Players.PlayerAdded:Connect(onPlayerAdded)
for _, p in Players:GetPlayers() do task.spawn(onPlayerAdded, p) end
Players.PlayerRemoving:Connect(onPlayerRemoving)
```

Character: `player.CharacterAdded` + check `player.Character` already present. Clean up per-character
connections on `CharacterRemoving` or `Humanoid.Died`.

## Cleanup

Use a Trove/Janitor/Maid object per lifetime (player session, character, round, UI screen):

```lua
local trove = Trove.new()
trove:Connect(humanoid.Died, onDied)
trove:Add(effectPart)
trove:Add(task.spawn(loop))  -- threads are cancelled
-- later
trove:Destroy()
```

Memory leaks in Roblox are almost always: connections never disconnected, tables keyed by Player/Instance never cleared on removal, or threads looping forever.

## State

- Server is source of truth. Replicate to client via: attributes on Player/instances (simple values),
  ValueObjects (legacy), or a remote that sends state snapshots/diffs (complex state).
- Shared data (e.g. tables of items) → one module in `Shared/Config` with `table.freeze`.
- Use `CollectionService` tags to bind behavior to instances: `CollectionService:GetInstanceAddedSignal("Door")` + iterate `GetTagged`. Behavior lives in one service, not a script per door.
- Use Attributes for per-instance config (`door:GetAttribute("OpenTime")`) so designers can tweak in Studio.

## Signals

Use a Lua signal implementation (GoodSignal / sleitnick Signal) for in-process events; avoid BindableEvents
(they copy tables and strip metatables). Expose events on services: `CombatService.PlayerKilled = Signal.new()`.

## Promises (optional)

evaera's Promise is useful for chains of async calls with timeouts/cancellation. Don't mix styles randomly — pick one per project.

## Common libraries (Wally)

Trove/Janitor (cleanup), Signal, Promise, t (runtime type checks), ProfileStore (data), Zap/Blink/ByteNet
(typed networking via buffers), React-lua/Fusion/Vide (UI), Jecs/Matter (ECS), Jest-Lua (testing).
Only add a dependency when it solves a real problem; vendor into `ReplicatedStorage.Packages`.

## Optional file-based toolchain

If the user wants code on disk/git: Rokit (tool manager) → Rojo (sync), Wally (packages), Selene (lint),
StyLua (format), luau-lsp (types), Lune (scripts/CI), Darklua (bundling). A `default.project.json` maps
folders to services. When Rojo is active, edit files on disk and let Rojo sync — don't edit synced scripts in
Studio via MCP (Rojo overwrites them). Ask before converting a Studio-only project.

## Round / match games

Model the game loop as an explicit state machine on the server (`Intermission → Starting → InRound → Ending`),
replicate the state + end time (`workspace:GetServerTimeNow()` based) via attributes so clients render countdowns
without per-second remote spam.
