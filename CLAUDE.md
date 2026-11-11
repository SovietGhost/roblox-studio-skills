# Roblox Studio Development — Project Guide

This workspace is for advanced Roblox experience development with Claude Code, driving Roblox Studio
directly through the **Roblox Studio MCP server** (`mcp__Roblox_Studio__*` tools). The game itself lives
in Studio (the DataModel), not on disk, unless a Rojo project is added later.

## Golden rules

1. **Server is authoritative.** Never trust the client. Every RemoteEvent/RemoteFunction handler validates
   type, range, rate, and permission. Clients send *intent* ("I want to swing"), never *results* ("I hit X for 50").
2. **Look before you edit.** Use `search_game_tree`, `script_grep`, `script_read` to understand existing
   structure before changing anything. Match existing conventions.
3. **Verify every change.** After edits: start a playtest, read `get_console_output`, inspect state on the
   `Server`/`Client` datamodels, then stop the playtest. A change is not "done" until it ran without errors.
4. **Never freeze Studio.** `execute_luau` runs on Studio's main thread. Chunk bulk work and `task.wait()`
   per chunk (~1000 items). Build large subtrees outside the DataModel and parent in small batches.
5. **Data safety first.** DataStore code must use `UpdateAsync`/session locking, retries with backoff,
   `pcall` around every `*Async` call, and save on `PlayerRemoving` + `game:BindToClose`.
6. **Purchases are idempotent.** `ProcessReceipt` returns `PurchaseGranted` only after the grant is persisted.
7. **Do not save the place** (`save_place`) or publish without explicit user approval.

## Code conventions (Luau)

- `--!strict` at the top of every new ModuleScript; `--!strict` or `--!nonstrict` on Scripts. Type all public APIs.
- Use the `task` library (`task.spawn`, `task.defer`, `task.delay`, `task.wait`, `task.cancel`). Never `wait()`, `spawn()`, `delay()`.
- Get services once at the top: `local Players = game:GetService("Players")`.
- PascalCase for modules/classes/services, camelCase for locals/functions, SCREAMING_SNAKE for constants.
- Every `:Connect` that can outlive its owner is tracked and disconnected (Trove/Janitor pattern or a cleanup table).
- Use modern APIs: `workspace:Raycast`/`Blockcast`/`Spherecast`, `GetPartBoundsInBox`, `LinearVelocity`/`AlignPosition`
  (not BodyMovers), `Animator:LoadAnimation` (not `Humanoid:LoadAnimation`), `TextChatService`, attributes + CollectionService tags.
- Don't use `Instance.new(class, parent)`: set properties first, then set `Parent` last.
- Script `RunContext` may be used for scripts outside the standard containers, but prefer standard containers.

## Default DataModel layout

```
ReplicatedStorage/
  Shared/            -- ModuleScripts used by both sides (types, config, pure utils)
  Packages/          -- third-party libs (Trove, Signal, Promise, etc.)
  Remotes/           -- RemoteEvents / RemoteFunctions / UnreliableRemoteEvents (created by server)
  Assets/            -- models, effects, sounds cloned at runtime
ServerScriptService/
  Server/            -- Bootstrap Script + Services/ (ModuleScripts, server-only)
ServerStorage/       -- server-only assets & modules (never replicated)
StarterPlayer/
  StarterPlayerScripts/Client/   -- Bootstrap LocalScript + Controllers/
  StarterCharacterScripts/       -- per-character scripts (use sparingly)
StarterGui/          -- ScreenGuis (ResetOnSpawn = false unless needed)
```

One `Script` + one `LocalScript` bootstrap that require and `Init`/`Start` service modules — avoid dozens of loose scripts.

## Skills in this repo (`.claude/skills/`)

| Skill | Use when |
|---|---|
| `roblox-mcp-workflow` | Any task that touches Studio via MCP — editing scripts, building, playtesting, verifying |
| `luau-advanced` | Writing/refactoring Luau: strict types, generics, OOP, performance idioms, Parallel Luau, buffers |
| `roblox-architecture` | Structuring a game: services/controllers, bootstrap, module patterns, lifecycle, cleanup |
| `roblox-networking-security` | Remotes, validation, rate limiting, anti-exploit, replication, network ownership |
| `roblox-data-persistence` | DataStores, session locking, ProfileStore, MemoryStore, migrations, GDPR |
| `roblox-performance` | Lag, memory, frame time, streaming, MicroProfiler, instance budgets |
| `roblox-gameplay-systems` | Combat/hitboxes, abilities, physics, characters, animation, NPC AI, pathfinding |
| `roblox-ui` | ScreenGuis, responsive layout, cross-platform input, UI frameworks (React-lua/Fusion/Vide) |
| `roblox-monetization` | Developer products, game passes, ProcessReceipt, subscriptions, premium payouts |

## Subagents (`.claude/agents/`)

| Agent | Purpose |
|---|---|
| `roblox-security-auditor` | Audits every remote & server handler for exploitable trust issues. Read-only. |
| `luau-code-reviewer` | Reviews Luau for bugs, leaks, type issues, deprecated APIs. Read-only. |
| `roblox-playtester` | Runs a playtest, exercises a feature, reads console/server state, reports pass/fail. |
| `roblox-perf-profiler` | Measures instance counts, script activity, memory, part/physics load; recommends fixes. |

Use them when the user asks for an audit/review/test, or after a large feature lands (security + review in parallel).

## MCP quick reference

- Always call `list_roblox_studios` first if `studio_id` is unknown, then `get_studio_state`.
- `datamodel_type`: `Edit` when stopped; `Server`/`Client` during a playtest (`Edit` is rejected while playing).
- Script edits (`multi_edit`) only work in `Edit` mode — stop the playtest before editing.
- Paths use dot notation: `game.ServerScriptService.Server.Services.DataService`.
