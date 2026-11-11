---
name: luau-advanced
description: Advanced Luau language guidance — strict typing, generics, type functions, OOP/class patterns, metatables, modern syntax, the task scheduler, performance idioms, native codegen, buffers, and Parallel Luau (Actors). Use when writing, refactoring, or reviewing any Luau code.
---

# Advanced Luau

## Modern syntax checklist

```lua
--!strict
local x = if cond then a else b                 -- if-expression
count += 1                                       -- compound assignment
for i, v in items do end                         -- generalized iteration (no ipairs/pairs needed)
local msg = `Player {player.Name} has {coins} coins`  -- string interpolation
for _, p in parts do if not p:IsA("BasePart") then continue end end
local frozen = table.freeze({ MAX = 10 })        -- immutable config
local copy = table.clone(t)
local n = table.find(list, value)
local s = string.split("a,b", ",")
```

`ipairs` is still useful when you must stop at the first nil. `pairs` is unnecessary.

## Types

```lua
--!strict
export type Weapon = {
	name: string,
	damage: number,
	cooldown: number,
	tags: { string }?,
}

type Map<K, V> = { [K]: V }
type Callback<T...> = (T...) -> ()

local function first<T>(list: { T }): T?
	return list[1]
end

-- Narrowing
local function f(v: unknown)
	if typeof(v) == "number" then
		-- v: number
	end
end

-- Instance narrowing
local part = workspace:FindFirstChild("Door")
if part and part:IsA("BasePart") then part.Anchored = true end
```

- Use `unknown` for untrusted input (remote args) and narrow with `typeof`. Avoid `any` except at boundaries.
- `typeof(x)` (Roblox types: `"Vector3"`, `"CFrame"`, `"Instance"`) vs `type(x)` (Lua primitives only). Use `typeof`.
- `::` casts: `local hum = model:FindFirstChildOfClass("Humanoid") :: Humanoid?`
- Read-only/write-only props: `{ read name: string }` (new solver).
- `typeof(setmetatable({} :: Fields, Class))` for class instance types.
- Type functions (new solver) allow computing types — use sparingly; keep public APIs readable.

## Class pattern (strict-typed)

```lua
--!strict
local Trove = require(path.to.Trove)

local Enemy = {}
Enemy.__index = Enemy

type Fields = {
	model: Model,
	health: number,
	_trove: typeof(Trove.new()),
}
export type Enemy = typeof(setmetatable({} :: Fields, Enemy))

function Enemy.new(model: Model): Enemy
	local self = setmetatable({
		model = model,
		health = 100,
		_trove = Trove.new(),
	} :: Fields, Enemy)
	self._trove:Add(model)
	return self
end

function Enemy.TakeDamage(self: Enemy, amount: number)
	self.health = math.max(0, self.health - amount)
	if self.health == 0 then self:Destroy() end
end

function Enemy.Destroy(self: Enemy)
	self._trove:Destroy()
end

return Enemy
```

Prefer composition over deep inheritance. Prefer plain modules with functions over classes when there's no per-instance state.

## Task scheduler

| Use | Instead of |
|---|---|
| `task.wait(s)` | `wait()` (throttled, inaccurate) |
| `task.spawn(f, ...)` | `spawn` / `coroutine.wrap` (errors swallowed) |
| `task.defer(f, ...)` | run at end of current resumption cycle (good for avoiding re-entrancy) |
| `task.delay(s, f, ...)` | `delay` |
| `task.cancel(thread)` | flag-polling loops |

- Don't create unbounded `while true do task.wait() end` loops per object — use one `RunService.Heartbeat` loop that iterates a set.
- `RunService` events: `PreRender` (client, before render — camera), `PreAnimation`, `PreSimulation` (before physics), `PostSimulation`/`Heartbeat` (after physics). `RenderStepped` = legacy `PreRender`.
- Signal timing: Workspace.SignalBehavior is `Deferred` by default — handlers run later in the frame, not immediately on fire. Don't rely on immediate side-effects of `:Fire`/property changes.

## Error handling

```lua
local ok, result = pcall(DataStore.GetAsync, DataStore, key) -- avoid closure allocation
if not ok then warn(`GetAsync failed: {result}`) end

local ok2, err = xpcall(fn, debug.traceback)  -- stack traces for logging
```

Every `*Async` call that hits the network (DataStore, HttpService, MarketplaceService, TeleportService,
Players:GetUserIdFromNameAsync, etc.) must be `pcall`ed, ideally with retries + exponential backoff.

## Performance idioms

- Cache globals/services/methods in hot paths: `local v3 = Vector3.new`, `local sqrt = math.sqrt`.
- Compare squared distances; or `(a - b).Magnitude` once, not repeatedly.
- `table.create(n, default)` to preallocate; `table.clear(t)` to reuse.
- Avoid `..` concatenation in loops; use `table.concat` or `buffer`.
- Avoid creating closures/tables per frame in hot loops.
- `--!native` (whole script) or `@native` (per function) enables native code generation on the server and
  for compute-heavy pure Luau. It helps math/loops, not Roblox API calls. Measure before/after.
- `--!optimize 2` is default in live games.
- Use `vector` library (`vector.create`, `vector.dot`) for fast native vector math when appropriate.

## Buffers

`buffer` is a fixed-size byte array — great for networking compression and binary serialization.

```lua
local b = buffer.create(12)
buffer.writef32(b, 0, pos.X); buffer.writef32(b, 4, pos.Y); buffer.writef32(b, 8, pos.Z)
remote:FireServer(b)  -- buffers replicate compactly
```

Server must validate `buffer.len(b)` before reading (reads past the end error).

## Parallel Luau

- Put scripts under an `Actor` instance. Inside, `task.desynchronize()` to run in parallel, `task.synchronize()` before touching most of the DataModel (writes are mostly forbidden in parallel phase).
- `signal:ConnectParallel(fn)` runs handler in parallel phase.
- Communicate across actors with `Actor:SendMessage(topic, ...)` / `actor:BindToMessageParallel(topic, fn)` or `SharedTable`.
- Good for: pathfinding math, procedural generation, raycast-heavy AI perception, terrain noise.
- Bad for: anything dominated by instance writes.
- Split work over N actors (≈ core count, e.g. 4–8), not one actor per NPC by default.

## Gotchas

- `#t` on non-sequential tables is undefined. Track counts yourself.
- Floating `nil` in arrays breaks `#` and `ipairs`.
- `Instance:Destroy()` locks Parent and disconnects connections to *that instance's (and descendants') signals*. It does NOT disconnect connections the instance's owner made to *other* objects' signals (e.g. `RunService.Heartbeat`), and Lua tables/closures still referencing it keep it in memory. Disconnect those and remove references yourself.
- `WaitForChild` without timeout can yield forever; on the server, prefer direct indexing for things that are guaranteed to exist.
- String keys on Instances (`part.Name`) aren't unique — key tables by Instance, not by name.
- `math.random` is fine; use `Random.new(seed)` for reproducible streams.
