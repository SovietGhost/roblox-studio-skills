---
name: roblox-performance
description: Diagnosing and fixing Roblox performance problems — frame time, server heartbeat, memory, network bandwidth, instance and part counts, physics, rendering, StreamingEnabled, MicroProfiler and Developer Console usage, and measuring via execute_luau. Use when the game lags, memory grows, the server Heartbeat drops, or before shipping a large map/system.
---

# Roblox performance

## Measure first (via MCP)

Run during a playtest on the `Server` and `Client` datamodels:

```lua
local Stats = game:GetService("Stats")
local RunService = game:GetService("RunService")
local t, n = 0, 0
local conn = RunService.Heartbeat:Connect(function(dt) t += dt; n += 1 end)
task.wait(5); conn:Disconnect()
return {
	avgFps = n / t,
	memMB = Stats:GetTotalMemoryUsageMb(),
	instances = Stats.InstanceCount,
	parts = #workspace:GetDescendants(), -- rough; filter BaseParts if needed
	primitives = Stats.PrimitivesCount,
	movingPrimitives = Stats.MovingPrimitivesCount,
	dataSendKbps = Stats.DataSendKbps,
	dataReceiveKbps = Stats.DataReceiveKbps,
}
```

Per-category memory: `Stats:GetMemoryUsageMbForTag(Enum.DeveloperMemoryTag.Script)` (also `PhysicsParts`,
`GraphicsTexture`, `Instances`, `Signals`, `LuaHeap`, …). `gcinfo()` gives Lua heap KB.
Tell the user to use the MicroProfiler (Ctrl+F6) and Developer Console (F9) for frame-level detail; label
your own hot code with `debug.profilebegin("Name")` / `debug.profileend()`.

## Common culprits & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Server Heartbeat < 60 | Many per-object loops/scripts; heavy Touched handlers; unanchored physics | One central loop over a set; anchor static parts; CollectionService instead of script-per-object |
| Memory climbs over time | Leaked connections/tables/threads | Trove per lifetime; clear `[player]` tables on PlayerRemoving; check `DeveloperMemoryTag.Signals` growth |
| Client FPS low | Too many parts/draw calls, transparency, particles, shadows, high-poly meshes | Merge/MeshPart, `CastShadow=false` on small parts, `RenderFidelity=Automatic`, limit ParticleEmitter rates, LOD via streaming |
| Network lag | Remote spam, large payloads, many moving physics parts | Batch/throttle, buffers, UnreliableRemoteEvent for cosmetic, attributes for state, anchor/sleep parts |
| Spikes on spawn | Cloning large models synchronously | Pre-clone pools, spread over frames, `task.defer` |
| Long join time | Huge map without streaming | `Workspace.StreamingEnabled = true` |

## StreamingEnabled

- Enable for large maps. Client only has nearby parts → **client code must handle instances not existing**
  (`WaitForChild` with timeout, or `CollectionService` added/removed signals).
- `Model.ModelStreamingMode`: `Atomic` (stream whole model together), `Persistent` (always on client — use sparingly), `PersistentPerPlayer`, `Nonatomic`.
- `StreamingTargetRadius`/`StreamingMinRadius` tune load distance. `player:RequestStreamAroundAsync(pos)` before teleports.
- Server sees everything; streaming only affects clients.

## Physics

- Anchor everything that doesn't move. Use `CanCollide=false`, `CanQuery=false`, `CanTouch=false` on decorative parts (reduces broadphase & touch work).
- Use `CollisionGroups` (PhysicsService:RegisterCollisionGroup) instead of many filtered Touched checks.
- Prefer constraint movers (`LinearVelocity`, `AlignPosition`, `AlignOrientation`) over per-frame CFrame writes of unanchored parts.
- For many moving visual objects, animate on the client with CFrame on anchored parts (or `workspace:BulkMoveTo`).

## Scripts

- Avoid `while task.wait() do` per instance; avoid `RenderStepped` work that isn't render-related.
- Don't call `GetDescendants`/`FindFirstChild` chains in hot loops — cache references.
- Spatial queries: `workspace:GetPartBoundsInRadius` with `OverlapParams` filter is cheaper than looping all parts.
- Object pooling for projectiles/VFX: reuse parts (move to far CFrame / parent nil) instead of create/destroy each shot.
- `--!native` for compute-heavy modules (measure).
- Parallel Luau (Actors) for parallelizable compute (see `luau-advanced`).

## Rendering budgets (rules of thumb)

- Mobile matters: target low-end devices. Keep visible part count modest (low thousands), limit dynamic lights (PointLight/SpotLight with Shadows), limit transparent overlapping surfaces, limit `SurfaceGui`/`BillboardGui` counts (or `MaxDistance`).
- Textures: reuse; prefer smaller resolutions; `MaterialVariant` instead of many unique textures.
- Use `Lighting.Technology = Future` only if the visual benefit justifies the cost; `ShadowMap` is a common balance.

## Report format

When asked to profile, report: measured numbers (before), top 3–5 causes with evidence (counts, paths),
concrete fixes ranked by impact/effort, and numbers after if you applied fixes.
