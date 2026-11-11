---
name: roblox-networking-security
description: Secure client-server networking in Roblox — RemoteEvent/RemoteFunction/UnreliableRemoteEvent design, argument validation, rate limiting, server-authoritative combat and economy, network ownership, replication rules, anti-exploit patterns, and backdoor detection. Use whenever adding or reviewing remotes, economy, combat, or anything a player could cheat.
---

# Networking & security

## Threat model

Exploiters fully control their client: they can read all replicated code, fire any remote with any
arguments at any rate, call RemoteFunctions, change their character's physics (they own it), teleport,
fly, noclip, and edit their local UI/state. **They cannot** run server code, read ServerStorage/
ServerScriptService, or change server-side values directly (except for physics of parts they own).

## Remote types

| Type | When |
|---|---|
| `RemoteEvent` | Default. Fire-and-forget, ordered, reliable |
| `UnreliableRemoteEvent` | High-frequency, loss-tolerant data (cosmetic VFX, look direction). ~900 byte payload limit, may drop/reorder |
| `RemoteFunction` | Client→server request needing a response. **Never `InvokeClient`** — client can yield forever or error your server thread |

Name remotes by intent (`RequestPurchase`, `RequestAttack`), group them in `ReplicatedStorage.Remotes`.

## Validation template

```lua
--!strict
local RATE = { window = 1, max = 10 }
local calls: { [Player]: { t: number, n: number } } = {}

local function rateLimited(player: Player): boolean
	local now = os.clock()
	local c = calls[player]
	if not c or now - c.t > RATE.window then
		calls[player] = { t = now, n = 1 }
		return false
	end
	c.n += 1
	return c.n > RATE.max
end
Players.PlayerRemoving:Connect(function(p) calls[p] = nil end)

Remotes.RequestPurchase.OnServerEvent:Connect(function(player: Player, itemId: unknown, qty: unknown)
	if rateLimited(player) then return end
	if typeof(itemId) ~= "string" or typeof(qty) ~= "number" then return end
	if qty ~= qty or qty < 1 or qty > 99 or qty % 1 ~= 0 then return end -- NaN, range, integer
	local item = ShopConfig[itemId]
	if not item then return end
	ShopService.TryPurchase(player, item, qty) -- server computes price, checks balance, persists
end)
```

Checklist for every handler:
1. **Type** every arg with `typeof` (args are `unknown`). Tables: validate every field, reject metatables, cap size/depth.
2. **NaN/inf**: `x ~= x` catches NaN; `math.abs(x) == math.huge` catches inf. Vector3/CFrame components too.
3. **Range & enum**: check against an allowlist (`local item = Config[id]; if not item then return end`). Never use client strings to index Instances (`workspace[clientName]`); use `FindFirstChild` and then check the result.
4. **Instances from client**: verify `IsA`, `IsDescendantOf(expectedContainer)`, and ownership (is it *their* tool/plot?).
5. **State**: is the player alive, in the right round/zone, holding the tool, not on cooldown (server-tracked)?
6. **Distance**: `(char.HumanoidRootPart.Position - target.Position).Magnitude <= MAX + latencyTolerance`.
7. **Rate**: per-player, per-remote limits; cooldowns stored server-side.
8. **Strings**: cap length; filter any user text shown to others with `TextService:FilterStringAsync` (or rely on TextChatService for chat).
9. Fail silently or log; don't kick on a single anomaly (lag causes false positives) — accumulate a suspicion score.

## Server-authoritative patterns

- **Economy**: client requests; server owns price, balance, inventory. Never accept amounts/prices from the client.
- **Combat**: client sends "attack with tool X toward direction D" + optionally the target it thinks it hit. Server checks cooldown, range, line of sight (raycast), and applies damage. See `roblox-gameplay-systems` for lag-compensated hit validation.
- **Projectiles**: server decides spawn & damage; client renders visuals (fast-cast style). Validate fire rate and origin near the character.
- **Movement**: client owns its character physics. Detect speed/teleport hacks server-side by sampling HumanoidRootPart position deltas over time vs WalkSpeed + tolerance; correct by resetting position, not instant kicks.
- **Touched**: run Touched handlers on the server for gameplay effects; verify the touching part belongs to a live character.
- **Tools**: `Tool.Activated` fires on the client; use a remote for the server effect and validate it.

## Network ownership

- Unanchored parts near a player may be owned by that client → they can move them arbitrarily.
- For gameplay-critical physics (vehicles' damage, physics puzzles, NPCs): `part:SetNetworkOwner(nil)` (server) or validate results.
- Vehicles: give ownership to the driver for smoothness, but validate speed/position server-side.

## Replication facts

- Client changes to Workspace don't replicate (FilteringEnabled is always on), **except** physics of owned parts, character animations (Animator), and some Humanoid state.
- `ReplicatedStorage` module source is visible to clients. Don't put secret logic there.
- Server→client: properties/attributes replicate automatically; avoid per-frame remotes when attributes suffice.
- Remotes throttle around ~50 KB/s per client; batch and compress (buffers) for high-frequency data.

## Backdoor / malicious model scan

grep for: `require(%d`, `getfenv`, `setfenv`, `loadstring`, `string.reverse`, `\\%d%d%d` (escaped bytes),
`HttpService` calls to unknown domains, `MarketplaceService:PromptPurchase` in unexpected scripts,
`InsertService:LoadAsset`, scripts named like engine objects ("Weld", "Script ", "Vaccine"), scripts in
odd places (inside meshes, lighting). Report findings; don't silently delete user content without saying.

## Secrets

Use `HttpService:GetSecret("name")` (configured in Creator Dashboard) for API keys. Never hardcode keys
in scripts, never in ReplicatedStorage.
