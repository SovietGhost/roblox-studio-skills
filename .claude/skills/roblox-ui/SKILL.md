---
name: roblox-ui
description: Building Roblox user interfaces — ScreenGui setup, responsive scaling across phone/tablet/PC/console, layout objects, safe areas, cross-platform input (UserInputService, ContextActionService, Input Action System, gamepad selection), UI animation, and declarative UI frameworks (React-lua, Fusion, Vide). Use when creating or fixing HUDs, menus, shops, inventories, or input handling.
---

# Roblox UI

## ScreenGui defaults

```lua
gui.ResetOnSpawn = false                      -- keep UI across respawns
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.ScreenInsets = Enum.ScreenInsets.CoreUISafeInsets  -- avoid notches & top bar
gui.IgnoreGuiInset = false
```

UI is client-only. The server can clone ScreenGuis into PlayerGui, but logic lives in client controllers.
Server sends state; UI renders it. UI never decides outcomes (purchases, rewards).

## Responsive layout

- Use **Scale** for size/position of containers; Offset only for small fixed elements (borders, icons) inside them.
- `UIAspectRatioConstraint` to keep shapes; `UISizeConstraint` / `UITextSizeConstraint` to bound sizes.
- `AnchorPoint = Vector2.new(0.5, 0.5)` + `Position = UDim2.fromScale(0.5, 0.5)` to center.
- Layouts: `UIListLayout` (with `Wraps`, `HorizontalFlex`/`VerticalFlex` + `UIFlexItem` for flexbox-style), `UIGridLayout`, `UIPadding`, `UICorner`, `UIStroke`, `UIGradient`.
- `TextScaled = true` + `UITextSizeConstraint(MaxTextSize)`; or compute text size from `AbsoluteSize`.
- Test with Studio's Device Emulator (phone portrait/landscape, tablet, 1080p, console 10-foot UI). Via MCP,
  use `screen_capture` during a playtest; check `workspace.CurrentCamera.ViewportSize`.
- Mobile: touch targets ≥ ~44px, keep center/bottom-right clear for joystick/jump button.

## Input

| API | Use |
|---|---|
| `ContextActionService:BindAction(name, fn, createTouchButton, Enum.KeyCode.E, Enum.KeyCode.ButtonX)` | Gameplay actions with multi-device bindings and auto touch buttons; unbind when context ends |
| `UserInputService.InputBegan` | Raw input; **check `gameProcessedEvent`** to ignore typing in chat/textboxes |
| Input Action System (`InputContext`, `InputAction`, `InputBinding`) | Newer declarative binding system; use if the project already does |
| `UserInputService.PreferredInput` / `LastInputTypeChanged` | Switch button prompts (keyboard ↔ gamepad ↔ touch) |
| `ProximityPrompt` | World interactions — handles all platforms, use `Triggered` on the server |

Gamepad: set `GuiObject.Selectable`, use `GuiService.SelectedObject` to focus first button when a menu opens,
`NextSelectionUp/Down/Left/Right` for custom navigation.

## Animation

`TweenService` on `Position`/`Size`/`BackgroundTransparency`/`UIScale.Scale` (use a `UIScale` child for pop-in
effects instead of tweening Size). Cancel previous tweens on the same property before starting new ones.

## Structure

- One UIController that mounts screens; each screen is a module with `Mount/Unmount` or a component.
- Bind to state changes (attributes on LocalPlayer, a client state store) instead of polling.
- For complex UIs, a declarative framework helps: **React-lua** (`jsdotlua/react` + `react-roblox`), **Fusion**, or **Vide**. Match the project's existing choice; don't mix frameworks.

Example (no framework) — reactive coin label:

```lua
local player = Players.LocalPlayer
local label = gui.HUD.Coins
local function render() label.Text = `{player:GetAttribute("Coins") or 0}` end
player:GetAttributeChangedSignal("Coins"):Connect(render)
render()
```

## Text & localization

- Any user-generated text shown to other players must be filtered on the server (`TextService:FilterStringAsync` → `GetNonChatStringForBroadcastAsync`).
- Use `LocalizationTable`s + automatic translation for strings; avoid baking text into images.
- Use `RichText` for inline styling; escape user content (`<`, `>`, `&`).

## Accessibility / polish

Sufficient contrast, don't rely on color alone, respect `GuiService.PreferredTextSize`/reduced motion settings where available,
sound feedback on button press, hover/press states, disable buttons while a request is pending (prevents double-purchase spam).
