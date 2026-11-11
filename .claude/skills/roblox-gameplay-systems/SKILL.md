---
name: roblox-gameplay-systems
description: Building core gameplay systems in Roblox — combat and lag-compensated hit validation, hitboxes with spatial queries/shapecasts, projectiles, abilities and cooldowns, status effects, character control, Animator and animation events, physics constraints, NPC AI with PathfindingService, tweening/VFX, sounds, and camera. Use when implementing or debugging gameplay mechanics.
---

# Gameplay systems

## Combat flow (server authoritative)

```
Client: input → play local anim/VFX immediately (feel) → Remotes.RequestAttack:FireServer(weaponId, aimDir)
Server: validate (alive, has weapon, cooldown, rate) → compute hitbox at server's view → apply damage → replicate result
Clients: receive hit result (remote or attribute change) → hit VFX, damage numbers
```

### Hit detection options

| Method | API | Use |
|---|---|---|
| Raycast | `workspace:Raycast(origin, dir, params)` | Hitscan guns, line of sight |
| Shapecast | `workspace:Blockcast(cf, size, dir, params)`, `Spherecast(pos, radius, dir, params)`, `Shapecast(part, dir, params)` | Thick swings, fast melee sweeps |
| Overlap | `workspace:GetPartBoundsInBox(cf, size, overlapParams)`, `GetPartBoundsInRadius`, `GetPartsInPart` | Instant AoE / melee box |
| Touched | `BasePart.Touched` | Avoid for combat (unreliable, client-owned physics); fine for pickups/zones |

```lua
local params = OverlapParams.new()
params.FilterType = Enum.RaycastFilterType.Exclude
params.FilterDescendantsInstances = { attackerCharacter }
local hitHumanoids: { [Humanoid]: true } = {}
for _, part in workspace:GetPartBoundsInBox(hrp.CFrame * CFrame.new(0, 0, -3), Vector3.new(5, 5, 6), params) do
	local model = part:FindFirstAncestorOfClass("Model")
	local hum = model and model:FindFirstChildOfClass("Humanoid")
	if hum and hum.Health > 0 and not hitHumanoids[hum] then
		hitHumanoids[hum] = true -- dedupe multi-part hits
	end
end
```

### Lag compensation

For fast shooters, the client may report the target it hit. Server validates against a short position
history (store each character's HRP position every Heartbeat for ~1s; rewind to `now - playerPing`
(`player:GetNetworkPing()`)) and accepts if within tolerance + line of sight is clear. Never accept
damage values from the client.

### Cooldowns & abilities

- Store cooldowns server-side: `lastUsed[player][abilityId] = os.clock()`. Client mirrors for UI only.
- Ability definitions in a frozen shared config (`cooldown`, `damage`, `range`, `animId`); server reads config, client reads the same for UI.
- Status effects (stun, slow, burn) as server-owned data with expiry; replicate via attributes (`char:SetAttribute("Stunned", true)`) so clients and other systems can react. Stack rules explicit.

### Damage

Centralize in one `DamageService.Apply(source, target, amount, kind)` — handles teams/friendly fire, i-frames,
shields, kill credit (`creator` tag / attribute), and fires a signal for kill feeds/stats.

## Projectiles

Server: validate fire rate/origin, simulate with raycast steps per Heartbeat (`pos += vel*dt`, `vel += gravity*dt`,
raycast segment). Clients: render a visual-only projectile from the same params (remote broadcast with
`workspace:GetServerTimeNow()` timestamp so they can fast-forward). Pool visual parts.

## Characters

- `Humanoid` properties (WalkSpeed, JumpPower/JumpHeight) — set on server; client owns physics so verify.
- Use `Animator` (`humanoid:FindFirstChildOfClass("Animator")`) → `animator:LoadAnimation(anim)`; cache tracks per character; `track:GetMarkerReachedSignal("Hit")` for hit frames (play on client for responsiveness; server uses timing from config).
- Animations played on the client by the owner replicate to others via Animator.
- Custom movement: `Humanoid:Move`, `ControllerManager` + `GroundController/AirController` for physics-based custom controllers.
- `StarterPlayer.CharacterUseJumpPower`, `HumanoidDescription` + `Humanoid:ApplyDescription` for avatar customization (server).
- Ragdoll: disable `Humanoid.BreakJointsOnDeath`, swap Motor6D for BallSocketConstraints on death.

## Physics & movers

Use constraints: `LinearVelocity`, `AngularVelocity`, `AlignPosition`, `AlignOrientation`, `VectorForce`,
`HingeConstraint`, `SpringConstraint`, `RopeConstraint`. BodyVelocity/BodyGyro/BodyPosition are deprecated.
Doors/elevators: anchored + `TweenService` on server (or client-side tweens of a server-authoritative state for smoothness).

## NPC AI

```lua
local path = PathfindingService:CreatePath({ AgentRadius = 2, AgentHeight = 5, AgentCanJump = true,
	Costs = { Water = 20, DangerZone = math.huge } })
local ok = pcall(path.ComputeAsync, path, npcRoot.Position, goal)
if ok and path.Status == Enum.PathStatus.Success then
	for _, wp in path:GetWaypoints() do
		if wp.Action == Enum.PathWaypointAction.Jump then humanoid.Jump = true end
		humanoid:MoveTo(wp.Position)
		humanoid.MoveToFinished:Wait() -- add timeout; MoveTo auto-times out after 8s
	end
end
```

- Recompute on `path.Blocked`; throttle recomputation (not every frame).
- State machine per NPC (`Idle → Patrol → Chase → Attack → Return`), ticked by one central AI loop at 5–10 Hz, not per-NPC `while` loops.
- `npcRoot:SetNetworkOwner(nil)` so the server simulates NPCs (prevents player-owned NPC exploits & jitter on handoff).
- Perception: distance check first (cheap), then raycast for LOS. Many NPCs → Parallel Luau for perception.
- PathfindingModifier / PathfindingLink instances for designer-authored costs and jumps/ladders.

## Tweening & VFX

- `TweenService:Create(inst, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), { ... }):Play()`.
- Cosmetic VFX should run on clients (broadcast an event; each client spawns the effect) — server spawning particles costs replication.
- `ParticleEmitter:Emit(n)` for bursts. Clean up with `Debris:AddItem` or Trove/task.delay + Destroy.

## Sound

Use `SoundService` + `SoundGroup`s for volume categories; 3D sounds parented to parts; `Sound:Play()` on the client for local feedback. Use the newer audio API (`AudioPlayer`, `AudioEmitter`, `AudioListener`, `Wire`) when the project already does.

## Camera

Client only. `workspace.CurrentCamera`; set `CameraType = Scriptable` for cutscenes; update in `RunService:BindToRenderStep("Cam", Enum.RenderPriority.Camera.Value + 1, fn)`. Restore `Custom` afterwards.

## Zones/triggers

Prefer spatial queries polled at ~5–10 Hz (`GetPartBoundsInBox` over the zone, filtered to characters) over Touched/TouchEnded, which flicker.
