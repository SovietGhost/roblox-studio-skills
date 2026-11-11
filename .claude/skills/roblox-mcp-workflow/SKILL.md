---
name: roblox-mcp-workflow
description: How to drive Roblox Studio through the Roblox Studio MCP tools — discovering the DataModel, editing scripts, building instances safely with execute_luau, playtesting, reading console output, and verifying changes. Use for any task that reads or changes a place in Studio.
---

# Roblox Studio MCP workflow

All game work happens through `mcp__Roblox_Studio__*` tools. Follow this loop: **Discover → Plan → Edit → Playtest → Verify → Stop**.

## 0. Connect

1. `list_roblox_studios` → pick the `studio_id` (ask the user if several are open and it's ambiguous).
2. `get_studio_state` → is it in Edit or Play? Which datamodels exist (`Edit`, `Server`, `Client`)?
3. If a playtest is running and you need to edit scripts, `start_stop_play(is_start=false)` first.

## 1. Discover (never skip)

| Need | Tool |
|---|---|
| Overall layout | `search_game_tree(datamodel_type="Edit", max_depth=3)` |
| All scripts | `search_game_tree(instance_type="LuaSourceContainer")` |
| Find a symbol / remote / API use | `script_grep(query="OnServerEvent")` (50-match cap — narrow the query) |
| Find scripts by name/topic | `script_search` |
| Read code | `script_read(target_file="game.ServerScriptService.X")` — read in parallel |
| Properties/attributes of one instance | `inspect_instance(path=...)` |
| Large/complex questions | `subagent(subagent_type="explore", ...)` — Studio-side explore agent |

Before writing new code, grep for existing patterns: `require(`, `RemoteEvent`, `DataStoreService`, `ProcessReceipt`, `Trove`, `Signal`. Reuse what's there.

## 2. Edit scripts — `multi_edit`

- Works only on `datamodel_type="Edit"`.
- Existing script: `old_string` must match exactly (including indentation/tabs). Read the script first.
- New script: pass `className` (`Script` | `LocalScript` | `ModuleScript`) and a first edit with `old_string=""`.
  Parent folders must exist — create them with `execute_luau` first if needed.
- One `multi_edit` per script; batch all changes to that script in one call (atomic).
- Prefer editing modules over adding loose scripts.

## 3. Build / mutate instances — `execute_luau`

Runs on Studio's main thread in the chosen datamodel. Rules:

- Return a value (string/table) so you get feedback: `return #workspace:GetDescendants()`.
- Wrap destructive work: check the target exists, print what you changed.
- **Chunk bulk work**: ~500–1000 instances per chunk, `task.wait()` between chunks, never per item.
- Build each chunk in an unparented `Folder`/`Model`, then parent that chunk. Parenting a huge subtree in one write freezes Studio.
- Any polling loop must `task.wait()` inside it.
- Use `ChangeHistoryService` for undoable edit-mode changes:

```lua
local CHS = game:GetService("ChangeHistoryService")
local id = CHS:TryBeginRecording("Add spawn pads")
-- ... changes ...
if id then CHS:FinishRecording(id, Enum.FinishRecordingOperation.Commit) end
```

- To set script source, prefer `multi_edit`. (`Script.Source` is writable from execute_luau in Edit, but
  `multi_edit` is safer and diffable.)
- Remotes: create them in Edit mode under `ReplicatedStorage.Remotes` *or* have the server create them at
  runtime — be consistent with the existing project.

## 4. Playtest & verify

1. `start_stop_play(is_start=true)`; then `get_studio_state` until `Server` and `Client` are available.
2. `get_console_output` — look for errors, warnings, `Infinite yield possible`, stack traces.
3. Inspect runtime state:
   - Server: `execute_luau(datamodel_type="Server", code="return game.Players:GetPlayers()[1]:GetAttribute('Coins')")`
   - Client: `execute_luau(datamodel_type="Client", ...)` for UI/local state.
   - `search_game_tree(datamodel_type="Client", path="Players")` to confirm PlayerGui contents.
4. Exercise the feature:
   - `character_navigation(datamodel_type="Client", instance_path="game.Workspace.Shop")` to walk somewhere / trigger Touched.
   - `user_keyboard_input` / `user_mouse_input` to press keys and click UI.
   - Fire remotes from the Client datamodel to simulate a player (and to simulate an exploiter sending bad args!).
5. `screen_capture` for visual confirmation of builds/UI.
6. `start_stop_play(is_start=false)` when done. Always leave Studio in Edit mode unless asked otherwise.

Console output accumulates — note the last line before an action, or print a unique marker (`print("[VERIFY-1]")`) to separate runs.

## 5. Assets & generation

- `search_asset` / `insert_asset` for Creator Store models. Inspect inserted models for scripts
  (`search_game_tree(path=..., instance_type="LuaSourceContainer")`) — free models can contain backdoors
  (`require(<number>)`, `getfenv`, `loadstring`, obfuscated strings). Delete or audit them.
- `generate_mesh`, `generate_material`, `generate_texture`, `generate_procedural_model`, `generate_scene`,
  `generate_animation`, `generate_avatar` for AI generation. Use `wait_job_finished` for async jobs.

## 6. Saving

`save_place` only with explicit user approval. Never publish.

## Common failures

| Symptom | Cause / fix |
|---|---|
| `Edit` datamodel rejected | A playtest is running — stop it or use `Server`/`Client` |
| `multi_edit` "old_string not found" | Whitespace mismatch — re-read the script and copy exactly |
| MCP connection drops | `execute_luau` didn't yield — chunk the work |
| Changes vanish after playtest | You edited the `Server`/`Client` datamodel; those are discarded on stop. Edit in `Edit` |
| `Infinite yield possible on WaitForChild` | Name/path wrong, or object created by the other side later; check creation order |
