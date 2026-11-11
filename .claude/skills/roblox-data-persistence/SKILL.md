---
name: roblox-data-persistence
description: Persistent and cross-server data in Roblox — DataStoreService best practices, session locking, ProfileStore, retries and request budgets, schema versioning/migrations, ordered leaderboards, MemoryStoreService (sorted maps, queues, hash maps), MessagingService, and GDPR/right-to-erasure. Use for any player data, saving/loading, leaderboards, trading, matchmaking queues, or cross-server features.
---

# Data persistence

## Non-negotiables

1. `pcall` every DataStore call; retry with exponential backoff (e.g. 1s, 2s, 4s, max ~5 tries).
2. **Session locking** — prevent two servers writing the same player (teleport/rejoin race = item dupes).
   Use ProfileStore (recommended if the project allows packages) or implement locking with `UpdateAsync`.
3. Use `UpdateAsync` for writes (atomic read-modify-write); `SetAsync` overwrites blindly.
4. Save on: periodic autosave (every 60–300s), `PlayerRemoving`, and `game:BindToClose` (≈30s budget; save all in parallel with `task.spawn` and wait).
5. Never save before load succeeded — if load failed, don't overwrite with defaults. Kick the player with a "data failed to load, rejoin" message rather than letting them play on blank data.
6. Store **data**, not Instances. Serialize Vector3/Color3/CFrame to tables or numbers.
7. Limits: 4 MB per key value; key names ≤ 50 chars; respect `DataStoreService:GetRequestBudgetForRequestType(Enum.DataStoreRequestType.X)`.
8. Pass `{player.UserId}` to `SetAsync/UpdateAsync` `userIds` param (via `DataStoreSetOptions`/UpdateAsync return) so GDPR erasure tooling works.
9. Studio: enable "Enable Studio Access to API Services" to test; use a separate datastore name (e.g. `PlayerData_DEV`) when `RunService:IsStudio()` to avoid corrupting live data.

## Schema

```lua
local TEMPLATE = table.freeze({
	version = 3,
	coins = 0,
	inventory = {},   -- { [itemId]: count }
	stats = { kills = 0, deaths = 0 },
	purchases = {},   -- { [purchaseId]: true } for ProcessReceipt idempotency (cap size!)
})
```

- Reconcile on load: fill missing keys from TEMPLATE (deep), so adding fields is safe.
- Migrations: `while data.version < TEMPLATE.version do data = MIGRATIONS[data.version](data) end`.
- Keep keys short for large arrays; avoid storing derived values.

## Minimal session-locked load (no library)

```lua
local LOCK_TIMEOUT = 30 * 60
local function load(store: DataStore, userId: number, jobId: string)
	local key = `u_{userId}`
	return pcall(store.UpdateAsync, store, key, function(old)
		old = old or { data = nil, lock = nil }
		local lock = old.lock
		if lock and lock.jobId ~= jobId and os.time() - lock.time < LOCK_TIMEOUT then
			return nil -- abort: another server holds it; caller retries/waits
		end
		old.lock = { jobId = jobId, time = os.time() }
		return old
	end)
end
```

Release the lock on final save (set `lock = nil`), refresh `lock.time` on autosave. Retry acquiring for
several seconds when locked (previous server is still saving). This is subtle — prefer ProfileStore.

## ProfileStore (loleris) usage sketch

```lua
local ProfileStore = require(ServerStorage.Packages.ProfileStore)
local Store = ProfileStore.New("PlayerData", TEMPLATE)
local profiles: { [Player]: any } = {}

local function onPlayerAdded(player: Player)
	local profile = Store:StartSessionAsync(`{player.UserId}`, {
		Cancel = function() return player.Parent ~= Players end,
	})
	if not profile then player:Kick("Data failed to load, please rejoin."); return end
	profile:AddUserId(player.UserId)
	profile:Reconcile()
	profile.OnSessionEnd:Connect(function()
		profiles[player] = nil
		player:Kick("Session ended — please rejoin.")
	end)
	if player.Parent == Players then profiles[player] = profile else profile:EndSession() end
end
Players.PlayerRemoving:Connect(function(p) local pr = profiles[p]; if pr then pr:EndSession() end end)
```

Verify the exact ProfileStore API against the vendored version in the project before using it.

## Leaderboards

`OrderedDataStore`: integer values only; `GetSortedAsync(false, 100)` → pages. Update on save, not every
change; refresh the board every 60s+. For real-time/short-lived boards use MemoryStore SortedMap.

## MemoryStoreService (cross-server, ephemeral, fast)

- `GetSortedMap(name)`: leaderboards, matchmaking pools, server lists. `UpdateAsync` for atomics. Items expire (max 45 days).
- `GetQueue(name)`: matchmaking queues — `AddAsync`, `ReadAsync` + `RemoveAsync(id)` (invisibility timeout).
- `GetHashMap(name)`: key/value cache, global state, auction listings.
- Quota scales with player count; batch reads; pcall + retry.

## MessagingService

Cross-server pub/sub (global announcements, shutdown signals, cross-server trade notifications).
~1 KB message limit, delivery not guaranteed, rate limited — never use for authoritative state; pair with
DataStore/MemoryStore for truth.

## Trading / anti-dupe

Both players' data must change atomically from the server's point of view: lock both sessions (both players
in the same server with live profiles), validate items exist, mutate both, then save both. Log trades.
Never trust client-provided item lists without checking current ownership at commit time.

## GDPR

Handle Right to Erasure requests: `DataStore:RemoveAsync(key)` for the user's keys across all stores
(including ordered stores). Keep a list of all stores keyed by userId.
