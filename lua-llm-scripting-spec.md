# ValidusBot Lua Scripting Spec for LLMs

## Purpose
This document is a single-source reference for generating Lua scripts for ValidusBot.
It is designed for LLM usage (ChatGPT/Claude/etc.) and includes:
- runtime assumptions,
- available core libraries,
- known constraints,
- safe scripting rules,
- output quality rules for generated scripts.

Important: No document can guarantee 100% perfect code in all scenarios. This file is intended to maximize correctness and safety.

## Will The LLM Know Everything?
Short answer: it will know what is explicitly documented here.

For highest accuracy:
1. Attach this file.
2. Instruct the LLM to use only APIs documented in this file.
3. Instruct the LLM to treat undocumented APIs as unavailable.
4. Ask for strict argument validation and nil-safe logic.

This file includes a curated behavior guide plus an auto-generated public API
appendix derived from `docs/Scripts/core`. Every listed function has explicit
parameter and return types; structured values use the exact named records in
this document. The contracts were cross-checked against the Lua wrappers and
native binding validators instead of trusting legacy annotations alone. The
appendix deliberately omits local helpers, compatibility aliases, raw bindings,
protocol details, and bootstrap internals.

## Runtime Model

- Lua runs cooperatively on the bot/game thread. Managed coroutines provide
  yielding and fairness; they are not background OS threads. Game reads/actions,
  script callbacks, and Lua state access therefore remain serialized.
- The runtime loads `Scripts/core` before the user file. Generated scripts should
  use the canonical PascalCase globals from this document, such as `Self`,
  `Creature`, `Map`, `Cavebot`, `Module`, `Storage`, and `Engine`.
- With the sandbox enabled, each user file and required user module has a private
  `_ENV`. `_G` points to that private environment. Supported native/core globals
  are available through a restricted fallback, but user assignments do not
  modify the shared core environment.
- The top-level script body is a managed coroutine. After it finishes, the
  runtime resolves and starts `init()` from the same private `_ENV`, if present.
  `init()` is also managed and may call `wait`, HTTP, or WebSocket APIs.
- `Module` callbacks and scheduled callbacks run as managed coroutines and may
  yield cooperatively. Registered event/hook callbacks also use managed
  coroutines, but they are latency-sensitive producers: they must be
  constant-time, non-yielding, and return immediately after copying only the
  minimum scalar data into a coalesced slot or explicitly bounded queue.
- `terminate()` is different: it is protected synchronous cleanup, cannot yield,
  has a 50 ms execution limit, and runs at most once. Do not start HTTP,
  WebSocket, scheduled, or module work from it.
- A normal script remains alive while it owns runnable/sleeping coroutines,
  registered callbacks, scheduled events, HUD elements, or WebSockets. A script
  with no remaining work/resources finishes automatically.

### Failure isolation and diagnostics

- `pcall` and `xpcall` retain normal Lua behavior. An expected error caught by
  the script does not count as an uncaught runtime failure.
- An uncaught top-level-body or `init()` error stops only that script.
- An uncaught repeating-module error stops that module. If another healthy
  repeating module exists, it keeps running; if the failed module was the
  script's only module, the runtime stops the script.
- An uncaught `Events.Schedule` error ends only that one-shot callback.
- A key, packet, or Walker event registration is disabled after exactly three
  consecutive uncaught callback failures. One successful completion resets that
  registration's failure counter.
- A Walker interceptor always releases its owned pause before callback failure
  policy is applied, so a broken script cannot leave Walker indefinitely held.
- Supported C++ exceptions from native bindings become contextual Lua errors and
  follow the same component policy. An ordinary C++ exception escaping a script
  tick quarantines that script and does not prevent later scripts from ticking.
  A native access violation is the exceptional emergency case: Lua execution is
  disabled for the client session, Walker ownership is released, and a dump is
  written because continuing through corrupted native state is unsafe.
- Uncaught diagnostics include script, component kind/name, callback source,
  source line, safe error value, traceback, `failure_scope`, `runtime_action`,
  and `policy_reason`. Complete output is written to the per-script log even
  when UI notifications for identical errors are rate-limited.
- Error values may be strings, numbers, tables, userdata, threads, `nil`, or
  values with hostile `__tostring` metamethods. Runtime diagnostics serialize
  them without invoking user metamethods.

Do not wrap an entire repeating module in `pcall` merely to print and continue.
That converts a genuine component failure into an apparent success and prevents
the runtime from stopping the broken module. Catch only errors that the script
can handle meaningfully, then either recover or rethrow with context.

### Ownership, cleanup, and budgets

- Key/packet/Walker events, scheduled events, HUD elements/callbacks, HTTP
  results, WebSockets, sounds, and Walker holds are owned by the creating script.
  Teardown removes or cancels only that owner's resources.
- Explicit cleanup in `terminate()` is useful for restoring feature settings the
  script deliberately changed, but native owner-scoped resources are also
  released automatically.
- Per script: 64 MiB Lua memory, 256 managed coroutines, 64 active-plus-pending event
  callbacks, and 32 outstanding async result tokens.
- Scheduler limits are 3 ms per coroutine resume, 4 ms per script per frame, and
  8 ms for all Lua work per frame. Yieldable Lua is time-sliced. Deadline misses
  remain available as telemetry, while three consecutive unpreempted resumes of
  at least 5 ms or one unpreempted 50 ms resume circuit-breaks the offending
  component.
- Delays are integer milliseconds from 0 through 86,400,000 unless a narrower
  function-specific range is documented.

## Hard Rules for Generated Scripts

1. Use only functions and constants explicitly documented here. Never invent a
   plausible API name; treat an absent function as unavailable.
2. Use canonical PascalCase modules. Do not generate lower-camel compatibility
   calls such as `self.*`, `map.*`, `CaveBot`, `_G.position`, or old
   `engine.ammoRefill.*` paths.
3. Do not busy-loop. Repeating work must use `Module.Every`, `Events.Schedule`,
   or a loop that calls `wait(ms)`.
4. Do not use `os.execute`, `os.getenv`, or `os.tmpname`. `os.exit()` is reserved
   for an explicit emergency full-client shutdown policy. It audit-logs and
   immediately terminates the client without a logout prompt or normal cleanup.
5. Never expose, query, or toggle the reserved Objects Dumper feature through
   `Features`/`Engine.Features`.
6. Validate external input and option-table fields. Treat game-derived objects,
   capabilities, and snapshots as potentially `nil` or stale.
7. Use `pcall` only around an operation whose failure has a defined local
   recovery path. Let unexpected module/event errors reach the runtime policy.
8. Getter tables from `Engine` are detached snapshots. Mutating them does
   nothing; use the matching validated setter.
9. Keep feature changes idempotent. Prefer `Features.SetActive/Enable/Disable`
   over assuming the previous state and toggling blindly.
10. Use `Time.MonotonicMs()` for elapsed time. Never use `os.clock()` for
    wall-time timeouts.
11. Every event/hook callback must follow the critical-path rule below. It must
    be constant-time and non-yielding; it may only capture minimal scalar data
    into a coalesced slot or hard-bounded queue and then return.
12. If the script changes persistent bot configuration, state whether the
    change should remain after the script stops and restore it in `terminate()`
    when appropriate.

## Event And Hook Callback Critical-Path Rule

This rule applies to every callback registered through `Events`, `Hotkeys`,
`Cavebot`/`Walker`, `SharedStorageScope:OnChanged`, an event proxy, HUD
click/drag APIs, or another callback-taking event API.
Every covered callback must be constant-time and non-yielding: capture only the
minimum already-decoded scalar state and return immediately.

The native hook normally validates, copies, and queues a managed event
coroutine; the user callback is resumed later by the serialized Lua scheduler
on the game thread. Yielding releases the game thread while the coroutine is
asleep, but it does **not** finish that event callback. The callback continues
to occupy one of the script's 64 active-plus-pending event slots. A burst of
callbacks that call `wait`, HTTP, WebSocket receive/connect, or another
yielding API can therefore reach the limit and produce
`runtime_action=drop_newest_event_callback`. Synchronous native work executes
while Lua is on the game-thread path and can directly cause frame-time spikes.

An event callback may only:

- validate fields already present in its callback arguments;
- copy the minimum required strings, numbers, and booleans;
- set a flag, sequence number, or monotonic timestamp;
- overwrite one pending/coalesced state slot, or append to an explicitly
  bounded in-memory queue with a deliberate overflow policy;
- optionally issue one documented constant-time, non-blocking game action;
- return immediately.

An event callback must never:

- call `wait`, `Http.*`, `WebSocket.Connect`,
  `WebSocketConnection:Receive`, yielding HUD getters, or sound wait helpers;
- call `Storage`, shared-storage reads/writes/updates, profile save/load,
  `io.*`, filesystem APIs, or other lock/file-backed operations;
- call or invent a `Synchronize`, `RunOnMainThread`, mutex, condition-variable,
  or other cross-thread locking helper. The callback is already dispatched on
  the serialized Lua/game-thread path; synchronizing it again can block frames
  or deadlock;
- encode/decode large JSON, recursively traverse an event table, scan all
  creatures/containers/map tiles, perform pathfinding, or do other
  variable-cost work;
- contain retry, polling, or unbounded loops, or allocate an unbounded queue;
- create one `Events.Schedule`, `Module.After`, module, or coroutine per
  incoming event. That merely moves the backlog to another quota.

Create one stable `Module.Every` worker in `init()`. The callback should feed
the worker through either a single latest-value slot (best for level-triggered
state and duplicate alerts) or a hard-bounded FIFO (only when every event
matters). The worker drains a fixed maximum amount per invocation. Network
waits belong in that worker. Persistent storage and other synchronous work must
also be batched/debounced and kept infrequent because the worker still resumes
Lua on the game thread.

Use native packet filters whenever possible. In particular,
`message_contains` on one incoming
`GameServerOpcodes.GAME_SERVER_TEXT_MESSAGE` registration filters before a
Lua packet table or callback coroutine is allocated.

Walker observers follow the same enqueue-and-return rule. Label/action
interceptors pause Walker until their callback finishes. If a decision truly
requires slower work, call `Cavebot.Defer(timeoutMs)` inside the interceptor,
put the returned handle plus minimal scalar inputs into the bounded worker
queue, return immediately, and call `handle:Complete()` (or `:Cancel()`) on
every expected completion/failure path. Do not call `wait` or HTTP directly in
the interceptor.

### Safe packet-to-worker pattern

```lua
local SCRIPT_ID = "discord_afk_alert"
local WEBHOOK_URL = "REPLACE_WITH_DISCORD_WEBHOOK_URL"
local pendingMessage = nil

Events.RegisterPacketEvent({
    id = SCRIPT_ID,
    packet_id = GameServerOpcodes.GAME_SERVER_TEXT_MESSAGE,
    incoming = true,
    message_contains = "afk check started",
    message_case_sensitive = false,

    callback = function(packet)
        -- Critical path: use the documented direct field, coalesce, return.
        if pendingMessage == nil and type(packet.message) == "string" then
            pendingMessage = packet.message
        end
    end
})

function init()
    Module.Every(SCRIPT_ID .. "_worker", function()
        local message = pendingMessage
        if message == nil then
            return
        end

        pendingMessage = nil
        local response = Http.PostJson(WEBHOOK_URL, {
            content = message
        }, {
            timeoutMs = 8000
        })

        if not response.ok then
            print("Discord request failed: " .. tostring(response.error))
        end
    end, 100)
end
```

## Public Module Naming
- Module tables are exposed for user scripts in PascalCase form.
- Some modules also expose grouped namespaces such as `Cavebot.Walker`, `Cavebot.Actions`, `Cooldowns.Spell`, and `Engine.Healer`.
- Use the direct canonical methods documented here. Do not infer a `Query` or `Actions` namespace unless it is explicitly listed.
- Compatibility aliases are not part of the supported script-writing surface; generated scripts must use the exact names documented in this file.

## Known Constraints

- Hotkeys: alt cannot be used with Events.RegisterKeyEvent/Hotkeys.RegisterCombo, but it is supported by Hotkeys.SendCombo.
- Standard Lua `table.concat` is allowed in sandboxed user scripts and can be used for safe string assembly.
- Extended keys: insert/delete/home/end/pageup/pagedown/arrows default to extended=true in Hotkeys.ParseCombo.
- Features API excludes Objects Dumper from public get/set/list/status paths.
- HTTP and WebSocket operations must be started from a managed script coroutine. They yield cooperatively while waiting.
- HTTP has no callback lifecycle API. WebSockets use explicit `Receive`; there are no `onOpen`, `onClose`, `onError`, `onRedirect`, or automatic reconnect callbacks.
- Use `Time.MonotonicMs()` for elapsed time, retry backoff, timeouts, and
  telemetry cadence. It advances while the process is idle and is unaffected by
  system wall-clock changes. Its epoch is unspecified, so compare two returned
  values. Do not use `os.clock()` for elapsed time; Lua defines it as process CPU
  time.
- Capability wrappers such as Inventory and NPC trade can return nil/false when the corresponding game state is unavailable. Check `IsAvailable`/capability methods where provided.
- Sandboxed `io.open`, `os.remove`, and `os.rename` resolve inside the current
  product's user `Scripts` directory and cannot access `Scripts/core`.
  Individual file reads/writes are limited to 1 MiB. Prefer `Storage` for normal
  script state.
- Sandboxed `require` loads text-only Lua modules from the allowed script/library
  roots under the same restricted environment. Do not depend on native C-module
  loading or DLL search paths.
- `Settings.Save`/`Settings.Load` and `Cavebot.Save`/`Cavebot.Load` are the
  supported profile-file APIs. An explicit path is used exactly once; a bare
  settings filename is searched beside the running script and then in the
  product `Settings` folder, while a bare waypoint filename is searched beside
  the script and then in `Waypoints`. Sandboxed calls remain confined to the
  script tree and the matching product profile folder. Loads are queued until
  Lua dispatch has unwound; loading Scripter settings can stop/restart the
  caller, and loading a waypoint bundle can stop waypoint scripts.
- Current builds do not export `Game.GetMinimapTilePixelColor`.
  `Minimap.GetTilePixelColor` and `Minimap.IsWalkableByColor` are compatibility
  capability calls and return `nil`; use `Minimap.GetTileFlags`,
  `Minimap.IsWalkable`, or `Map` pathfinding.

## Profile File Save/Load

Use `Engine.Settings.Save/Load` (or the equivalent global `Settings` methods)
for `.json` settings and `Cavebot.Save/Load` for `.validuswpt` bundles. Each
returns `true, resolvedPath`; path/queue errors raise a normal Lua error.

```lua
Engine.Settings.Save("mage.json", {
    Engine.Settings.Feature.Healer,
    Engine.Settings.Feature.Targeting,
    Engine.Settings.Feature.Scripter
})

Cavebot.Load("hunt.validuswpt", {
    Cavebot.BundleFeature.Targeting,
    Cavebot.BundleFeature.Looter
})
```

For settings, `features == nil` selects all 19 GUI feature sections. An empty
table selects none; `{ all = true, scripter = false }` selects all except
Scripter. Key Events are always saved/loaded. For waypoint bundles, Walker and
Lure are always included and the optional table selects Targeting, Magic
Shooter, and Looter; `nil` selects none of those optional sections.

A bare load name checks the running script's directory first and the matching
product `Settings`/`Waypoints` folder second. A path containing directories is
used only at that script-relative location; an absolute path is used only as
given. Missing files report every searched location. Bare saves target the
running script's directory (or the matching product folder for an embedded
script). The Lua sandbox still confines profile access to `Scripts` and the
matching product folder; disabling it permits other local paths, but network,
UNC, device, and mapped-remote paths remain rejected.

## Critical Constants (Explicit Values)
Use these exact values to avoid numeric mapping mistakes.

### MoveDirection
- MoveDirection.NORTH = 0
- MoveDirection.EAST = 1
- MoveDirection.SOUTH = 2
- MoveDirection.WEST = 3
- MoveDirection.NORTHEAST = 5
- MoveDirection.SOUTHEAST = 6
- MoveDirection.SOUTHWEST = 7
- MoveDirection.NORTHWEST = 8
- MoveDirection.INVALID = 9

Important:
- Value 4 is not a valid MoveDirection in this runtime.
- For diagonals, use constants from lua_consts.lua (5..8), not hardcoded legacy 4..7 mappings.

### RotateDirection
- RotateDirection.NORTH = 0
- RotateDirection.EAST = 1
- RotateDirection.SOUTH = 2
- RotateDirection.WEST = 3

### PathFindResult
- PathFindResult.OK = 0
- PathFindResult.SAME_POSITION = 1
- PathFindResult.IMPOSSIBLE = 2
- PathFindResult.TOO_FAR = 3
- PathFindResult.NO_WAY = 4
- PathFindResult.GOAL_BLOCKED = 5
- PathFindResult.NONE = 6

### PathFindFlags (bit flags)
- PathFindFlags.ALLOW_NOT_SEEN_TILES = 1
- PathFindFlags.ALLOW_CREATURES = 2
- PathFindFlags.ALLOW_NON_PATHABLE = 4
- PathFindFlags.ALLOW_NON_WALKABLE = 8
- PathFindFlags.IGNORE_CREATURES = 16
- PathFindFlags.IGNORE_GOAL_POSITION = 32
- PathFindFlags.CHECK_GOAL_POSITION = 64
- PathFindFlags.PRIORITIZE_DIAGONAL_MOVEMENTS = 128

The eight flag values and the existing Lua default mask (85: check goal,
allow non-pathable, allow not-seen, ignore creatures) are unchanged. Missing
minimap pixels remain excluded unless allowSpeculativeUnknown is explicitly
true and ALLOW_NOT_SEEN_TILES is set. Existing scripts therefore keep their
old map coverage. PRIORITIZE_DIAGONAL_MOVEMENTS now lowers diagonal search
cost. IGNORE_CREATURES excludes occupied intermediate tiles even when
ALLOW_CREATURES is set; the latter can still permit an occupied goal.
ALLOW_NON_WALKABLE may also admit occupied intermediate tiles when
IGNORE_CREATURES is off. None of these flags bypasses a separate live
movement check.

### CooldownGroupId
- CooldownGroupId.ATTACK = 1
- CooldownGroupId.HEALING = 2
- CooldownGroupId.SUPPORT = 3
- CooldownGroupId.SPECIAL = 4
- CooldownGroupId.CRIPPLING = 5
- CooldownGroupId.FOCUS = 7
- CooldownGroupId.ULTIMATE = 8
- CooldownGroupId.GREAT_BEAMS = 9
- CooldownGroupId.BURST_OF_NATURE = 10
- CooldownGroupId.VIRTUE = 11

## Recorded ImGui UI Windows

UI (also available as Core.UI) creates detached ImGui windows owned by the
calling script. This is a recorded API, not a per-render-frame Lua callback:

1. ValidusBot runs the window builder on the normal game/Lua update thread.
2. The builder records an immutable native widget tree and may safely read
   Self, Game, Engine, and other public game APIs.
3. The render thread replays only that native tree. It never enters Lua, waits
   for a Lua-state lock, or calls a game API.
4. Widget interactions are queued and their callbacks run as managed Lua
   coroutines on the next game update. Callback dispatch and completion both
   invalidate the owning window.

Builders are descriptive and must not call wait or another yielding API.
Container functions take nested builder functions so scripts cannot produce
unbalanced ImGui Begin/End scopes. Do not run persistent game actions from a
builder; use a button or change callback for actions.

Scripts can queue the same native notifications used by ValidusBot with
`UI.Notify(title, message, type, options)`. The type is `success`, `warning`,
`error`, or `info` (the default), and `options.width` is an optional logical
pixel width from 180 through 800; omit it for automatic sizing. Notifications
may be sent from ordinary script code or widget event callbacks, but not from a
window builder because builders can run periodically. Each script may queue up
to eight notifications per ten-second interval; the function returns false
when that limit is reached.

    UI.Notify("Targeting", "Demon rule saved.", "success")

    UI.Button("test_warning", "Test warning", function()
        UI.Notify("Targeting", "No rune configured for this rule.", "warning")
    end)

    local statusWindow = UI.CreateWindow("character_status", "Character Status", function()
        UI.Text("Level: " .. tostring(Self.GetLevel()))
        UI.Text("Health: " .. tostring(Self.GetHealthPercentage()) .. "%")
        UI.Text("Mana: " .. tostring(Self.GetManaPercentage()) .. "%")
    end, {
        size = { x = 330, y = 180 },
        min_size = { x = 260, y = 130 },
        refresh_ms = 100,
    })

The default refresh_ms is 100, so visible dynamic windows rebuild at most ten
times per second. refresh_ms = 0 is dirty-only. Values other than zero must be
from 16 through 60000. Hidden windows do not rebuild. Calling Show() after
hiding or closing a window requests an immediate build. Invalidate() requests a
build on the next eligible Lua update.

The title-bar X hides a window; it does not destroy the registration. Give a
window a managed show/hide hotkey with the `hotkey` option or call
`SetHotkey()` later. That registration remains active while the window is
hidden, so the same key can always reopen a window closed with X. Pass nil to
`SetHotkey` to clear it. `GetHotkeyEventId()` returns the managed event ID for
the native `UI.Hotkey` binding card. Window visibility hotkeys toggle native
presentation state directly: they do not wait for a Lua callback or a window
rebuild, and remain usable when the last recorded build is suspended after an
error.

    local toolsWindow
    toolsWindow = UI.CreateWindow("tools", "Script Tools", function()
        UI.Text("Ctrl+F10 hides or shows this window")
        UI.Hotkey("toggle_key", "SHOW / HIDE WINDOW",
            toolsWindow:GetHotkeyEventId())
    end, {
        hotkey = "ctrl+f10",
        hotkey_options = { name = "Toggle script tools" },
        resizable = true,
    })

    local message = "ready"
    local dirtyWindow

    dirtyWindow = UI.CreateWindow("manual", "Manual Refresh", function()
        UI.Text("State: " .. message)
        UI.Button("use_item", "Use item", function()
            -- Game actions run here on the managed game-thread callback path.
            Game.UseItem(3031)
            message = "item action requested"
        end)
        UI.SameLine()
        UI.Button("refresh", "Refresh", function()
            message = "refreshed at " .. tostring(Time.MonotonicMs())
            dirtyWindow:Invalidate()
        end)
    end, { refresh_ms = 0 })

Controls use stable, non-empty IDs. IDs must be unique within one window tree;
the runtime namespaces them by script owner and window ID, so separate scripts
may use the same IDs safely. Values are passed into controls, and callbacks
receive the new value instead of the control returning an immediate-mode
result. Text and continuous controls commit on Enter/deactivation by default.
Set live = true to receive coalesced editing updates no more than once every
50 ms. Buttons, checkboxes, radio buttons, selections, and combo selection are
immediate.

    local settings = {
        enabled = true,
        label = "hunt",
        distance = 4,
        color = { r = 220, g = 45, b = 55, a = 255 },
    }

    UI.CreateWindow("settings", "Script Settings", function()
        UI.Checkbox("enabled", "Enabled", settings.enabled, function(value)
            settings.enabled = value
        end)
        UI.InputText("label", "Label", settings.label, function(value)
            settings.label = value
        end)
        UI.SliderInt("distance", "Distance", settings.distance, 1, 8, function(value)
            settings.distance = value
        end)
        UI.ColorEdit("accent", "Accent", settings.color, function(value)
            settings.color = value -- r, g, b, a from 0 through 255
        end)
    end)

Sizes and positions use DPI-scaled logical pixels. Colors are
{ r, g, b, a } tables with channels from 0 through 255; a defaults to 255.
Window options are:

- visible, size, min_size, position, resizable, closable;
- opacity from 0.05 through 1.0 (window shell only; recorded content stays crisp);
- decorations (default true) to enable or disable the ValidusBot backdrop art;
- style = { colors = {...}, vars = {...}, palette = {...} };
- on_close = function() ... end;
- refresh_ms;
- hotkey and hotkey_options = { name, trigger_on_keydown }.
- persist_geometry (default false) and an optional persistence_key. When
  enabled, the last dragged position and resized dimensions survive hide/show
  and script reloads for the current ValidusBot runtime. Persistence is scoped
  to the script path and never shared with another script.

Window methods are Show, Hide, Toggle, IsVisible, Focus, Destroy, Invalidate,
SetRefreshInterval, SetTitle, SetSize, SetResizable, SetOpacity,
SetDecorations, SetPosition, SetHotkey, GetHotkeyEventId, SetStyle, SetOnClose,
OpenPopup, and ClosePopup. A resizable
window can be dragged from its native resize grip; `SetSize()` also resizes it
programmatically in logical pixels.

Available recorded content includes text, wrapped/bulleted text, separators,
spacing and same-line layout, buttons, checkboxes, radio buttons, selectable
rows, text/integer/float inputs, sliders, drags, combos, color editors, progress
bars, groups, disabled/style scopes, child panels, tabs, tables, trees,
collapsing headers, popups, modals, and tooltips. Native ValidusBot components
also include icon-font buttons, selectable action-entry rows, editable hotkey cards, item previews, raw client
sprites, curated monster previews, and raw or curated outfit previews. Texture
decoding and outfit composition use the same bounded asynchronous native caches
as the main UI; Lua records only IDs and options.

The extended layout API adds `Spacer`/`Dummy`, exact `Padding`, `Row`, `Column`,
`Center`, `AlignRight`, `Indent`, and `Baseline`. Widget/container options may
use numeric `width`/`height`, or the policies `"auto"`, `"content"`, and
`"fill"`; `min_width`, `max_width`, `aspect_ratio`, and row-child `grow` are
also available. Rows support `gap`, `align`, `justify = "space-between"`, and
`wrap`. The higher-level helpers `SectionHeader`, `MetricCard`, `SearchField`,
`Callout`, `Toolbar`, `SplitPane`, `FilterChip`, `Breadcrumbs`, `PropertyRow`,
and `EmptyState` compose those same recorded nodes and do not add a per-frame
Lua callback.

Interactive game resources are available as `ItemButton`, `TextureButton`,
`MonsterButton`, and `OutfitButton`, plus composed `ItemSlot` and
`EquipmentSlot` helpers. `ReorderList`, `DragSource`, and `DropTarget` use
bounded native ImGui payloads and deliver their final events to Lua on the next
game update. A `ContextMenu` must be recorded immediately after the item it
belongs to.

`Canvas` provides a deliberately bounded custom drawing surface. Its nested
builder may record only `CanvasLine`, `CanvasRect`, `CanvasCircle`,
`CanvasPath`, `CanvasText`, and `CanvasTexture` commands. Coordinates are
DPI-scaled logical pixels and drawing is clipped to the canvas. The canvas
does not expose ImGui draw-list pointers, arbitrary texture handles, callbacks,
or backend state. Item IDs, raw sprite IDs, and curated monster names use the
same bounded asynchronous caches as ordinary texture controls.

    UI.IconButton("reload", "refresh", function()
        toolsWindow:Invalidate()
    end, { tooltip = "Refresh", size = { x = 28, y = 28 } })
    UI.ItemTexture("potion", 268, { size = 54 })
    UI.MonsterTexture("demon", "Demon", { size = 54 })
    UI.OutfitTexture("citizen", LoadOutfitById(128, {
        head = 78, body = 68, legs = 58, feet = 76, addons = "all"
    }), { size = 54 })

Custom style scopes are balanced natively and cannot leak into later native
ValidusBot windows. `colors` controls ordinary ImGui slots, `vars` controls
geometry/spacing, and `palette` controls the ValidusBot-native components used
by recorded windows. Palette slots are Header, Layout, Border, Accent, Section,
Element, FieldActive, FieldInactive, Input, InputHovered, InputActive, Tooltip,
TooltipBorder, CheckboxInactive, CheckboxCircleInactive, Text, TextHover,
TextInactive, TextMuted, Warning, NotifyText, White, Black, HeaderText,
Selection, and Favorite.

    local redStyle = {
        colors = {
            Button = { r = 150, g = 25, b = 35, a = 255 },
            ButtonHovered = { r = 200, g = 40, b = 55, a = 255 },
        },
        vars = {
            FrameRounding = 6,
            ItemSpacing = { x = 8, y = 6 },
        },
    }

    UI.CreateWindow("styled", "Styled Script Window", function()
        UI.Style(redStyle, function()
            UI.Button("action", "Script-only style")
        end)
    end, {
        style = redStyle,
        opacity = 0.82,
        decorations = false,
    })

`opacity` is applied independently from text/widget alpha. When decorations are
enabled, the backdrop art follows the shell opacity. When they are disabled,
the script's `colors.WindowBg` and scoped native palette define the complete
appearance. `SetOpacity()` and `SetDecorations()` update these properties at
runtime. See `ui_test_08_neon_glass_dashboard.lua` for a teal/violet theme with
nested palette scopes, game textures, controls, and live opacity adjustment.

Limits are 8 registered windows per script, 32 visible Lua windows globally,
2,048 nodes and 64 KiB of recorded text per window tree, and 128 pending UI
events per script. UI builds share the ordinary Lua scheduler budget and have a
3 ms per-build limit. A deferred or failed build leaves the last good tree
visible. Failures retry after one second; after three consecutive failures,
periodic rebuilding is suspended and the window shows an error until
Invalidate, Show, or script reload retries it. Registered windows are active
script resources and are removed automatically on destroy, unload, reload, or
script failure.

Raw ImGui pointers, draw lists, backend/font-atlas access, docking,
multi-viewport mutation, and shared-context access are intentionally
unavailable.

## Canonical Types And Exact Table Contracts

This section is authoritative for every `table`, object, callback payload, or
structured option referenced by the API index. A field is required unless its
name ends in `?`. `T[]` means a contiguous 1-based Lua array;
`table<K, V>` means a key/value map. Field names are case-sensitive. Never
guess aliases such as `container.index`, `item.id`, or `item.slot`.

The audit covers every file loaded from `docs/Scripts/core`:
`cavebot.lua`, `cavebot_actions.lua`, `chat_channel.lua`,
`chat_channel_storage.lua`, `client.lua`, `container.lua`, `cooldowns.lua`,
`creature.lua`, `creature_iterators.lua`, `engine.lua`,
`event_proxies.lua`, `features.lua`, `game.lua`, `hotkeys.lua`,
`http.lua`, `hud_wrapper.lua`, `inventory.lua`, `item.lua`,
`json.lua`, `lua_consts.lua`, `map.lua`, `minimap.lua`, `module.lua`,
`npc_trade_storage.lua`, `position.lua`, `self.lua`, `sound.lua`,
`spells.lua`, `storage.lua`, `ui.lua`, `vip.lua`, and `websocket.lua`.
`zz_api_surface.lua` was also audited. It creates additional callable paths:
Pascal-case aliases, `Query`/`Actions` buckets, and `Core.<Module>` mirrors.
`zz_api_surface.lua` adds no new behavior or independent contract. To keep the index
deterministic, Appendix A lists only the canonical function once; generated
scripts must use those canonical names instead of the bootstrap aliases.

### Common value types

```text
JsonPrimitive = nil | boolean | finite number | string | Json.Null
JsonValue = JsonPrimitive | JsonValue[] | table<string, JsonValue>

Position = {
    x: integer,
    y: integer,
    z: integer
}

PositionLike = Position | a plain table with the same x/y/z fields

ScreenPosition = {
    x: number,
    y: number
}

ColorRGBA = {
    r: integer,  -- 0..255
    g: integer,  -- 0..255
    b: integer,  -- 0..255
    a: integer   -- 0..255
}

ProfileSummary = {
    index: integer,  -- 1-based
    name: string
}

FeatureIdentifier = integer | string

SettingsFeature = "healer"|"conditions"|"heal_friends"|"targeting"|
    "alarms"|"magic_shooter"|"extras"|"equipment_manager"|
    "channels_manager"|"pvp_tools"|"looter"|"combo_bot"|"hud"|
    "delays"|"ammo_refill"|"tank_mode"|"timer_actions"|
    "supplies_sorter"|"scripter"

SettingsFeatureSelection = SettingsFeature[] | table<string, boolean>

UIVector2 = {
    x: number,
    y: number
}

UIEdgeInsets = {
    left: number,
    top: number,
    right: number,
    bottom: number
}

UIPadding = number | UIVector2 | UIEdgeInsets

UIReorderCallback = function(fromIndex: integer, toIndex: integer)
UIBreadcrumbSelectCallback = function(index: integer, label: string)

UIStyle = {
    colors?: table<string, ColorRGBA>,
    vars?: table<string, number | UIVector2>,
    palette?: table<string, ColorRGBA>
}

UIWindowOptions = {
    visible?: boolean,
    size?: UIVector2,
    min_size?: UIVector2,
    position?: UIVector2,
    resizable?: boolean,
    closable?: boolean,
    opacity?: number,
    decorations?: boolean,
    style?: UIStyle,
    on_close?: function(),
    refresh_ms?: integer,
    hotkey?: string,
    hotkey_options?: UIWindowHotkeyOptions,
    persist_geometry?: boolean,
    persistence_key?: string
}

UIWindowHotkeyOptions = {
    name?: string,
    trigger_on_keydown?: boolean
}

UIWidgetOptions = {
    size?: UIVector2,
    width?: number | "auto" | "content" | "fill",
    height?: number | "auto" | "content" | "fill",
    grow?: number,
    min_width?: number,
    max_width?: number,
    aspect_ratio?: number,
    tooltip?: string,
    format?: string,
    speed?: number,
    live?: boolean
}

UIActionRowOptions = {
    size?: UIVector2,  -- size.y controls logical row height
    tooltip?: string,
    show_toggle?: boolean,
    show_delete?: boolean,
    show_edit?: boolean
}

UITextureOptions = {
    size?: number,
    tooltip?: string,
    speed?: number,
    opacity?: number,
    border?: boolean
}

UIContainerOptions = {
    size?: UIVector2,
    width?: number | "auto" | "content" | "fill",
    height?: number | "auto" | "content" | "fill",
    grow?: number,
    min_width?: number,
    max_width?: number,
    aspect_ratio?: number,
    border?: boolean,
    open?: boolean,
    headers?: boolean,
    padding?: number | UIVector2 | { left: number, top: number, right: number, bottom: number },
    gap?: number,
    align?: "start" | "center" | "end" | "right" | "baseline",
    justify?: "start" | "space-between",
    wrap?: boolean,
    background?: ColorRGBA
}

UICanvasShapeOptions = {
    filled?: boolean,
    closed?: boolean,
    thickness?: number,
    rounding?: number
}

UISectionHeaderOptions = {
    gap?: number,
    color?: ColorRGBA,
    separator?: boolean
}

UIMetricCardOptions = {
    size?: UIVector2,
    grow?: number,
    border?: boolean,
    padding?: UIPadding,
    label_color?: ColorRGBA,
    progress?: number,
    progress_text?: string
}

UICalloutOptions = {
    size?: UIVector2,
    padding?: UIPadding
}

UIEmptyStateOptions = {
    icon?: string,
    color?: ColorRGBA
}

UINotificationType = "success"|"warning"|"error"|"info"

UINotificationOptions = {
    width?: number  -- 0/omitted for automatic, otherwise 180..800 logical pixels
}

CavebotBundleFeature = "targeting"|"magic_shooter"|"looter"
CavebotBundleFeatureSelection = CavebotBundleFeature[] |
    table<string, boolean>
```

`Position.New(x, y, z)` and `Position.New(positionLike)` return a `Position`
object with those three public fields plus the documented methods. Game,
Map, Minimap, Creature, Self, Walker, packet, and world-HUD position records use
the same exact `x`, `y`, and `z` names.

### Item, map, and minimap records

```text
ObjectInfo = {
    itemId: integer,
    name: string,
    description: string,
    isCreature: boolean,
    isWalkable: boolean,
    isPassable: boolean,
    isPathable: boolean,
    isShootable: boolean,
    isMovable: boolean,
    isGround: boolean,
    isBottom: boolean,
    isTop: boolean,
    isClip: boolean,
    isForceUsable: boolean,
    isLiquidPool: boolean,
    isContainer: boolean,
    isAutoMap: boolean,
    isFloorChange: boolean,
    isTeleport: boolean,
    isUsable: boolean,
    isMultiUsable: boolean,
    isCumulative: boolean,
    isHangable: boolean,
    isRotatable: boolean,
    isTakable: boolean,
    isWritable: boolean,
    isWriteOnce: boolean,
    isReportable: boolean,
    isWrapable: boolean,
    isUnwrapable: boolean,
    isTopEffect: boolean,
    isPlayerCorpse: boolean,
    isCreatureCorpse: boolean,
    isLiquidContainer: boolean,
    hasNoMovementAnimation: boolean,
    isFlagsEmpty: boolean,
    showsInCyclopedia: boolean,
    tileSpeed: integer,
    itemCategory: integer,
    cyclopediaItemId: integer,
    equipmentSlot: integer,
    equipmentSlotByCategory: integer
}

MapTileFlags = {
    isWalkable: boolean,
    isPassable: boolean,
    isPathable: boolean,
    isBlockingPath: boolean,
    hasCreature: boolean,
    hasNpc: boolean,
    hasGroundItem: boolean,
    hasBlockingItem: boolean,
    hasAutoMap: boolean,
    hasTeleport: boolean,
    speed: integer
}

MapTileItem = ObjectInfo plus {
    stackPosition: integer,  -- 0-based tile stack position
    count: integer
}

MapPathResult = {
    Directions: integer[],  -- MoveDirection values; capital D is intentional
    pathFindResult: integer -- PathFindResult value
}

MinimapTileInfo = {
    position: Position,
    flags: MapTileFlags|nil,
    items: MapTileItem[],
    pixelColor: integer|nil,
    walkableByColor: boolean|nil
}
```

Exact mappings:

- `Item.GetInfo` and `Map.GetObjectInfo` return `ObjectInfo|nil`.
- `Map.GetTileFlags` and `Minimap.GetTileFlags` return
  `MapTileFlags|nil`.
- `Map.GetTileItems` and `Minimap.GetTileItems` return `MapTileItem[]`.
- `Map.FindPath` and `Minimap.FindPath` return `MapPathResult`.
- `Minimap.GetTileInfo` returns `MinimapTileInfo`.
- All documented Item/Map use, look, move, buy, and sell operations return a
  boolean indicating whether the action was accepted/dispatched.

### Container records

```text
ContainerItem = {
    itemId: integer,
    name: string,
    slotIndex: integer,  -- 0-based
    count: integer,
    tierLevel: integer,
    isUpgradable: boolean,
    objectInfo: ObjectInfo|nil
}

ContainerSummary = {
    containerNumber: integer,
    containerId: integer,
    name: string,
    size: integer,
    itemsCount: integer,
    freeSlots: integer,
    hasItems: boolean
}

ContainerSnapshot = ContainerSummary plus {
    items: ContainerItem[]
}

ContainerFindResult = {
    itemId: integer,
    itemSlot: integer,       -- 0-based slot; not slotIndex
    itemQuantity: integer,   -- not count
    containerNumber: integer
}
```

Function-to-record mapping:

- `Container.GetOpenContainers() -> ContainerSummary[]`. These are summaries;
  they intentionally do **not** contain an `items` field.
- `Container.GetByNumber/GetByName/GetById -> ContainerSnapshot|nil`.
- `Container.GetItems -> ContainerItem[]`.
- `Container.GetItem` and `Item.GetFromContainer -> ContainerItem|nil`.
- `Container.FindItem`, `Container.FindItemInOpenContainers`, and
  `Item.FindInContainer -> ContainerFindResult|nil`.
- Container move/use/look methods return `boolean`.

Correct vial/container traversal:

```lua
for _, container in ipairs(Container.GetOpenContainers()) do
    for _, item in ipairs(Container.GetItems(container.containerNumber)) do
        if item.itemId == VIAL_ID then
            Container.MoveItemToFloor(
                container.containerNumber,
                item.slotIndex,
                item.itemId,
                destination,
                item.count)
        end
    end
end
```

Do not substitute `container.index`, `container.containerIndex`,
`item.id`, `item.itemSlot`, or `item.slot` in this traversal.
`itemSlot` exists only on `ContainerFindResult`.

### Equipment and inventory records

```text
EquipmentItem = {
    itemId: integer,
    item_id: integer,       -- exact compatibility alias of itemId
    slot: integer,
    slotIndex: integer,     -- exact compatibility alias of slot
    count: integer,
    itemCount: integer,     -- exact compatibility alias of count
    tierLevel: integer,
    isUpgradable: boolean,
    name: string,
    objectInfo: ObjectInfo|nil
}

EquipmentSlotConstants = {
    NONE: 0, HELMET: 1, AMULET: 2, BACKPACK: 3, ARMOR: 4,
    RIGHT_HAND: 5, LEFT_HAND: 6, LEGS: 7, BOOTS: 8, RING: 9,
    ARROW: 10
}

InventorySnapshot = {
    canReadEquipment: boolean,
    canMoveEquipment: boolean,
    slotIds: integer[],
    slots: table<integer, EquipmentItem>
}
```

`Inventory.GetSlotItem -> EquipmentItem|nil`;
`Inventory.GetAllSlotItems -> table<integer, EquipmentItem>`;
`Inventory.GetEquipmentSlotConstants -> EquipmentSlotConstants`; and
`Inventory.GetSnapshot -> InventorySnapshot`. Only occupied equipment slots
are present in the slot map. Equip, look, and movement calls return `boolean`.

### Creature and local-player records

```text
CreatureOutfit = {
    outfit_id: integer,
    is_mounted: boolean
}

Creature = {
    _id: integer,
    _cached_name: string|nil
}

CreatureIterator = function() -> integer|nil, Creature|nil

SelfStatusFlags = {
    isHungry: boolean|nil,
    isInRestingArea: boolean|nil,
    isPoisoned: boolean|nil,
    isBurning: boolean|nil,
    isElectrified: boolean|nil,
    isDrunk: boolean|nil,
    isManaShielded: boolean|nil,
    isParalyzed: boolean|nil,
    isHasted: boolean|nil,
    isInCombat: boolean|nil,
    isDrowning: boolean|nil,
    isFreezing: boolean|nil,
    isDazzled: boolean|nil,
    isCursed: boolean|nil,
    isStrengthened: boolean|nil,
    isPzBlocked: boolean|nil,
    isInProtectionZone: boolean|nil,
    isBleeding: boolean|nil,
    hasLesserHex: boolean|nil,
    hasIntenseHex: boolean|nil,
    hasGreaterHex: boolean|nil,
    isRooted: boolean|nil,
    isFeared: boolean|nil,
    hasGoshnarTaint1: boolean|nil,
    hasGoshnarTaint2: boolean|nil,
    hasGoshnarTaint3: boolean|nil,
    hasGoshnarTaint4: boolean|nil,
    hasGoshnarTaint5: boolean|nil,
    hasNewManaShield: boolean|nil,
    hasAgony: boolean|nil,
    isPowerless: boolean|nil,
    hasMentorOther: boolean|nil
}

SelfStatsSnapshot = {
    health: integer|nil,
    maxHealth: integer|nil,
    mana: integer|nil,
    maxMana: integer|nil,
    capacity: number|nil,
    stamina: number|nil,
    online: boolean|nil,
    isAlive: boolean|nil,
    isAttacking: boolean|nil,
    isFollowing: boolean|nil,
    manaShieldCapacity: integer|nil,
    maxManaShieldCapacity: integer|nil,
    targetId: integer|nil,
    followId: integer|nil,
    mousePosition: Position|nil,
    mouseWorldX: integer|nil,
    mouseWorldY: integer|nil,
    mouseWorldZ: integer|nil,
    capacityFloor: integer|nil,
    level: integer|nil,
    soul: integer|nil,
    staminaHours: integer|nil,
    staminaDays: integer|nil,
    levelPercent: integer|nil,
    healthPercent: number|nil,
    manaPercent: number|nil,
    hasTarget: boolean|nil,
    hasFollow: boolean|nil,
    statusFlags: SelfStatusFlags
}
```

Creature iterator collection methods return `Creature[]`;
`Creatures.GetCreatureByName -> Creature|nil`;
`Creature:GetPosition -> Position` (an invalid wrapper produces zero
coordinates); `Creature:GetOutfit -> CreatureOutfit`; and
`Self.GetMousePositionInWorld -> Position|nil`.
`Self.GetStatusFlagsSnapshot -> SelfStatusFlags` and
`Self.GetStatsSnapshot -> SelfStatsSnapshot`.
`Self.HasCharacterFlag(flag)` accepts a named `CharacterFlag` value or a raw
integer bit position from 0 through 31. It returns `nil` when live player data
is unavailable and otherwise returns a boolean. `CharacterFlag.COUNT` is an
enum sentinel, not a named gameplay condition, even though raw position 30 can
be queried safely.

### JSON, network, scheduler, hotkey, and sound records

`OpaqueRuntimeValue` below means a native table or userdata value retained only
for diagnostics. Its internal fields are not a public contract and generated
scripts must not inspect them.

```text
OpaqueRuntimeValue = native table | userdata

HttpRequestOptions = {
    url: string,
    method?: "GET"|"POST"|"PUT"|"PATCH"|"DELETE"|"HEAD", -- default GET
    headers?: table<string, string>,
    body?: string,
    timeoutMs?: integer,
    maxResponseBytes?: integer,
    followRedirects?: boolean
}

HttpConvenienceOptions = {
    headers?: table<string, string>,
    body?: string,                   -- effective only for Http.Get/GetJson
    timeoutMs?: integer,
    maxResponseBytes?: integer,
    followRedirects?: boolean
}

HttpHeader = {
    name: string,
    value: string
}

HttpResponse = {
    ok: boolean,
    status: integer,
    redirects: integer,
    body: string,
    error: string,
    url: string,
    headers: table<string, string>, -- lower-case names; repeated values combined
    headerList: HttpHeader[]        -- ordered and preserves repeated headers
}

WebSocketConnectOptions = {
    headers?: table<string, string>,
    subprotocol?: string,
    timeoutMs?: integer,
    maxMessageBytes?: integer
}

WebSocketEvent = {
    ok: boolean,
    connectionId: integer,
    closeCode: integer,
    type: "text"|"binary"|"close"|"error"|"timeout",
    data: string,
    error: string,
    url: string
}

WebSocketConnection = {
    url: string,
    Send: function(data: string, binary?: boolean) -> boolean, string|nil,
    Receive: function(timeoutMs?: integer) -> WebSocketEvent,
    Close: function(closeCode?: integer, reason?: string) -> boolean, string|nil,
    IsOpen: function() -> boolean
}

ParsedHotkey = {
    keycode: integer,
    ctrl: boolean,
    shift: boolean,
    alt: boolean,
    trigger_on_keydown: boolean,
    extended: boolean
}

HotkeyRegistrationOptions = {
    id: string,
    combo: string,
    callback: function(),
    name?: string,
    trigger_on_keydown?: boolean,
    extended?: boolean
}

ModuleRecord = {
    mode: "every"|"after",
    delayMs: integer,
    active: boolean
}

ModuleListRecord = ModuleRecord plus {
    name: string
}

SoundPlaybackOptions = exactly one of {
    sound_id: integer,
    sound_name: string,
    file_path: string
} plus {
    instant?: boolean
}
```

Exact network and scheduler returns:

- `Http.Request -> HttpResponse` and takes `HttpRequestOptions`, including its
  required `url`. `Http.Get/Post/PostJson -> HttpResponse` and take
  `HttpConvenienceOptions`; their separate arguments supply the URL and, for
  Post/PostJson, replace the request body.
- `Http.GetJson -> JsonValue|nil, HttpResponse, string|nil`. The third result
  is a decode error; inspect it when the first result is `nil`.
- `WebSocket.Connect -> WebSocketConnection|nil, string|nil`.
- `WebSocketConnection:Receive -> WebSocketEvent`; `Send` and `Close` return
  `boolean, string|nil`; `IsOpen -> boolean`.
- `Hotkeys.ParseCombo -> ParsedHotkey|nil, string|nil` and
  `Hotkeys.RegisterCombo -> boolean, string`, where the string is the opaque
  owner-scoped registration ID. A registered hotkey callback takes no
  arguments and must obey the event critical-path rule.
- Raw `Module.New/Stop/Pause/Resume -> nil`.
  `Module.Every/After/Cancel/PauseManaged/ResumeManaged -> boolean`;
  `Module.Exists -> boolean`; `Module.Get -> ModuleRecord|nil`; and
  `Module.List -> ModuleListRecord[]`.
- `Sound.Play/Stop/ClearQueue/SetMinDelay -> nil`.
  The helper `PlayById/PlayByName/PlayFile/PlayBotSound/StopAll -> nil`;
  smart and wait helpers return `boolean`. `Sound.IsQueued` accepts the same
  exact source-selection fields as `Sound.Play`.

`Json.Encode/TryEncode` accept `JsonValue`; `Json.Decode/TryDecode` return
`JsonValue`. `Json.Array` takes a contiguous `JsonValue[]` and returns the same
array tagged for JSON-array encoding. `Json.Object` takes and returns a
`table<string, JsonValue>` tagged for JSON-object encoding. Do not pass native
userdata, functions, threads, cyclic tables, non-finite numbers, or mixed
array/object keys.

### Chat channel, NPC trade, and VIP records

```text
ChatChannelRecord = {
    id: integer,
    name: string,
    canSend: boolean,
    isOpened: boolean,
    isLocal: boolean,
    isServerLog: boolean,
    raw: OpaqueRuntimeValue|nil
}

ChatChannelInput = {
    id?: integer,          -- channelId is also accepted as an input alias
    name?: string,         -- channelName is also accepted as an input alias
    canSend?: boolean,
    isOpened?: boolean,
    isLocal?: boolean,
    isServerLog?: boolean,
    raw?: OpaqueRuntimeValue
}

ChatChannelIdentifier = integer | string | ChatChannelRecord |
    { id: integer } | { channelId: integer } |
    { name: string } | { channelName: string }

ChatChannelStorageSnapshot = {
    available: boolean,
    openedCount: integer,
    chatCount: integer,
    openedNames: string[],
    sendableNames: string[],
    localChannel: ChatChannelRecord|nil,
    serverLogChannel: ChatChannelRecord|nil,
    openedChannels: ChatChannelRecord[],
    chatChannels: ChatChannelRecord[]
}

NpcTradeOffer = {
    itemId: integer,
    name: string,
    buyPrice: number,
    sellPrice: number,
    capacity: number,
    raw: OpaqueRuntimeValue
}

NpcTradeSnapshot = {
    available: boolean,
    isOpen: boolean|nil,
    npcName: string|nil,
    offerCount: integer,
    offers: NpcTradeOffer[]
}

VIPEntry = {
    name: string,
    description: string,
    type: integer,
    online: boolean,
    notifyOnLogin: boolean,
    raw: OpaqueRuntimeValue
}

VIPSnapshot = {
    available: boolean,
    count: integer,
    onlineCount: integer,
    heartCount: integer,
    names: string[],
    onlineNames: string[],
    vips: VIPEntry[]
}
```

`ChatChannel.New(ChatChannelInput|integer, channelName?) -> ChatChannel`, whose
public data fields match `ChatChannelRecord` and whose methods are listed in the
API index. `ChatChannel.FromIdentifier/GetById/GetByName -> ChatChannel|nil`;
`ChatChannel:ToTable -> ChatChannelRecord`.
`ChatChannelStorage.GetOpenedChannels/GetChatChannels -> ChatChannelRecord[]`;
all single-channel getters and `ResolveChannel` return
`ChatChannelRecord|nil`; `ToNameLookupTable -> table<string,
ChatChannelRecord>`; `ToIdLookupTable -> table<integer, ChatChannelRecord>`;
and `GetSnapshot -> ChatChannelStorageSnapshot`.

`NpcTradeStorage.GetOffers -> NpcTradeOffer[]`; its offer getters return
`NpcTradeOffer|nil`; `GetSnapshot -> NpcTradeSnapshot`; and `Buy/Sell ->
boolean`. `VIP.GetAll/GetByType/GetHearts/FindByPrefix -> VIPEntry[]`;
`VIP.Get -> VIPEntry|nil`; `VIP.ToLookupTable -> table<string, VIPEntry>`; and
`VIP.GetSnapshot -> VIPSnapshot`.

### Cooldown and spell records

```text
CooldownSpellStatus = {
    inCooldown: boolean,
    timeLeft: integer
}

CooldownStatus = {
    spells: table<string, CooldownSpellStatus>,
    groups: {
        attack: integer,
        healing: integer,
        support: integer,
        special: integer,
        crippling: integer,
        focus: integer,
        ultimate: integer
    },
    useWith: boolean
}

SpellInfo = {
    id: integer|nil,
    words: string,
    cooldownId: integer|nil,
    groupIds: integer[],
    inCooldown: boolean,
    leftCooldownTime: integer
}

ItemSpellInfo = {
    id: integer,
    cooldownId: integer|nil,
    groupIds: integer[],
    inCooldown: boolean,
    leftCooldownTime: integer
}
```

All cooldown-time functions return integer milliseconds and all `Is*` or
`WillBeReady` functions return booleans. `Cooldowns.Utils.GetStatus ->
CooldownStatus`; `Spells.GetInfo -> SpellInfo`; `Spells.Item.GetInfo ->
ItemSpellInfo`; both `GetGroupIds` functions return `integer[]`;
`Spells.GetIdByWords/GetIdByName/Spells.Item.GetCooldownId -> integer|nil`; and
`Spells.GetWordsById -> string|nil`. A `spellOrWordsOrId` argument is exactly a
non-empty spell-words `string` or a numeric spell cooldown ID.

### Cavebot, Walker, Lure, and action records

```text
WalkerWaypointInput = {
    type?: integer,                 -- default WaypointType.Node
    x?: integer,
    y?: integer,
    z?: integer,
    labelName?: string,
    useWithItemId?: integer,
    delayMs?: integer,
    scriptContent?: string
}

RawWalkerWaypointInput = WalkerWaypointInput plus {
    useItemId?: integer,
    useTarget?: integer,
    actionWaypointKind?: integer,
    actionKind?: string,
    actionVersion?: string,
    actionConfig?: table<string, JsonValue>,
    failurePolicy?: table<string, JsonValue>
}

WalkerNavigationMode = "waypoints"|"auto_explore"

WalkerWaypoint = {
    index: integer,                 -- 1-based current route index
    type: integer,
    x: integer,
    y: integer,
    z: integer,
    useWithItemId: integer,
    useItemId: integer,
    useTarget: integer,
    actionWaypointKind: integer,
    delayMs: integer,
    labelName: string,
    scriptContent: string,
    actionKind: string,
    actionVersion: string,
    actionConfig: JsonValue,
    failurePolicy: JsonValue,
    uniqueId: integer
}

WalkerSpecialAreaInput = {
    x: integer,
    y: integer,
    z: integer,
    width?: integer,
    height?: integer,
    featureMask?: integer,
    enabled?: boolean
}

WalkerSpecialAreaPatch = {
    x?: integer,
    y?: integer,
    z?: integer,
    width?: integer,
    height?: integer,
    featureMask?: integer,
    enabled?: boolean
}

WalkerSpecialArea = {
    index: integer,
    id: integer|string,
    uniqueId: integer|string,       -- exact alias of id
    x: integer,
    y: integer,
    z: integer,
    width: integer,
    height: integer,
    featureMask: integer,
    enabled: boolean
}

WalkerAutoRecorderOptions = {
    recordMovementActions: boolean,
    recordUseItemActions: boolean,
    recordUseWithActions: boolean
}

WalkerAutoRecorderOptionsPatch = the same fields, all optional

WalkerAutoExploreSettings = {
    style: "natural"|"thorough"|"wide_roam",
    areaMode: "painted"|"connected_walkable",
    maximumFloorsUp: integer,
    maximumFloorsDown: integer,
    allowWalkOn: boolean,
    allowLadder: boolean,
    allowRope: boolean,
    allowHole: boolean,
    allowTeleport: boolean,
    autoOpenDoors: boolean
}

WalkerAutoExploreSettingsPatch = the same fields, all optional

WalkerAutoExploreVisitHeat = {
    position: Position,
    visitCount: integer,
    lastVisitSequence: integer
}

WalkerAutoExploreStatus = {
    phase: "idle"|"exploring"|"fighting"|"transition"|"recovering"|
        "paused"|"stuck",
    hasBasePosition: boolean,
    basePosition: Position,
    currentPosition: Position,
    hasTarget: boolean,
    targetPosition: Position,
    visitedTileCount: integer,
    paintedTileCount: integer,
    activeConnectorId: integer|string,
    outsideMask: boolean,
    status: string,
    latestFailureReason: string,
    plannedPath: Position[],
    recentTrail: Position[],
    recentVisitHeat: WalkerAutoExploreVisitHeat[]
}

WalkerAutoExploreConnectorInput = {
    enabled?: boolean,
    kind: "walk_on"|"ladder"|"rope"|"hole"|"teleport",
    source: Position,
    destination: Position,
    pairedConnectorId?: integer|string
}

WalkerAutoExploreConnectorPatch = the same fields, all optional

WalkerAutoExploreConnector = {
    id: integer|string,
    enabled: boolean,
    kind: "walk_on"|"ladder"|"rope"|"hole"|"teleport",
    source: Position,
    destination: Position,
    pairedConnectorId: integer|string
}

LureSettingInput = {
    lureMonstersCount?: integer,
    leaveMonstersCount?: integer,
    dontLeaveMonstersUnderHpPerc?: integer,
    monsterDangerLevel?: integer,
    considerDangerLevelsAbove?: boolean,
    enabled?: boolean
}

LureSetting = {
    index: integer,
    lureMonstersCount: integer,
    leaveMonstersCount: integer,
    dontLeaveMonstersUnderHpPerc: integer,
    monsterDangerLevel: integer,
    considerDangerLevelsAbove: boolean,
    enabled: boolean
}

CavebotStatus = {
    walkerEnabled: boolean,
    lureEnabled: boolean,
    walkerStuck: boolean,
    lureState: integer,
    lureMonsterCount: integer,
    waypointCount: integer,
    selectedWaypointIndex: integer|nil
}

CavebotDeferredHandle = {
    token: integer,
    Complete: function(self) -> boolean,
    Cancel: function(self) -> boolean
}

CavebotWaypointChangeEvent = {
    previousIndex: integer|nil,
    index: integer,
    type: integer,
    x: integer,
    y: integer,
    z: integer,
    label: string,
    labelName: string,              -- exact alias of label
    uniqueId: integer
}

CavebotActionWaypointRef = {
    index: integer,
    uniqueId: integer,
    x: integer,
    y: integer,
    z: integer
}

CavebotActionStartedEvent = {
    executionId: integer,
    action: string,
    name: string,                   -- exact alias of action
    kind: integer,
    waypoint: CavebotActionWaypointRef
}

CavebotActionCompletedEvent = CavebotActionStartedEvent plus {
    ok: boolean,
    outcome: string,
    description: string,
    durationMs: integer,
    result?: string,                -- present only when ok is true
    error?: string                  -- present only when ok is false
}

WalkerEventCallback = exactly one callback shape selected by eventId:
    ON_LABEL or OBSERVE_LABEL:
        function(labelName: string)
    ON_WAYPOINT_CHANGE:
        function(previousIndex: integer, index: integer, type: integer,
            x: integer, y: integer, z: integer, labelName: string,
            uniqueId: integer)
    ON_ACTION or OBSERVE_ACTION:
        function(actionName: string)
    ACTION_STARTED:
        function(executionId: integer, actionName: string, kind: integer,
            waypointIndex: integer, waypointUniqueId: integer,
            x: integer, y: integer, z: integer)
    ACTION_COMPLETED:
        function(executionId: integer, actionName: string, kind: integer,
            waypointIndex: integer, waypointUniqueId: integer,
            x: integer, y: integer, z: integer, ok: boolean,
            outcome: string, description: string, durationMs: integer)
    MOVEMENT_CONFIRMED:
        function(oldX: integer, oldY: integer, oldZ: integer,
            newX: integer, newY: integer, newZ: integer,
            fullMapUpdate: boolean)
    MOVEMENT_REJECTED:
        function()
```

`WalkerWaypointInput` is the exact high-level Cavebot wrapper input. It
has no index or uniqueId input; all supported action fields pass through to native Walker. A GetWaypoints snapshot can be used as input after omitting output-only identity fields. RawWalkerWaypointInput remains an alias for the same shape.
Walker.AddWaypoint -> integer|false` returns the new 1-based index. Insert,
replace, delete, clear, move, and waypoint-position mutations return `boolean`.
`Walker.GetWaypoints -> WalkerWaypoint[]`.
`Walker.GetSelectedWaypointIndex -> integer|nil`; all other waypoint count or
distance getters return integers.

`Walker.GetSpecialAreas -> WalkerSpecialArea[]` and
`AddSpecialArea -> integer|string|false`. Update accepts only
`WalkerSpecialAreaPatch`; unknown keys are rejected. Auto-explore getters
return the named records above, connector add returns
`integer|string|false`, and the mutation functions return booleans.
`SetAutoRecorderOptions` accepts `WalkerAutoRecorderOptionsPatch`; omitted
fields retain their previous values. Its getter returns the required-field
`WalkerAutoRecorderOptions` snapshot.

`Lure.GetSettings -> LureSetting[]`; `Lure.AddSetting -> integer|false`;
`UpdateSetting/RemoveSetting/ClearSettings -> boolean`. Lure boolean getters
return booleans, numeric state/count/option/range/delay getters return integers,
and setters other than raw `Lure.SetEnabled` return booleans. Raw
`Walker.SetEnabled`, `Lure.SetEnabled`, `Walker.Resume`, and `Walker.GoTo`
return no values (`nil`).

The high-level Cavebot callback signatures are exact:

- `OnLabel/InterceptLabel/ObserveLabel(function(labelName: string))`.
- `OnAction/InterceptAction/ObserveAction(function(actionName: string))`.
- `OnWaypointChange/ObserveWaypointChange(function(event:
  CavebotWaypointChangeEvent))`.
- `OnActionStarted(function(event: CavebotActionStartedEvent))`.
- `OnActionCompleted(function(event: CavebotActionCompletedEvent))`.

Each registration returns `integer|nil`. Use these named helpers instead of
`RegisterEvent` when possible because the raw callback argument list depends on
the selected `WalkerEvent`. Every callback above is an event critical path.
Observer callbacks must return immediately. Legacy label/action interceptors
also pause Walker; if asynchronous work is required, create a
`CavebotDeferredHandle`, enqueue only its token plus minimal scalar data, return,
and complete/cancel it from the stable worker.

`Cavebot.SetEnginesEnabled/GetStatus -> CavebotStatus`;
`Cavebot.Defer -> CavebotDeferredHandle`; `Cavebot.Pause -> string|nil`
(scheduled-event ID only when auto-resume is requested); Cavebot enable/disable,
resume, and go-to calls return `nil`; `UnregisterAllEvents -> boolean`.

#### Cavebot action context and results

```text
CavebotSupplyItem = {
    itemId: integer,
    name?: string,
    enabled?: boolean,
    min?: integer,
    target?: integer,
    buy?: { enabled?: boolean },
    ignoreCapacity?: boolean,
    buyInShoppingBags?: boolean,
    sellEquipped?: boolean,
    keep?: integer,
    amount?: integer
}

CavebotSupplyProfile = {
    checkCapacity?: boolean,
    minCapacity?: number,
    checkStamina?: boolean,
    minStaminaMinutes?: integer,
    items?: CavebotSupplyItem[]
}

CavebotVendorProfile = {
    talkSequence?: string[]
}

CavebotActionContext = {
    actionType?: string,             -- action_type is accepted as an alias
    actionConfig?: table<string, JsonValue>, -- action_config alias accepted
    vendorProfile?: CavebotVendorProfile, -- vendor_profile alias accepted
    supplyProfile?: CavebotSupplyProfile, -- supply_profile alias accepted
    successLabel?: string,
    failureLabel?: string,
    configRef?: JsonValue,           -- config_ref alias accepted
    messages?: string|string[],
    message?: string,
    sellItems?: CavebotSupplyItem[],
    script?: string
}

CavebotActionItemResult = {
    itemId: integer,
    amount?: integer,
    count?: integer,
    min?: integer,
    target?: integer,
    keep?: integer,
    low?: boolean,
    countAvailable?: boolean,
    reason?: string,
    error?: string,
    name?: string
}

CavebotActionResult = {
    ok: boolean,
    actionType?: string,
    error?: string,
    pending?: boolean,
    goToLabel?: string,              -- gotoLabel also accepted from custom handlers
    configRef?: JsonValue,
    value?: JsonValue,
    sent?: string[],
    failedMessage?: string,
    needsRefill?: boolean,
    reasons?: string[],
    capacity?: number,
    staminaMinutes?: integer,
    items?: CavebotActionItemResult[],
    npcTalk?: string[],
    bought?: CavebotActionItemResult[],
    sold?: CavebotActionItemResult[],
    skipped?: CavebotActionItemResult[],
    errors?: CavebotActionItemResult[]
}
```

`actionConfig` is deliberately an open, JSON-compatible action-specific map.
Built-in handlers currently read keys such as `params`, `messages`, and
`talkSequence`, and custom handlers may define additional keys. Do not assume
those examples are the only legal fields.

`Cavebot.Actions.Register(actionType, handler)` expects
`handler(context: CavebotActionContext) -> CavebotActionResult` and returns
`boolean`. `Cavebot.Actions.Run -> CavebotActionResult` and
`GetLastResult -> CavebotActionResult|nil`. A custom handler may add fields,
but should keep them JSON-compatible and must preserve the common `ok` result
contract.

### Engine feature snapshot records

Every Engine getter below returns a detached snapshot. Mutating it never changes
the running feature; use the matching setter. Entry and profile indexes are
1-based.

```text
TimerActionEntry = {
    index: integer,
    spellWords: string,
    itemId: integer,
    type: integer,
    delay: integer,
    timeUnit: integer,
    enabled: boolean,
    useInProtectionZone: boolean
}

SuppliesSorterEntry = {
    index: integer,
    destinationContainerId: integer,
    itemIds: integer[],
    enabled: boolean
}

ChannelManagerEntry = {
    index: integer,
    name: string,
    message: string,
    intervalSeconds: integer,
    channelId: integer,
    talkAction: integer,
    enabled: boolean
}

ConditionSpellEntry = {
    index: integer,
    spellWords: string,
    manaCost: integer,
    characterFlag: integer,
    enabled: boolean
}

HealerSpellInput = {
    spell_words?: string,
    cast_value?: integer,
    mana_cost?: integer,
    attribute?: "health"|"healthpercent"|"mana"|"manapercent",
    condition?: "below"|"above",
    enabled?: boolean
}

HealerSpellPatch = {
    cast_value?: integer,
    mana_cost?: integer,
    enabled?: boolean
}

HealerSpellEntry = {
    spell_words: string,
    cast_value: integer,
    mana_cost: integer,
    attribute: "health"|"health_percent"|"mana"|"mana_percent"|"unknown",
    condition: "below"|"above"|"unknown",
    enabled: boolean,
    display_string: string
}

IndexedHealerSpellEntry = HealerSpellEntry plus {
    index: integer -- added only by Engine.Healer.FindSpellByWords
}

HealerItemInput = {
    item_id?: integer,
    cast_value?: integer,
    delay_ms?: integer,
    attribute?: "health"|"healthpercent"|"mana"|"manapercent",
    condition?: "below"|"above",
    action?: "useonself"|"useincontainer",
    use_when_feared?: boolean,
    enabled?: boolean
}

HealerItemPatch = {
    cast_value?: integer,
    delay_ms?: integer,
    enabled?: boolean
}

HealerItemEntry = {
    index: integer, -- added by Engine.Healer.GetItems/FindItemById
    item_id: integer,
    cast_value: integer,
    delay_ms: integer,
    attribute: "health"|"health_percent"|"mana"|"mana_percent"|"unknown",
    condition: "below"|"above"|"unknown",
    action: "use_on_self"|"use_in_container"|"unknown",
    use_when_feared: boolean,
    enabled: boolean,
    display_string: string
}

AmmoRefillInput = {
    item_id?: integer,
    refill_at_count?: integer,
    refill_in_left_hand?: boolean,
    equip_from_hotkey?: boolean,
    enabled?: boolean
}

AmmoRefillEntry = AmmoRefillInput with all fields required plus {
    display_string: string
}

IndexedAmmoRefillEntry = AmmoRefillEntry plus {
    index: integer -- added only by Engine.AmmoRefill.FindByItemId
}

HealFriendAction = {
    index: integer,
    spellWords: string,
    manaCost: integer,
    itemId: integer,
    healthPercentage: integer,
    method: integer,
    enabled: boolean
}

HealFriendVocationEntry = {
    index: integer,
    vocation: integer,
    priority: integer,
    enabled: boolean,
    actions: HealFriendAction[]
}

HealFriendArea = {
    spellWords: string,
    manaCost: integer,
    vocation: integer,
    playersNeeded: integer,
    healthPercentage: integer,
    minimumHarmony: integer,
    extended: boolean,
    enabled: boolean,
    knightRequired: boolean,
    paladinRequired: boolean,
    sorcererRequired: boolean,
    druidRequired: boolean,
    monkRequired: boolean
}

FeatureProfile = {
    index: integer,
    name: string,
    active: boolean,
    entryCount: integer
}

EquipmentManagerProfile = FeatureProfile  -- compatibility documentation alias

EquipmentCondition = {
    type: integer,
    monstersAround: integer,
    playersAround: integer,
    creaturesCount: integer,
    targetName: string,
    creatureNames: string
}

EquipmentManagerEntry = {
    index: integer,
    itemId: integer,
    secondaryItemId: integer,
    excludedItemIds: string,
    excludedItemIdsEnabled: boolean,
    tier: integer,
    equipFromHotkey: boolean,
    equipAction: boolean,
    enabled: boolean,
    delayMs: integer,
    hasDelay: boolean,
    useExtraConditions: boolean,
    checkHealthRange: boolean,
    checkManaRange: boolean,
    healthManaOperator: integer,
    minimumHealthPercentage: integer,
    maximumHealthPercentage: integer,
    minimumManaPercentage: integer,
    maximumManaPercentage: integer,
    keepEquipped: boolean,
    keepEquippedMs: integer,
    slot: integer,
    conditionOperator: integer,
    firstCondition: EquipmentCondition,
    secondCondition: EquipmentCondition
}

AlarmsConfig = {
    lowHealthPercentage: integer,
    lowManaPercentage: integer,
    flashWindow: boolean,
    bringToFocus: boolean,
    ignoreAllyPlayers: boolean,
    gmCheckChatMessages: boolean,
    damageTakenMinimum: integer,
    damageTakenMaximum: integer,
    playerAttackFilterMode: integer,
    playerDetectedFilterMode: integer,
    skullFilterMode: integer,
    creatureDetectedNames: string,
    alarmMessages: string,
    playerAttackNames: string,
    playerDetectedNames: string,
    skullNames: string,
    enemyNames: string,
    gmNames: string
}

PVPTrashItem = {
    index: integer,
    itemId: integer,
    quantity: integer
}

PVPConfig = {
    holdTarget: boolean,
    trashOnMouse: boolean,
    antiPush: boolean,
    killTarget: boolean,
    magicWallKeeper: boolean,
    wildGrowthKeeper: boolean,
    previousSpotWall: boolean,
    pushmax: boolean,
    pushAttackedPlayer: boolean,
    killTargetManaCost: integer,
    killTargetHealthPercentage: integer,
    killTargetSpellWords: string,
    pushmaxDisintegrateRuneId: integer,
    pushmaxNonDisintegrateRuneId: integer,
    delayBetweenRuneAndPush: integer,
    wallKeeperRuneIds: integer[],
    wildGrowthKeeperRuneIds: integer[],
    previousSpotRuneIds: integer[],
    antiPushTrashItems: PVPTrashItem[],
    mouseTrashItems: PVPTrashItem[]
}
```

Important Healer distinction: add inputs use `healthpercent`, `manapercent`,
`useonself`, and `useincontainer` without underscores. Getter snapshots use
`health_percent`, `mana_percent`, `use_on_self`, and
`use_in_container`. Generated scripts must not feed a getter's strings back to
`AddSpell/AddItem` without translating them.

Exact mappings include:

- `Engine.TimerActions.GetEntries -> TimerActionEntry[]`;
  `Engine.SuppliesSorter.GetEntries -> SuppliesSorterEntry[]`;
  `Engine.Channels.GetEntries -> ChannelManagerEntry[]`.
- `Engine.Conditions.GetSpells/GetHoldSpells -> ConditionSpellEntry[]`.
- `Engine.Healer.GetSpells -> HealerSpellEntry[]` and
  `GetSpellByIndex -> HealerSpellEntry|nil`; these native spell records do not
  contain an `index`. `FindSpellByWords -> IndexedHealerSpellEntry|nil` adds
  the matched 1-based index. Item list/find records use `HealerItemEntry` and
  always include their index. Add calls accept their named input records and
  return the new 1-based integer index, or `false` if the native feature is
  unavailable. `ClearAllSpells/ClearAllItems -> boolean`.
- `Engine.AmmoRefill.Get/GetAll -> AmmoRefillEntry|nil` and
  `AmmoRefillEntry[]`; those native records do not include an index.
  `FindByItemId -> IndexedAmmoRefillEntry|nil` adds the matched 1-based index.
  Add accepts `AmmoRefillInput` and returns the new index or `false` when no
  active profile/feature is available. `ClearAll -> boolean`. Profile summaries,
  profile names, and profile lifecycle operations follow the common contract
  described below.
- `Engine.HealFriend.GetVocations -> HealFriendVocationEntry[]` and
  `GetArea -> HealFriendArea`.
- `Engine.EquipmentManager.GetProfiles -> FeatureProfile[]` keeps its existing
  record fields, and `GetEntries -> EquipmentManagerEntry[]`.
- `Engine.Alarms.GetConfig -> AlarmsConfig`;
  `Engine.PVPTools.GetConfig -> PVPConfig`.
- `Engine.Equipment.GetSlotItem/GetAllSlotItems/GetSlotConstants/GetSnapshot`
  use the same `EquipmentItem`, `EquipmentSlotConstants`, and
  `InventorySnapshot` records documented above.

```text
ComboClientEntry = {
    index: integer,
    leaderName: string,
    leaderSpellWords: string,
    mySpellWords: string,
    myRuneId: integer,
    leaderAction: integer,
    myAction: integer,
    focusOption: integer,
    shootType: integer,
    range: integer,
    enabled: boolean,
    requiresTarget: boolean
}

ComboRoomEntry = {
    index: integer,
    leaderSpellWords: string,
    mySpellWords: string,
    leaderRuneId: integer,
    myRuneId: integer,
    leaderAction: integer,
    myAction: integer,
    equipMode: integer,
    range: integer,
    enabled: boolean,
    requiresTarget: boolean
}

ComboRoomState = {
    inRoom: boolean,
    leader: boolean,
    memberCount: integer,
    roomId: string,
    lastMessage: string,
    leaderCharacterName: string,
    memberNames: string[]
}

HudFeatureConfig = {
    magicWallTimers: boolean,
    xray: boolean,
    targetingAnchor: boolean,
    levelSpy: boolean,
    magicWallIds: string,
    wildGrowthIds: string,
    timerColor: number[4] -- r,g,b,a normalized to 0..1
}

HudSpecialFoodCounter = {
    index: integer,
    itemId: integer,
    delaySeconds: integer
}

MagicShooterEntry = {
    index: integer,
    enabled: boolean,
    kind: "rune"|"spell",
    actionType: "targetedSpell"|"areaRune"|"targetedRune"|"empowerment"|
        "absoluteSpell"|"avatars"|"exetaChallenges"|"unknown",
    range: integer,
    option: integer,
    condition: integer,
    manaPercentage: integer,
    healthPercentage: integer,
    healthCondition: integer,
    harmony?: integer,                  -- x64 only
    harmonyCondition?: integer,         -- x64 only
    monsterCount: integer,
    monsterCountCondition: integer,
    minimumMonsterHealthPercentage: integer,
    maximumMonsterHealthPercentage: integer,
    dangerLevel: integer,
    customDelayMs: integer,
    shootAfterWalkDelayMs: integer,
    momentumDelayMs: integer,
    meleeSkillIncreasePercentage: integer,
    distanceSkillIncreasePercentage: integer,
    requiresTarget: boolean,
    pvpSafe: boolean,
    shootOverAllies: boolean,
    customSpell: boolean,
    attackSkillBuffSpell: boolean,
    dontCastWhileWalking: boolean,
    prioritizeWithMomentum: boolean,
    monsterNames: string,
    castMethod: integer,
    patternAnchor: integer,
    patternSource: integer,
    patternVariant: integer,
    effectType: integer,
    priorityLane: integer,
    targetPolicy: integer,
    hitCountMode: integer,
    equipmentRequirement: integer,
    trackedEffect: integer,
    chainMaxTargets: integer,
    chainJumpRange: integer,
    chainSelector: integer,
    patternId: string,
    stanceGroup?: string,               -- x64 15.25+ only
    stanceId?: string,                  -- x64 15.25+ only
    forceUnknownStance?: boolean,        -- x64 15.25+ only
    runeId?: integer,                   -- present when kind is rune
    spellWords?: string                 -- present when kind is spell
}

TargetingEntry = {
    index: integer,
    monsterName: string,
    monstersIgnoreList: string,
    priority: integer,
    dangerLevel: integer,
    attackOption: integer,
    keepDistanceOption: integer,
    minimumHealthPercentage: integer,
    maximumHealthPercentage: integer,
    keepDistanceRange: integer,
    anchoringRange: integer,
    lootMonster: boolean,
    stayDiagonal: boolean,
    mustBeShootable: boolean,
    mustBeReachable: boolean,
    anchoring: boolean,
    enabled: boolean
}
```

`Engine.ComboBot.GetClientEntries -> ComboClientEntry[]`,
`GetRoomEntries -> ComboRoomEntry[]`, and
`GetRoomState -> ComboRoomState`. `Engine.HUD.GetConfig -> HudFeatureConfig`
and `GetSpecialFoodCounters -> HudSpecialFoodCounter[]`.

All four profile-backed feature namespaces (`Engine.AmmoRefill`,
`Engine.EquipmentManager`, `Engine.MagicShooter`, and `Engine.Targeting`) expose
the same profile API: `GetProfileCount()`, `GetProfileNames()`,
`GetProfiles()`, `FindProfileByName(profileName)`,
`GetActiveProfile()`/`GetCurrentProfile()`,
`SetActiveProfile(profile)`/`SetCurrentProfile(profile)`, `NextProfile()`,
`AddProfile(profileName?)`, `RemoveProfile(profile)`, and
`RenameProfile(profile, newName)`.
Setters accept a 1-based integer index or non-empty exact profile name and
return `false` when it is not found. Active/current/next getters return
`ProfileSummary|nil`; next wraps to the first profile and returns nil when the
feature has no profiles. `GetProfiles -> FeatureProfile[]` always uses the
exact fields `index`, `name`, `active`, and `entryCount`.
`FindProfileByName` performs an exact match. Add selects the new profile and
uses a generated unique name when its argument is nil or empty. Remove and
rename accept the same index-or-name selector; removing the active profile
leaves no profile selected, while removing an earlier profile preserves the
same logical selection by shifting its index. Duplicate add/rename operations
return `false, "Profile name already exists"`. `Engine.AmmoRefill.SetProfile()`
remains a compatibility alias.

`Engine.MagicShooter.GetEntries(profile?) -> MagicShooterEntry[]|nil,
string|nil`. On success the error is nil; on profile resolution failure the
entry list is nil and the error explains why. Exactly one of `runeId` or
`spellWords` is present according to `kind`. The harmony and stance fields
are build-dependent and must be feature-detected.

`Engine.Targeting.GetEntries(profile?) -> TargetingEntry[]|nil`.
`Engine.Scripter.GetAvailableScripts/GetRunningScripts -> string[]`, while
`GetOutput -> string`.

### HUD parameter and getter records

```text
HudRenderLayer = "map"|"overlay"
HudImageBytes = string | integer[]

HudScreenTextParams = {
    id: string,
    text: string,
    color?: ColorRGBA,
    font_family?: string,
    font_size?: integer,
    render_layer?: HudRenderLayer,
    h_align?: integer,
    v_align?: integer,
    is_draggable?: boolean,
    is_clickable?: boolean,
    on_click?: function(),
    z_index?: integer,               -- zIndex is an accepted alias
    enabled?: boolean
}

HudScreenImageParams = {
    id: string,
    source?: string,
    source_base64?: string,
    source_bytes?: HudImageBytes,
    item_id?: integer,
    item_name?: string,
    outfit_id?: integer,
    outfit_name?: string,
    outfit_head?: integer,
    outfit_body?: integer,
    outfit_legs?: integer,
    outfit_feet?: integer,
    outfit_addons?: integer|string,  -- 0..255 mask, or "all"
    outfit_direction?: integer|string, -- 0 north, 1 east, 2 south, 3 west
    outfit_mount_id?: integer,
    width?: number,
    height?: number,
    source_width?: integer,
    source_height?: integer,
    opacity?: number,
    smooth?: boolean,
    label?: string,
    label_color?: ColorRGBA,
    label_offset_x?: number,
    label_offset_y?: number,
    render_layer?: HudRenderLayer,
    h_align?: integer,
    v_align?: integer,
    is_draggable?: boolean,
    is_clickable?: boolean,
    on_click?: function(),
    z_index?: integer,               -- zIndex is an accepted alias
    enabled?: boolean
}

HudWorldTextParams = {
    id: string,
    x: integer,
    y: integer,
    z: integer,
    text: string,
    color?: ColorRGBA,
    lifetime_ms?: integer,
    font_family?: string,
    font_size?: integer,
    render_layer?: HudRenderLayer,
    enabled?: boolean,
    offset_x?: number,
    offset_y?: number,
    z_index?: integer                -- zIndex is an accepted alias
}

HudWorldImageParams = {
    id: string,
    x: integer,
    y: integer,
    z: integer,
    source?: string,
    source_base64?: string,
    source_bytes?: HudImageBytes,
    item_id?: integer,
    item_name?: string,
    outfit_id?: integer,
    outfit_name?: string,
    outfit_head?: integer,
    outfit_body?: integer,
    outfit_legs?: integer,
    outfit_feet?: integer,
    outfit_addons?: integer|string,  -- 0..255 mask, or "all"
    outfit_direction?: integer|string, -- 0 north, 1 east, 2 south, 3 west
    outfit_mount_id?: integer,
    width?: number,
    height?: number,
    source_width?: integer,
    source_height?: integer,
    opacity?: number,
    smooth?: boolean,
    label?: string,
    label_color?: ColorRGBA,
    label_offset_x?: number,
    label_offset_y?: number,
    render_layer?: HudRenderLayer,
    lifetime_ms?: integer,
    enabled?: boolean,
    offset_x?: number,
    offset_y?: number,
    z_index?: integer                -- zIndex is an accepted alias
}

HudWorldBoxParams = {
    id: string,
    x: integer,
    y: integer,
    z: integer,
    width?: number,
    height?: number,
    color?: ColorRGBA,
    border_width?: number,
    border_color?: ColorRGBA,
    lifetime_ms?: integer,
    render_layer?: HudRenderLayer,
    enabled?: boolean,
    z_index?: integer                -- zIndex is an accepted alias
}

HudImageLabelUpdate = {
    id: string,
    label?: string,
    label_color?: ColorRGBA,
    label_offset_x?: number,
    label_offset_y?: number
}

HudWorldPositionUpdate = {
    id: string,
    x: integer,
    y: integer,
    z: integer
}

HudScreenPositionUpdate = {
    id: string,
    x: number,
    y: number
}

HudScreenPosition = {
    x: number,
    y: number,
    POS_X: number,                   -- exact compatibility alias of x
    POS_Y: number,                   -- exact compatibility alias of y
    POS_y: number                    -- exact legacy-case alias of y
}
```

Image add records require exactly one source selector: `source`,
`source_base64`, `source_bytes`, `item_id`, `item_name`, `outfit_id`,
or `outfit_name`. Outfit palette fields accept Tibia color indexes from 0
through 132. `outfit_addons` accepts a bit mask from 0 through 255 or
`"all"`. `outfit_direction` accepts `north`, `east`, `south`, `west`, or the
corresponding integer `0..3`, and defaults to south. `outfit_mount_id = 0`
explicitly removes a curated mount. When
`is_clickable` is true, `on_click` is required. `HUD.SetPointerCallbacks(id,
on_mouse_down?, on_mouse_up?, on_mouse_leave?)` adds press/release/leave
callbacks to an existing screen text or image. Mouse-leave fires once when a
pressed pointer leaves the element, including canceled interactions; mouse-up
fires on release while the interaction is still owned. All pointer callbacks
take no arguments. Click, pointer, and drag-end callbacks are event critical
paths and must return immediately; drag-end receives `(x: number, y: number)`.

`Engine.HUD.AddScreenText/AddScreenImage/AddWorldText/AddWorldImage/AddWorldBox`
return the same validated parameter table supplied by the caller. Position,
label, and color update calls return `nil`. The yielding HUD getters return
`ColorRGBA`, `Position`, or `HudScreenPosition` exactly; they must never be
called from an event/hook callback.

The wrapper getters map as follows:

- `ScreenText:GetColor`, `WorldText:GetColor`, and `WorldBox:GetColor` return
  `ColorRGBA`.
- `ScreenText:GetPosition` and `ScreenImage:GetPosition` return
  `HudScreenPosition`, including all three compatibility aliases.
- `WorldText:GetPosition`, `WorldBox:GetPosition`, and
  `WorldImage:GetPosition` return `Position`.
- Wrapper setters and `Create` return the same wrapper object for chaining;
  `Remove -> nil`; boolean state getters return booleans; width and height
  getters return numbers.

### Persistent storage records

```text
StorageScope = private per-script logical namespace object
SharedStorageScope = named cross-script namespace object

SharedStorageChangeWriter = {
    id: string,
    name: string,
    type: "script"|"walker"|"one_shot"|"unknown"
}

SharedStorageChangeEvent = {
    namespace: string,
    operation: "set"|"remove"|"clear",
    scope: "global"|"character",
    revision: integer,
    timestampUnixMs: integer,
    writer: SharedStorageChangeWriter,
    character?: string,
    key?: string,
    previousExists: boolean,
    newExists: boolean,
    previousValueIncluded: boolean,
    newValueIncluded: boolean,
    previousValue?: JsonValue,
    newValue?: JsonValue,
    changedCount?: integer,
    changedKeys?: string[],
    changedKeysTruncated?: boolean,
    valueOmissionReason?: string
}
```

Set/remove events always include `key`. An unfiltered clear event instead
includes `changedCount`, `changedKeys`, and `changedKeysTruncated`; a key-filtered
clear includes that subscribed `key` and may include its previous value.
`SharedStorageScope:OnChanged` takes
`function(event: SharedStorageChangeEvent)` and returns
`string|nil, string|nil` (`subscriptionId, errorMessage`). Its callback is an event
critical path and must only hand minimal scalar state to the stable worker.

## Core Libraries Overview

### json.lua
Native-backed JSON encoding and decoding. It does not load a Lua C module or an additional dependency DLL.

Primary API:
- Json.Encode(value, pretty?)
- Json.Decode(text)
- Json.TryEncode(value, pretty?)
- Json.TryDecode(text)
- Json.Array(table)
- Json.Object(table)
- Json.Null

Notes:
- Json.Encode and Json.Decode raise normal, catchable Lua errors when encoding or decoding fails.
- Json.TryEncode and Json.TryDecode do not throw for codec failures; they return `nil, error`.
- Decoded JSON null values are represented by Json.Null so arrays and objects round-trip without losing null entries.
- Empty Lua tables encode as objects by default. Use Json.Array({}) to force an empty array.
- Input/output is limited to 2 MB, nesting to 32 levels, and values to 65,536 nodes.
- Only nil, booleans, finite numbers, strings, tables, and Json.Null are supported.

### http.lua
Coroutine-friendly HTTP and HTTPS requests. No extra dependency DLL is required.

Primary API:
- Http.Request(options)
- Http.Get(url, options?)
- Http.Post(url, body?, options?)
- Http.GetJson(url, options?)
- Http.PostJson(url, value, options?)

Request options:
- url: string (required)
- method: GET, POST, PUT, PATCH, DELETE, or HEAD (default GET)
- headers: table of string names to string values
- body: string (default empty; maximum 1 MB)
- timeoutMs: 250..30000 (default 5000; one total active-request deadline, including redirect handling)
- maxResponseBytes: 1..4194304 (default 2 MB)
- followRedirects: boolean (default true; up to five redirects)

The returned response contains `ok`, `status`, `headers`, `headerList`, `body`, `error`, `url`, and `redirects`. `headers` uses lower-case names for convenient lookup; `headerList` preserves repeated headers as ordered `{ name, value }` entries. `GetJson` returns `decodedValue, response, decodeError`; `PostJson` JSON-encodes the request value and returns the normal response table.

Calls must run inside a managed script coroutine; `init()` is managed and may call them. A script may have at most four pending HTTP requests. Local/private endpoints are allowed. `validusbot.net`, its subdomains, and the registered service host are blocked, including redirect destinations. TLS certificate validation remains enabled, HTTPS-to-HTTP redirects are rejected, and sensitive/custom headers are not forwarded across origins except for a small safe set such as `Accept` and `User-Agent`. Cookies and automatic authentication are not provided.

Per-request User-Agent:
- Set `headers = { ["User-Agent"] = "MyScript/1.0" }` in that request's options.
- The value is scoped to that request and is retained across allowed redirects.
- Header names/values must be strings; at most 64 request headers and 32 KB of combined header text are accepted.

### websocket.lua
Independent `ws://` and `wss://` client connections. No extra dependency DLL is required.

Primary API:
- WebSocket.Connect(url, options?)
- connection:Send(data, binary?)
- connection:Receive(timeoutMs?)
- connection:Close(closeCode?, reason?)
- connection:IsOpen()

`Connect` options support `headers`, `subprotocol`, `timeoutMs` (default 10000), and `maxMessageBytes` (default 1 MB, maximum 4 MB). `maxMessageBytes` limits both messages sent by the script and messages received from the server. `Connect` returns `connection, nil` on success or `nil, error` on failure. Headers, including a custom `User-Agent`, are scoped to that connection attempt.

`Receive` yields and returns an event table with `type` equal to `text`, `binary`, `close`, `error`, or `timeout`; relevant fields include `data`, `closeCode`, `error`, and `url`. Its default timeout is 30000 ms and values are clamped to 0..60000. `Send` and `Close` return `boolean, errorOrNil`.

A script may own up to four simultaneous WebSockets; eight are allowed across all scripts. Queues and message sizes are bounded, text/reason values must be valid UTF-8, and close codes/reasons are validated. Connections and outstanding operations are cancelled when their owning script stops. `validusbot.net`, its subdomains, and the registered service host are blocked. `wss://` uses normal TLS certificate validation. Reconnect is explicit: after a close/error, call `WebSocket.Connect` again from script logic.

### features.lua
Public feature control wrappers.

- Supports identifiers by numeric BotFeatureId or feature name string.
- Public IDs are HEALER..TIMER_ACTIONS.
- Objects Dumper is reserved and intentionally blocked.

Primary API:
- Features.IsActive(featureIdentifier)
- Features.Enable(featureIdentifier)
- Features.Disable(featureIdentifier)
- Features.Toggle(featureIdentifier)
- Features.SetActive(featureIdentifier, activeStatus)
- Features.GetName(featureIdentifier)
- Features.GetAllFeatureIds()
- Features.GetActiveFeatures()
- Features.EnableMultiple(featureList)
- Features.DisableMultiple(featureList)
- Features.DisableAllExcept(excludeList)
- Features.PrintStatus()

`Enable` and `Disable` use the native idempotent `SetActive` operation and return
the resulting active state. `Toggle` also returns the new state, but should be
used only when inversion is the intended operation. Feature activation changes
live bot state; the feature's own settings remain intact.

### Magic Shooter entries
Lua scripts can inspect and configure existing Magic Shooter entries without
rebuilding profiles. Magic Shooter uses one normalized action model for spells,
runes, directional attacks, target-centered areas, chains, support effects, and
stances:

- `Engine.MagicShooter.GetEntries(profile?) -> entries|nil, error?`
- `Engine.MagicShooter.SetEntryRune(entryIndex, runeId, profile?) -> success, error?`
- `Engine.MagicShooter.SetEntrySpell(entryIndex, spellWords, profile?) -> success, error?`
- `Engine.MagicShooter.SetEntryEnabled(entryIndex, enabled, profile?) -> boolean`
- `Engine.MagicShooter.SetEntryMonsterNames(entryIndex, names, profile?) -> boolean`
- `Engine.MagicShooter.SetEntryRange(entryIndex, range, profile?) -> boolean`
- Invocation and placement: `SetEntryCastMethod`, `SetEntryPatternAnchor`, `SetEntryPatternSource`, `SetEntryPatternVariant`, and `SetEntryPatternId`.
- Evaluation and ordering: `SetEntryEffectType`, `SetEntryPriorityLane`, `SetEntryTargetPolicy`, `SetEntryHitCountMode`, `SetEntryMonsterCount`, and `SetEntryMonsterCountCondition`.
- Chain behavior: `SetEntryChainMaxTargets`, `SetEntryChainJumpRange`, and `SetEntryChainSelector`.
- Requirements/effect tracking: `SetEntryEquipmentRequirement` and `SetEntryTrackedEffect`.
- Explicit setters also exist for option, condition, mana/health thresholds, monster HP range, danger, PvP safety, ally shooting, target requirement, custom/walk/momentum delays, skill-buff percentages, and movement/momentum flags.

`profile` defaults to the active Magic Shooter profile. It may be a 1-based profile index or an exact profile name. Entry indexes are also 1-based and match the order shown in the selected profile. `GetEntries` returns a detached snapshot with all public condition, threshold, delay, safety, skill-buff, and pattern fields. Editing that returned table has no effect; call the matching setter.

The enum tables live under `Engine.MagicShooter`: `Option`, `Condition`,
`CastCondition`, `MonsterCountCondition`, `CastMethod`, `PatternAnchor`,
`PatternSource`, `PatternVariant`, `EffectType`, `PriorityLane`, `TargetPolicy`,
`HitCountMode`, `EquipmentRequirement`, `TrackedEffect`, and `ChainSelector`.
Use these named constants instead of hard-coded integers. `PatternVariant`
exposes only `Default` and `Custom`; numeric value `1` is reserved and rejected.

Runtime selection is strict vector order. On every pass Magic Shooter starts at
entry 1, checks that entry's complete type-specific conditions and cooldowns,
then continues downward only when it cannot use that entry. The first fully
eligible entry is used. Spell/rune type, area size, `PriorityLane`, and
`dangerLevel` do not reorder evaluation. Those advanced fields remain readable
and settable for profile/model compatibility, but scripts must arrange actual
priority by moving entries in the profile UI.

Conditions are evaluated per entry, including `DontCastWhileWalking`; enabling
that option on one entry does not suppress a later entry that permits casting
while walking. Creature candidates are read from current game storage while the
entry is being evaluated. There is no shared Lua-style creature snapshot whose
contents can be edited or reused to change the selection.

Chain entries can count guaranteed hits or possible hits. `ChainSelector` provides `Closest`, `Random`, and `HighestHealth`; use the named constant because historical and current chain spells may use different selectors. Monster name, HP, and tracked-effect filters decide which chained monsters count toward the condition, but excluded creatures may still physically relay a chain. PvP safety follows the complete possible chain and rejects a cast that could reach a non-allowed player. Current-target chains fall back to a valid in-range creature when necessary.

For directly targeted actions (runes and supported crosshair spells), `SetEntryRequiresTarget(false)` allows the configured target-selection policy to choose a valid creature without requiring an existing client attack target. Ordinary spoken targeted spells still resolve against Tibia's current attack target and cannot be redirected by a bot-only policy.

Monster names are case-insensitive and accept comma, semicolon, or newline separators. Range, floor, shootability, monster HP, and Monster Name filters are applied consistently to spells and runes. PvP safety is evaluated independently of monster-name and HP filters.

Momentum is the sole intentional priority exception. When the Momentum effect is
triggered, Magic Shooter scans from the top for the first enabled attack spell
or attack rune with `PrioritizeWithMomentum` enabled whose individual cooldown
is at most its configured `MomentumDelay`. That entry is reserved for the
Momentum window. Until it fires or the window expires, other attack spells and
runes are skipped. Stances, empowerment/support entries, challenge/exeta, and
avatar entries may still run. The option is meaningful only for attack spells
and runes; scripts should not enable it for support/stance entries.

`SetEntryCustomDelay` also provides the retry window for custom actions whose
cooldown is not exposed by the client; when a normal entry is waiting, scanning
continues to the next entry. `SetEntryAttackSkillBuffSpell` remains for old
support entries, but new scripts should prefer `TrackedEffect.SkillBuff`.

x64-only controls are exported only when supported by the native client build.
Harmony and Monk-specific controls belong to x64 clients from 15.00 onward.
`CrossHairSpell`, `TargetOrSelf`, stance behavior/effect tracking, and
`SetEntryStanceGroup`, `SetEntryStanceId`, and
`SetEntryForceUnknownStance` require the newer x64 client features from 15.25
onward. None exist in the x86 API. Check
`type(Engine.MagicShooter.SetEntryStanceId) == "function"` when targeting both
architectures.

The stance tracker observes outgoing manual and scripted stance words even while
Magic Shooter is disabled. It groups mutually exclusive stances, tracks a
pending cast, confirms it from spell cooldown/exhaustion, and abandons an
unconfirmed attempt after two seconds. Casting the currently active stance is
tracked as toggling that group off. State resets to unknown after injection or a
character change. By default an unknown group does not force a cast; enable
`SetEntryForceUnknownStance` only for the entry that should establish the
initial state.

To build a sequence such as `uteta flam`, then a fire attack, then another
attack, place those entries in that order. The stance entry runs only when its
group is not already in the requested state. Subsequent strict-order passes can
then reach the dependent attack and later fallback entries as their own
conditions/cooldowns allow.

Replacing an action preserves that entry's enabled state, position, monster
names, creature-count/health/mana conditions, delays, momentum option, stance
fields, and advanced settings. A recognized rune or spell also updates its
action type, range, and built-in pattern using the same automatic detection as
the Magic Shooter UI. An unknown custom action can replace an entry that is
already the same kind, but cannot cross from rune to spell or spell to rune
because the missing action metadata would be ambiguous.

Entry/profile fields changed through Lua affect live in-memory settings and are
included in a later normal settings save. Older settings files load with safe
defaults for newly added fields, and the next save writes the new fields.
Momentum timestamps, pending/active stance tracking, per-entry retry timers, and
other transient runtime state are intentionally not persisted.

### Targeting entries

- `Engine.Targeting.GetEntries(profile?) -> table[]|nil`
- `Engine.Targeting.SetEntryEnabled(entryIndex, enabled, profile?) -> boolean`
- `Engine.Targeting.SetEntryMonsterName(entryIndex, name, profile?) -> boolean`
- `Engine.Targeting.SetEntryMonstersIgnoreList(entryIndex, names, profile?) -> boolean`
- Explicit setters cover priority, danger, attack option, keep-distance option/range, HP range, anchoring/range, looting, diagonal movement, shootable, and reachable requirements.

Targeting getters return detached snapshots. Monster-name and ignore-list setters rebuild the same lowercase parsed caches used by targeting. Ignore lists accept comma, semicolon, or newline separators.

### Other feature control namespaces

The following are grouped under `Engine` and use explicit getters/setters or feature actions:

- `Engine.Walker` and `Engine.Lure`: waypoint/lure configuration and runtime actions already available through their native APIs.
- `Engine.Extras`: individual toggles, follow settings, training actions, and copied item-ID lists.
- `Engine.TankMode`: mana-shield/cancel spells, thresholds, costs, potion ID, and individual toggles.
- `Engine.Looter`: mode, action type, minimum capacity, and `LootAroundCharacter()`.
- `Engine.TimerActions`: copied entries, individual entry setters, add/remove/clear.
- `Engine.SuppliesSorter`: copied entries, individual entry setters, add/remove/clear.
- `Engine.Channels`: copied entries, individual entry setters, global delay, add/remove/clear. Mutations clear stale queued messages where required.
- `Engine.HUD`: the HUD element API with its original per-script ownership semantics.

All `GetEntries()` results and ID lists are copies. Never modify those tables expecting bot state to change. Configuration changes remain in memory until the bot's normal settings save runs.

Example HUD callback logic for a rune entry:

```lua
local AREA_RUNE_ENTRY = 4

local function selectAvalanche()
    local ok, err = Engine.MagicShooter.SetEntryRune(AREA_RUNE_ENTRY, 3161)
    if not ok then
        print("Could not select avalanche rune: " .. tostring(err))
    end
end
```

### hotkeys.lua
Combo parser and event registration wrapper.

Primary API:
- Hotkeys.ParseCombo(combination)
- Hotkeys.RegisterCombo(params)
- Hotkeys.SendKey(key, clientOnly?)
- Hotkeys.SendCombo(combination, clientOnly?)

`SendKey` and `SendCombo` always queue a key-down/key-up sequence for the current injected Tibia client window. `clientOnly` defaults to `true`, which bypasses ImGui, Lua callback hotkeys, and ValidusBot feature hotkeys. Pass `false` only when the synthetic input should travel through the normal client window procedure. A `true` return means the sequence was queued, not that Tibia had an action assigned to it. No arbitrary window handles or system-wide input are exposed.

```lua
Hotkeys.SendKey("f1")
Hotkeys.SendCombo("ctrl+shift+f9")
Hotkeys.SendCombo("alt+f1")
Hotkeys.SendCombo("ctrl+f1", false) -- opt into normal ImGui/bot/Lua routing
```

Expected params for RegisterCombo:
- id: string (required)
- combo: string (required)
- callback: function (required)
- name: string (optional)
- trigger_on_keydown: boolean (optional)
- extended: boolean (optional override)

### module.lua
Module runtime plus scheduling helpers built on Module.New/Stop/Pause/Resume.

Primary API:
- Module.New(name, callback, delayMs)
- Module.Stop(name)
- Module.Pause(name)
- Module.Resume(name)
- Module.Every(name, callback, delayMs)
- Module.After(name, callback, delayMs)
- Module.Cancel(name)
- Module.PauseManaged(name)
- Module.ResumeManaged(name)
- Module.Exists(name)
- Module.Get(name)
- Module.List()

`Module.New` creates a repeating managed coroutine. Its callback may call
`wait(...)`; it must not busy-loop. `Module.Every` is the recommended repeating
helper: it stops an existing module with the same name before registering the new
one and records it in the core helper registry. `Module.After` is a one-shot
managed coroutine that waits, invokes the callback once, then removes itself.
`Module.Cancel`, `PauseManaged`, `ResumeManaged`, `Exists`, `Get`, and `List`
operate on modules created through `Every` or `After`; raw `Module.New` modules
are not added to that Lua-side registry.

Names must be non-empty and should be stable and script-specific. Delays are
integer milliseconds from 0 through 86,400,000. An uncaught callback error is
reported with script/module context and stops the offending module. It stops the
entire script only when no healthy sibling work remains. Do not wrap the whole
repeating callback in `pcall`; catch only an operation whose failure is expected
and recoverable.

### position.lua
Position object utilities and spatial checks.

Primary API:
- Position.New(x, y, z | table)
- Position:DistanceTo(otherPos)
- Position.IsReachable(fromPos, toPos)
- Position.IsShootable(fromPos, toPos)
- targetPosition:IsReachable(fromPos?)
- targetPosition:IsShootable(fromPos?)

Notes:
- Reach/shoot checks default source to local player when fromPos is nil.
- DistanceTo returns 9999 on different z-level.

### creature.lua
Creature wrapper for reading and interacting with visible game creatures.

Factory and static helpers:
- Creature:New(creatureId)
- Creature.GetFollowed()
- Creature.GetTarget()
- Creature.GetLocalPlayer()

Core methods include:
- identity/state: GetId, GetName, GetLowercaseName, IsValid, IsVisible
- relation/type: IsPlayer, IsMonster, IsNPC, IsSummon, IsGameMaster, IsMounted, IsInParty, IsPartyLeader, IsInGuild, IsWarEnemy, IsWarAlly, IsSkulled
- stats/meta: GetHealthPercent, GetDirection, GetSpeed, GetVocation, GetSkull, GetPartyShield, GetGuildShield, GetOutfit, GetMasterId
- spatial: GetPosition, DistanceTo, DistanceToCreature, IsAdjacentTo, IsSameFloor, IsReachable, IsShootable
- utilities: Equals, ToString, ClearCache

### creature_iterators.lua
Iterators and scan helpers for creature collections.

Iterators:
- Creature.ICreatures()
- Creature.IPlayers()
- Creature.IMonsters()
- Creature.INpcs()

Collection helpers:
- Creatures.GetVisibleCreatureIds()
- Creatures.GetVisibleCreatures()
- Creatures.GetCreatureByName(name)
- Creatures.GetLocalPlayerId()
- Creatures.GetPlayerIdUnderMouse()
- Creatures.GetFollowingCreatureId()
- Creatures.GetAttackingCreatureId()
- Creatures.IsCreatureOnScreen(...)
- Creatures.GetCreatureIdsByScan(...)
- Creatures.GetCreaturesByScan(...)
- Creatures.GetVisiblePlayers()
- Creatures.GetVisibleMonsters(ignoreSummons)
- Creatures.GetVisibleNpcs()

### chat_channel.lua and chat_channel_storage.lua
Normalized access to opened/sendable chat channels.

Object API:
- ChatChannel.New(channelOrId, channelName?)
- ChatChannel.FromIdentifier(identifier)
- ChatChannel.GetById(channelId) / GetByName(channelName)
- channel:GetId() / GetName() / CanSend() / IsOpened() / IsLocal() / IsServerLog() / IsValid()
- channel:Send(message) / Refresh() / ToTable() / ToString()

Storage/query API:
- ChatChannelStorage.IsAvailable()
- ChatChannelStorage.GetOpenedChannels() / GetChatChannels()
- ChatChannelStorage.GetLocalChatChannel() / GetServerLogChannel()
- ChatChannelStorage.GetChatChannelByName(name) / GetChatChannelById(id)
- ChatChannelStorage.HasChannelByName(name) / HasChannelById(id)
- ChatChannelStorage.GetOpenedChannelCount() / GetChatChannelCount()
- ChatChannelStorage.GetChannelNames(onlySendable?)
- ChatChannelStorage.ResolveChannel(identifier) / CanSend(identifier) / Send(message, identifier)
- ChatChannelStorage.ToNameLookupTable() / ToIdLookupTable() / GetSnapshot() / FormatChannel(channel)

A channel identifier may be an id, name, or normalized channel table. Public channel fields are `id`, `name`, `canSend`, `isOpened`, `isLocal`, and `isServerLog`. Check `CanSend` before sending; local chat and server-log pseudo-channels are not normal sendable channels.

### self.lua
Player-centric wrapper API.

Common state:
- Self.GetHealth / GetMaxHealth / GetHealthPercentage
- Self.GetMana / GetMaxMana / GetManaPercentage
- Self.GetCapacity / GetStamina
- Self.GetItemCount(itemId, tierLevel?)
- Self.GetCharacterWorld(characterName)
- Self.GetLevel / GetSoul / GetLevelPercentage
- Self.IsOnline / IsAlive / IsAttacking / IsFollowing
- Self.GetTargetId / GetFollowId / HasTarget / HasFollow

World/mouse:
- Self.GetMousePositionInWorld
- Self.GetMouseWorldX/Y/Z
- Self.GetMousePositionText

Actions:
- chat: Say, Whisper, Yell, SayOnChannel, SayToNpc, PrivateMessage
- combat: Attack, Follow, StopAttackAndFollow
- movement: Step, CancelWalk, Mount, Dismount
- interaction: UseItemInContainer, UseItemOnFloor, LookAtPosition, LookAtCreature
- npc trade: BuyItem, SellItem

Utilities:
- Self.GetStatsSnapshot()
- Self.FormatStatsSnapshot(...)
- Self.IsAvailable()

### game.lua
Account/session and window helpers.

Login/session API:
- Game.GetCharacterWorld(characterName)
- Game.LoginToPreviouslyLoggedCharacter()
- Game.LoginToAccount(email, password)
- Game.LoginToCharacter(characterName)
- Game.Logout()
- Game.EnterWorld()
- Game.OpenStore()
- Game.OpenContainerInNewWindow(equipmentSlotOrContainerId, fromContainerNumber?, fromContainerSlot?)

Notes:
- Login functions validate required string arguments and return boolean success from the Lua wrapper.
- `OpenContainerInNewWindow` accepts either one `EquipmentSlot.*` value or a container item id plus its source container number and source slot.
- Prefer Self wrappers for local-player convenience operations when they exist.

### map.lua
Map tile interaction wrappers.

Primary API:
- Map.UseItemOnFloor(position, stackPosition, itemId)
- Map.Look(position)
- Map.MoveItemFloorToContainer(itemId, fromPosition, containerIndex, slotIndex, itemCount)
- Map.MoveItemFloorToFloor(fromPosition, itemId, toPosition, itemCount)
- Map.GetTileFlags(position)
- Map.GetTileItems(position, includeCreatures)
- Map.GetObjectInfo(itemId)
- Map.FindPath(fromPosition, toPosition, maxComplexity?, flags?, allowSpeculativeUnknown?)

The final boolean defaults to false. Set it to true together with
PathFindFlags.ALLOW_NOT_SEEN_TILES only for speculative planning through
missing minimap pixels.

`Map.FindPath` preserves the legacy path topology. It rejects a visible teleport
as an intermediate node, but permits one when it is the exact requested goal.
For scripted one-step movement, inspect `Map.GetTileFlags(next).hasTeleport` and
the `isFloorChange` / `isTeleport` fields returned by `Map.GetTileItems(next)`;
then treat `Self.Step(...) == true` only as dispatch acceptance and confirm that
the observed player position reached the expected same-floor tile before consuming
the path direction. Replan on timeout or any unexpected/floor-changing position.

### minimap.lua
Read-only minimap tile queries and pathfinding helpers.

Primary API:
- Minimap.GetTileFlags(position)
- Minimap.GetTileItems(position, includeCreatures?)
- Minimap.IsWalkable(position) / IsPathable(position)
- Minimap.GetTilePixelColor(position)
- Minimap.IsPixelColorWalkable(pixelColorIndex)
- Minimap.IsWalkableByColor(position)
- Minimap.FindPath(fromPosition, toPosition, maxComplexity?, flags?, allowSpeculativeUnknown?)

The final boolean defaults to false and is forwarded to the game pathfinder.
- Minimap.GetTileInfo(position, includeCreatures?)

Tile checks may return nil when minimap data is unavailable. `FindPath` returns `{ Directions = integer[], pathFindResult = PathFindResult.* }`; note the capital `D` in `Directions`. `GetTileInfo` returns `position`, `flags`, `items`, `pixelColor`, and `walkableByColor`.

### item.lua
Item use and trade wrappers.

Primary API:
- Item.Use(itemId)
- Item.UseOnSelf(itemId)
- Item.UseOnCreature(itemId, creatureId)
- Item.Buy(itemId, itemCount, ignoreCapacity, buyInShoppingBags)
- Item.Sell(itemId, itemCount, sellEquipped)
- Item.UseFromContainerOnFloor(floorPosition, fromItemId, toItemId, toStackPosition)
- Item.UseFromFloorToContainer(floorPosition, fromItemId, fromStackPosition, toItemId)
- Item.UseFromContainerToContainer(fromContainer, fromSlot, fromItemId, toContainer, toSlot, toItemId)
- Item.GetInfo(itemId)
- Item.GetName(itemId)
- Item.GetDescription(itemId)
- Item.HasFlag(itemId, fieldName)
- Item.IsContainer(itemId)
- Item.IsCumulative(itemId)
- Item.IsUsable(itemId)
- Item.IsMultiUsable(itemId)
- Item.IsMovable(itemId)
- Item.IsTakable(itemId)
- Item.IsGround(itemId)
- Item.IsLiquidContainer(itemId)
- Item.IsCreature(itemId)
- Item.GetFromContainer(containerNumber, slotIndex)
- Item.FindInContainer(containerNumber, itemId, tierLevel)

### npc_trade_storage.lua
Capability-safe access to the currently opened NPC trade window.

Primary API:
- NpcTradeStorage.IsAvailable() / IsOpen()
- NpcTradeStorage.GetNpcName()
- NpcTradeStorage.GetOffers()
- NpcTradeStorage.GetOfferByItemId(itemId) / GetOfferByName(itemName)
- NpcTradeStorage.Buy(itemId, itemCount, ignoreCapacity?, buyInShoppingBags?)
- NpcTradeStorage.Sell(itemId, itemCount, sellEquipped?)
- NpcTradeStorage.FormatOffers()
- NpcTradeStorage.GetSnapshot()

Normalized offers contain `itemId`, `name`, `buyPrice`, `sellPrice`, and `capacity`. State/capability calls can return nil when no supported trade window is open; validate offers before buying or selling.

### container.lua
Container item movement and look wrappers.

Primary API:
- Container.MoveItemToContainer(...)
- Container.UseItem(...)
- Container.MoveItemToFloor(...)
- Container.MoveItemFromEquipmentToContainer(...)
- Container.MoveItemToEquipment(...)
- Container.LookItem(...)
- Container.GetOpenContainers()
- Container.GetByNumber(containerNumber)
- Container.GetByName(containerName)
- Container.GetById(containerId)
- Container.GetItems(containerNumber)
- Container.GetItem(containerNumber, slotIndex)
- Container.FindItem(containerNumber, itemId, tierLevel)
- Container.FindItemInOpenContainers(itemId, tierLevel)
- Container.GetSize(containerNumber)
- Container.GetItemsCount(containerNumber)
- Container.GetFreeSlots(containerNumber)
- Container.GetId(containerNumber)
- Container.GetName(containerNumber)

### inventory.lua
Equipment-slot reading and movement helpers.

Primary API:
- Inventory.GetEquipmentSlotConstants()
- Inventory.CanReadEquipment() / CanMoveEquipment()
- Inventory.GetSlotItem(equipmentSlot) / GetAllSlotItems()
- Inventory.Equip(itemId, tierLevel?)
- Inventory.LookSlotItem(itemId, equipmentSlot)
- Inventory.MoveFromContainerToSlot(containerIndex, slotIndex, itemId, equipmentSlot, itemCount?)
- Inventory.MoveFromSlotToContainer(equipmentSlot, containerIndex, slotIndex, itemId, itemCount?)
- Inventory.GetSlotItemId(equipmentSlot) / HasItemInSlot(equipmentSlot)
- Inventory.GetSlotIds() / GetSnapshot()

Use `EquipmentSlot.*` constants. Read helpers may return nil when equipment access is unavailable; call the capability methods and avoid assuming that a nil item means an empty slot.

### cooldowns.lua
Cooldown abstraction helpers.

Namespaces:
- Cooldowns.Spell: IsInCooldown, GetTimeLeft, WillBeReady, IsReady
- Cooldowns.Item: IsInCooldown, GetTimeLeft, WillBeReady, IsReady
- Cooldowns.Group: IsInCooldown, GetTimeLeft, WillBeReady, IsReady
- Cooldowns.UseWith: IsExhausted, IsReady
- Cooldowns.Utils: FormatTime, GetStatus, PrintStatus

### spells.lua
Primary API:
- Spells.GetIdByWords(words)
- Spells.GetIdByName(name)
- Spells.GetWordsById(spellId)
- Spells.IsInCooldown(spellWordsOrId)
- Spells.GetLeftCooldownTime(spellWordsOrId)
- Spells.IsReady(spellWordsOrId)
- Spells.WillBeReady(spellWordsOrId, timeMs)
- Spells.GetGroupIds(spellWordsOrId)
- Spells.GetLeftGroupCooldownTime(groupId)
- Spells.GroupIsInCooldown(groupId)
- Spells.IsUseWithItemExhausted()
- Spells.GetInfo(spellWordsOrId)
- Spells.Item.* for rune/item cooldown APIs

### Native Events API

Use the high-level `Hotkeys`, `Cavebot`, and proxy APIs when one matches the
task. The native `Events` table is available for direct scheduling and custom
registrations:

- `Events.Schedule(callback, delayMs, ...args) -> string` returns an owner-scoped event ID.
- `Events.GetScheduledEvents() -> string[]` returns this script's pending IDs.
- `Events.CancelScheduledEvent(eventId) -> boolean` cancels a pending callback.
- `Events.RegisterKeyEvent(options) -> string` registers a key callback and returns its opaque registration ID; use `Hotkeys.RegisterCombo` for normal combinations.
- `Events.RegisterPacketEvent(options) -> string` accepts `id`, `packet_id` (one opcode or an array), `callback`, and optional `incoming` (default `true`), then returns its opaque registration ID. For exactly one incoming `GAME_SERVER_TEXT_MESSAGE` opcode, a 1-1024-byte `message_contains` value adds a native literal substring filter before any per-packet Lua table or coroutine is allocated; optional `message_case_sensitive` defaults to `true`, and `false` uses ASCII case folding.
- `Events.RegisterWalkerEvent(eventId, callback) -> integer|nil` returns a function reference for singular unregistration.
- `Events.UnregisterKeyEvent(registrationId)`, `UnregisterPacketEvent(registrationId)`, and `UnregisterWalkerEvent(functionRef)` return booleans. Pass the exact opaque value returned at registration. A stale or foreign ID returns `false` and cannot remove another script's callback.
- `Events.UnregisterAllKeyEvents()`, `UnregisterAllPacketEvents()`, and `UnregisterAllWalkerEvents()` remove this script's registrations.

`Events.RegisterKeyEvent(options)` accepts exactly:

- `id: string` (required, non-empty);
- `keycode: integer` (required, 0..255);
- `callback: function()` (required; receives no arguments);
- `name?: string` (defaults to `id`);
- `trigger_on_keydown?: boolean`, `shift?: boolean`, `ctrl?: boolean`,
  and `extended?: boolean` (all default `false`).

`Events.RegisterPacketEvent(options)` accepts exactly:

- `id: string` (required, non-empty);
- `packet_id: integer|integer[]` (required byte opcode or non-empty array);
- `callback: function(packet: PacketEventPayload)` (required);
- `incoming?: boolean` (default `true`);
- `message_contains?: string` (1..1024 bytes; allowed only for exactly one
  incoming text-message opcode);
- `message_case_sensitive?: boolean` (default `true`; valid only with
  `message_contains`).

Every incoming and outgoing callback receives `opcode: integer`. Only the
following additional payload fields are decoded. Field names are exact and
case-sensitive:

- Incoming `GAME_SERVER_TEXT_MESSAGE`:
  `{ opcode, message: string, message_class: integer }`.
- Incoming talk:
  `{ opcode, statement_id: integer, creature_name: string,
  creature_message: string, position: Position, player_level: integer,
  channel_id: integer, speak_type: integer, player_is_traded: boolean }`.
- Incoming create-on-map:
  `{ opcode, position: Position, stack_position: integer,
  add_on_map_type: integer }`, plus either
  `item_id: integer, item_count: integer` for an object or
  `creature_id: integer, creature_name: string` for a creature.
- Incoming delete-on-map:
  `{ opcode, position: Position, stack_position: integer }`.
- Incoming move-creature:
  `{ opcode, creature_id: integer, old_position: Position,
  new_position: Position, old_stack_position: integer }`.
- Incoming full-map:
  `{ opcode, update_position: Position, sqm_positions: PacketMapSquare[] }`.
- Incoming top/right/bottom/left row:
  `{ opcode, sqm_positions: PacketMapSquare[] }`.
- `PacketMapSquare` is
  `{ position: Position, item_ids: integer[] }`.
- Incoming graphical-effect:
  `{ opcode, effects: PacketGraphicalEffect[] }`, where each effect is
  `{ from_position: Position, to_position: Position,
  magic_effect_class: integer, shoot_type: integer,
  tibia_effects_type: integer }`. Effect type is `1` for missile, `2` for
  graphical/magic, and `4` for sound; fields irrelevant to that type retain
  their zero/none value.
- Incoming player-inventory:
  `{ opcode, items: PacketInventoryItem[] }`, where each item is
  `{ item_id: integer, item_count: integer }`.
- Incoming unjustified-points:
  `{ opcode, full_kills_progress_in_day: integer,
  full_kills_left_in_day: integer, full_kills_progress_in_week: integer,
  full_kills_left_in_week: integer, full_kills_progress_in_month: integer,
  full_kills_left_in_month: integer,
  remaining_skull_time_in_days: integer }`.
- Any other incoming opcode currently receives only `{ opcode: integer }`.
- Outgoing auto-walk:
  `{ opcode, path_count: integer, path: integer[] }`.
- Outgoing directional walk and stop packets contain only `{ opcode }`.
- Outgoing equip-item:
  `{ opcode, item_id: integer, count_or_tier: integer,
  count_or_tier_bytes: 1|2 }`.
- Outgoing move-item:
  `{ opcode, from_position: Position, item_id: integer,
  from_stack_pos: integer, to_position: Position, item_count: integer,
  item_count_bytes: 1|2 }`.
- Outgoing use-item:
  `{ opcode, position: Position, item_id: integer, stack_pos: integer,
  use_index: integer }`.
- Outgoing use-item-with:
  `{ opcode, from_position: Position, from_item_id: integer,
  from_stack_pos: integer, to_position: Position, to_item_id: integer,
  to_stack_pos: integer }`.
- Outgoing use-on-creature:
  `{ opcode, from_position: Position, item_id: integer,
  from_stack_pos: integer, creature_id: integer }`.
- Outgoing look:
  `{ opcode, position: Position, item_id: integer, stack_pos: integer }`.
- Outgoing talk always exposes `speak_type` after its layout is long enough.
  Recognized valid say/whisper/yell/spell/private-to/channel layouts also expose
  `message`, with optional `receiver` or `channel_id` as appropriate.
- Outgoing attack:
  `{ opcode, target_id: integer, sequence?: integer }`.
- Any other outgoing opcode receives only `{ opcode: integer }`.

Fixed-layout outgoing decoders validate the exact protocol payload size first.
They accept the payload itself, one explicit trailing NUL, or the bounded
all-zero remainder of the current eight-byte outgoing transport block. Nonzero
suffix data and padding beyond one block are rejected. The native bridge obtains
that immutable view through Qt's `size()` and `constData()` APIs; it does not use
allocation-header flags or capacity as a packet length. Version-dependent one-
or two-byte item quantities report their width explicitly. A missing byte view,
opcode mismatch, malformed packet, or unknown layout deliberately falls back to
the fields that were safe to decode, usually just `opcode`, instead of
manufacturing zero-valued action data.

`IncomingOpcodeOnlyPacket = { opcode: integer }` names the fallback incoming
record used by Appendix A. `OutgoingOpcodeOnlyPacket` has the same shape.
`IncomingPacketPayload`, `OutgoingPacketPayload`, and `PacketEventPayload` name
the corresponding decoded-record unions above.

Do not recursively search a packet for strings or guess camelCase aliases. For
the AFK text-message example, read `packet.message` directly and use
`message_contains` so irrelevant text packets never enter Lua.

All registrations and scheduled callbacks are owned by the current script and
are removed automatically when it stops. Scheduled callbacks run as managed
coroutines and may yield. A scheduled callback failure ends only that callback.
Registered event callbacks must instead obey the non-yielding critical-path
rule. An event callback is disabled after exactly three consecutive uncaught
failures; one successful terminal invocation resets its failure count. At most
64 event callbacks may be active plus pending for a script. At the limit the
newest callback is dropped, so callbacks must coalesce or deliberately discard
replaceable telemetry instead of building a backlog.

### event_proxies.lua

`PacketEventProxy` is the common owner-safe wrapper around native packet
registration. `IncomingPacketProxy:New(name, opcodeOrArray)` and
`OutgoingPacketProxy:New(name, opcodeOrArray)` accept any byte opcode or a
non-empty opcode array. Use `OnReceive(function(proxy, packet) ...)` for an
incoming proxy, `OnSend(...)` for an outgoing proxy, or `OnPacket(...)` for
direction-independent code. All share `GetName()`, `GetRegistrationId()`,
`IsIncoming()`, and idempotent `Unregister()`.

The base constructor also supports this exact option shape:

```text
PacketEventProxyOptions = {
    name?: string,
    id?: string,
    packet_id: integer|integer[],
    incoming?: boolean,
    message_contains?: string,
    message_case_sensitive?: boolean
}
```

At least one of `name` or `id` is required. Native text filtering retains the
same restrictions as `Events.RegisterPacketEvent`.

Named incoming proxies, all using `OnReceive(proxy, packet)`, are:

- `TextMessagePacketProxy`
- `TalkPacketProxy`
- `CreateOnMapPacketProxy`
- `DeleteOnMapPacketProxy`
- `MoveCreaturePacketProxy`
- `FullMapPacketProxy`
- `MapRowPacketProxy` (all four row opcodes)
- `GraphicalEffectPacketProxy`
- `UnjustifiedPointsPacketProxy`
- `PlayerInventoryPacketProxy`

Named outgoing proxies, all using `OnSend(proxy, packet)`, are:

- `ClientMovementPacketProxy` (auto-walk, stop, and eight walk directions)
- `ClientEquipItemPacketProxy`
- `ClientMoveItemPacketProxy`
- `ClientUseItemPacketProxy`
- `ClientUseItemWithPacketProxy`
- `ClientUseOnCreaturePacketProxy`
- `ClientLookPacketProxy`
- `ClientTalkPacketProxy`
- `ClientAttackPacketProxy`

The historical text/container/stats/skills/creature/death proxies remain for
compatibility. `CreatureAddProxy` now correctly listens to create-on-map and
fires only for a parsed creature, using native snake-case fields. The protocol
delete-on-map record has no removed creature id, so `CreatureRemoveProxy`
preserves its first `creatureId` argument as `nil` and follows it with
`position`, `stackPosition`, and the complete packet. The container, stats, and
skills payloads remain signal/opcode-only until corresponding native parsers
are added; scripts must not invent their historical camelCase fields.

### engine.lua
The main high-level interface for querying and controlling configured bot features. Native state remains owned by the bot; Engine methods validate arguments and return Lua snapshots or operation results.

Primary namespaces:
- Engine.Healer
- Engine.Alarms
- Engine.AmmoRefill
- Engine.Features
- Engine.Equipment
- Engine.PVPTools
- Engine.MagicShooter
- Engine.Targeting
- Engine.Walker
- Engine.Lure
- Engine.Extras
- Engine.Channels
- Engine.Looter
- Engine.TankMode
- Engine.TimerActions
- Engine.SuppliesSorter
- Engine.HUD
- Engine.Scripter
- Engine.Delays
- Engine.Settings (canonical profile-file API; global `Settings` is equivalent)

Use `Engine.Features` to query, enable, disable, or toggle public bot features, including Supplies Sorter. IDs for internal object dumping, queue/event infrastructure, and the scripter are rejected by the native boundary. `Engine.Equipment` provides live equipped-slot data and equipment actions; it is distinct from Equipment Manager's saved configuration. Magic Shooter and Targeting expose profile and explicit entry control.

The older global `Features`, `Inventory`, `PVPTools`, `MagicShooter`, and `Targeting` tables remain available for compatibility. New scripts should prefer the corresponding `Engine.*` namespaces.

Getters return values or detached tables; editing a returned entry/list never
edits native feature state. Use the matching explicit setter so validation,
parsed caches, timers, queued work, packet subscriptions, and HUD refresh side
effects remain correct. `Engine.Walker.Defer(timeoutMs)` and
`CompleteDeferred(token)` implement Walker's blocking-decision
handshake. Complete or allow every accepted token before its timeout.
`Engine.Lure.UpdateSetting(index, setting)` updates an existing copied lure
setting using native validation.

### cavebot.lua
High-level cavebot orchestration wrappers over Walker and Lure Manager feature toggles.

Primary API:
- Cavebot.Save(path, features?)
- Cavebot.Load(path, features?)
- Cavebot.SetEnabled(enabled)
- Cavebot.Enable()
- Cavebot.Disable()
- Cavebot.IsEnabled()
- Cavebot.SetLureEnabled(enabled)
- Cavebot.EnableLure()
- Cavebot.DisableLure()
- Cavebot.IsLureEnabled()
- Cavebot.SetEnginesEnabled(walkerEnabled, lureEnabled)
- Cavebot.Resume()
- Cavebot.Defer(timeoutMs)
- Cavebot.GoTo(labelName)
- Cavebot.GoToLabel(labelName)
- Cavebot.Pause(milliseconds, autoResume)
- Cavebot.RegisterEvent(eventId, callback)
- Cavebot.ObserveLabel(callback)
- Cavebot.ObserveAction(callback)
- Cavebot.ObserveWaypointChange(callback)
- Cavebot.OnActionStarted(callback)
- Cavebot.OnActionCompleted(callback)
- Cavebot.OnWaypointChange(callback)
- Cavebot.InterceptLabel(callback)
- Cavebot.InterceptAction(callback)
- Cavebot.OnLabel(callback) (legacy blocking alias)
- Cavebot.OnAction(callback) (legacy blocking alias)
- Cavebot.UnregisterAllEvents()
- Cavebot.GetStatus()
- Cavebot.PrintStatus()

`ObserveLabel`, `ObserveAction`, and `ObserveWaypointChange` are non-blocking
telemetry callbacks and cannot pause Walker by registering. `ObserveAction`
reports that an Action waypoint was reached, before any blocking interceptor has
finished; it is not a success signal. `OnWaypointChange` is retained as a
compatibility alias for `ObserveWaypointChange`.
`ObserveWaypointChange` receives one table containing `previousIndex`, `index`,
`type`, `x`, `y`, `z`, `label`, `labelName`, and `uniqueId`; indices are
one-based and `previousIndex` is `nil` for the initial selection.

`OnActionStarted` and `OnActionCompleted` are the truthful, non-blocking Action
lifecycle APIs. Each receives one table. Both tables contain `executionId`,
`action`/`name`, the numeric Action `kind`, and a `waypoint` table with `index`,
`uniqueId`, `x`, `y`, and `z`. The completion table additionally contains:

- `ok`: `true` only when the Action reached a successful terminal result.
- `outcome`: `success`, `skipped`, `failure`, `timeout`, or `cancelled`.
- `description`: the result or error description.
- `result`: the description on success; otherwise `nil`.
- `error`: the description on failure; otherwise `nil`.
- `durationMs`: elapsed milliseconds from actual Action start to completion.

Use `executionId` to correlate a start with exactly one terminal completion.
Action start is dispatched only after legacy Action interceptors have released.
Changing the selected waypoint, replacing or deleting the Action, clearing or
reloading the route, or resetting Walker completes an active Action as
`cancelled`; it is never reported as successful merely because it started.

`OnLabel` and `OnAction` retain their historical blocking behavior.
`InterceptLabel` and `InterceptAction` are the preferred explicit names when a
script deliberately needs that behavior.

Blocking interceptors release automatically when their callback returns.
`Cavebot.Defer(timeoutMs)` may be called only inside an interceptor when work
must continue after the callback returns. It returns an owner-scoped handle with
`handle:Complete()` and `handle:Cancel()`; both release the hold, return `true`
only on the first successful release, and cannot release another script's hold.
`timeoutMs` must be between 1 and 60000, and expiration fails open.

Walker namespace (full runtime wrappers):
- Cavebot.Walker.SetEnabled(enabled)
- Cavebot.Walker.IsEnabled()
- Cavebot.Walker.Resume()
- Cavebot.Walker.Defer(timeoutMs)
- Cavebot.Walker.CompleteDeferred(token)
- Cavebot.Walker.GoTo(labelName)
- Cavebot.Walker.GetSelectedWaypointIndex()
- Cavebot.Walker.SetSelectedWaypointIndex(index)
- Cavebot.Walker.SetWaypointPosition(index, x, y, z)
- Cavebot.Walker.SelectClosestWaypoint()
- Cavebot.Walker.GetWaypointCount()
- Cavebot.Walker.GetWaypoints()
- Cavebot.Walker.GetSpecialAreas()
- Cavebot.Walker.GetSpecialAreaCount()
- Cavebot.Walker.AddSpecialArea(area)
- Cavebot.Walker.UpdateSpecialArea(id, updateData)
- Cavebot.Walker.DeleteSpecialArea(id)
- Cavebot.Walker.ClearSpecialAreas()
- Cavebot.Walker.ReorderSpecialArea(sourceIndex, targetIndex, dropAfterTarget?)
- Cavebot.Walker.IsPositionInsideSpecialArea(x, y, z, featureMask)
- Cavebot.Walker.AddWaypoint(waypoint)
- Cavebot.Walker.InsertWaypoint(index, waypoint)
- Cavebot.Walker.ReplaceWaypoint(index, waypoint)
- Cavebot.Walker.DeleteWaypoint(index)
- Cavebot.Walker.ClearWaypoints()
- Cavebot.Walker.MoveWaypointUp(index?)
- Cavebot.Walker.MoveWaypointDown(index?)
- Cavebot.Walker.IsStuck()
- Cavebot.Walker.SetStartFromNearestWaypoint(enabled)
- Cavebot.Walker.GetStartFromNearestWaypoint()
- Cavebot.Walker.SetNodeDistance(distance)
- Cavebot.Walker.GetNodeDistance()
- Cavebot.Walker.SetWalkToLureCenter(enabled)
- Cavebot.Walker.GetWalkToLureCenter()
- Cavebot.Walker.SetLeaveLureOnPlayer(enabled)
- Cavebot.Walker.GetLeaveLureOnPlayer()
- Cavebot.Walker.SetLeaveLurePlayerMode(mode)
- Cavebot.Walker.GetLeaveLurePlayerMode()
- Cavebot.Walker.SetDebugHud(enabled)
- Cavebot.Walker.GetDebugHud()
- Cavebot.Walker.SetNavigationMode(mode)
- Cavebot.Walker.GetNavigationMode()
- Cavebot.Walker.SetAutoExploreSettings(settings)
- Cavebot.Walker.GetAutoExploreSettings()
- Cavebot.Walker.ResetAutoExploreCoverage()
- Cavebot.Walker.GetAutoExploreStatus()
- Cavebot.Walker.IsAutoExplorePositionPainted(x, y, z)
- Cavebot.Walker.GetAutoExploreConnectors()
- Cavebot.Walker.AddAutoExploreConnector(connector)
- Cavebot.Walker.UpdateAutoExploreConnector(id, updateData)
- Cavebot.Walker.DeleteAutoExploreConnector(id)
- Cavebot.Walker.ClearAutoExploreConnectors()
- Cavebot.Walker.SetAutoExploreConnectorRecording(enabled)
- Cavebot.Walker.GetAutoExploreConnectorRecording()
- Cavebot.Walker.SetAutoRecorderEnabled(enabled)
- Cavebot.Walker.GetAutoRecorderEnabled()
- Cavebot.Walker.SetAutoRecorderOptions(options)
- Cavebot.Walker.GetAutoRecorderOptions()
- Cavebot.Walker.SetDistanceBetweenWaypoints(distance)
- Cavebot.Walker.GetDistanceBetweenWaypoints()
- Cavebot.Walker.SetPausedByLua(paused)
- Cavebot.Walker.IsPausedByLua()

Leave-lure player detection modes:
- `Cavebot.Walker.LeaveLurePlayerMode.NonAllyPlayers` (`0`, default) ignores party and guild/allied-guild players.
- `Cavebot.Walker.LeaveLurePlayerMode.AnyPlayer` (`1`) reacts to every visible player other than the local character.
- `Cavebot.Walker.LeaveLurePlayerMode.PlayersInList` (`2`) reacts to listed players.
- `Cavebot.Walker.LeaveLurePlayerMode.PlayersNotInList` (`3`) reacts to players outside the list.

Special Area records are detached snapshots with `index`, stable `id` and
`uniqueId`, `x`, `y`, `z`, `width`, `height`, `featureMask`, and `enabled`.
Adding requires `x`, `y`, and `z`; width and height default to 1 and are limited
to 1 through 10. Coordinates `x` and `y` must be 1 through 65535 and `z` must
be 0 through 15. `featureMask` defaults to Walker (`1`) and `enabled` defaults
to `true`. The feature mask values are available through
`Cavebot.Walker.SpecialAreaFeature` and `Engine.Walker.SpecialAreaFeature`:
Walker `1`, Targeting `2`, MagicShooter `4`, Looter `8`, and All `15`. Masks may
be combined. Update and delete use the stable ID, not the mutable row index;
updates are partial and preserve omitted fields. Every successful geometry or
scope mutation invalidates Walker's cached movement and refreshes its HUD.

Auto Explore navigation mode is `"waypoints"` or `"auto_explore"`, and can be
changed only while Walker is stopped. Styles are `"natural"`, `"thorough"`,
and `"wide_roam"`. Settings updates are partial and use
`maximumFloorsUp`, `maximumFloorsDown`, `allowWalkOn`, `allowLadder`,
`allowRope`, `allowHole`, `allowTeleport`, and `autoOpenDoors`. Painted areas
are intentionally edited only through World Map; scripts can query containment
with `IsAutoExplorePositionPainted`. Connector records use stable IDs, one of
`walk_on`, `ladder`, `rope`, `hole`, or `teleport`, and explicit
`source={x,y,z}` / `destination={x,y,z}` positions. Runtime status is a detached
snapshot containing phase, base/current/target positions, plannedPath,
recentTrail, bounded `recentVisitHeat` samples, coverage counts,
activeConnectorId, outsideMask, status, and `latestFailureReason`. Pairing uses
two reciprocal directed records; updating one paired record mirrors only its
endpoints onto the reverse record, while deleting one leaves the reverse record
valid and unpaired.

Waypoint script note:
- Script waypoints should call `Cavebot.Walker.Resume()` when they are done.
- If a waypoint script exits without resuming, the runtime resumes walking and reports a warning.
- Disabling the walker also stops an active waypoint script so it can later be enabled cleanly.

Position waypoint notes:
- Weak Horizontal Stand tries the exact target tile first, then accepts or moves to X - 1 or X + 1 on the same Y/Z.
- Weak Vertical Stand tries the exact target tile first, then accepts or moves to Y - 1 or Y + 1 on the same X/Z.

Actions namespace:
- Cavebot.Actions.GetLastResult()
- Cavebot.Actions.Register(actionType, handler)
- Cavebot.Actions.Run(context)

Built-in action types:
- check_supplies
- buy_supplies
- sell_loot
- open_depot
- deposit_items
- stash_items
- withdraw_supplies
- npc_say
- bank
- custom_script
- tasker
- imbuing

Action context conventions:
- actionType or action_type selects the handler.
- actionConfig or action_config contains action-specific config.
- successLabel and failureLabel can route the walker after action completion.
- Handlers return a table, commonly including ok, actionType, error, pending, goToLabel, and action-specific result fields.
- Cavebot.Actions.Run resumes the walker when the action completes unless a handler returns a route that changes the current waypoint.

Lure namespace (full runtime wrappers):
- Cavebot.Lure.SetEnabled(enabled)
- Cavebot.Lure.IsEnabled()
- Cavebot.Lure.GetState()
- Cavebot.Lure.IsLuring()
- Cavebot.Lure.IsFighting()
- Cavebot.Lure.SetForceLure(enabled)
- Cavebot.Lure.IsForceLure()
- Cavebot.Lure.EndForceLure()
- Cavebot.Lure.SetOption(option) -- 0 = Start/End, 1 = Dynamic, 2 = Kiting
- Cavebot.Lure.GetOption()
- Cavebot.Lure.SetNearRange(range)
- Cavebot.Lure.GetNearRange()
- Cavebot.Lure.SetAttackWhileLuring(enabled)
- Cavebot.Lure.GetAttackWhileLuring()
- Cavebot.Lure.SetConsiderOnlyReachable(enabled)
- Cavebot.Lure.GetConsiderOnlyReachable()
- Cavebot.Lure.SetSlowWalkDelayMs(delayMs)
- Cavebot.Lure.GetSlowWalkDelayMs()
- Cavebot.Lure.SetSlowWalkingCreaturesCount(count)
- Cavebot.Lure.GetSlowWalkingCreaturesCount()
- Cavebot.Lure.SetSlowWalkBurstSteps(steps)
- Cavebot.Lure.GetSlowWalkBurstSteps()
- Cavebot.Lure.SetKitingPreferredFarthestDistance(distance)
- Cavebot.Lure.GetKitingPreferredFarthestDistance()
- Cavebot.Lure.SetKitingMaximumFarthestDistance(distance)
- Cavebot.Lure.GetKitingMaximumFarthestDistance()
- Cavebot.Lure.SetKitingCloseMonsterDistance(distance)
- Cavebot.Lure.GetKitingCloseMonsterDistance()
- Cavebot.Lure.SetKitingCloseMonsterCount(count)
- Cavebot.Lure.GetKitingCloseMonsterCount()
- Cavebot.Lure.SetIgnoringMonsters(enabled)
- Cavebot.Lure.GetIgnoringMonsters()
- Cavebot.Lure.SetStartEndLureActive(enabled)
- Cavebot.Lure.GetStartEndLureActive()
- Cavebot.Lure.SetWaypointDynamicLureActive(enabled)
- Cavebot.Lure.GetWaypointDynamicLureActive()
- Cavebot.Lure.SetUnblocking(enabled)
- Cavebot.Lure.GetUnblocking()
- Cavebot.Lure.GetLuredCreaturesCount()
- Cavebot.Lure.HasActiveSettings()
- Cavebot.Lure.IsOtherPlayerOnScreen()
- Cavebot.Lure.GetSettings()
- Cavebot.Lure.GetSettingCount()
- Cavebot.Lure.AddSetting(setting)
- Cavebot.Lure.UpdateSetting(index, updateData)
- Cavebot.Lure.RemoveSetting(index)
- Cavebot.Lure.ClearSettings()

Kiting preferred, maximum, and close-monster distances accept integers from 1
through 7. Preferred may not exceed the current maximum, and maximum may not be
below the current preferred value, so adjust them in a valid order when changing
both. Close-monster count accepts 0 through 200. These getters/setters edit the
same values persisted by Lure Settings.

### sound.lua
Sound playback and queue control.

Primary API:
- Sound.Play(options)
- Sound.Stop()
- Sound.ClearQueue()
- Sound.GetQueueSize()
- Sound.IsPlaying()
- Sound.IsQueued(options)
- Sound.SetMinDelay(delayMs)
- Sound.GetCurrentDuration()
- Sound.GetFileDuration(filePath)
- Sound.PlayById(soundId, instant?)
- Sound.PlayByName(soundName, instant?)
- Sound.PlayFile(filePath, instant?)
- Sound.StopAll()
- Sound.GetQueueLength()
- Sound.PlayAndWait(options, maxWaitMs?)
- Sound.WaitForCompletion(maxWaitMs?)
- Sound.PlayByIdSmart(soundId, instant?)
- Sound.PlayByNameSmart(soundName, instant?)
- Sound.PlayFileSmart(filePath, instant?)
- Sound.PlayBotSound(nameOrPath, instant?)

`Sound.Play` and `Sound.IsQueued` options must identify exactly one source:
`sound_id = BotSoundId.*`, `sound_name = string`, or `file_path = string`.
`Sound.Play` also accepts `instant = boolean`. Built-in IDs are exactly 0 through
13 (`DISCONNECTED` through `UNJUSTIFIED_KILL`); there are no custom IDs 14
through 18. A bare `sound_name` such as `raid_warning` or `raid_warning.wav`
first searches the product `Data/Alarms` folder and then the calling script's
folder. Only bare filenames are accepted for lookup; subdirectories and
traversal are rejected. A fully qualified path supplied as `sound_name` is used
directly, while `file_path` always requires a full absolute file path and never
falls back to either folder. `PlayBotSound` accepts the same name-or-full-path
forms without rewriting the value. Missing files and invalid WAV data raise a
Lua error synchronously. `Sound.IsQueued` uses the identical resolver, so its
answer matches the request identity queued by `Sound.Play`. Scripts created
from embedded strings have no script-folder fallback because they have no
owning file path. Historical aliases such as `health` and `pm` are resolved
only after exact filename lookup in both folders; the alias succeeds only when
its canonical packaged alarm WAV exists and passes validation.

Playback, queue state, and stop/cleanup are owner-scoped, so one script cannot
claim another script's playback as its own. `SetMinDelay` changes the shared
sound manager delay and accepts 0 through 60,000 ms. The waiting helpers must run
inside a managed coroutine because they yield, and they use `Time.MonotonicMs`
instead of wall/CPU time. Smart helpers return `false` when the same source is
already queued or playing.

### Time

- `Time.MonotonicMs() -> integer`

Returns milliseconds from an unspecified monotonic epoch. Use differences
between readings for elapsed-time decisions. The runtime uses the same
`std::chrono::steady_clock` basis for scheduler deadlines.

### Script lifecycle and cross-script control

The current script's top-level body is executed first and its optional `init()`
is then invoked as a managed coroutine. Define `init()` for setup that may yield.
Define `terminate()` for short, synchronous cleanup only: it is protected,
non-yielding, called at most once, and has a 50 ms limit. Native owner-scoped
modules, schedules, events, HUD elements, sounds, network handles, and async
tokens are cleaned even if Lua cleanup fails.

- `Engine.Scripter.GetAvailableScripts()` lists exact runnable `.lua` filenames.
- `Engine.Scripter.Start(name)`, `Stop(name)`, and `Restart(name)` operate on those exact names.
- `Engine.Scripter.Refresh()` refreshes the script list.
- `Engine.Scripter.GetRunningScripts()` reports currently active script names.
- `Engine.Scripter.GetOutput(name)` returns current output or the most recently archived output.
- `Engine.Scripter.StopSelf()` is the only supported way for a running script to stop itself.
- `Script.Unload()` remains a compatibility self-stop call; prefer `StopSelf`.

A script cannot use `Stop(name)` or `Restart(name)` on itself and cannot disable
the sandbox or execute an arbitrary source string. Every script writes its own
runtime log under the product `UserData/BotLogs` area, including explicit PASS
and FAIL messages printed by test scripts. `GetOutput` is read-only and returns
an empty string when no current or archived output exists.

### storage.lua
Persistent JSON-compatible values with per-script scopes and explicit named
scopes shared between scripts. No file paths or manual serialization are
needed. Existing per-script behavior is unchanged.

Direct scopes:
- Storage.Global.Get(key, default?) / Set(key, value) / Remove(key) / Clear()
- Storage.Character.Get(key, default?) / Set(key, value) / Remove(key) / Clear()

Logical namespaces:
- Storage.Namespace(namespace, perCharacter?)
- Storage.ForCharacter(namespace)
- scope:Get(key, default?) / Set(key, value) / Remove(key)

Named cross-script scopes:
- Storage.Shared(namespace)
- Storage.SharedForCharacter(namespace)
- shared:Get(key, default?) -> value, errorMessage?
- shared:Set(key, value) -> success, errorMessage?
- shared:Remove(key) -> success, errorMessage?
- shared:Clear() -> success, errorMessage?
- shared:Update(key, updater, default?) -> success, newValue, errorMessage?
- shared:OnChanged(callback, key?, includeSelf?) -> subscriptionId?, errorMessage?
- shared:OffChanged(subscriptionId) -> success, errorMessage?

`Storage.Global`, `Storage.Character`, `Storage.Namespace`, and
`Storage.ForCharacter` remain private to the current script file. "Global" in
that API means every character running that one script; it does not mean every
Lua script.

`Storage.Shared("name")` opens a durable namespace available to every script,
including temporary Walker Lua waypoints. `Storage.SharedForCharacter("name")`
opens the same named file but selects data isolated by the currently logged-in
character. Any installed script that knows a shared namespace can access it, so
a namespace is a coordination boundary rather than a security boundary.

Use `Update` for read-modify-write operations which must not lose concurrent
changes:

```lua
local state = Storage.SharedForCharacter("cavebot.supplies")
local success, visits, updateError = state:Update("visits", function(current)
    return (current or 0) + 1
end)
```

The updater runs without a native or filesystem lock and may run again when a
different script or bot process wins a concurrent write. It should therefore
be deterministic and free of external side effects. Returning nil removes the
key. If `default` is a table, do not mutate that table in place; construct and
return a new table instead. Updates retry at most eight times and then return
an error instead of blocking indefinitely.

Shared operations coordinate across scripts and bot processes, use bounded
lock waits, revisions, and atomic file replacement. Calls may still perform
local disk I/O, so storage should persist meaningful state transitions rather
than act as a per-frame message bus. Keep frequently changing values in Lua
memory and persist them only when needed.

Use `OnChanged` to receive committed mutations from other scripts in the same
injected bot process. Pass a key to observe one field or nil to observe the
whole selected scope. Notifications exclude writes made by the subscribing
script unless `includeSelf` is true. The returned subscription is owned by the
script, keeps it alive as an event resource, and is removed automatically when
the script stops; `OffChanged` removes it early.

```lua
local state = Storage.Shared("cavebot.supplies")
local subscriptionId, subscribeError = state:OnChanged(function(event)
    print(event.operation, event.key, event.writer.name, event.revision)
    if event.newValueIncluded then
        print("new value:", event.newValue)
    end
end, "visits")
```

The callback receives a table with `namespace`, `operation` (`set`, `remove`,
or `clear`), `scope`, `revision`, `timestampUnixMs`, and a `writer` table with
stable process-local `id`, human-readable `name`, and `type` (`script`,
`walker`, or `one_shot`). Key-specific events also include `key`,
`previousExists`, `newExists`, `previousValueIncluded`, `newValueIncluded`, and
the corresponding `previousValue`/`newValue` fields when included. A whole-scope
clear instead reports `changedCount`, up to 256 `changedKeys`, and
`changedKeysTruncated`. Individual values larger than 256 KiB are omitted from
the notification while their existence flags remain accurate; the subscriber
can call `Get` when it needs the large current value.

Change callbacks are queued only after the storage lock is released and run as
managed event coroutines. Deadlock avoidance does not make storage access cheap:
the callback must not yield or call storage again. Copy only the event's minimal
scalar fields into a coalesced slot/bounded queue and let one stable worker
batch any follow-up read or write. The standard event failure policy disables a
subscription after three consecutive uncaught callback failures. Each script
may own at most 64 shared
storage subscriptions; the native notification queue is bounded to 32 events
and 8 MiB, the process accepts at most 512 subscriptions, and callback fan-out
is time-sliced to 128 admissions per manager pass. Remaining deliveries stay
queued, so a writer cannot grow DLL memory or monopolize the game thread
without limit.

Notifications are process-local: storage writes remain safe across bot
processes, but a change made by another injected client is observed on the next
explicit `Get`, not through `OnChanged`. Reads do not emit access events. That
avoids recursion (`Get` causing a callback which calls `Get` again), unnecessary
disk traffic, and leaking read activity between cooperating scripts. Use an
explicit audit key if scripts need to record meaningful reads.

Subscriptions and event metadata exist only at runtime. They do not change the
shared-storage JSON format, so existing storage files and Lua scripts remain
compatible.

Namespaces must be non-empty, at most 64 bytes, and contain only letters,
numbers, `_`, `-`, and `.`. Per-script namespace plus key combinations and
shared keys may not exceed 256 bytes; shared keys cannot contain NUL bytes.
Supported values are nil, booleans,
finite numbers, strings, and nested tables with string keys or contiguous
1-based array indexes. Cyclic tables are rejected. Each storage file is limited
to 2 MB, 12 nested levels, and 4,096 entries per table.

### vip.lua
Read-only VIP/contact queries through the canonical `VIP` table.

Primary API:
- VIP.IsAvailable()
- VIP.GetAll() / Get(name) / Exists(name)
- VIP.Count() / CountOnline()
- VIP.IsOnline(name) / GetType(name) / GetDescription(name) / GetNotifyOnLogin(name)
- VIP.GetNames(onlyOnline?) / GetByType(vipType) / GetHearts() / IsHeart(name)
- VIP.FindByPrefix(prefix, onlyOnline?)
- VIP.ToLookupTable() / GetSnapshot()

Normalized VIP entries contain `name`, `description`, `type`, `online`, and `notifyOnLogin`. `GetSnapshot` contains `available`, `count`, `onlineCount`, `heartCount`, `names`, `onlineNames`, and `vips`. Use `VipFlag.*` when filtering by type.

### hud_wrapper.lua
UI drawing wrapper classes for screen/world overlays.

Core classes:
- ScreenText
- ScreenImage
- WorldText
- WorldBox
- WorldImage

All classes support `New`, `Create`, `Remove`, `SetEnabled`, `SetZIndex`, `SetRenderLayer`, `SetParent`, `ClearParent`, `IsCreated`, `GetEnabled`, `GetVisible`, `GetPosition`, `GetWidth`, and `GetHeight` as applicable. Setters return the same object for chaining.

Class-specific methods:
- ScreenText: SetText, SetColor, SetFont, SetFontFamily, SetFontSize, SetAlignment, SetDraggable, SetDragTarget, SetOnDragEnd, SetPointerCallbacks, SetOnMouseDown, SetOnMouseUp, SetOnMouseLeave, SetClickable, SetScreenPosition, GetText, GetColor
- ScreenImage: SetSource, SetSourceBase64, SetSourceBytes, SetItemId, SetItemName, SetSize, SetLabel, SetAlignment, SetDraggable, SetDragTarget, SetOnDragEnd, SetPointerCallbacks, SetOnMouseDown, SetOnMouseUp, SetOnMouseLeave, SetClickable, SetScreenPosition
- WorldText: SetText, SetColor, SetFont, SetFontFamily, SetFontSize, SetPosition, SetLifetime, SetOffset, GetText, GetColor
- WorldBox: SetSize, SetWidth, SetHeight, SetColor, SetBorderWidth, SetBorderColor, SetPosition, SetLifetime, GetColor
- WorldImage: SetSource, SetSourceBase64, SetSourceBytes, SetItemId, SetItemName, SetSize, SetLabel, SetPosition, SetOffset, SetLifetime

Use HUD objects for visual diagnostics, status displays, labels, and map markers. IDs should be stable and unique within the script.

Render-layer notes:
- Every HUD element uses `HUDRenderLayer.MAP` by default, preserving the established game-view parent and clipping behavior.
- Pass `HUDRenderLayer.OVERLAY` as the final `New(...)` argument or call `SetRenderLayer(HUDRenderLayer.OVERLAY)` before `Create()` to draw over Tibia panels outside the map rectangle.
- A literal null Qt parent is not exposed because an unparented `QQuickItem` normally leaves the visual scene. The overlay layer safely uses the highest available item in Tibia's existing Qt scene.
- `SetRenderLayer` is creation-only. Choose the layer before `Create()`; changing it afterward raises a Lua error.
- `SetZIndex` still orders elements within their selected layer. Overlay elements receive an internal scene-layer bias so they remain above normal client QQuick items.
- `SetParent` and `ClearParent` are separate logical HUD relationships; they control visibility/removal/movement cascades and do not select the Qt render layer. Logical parents and children must use the same render layer.

```lua
local overlayTitle = ScreenText:New("overlay_title", HUDRenderLayer.OVERLAY)
    :SetText("Drawn above Tibia panels")
    :SetFont("Segoe UI", 20)
    :SetScreenPosition(700, 120)
    :SetZIndex(10)
    :Create()

-- Equivalent builder form:
local overlayIcon = ScreenImage:New("overlay_icon")
    :SetRenderLayer(HUDRenderLayer.OVERLAY)
    :SetSourceBase64(MY_PNG_OR_GIF_BASE64)
    :SetSize(32, 32)
    :SetScreenPosition(760, 120)
    :Create()

local warlock = ScreenImage:New("warlock")
    :SetSource(LoadOutfitIcon("warlock", {
        addons = "all",
        direction = "west",
        colors = { head = 78, body = 68, legs = 58, feet = 76 }
    }))
    :SetSize(64, 64)
    :Create()

local rawOutfit = ScreenImage:New("outfit_190")
    :SetSource(LoadOutfitById(190, { addons = { 1, 2 } }))
    :Create()
```

Screen image notes:
- ScreenImage:SetLabel(text, color, offsetX, offsetY) attaches or updates text on the image.
- ScreenImage:SetScreenPosition(x, y) positions image HUD elements in screen pixels.
- ScreenText:SetParent(parent_id) and ScreenImage:SetParent(parent_id) can parent complete HUD rows and icons to a draggable handle. All descendants move during each native drag update and keep their current offsets, so composed interfaces move as one unit without waiting for drag-end.
- `ScreenImage:SetSource(path)` and `WorldImage:SetSource(path)` load PNG, JPG, or animated GIF files from a full path. Asynchronous PNG/GIF loading requires a local path; remote/UNC PNG/GIF paths are rejected so script shutdown cannot be held by network filesystem I/O.
- `ScreenImage:SetSourceBase64(base64Image)` and `WorldImage:SetSourceBase64(base64Image)` embed PNG or animated GIF data directly in the script. Raw Base64, `data:image/png;base64,...`, and `data:image/gif;base64,...` values are accepted.
- `ScreenImage:SetSourceBytes(imageBytes)` and `WorldImage:SetSourceBytes(imageBytes)` accept either a contiguous Lua array of integer bytes or a binary Lua string, including hexadecimal PNG or GIF bytes.
- `LoadOutfitIcon(monsterName, options?)` preserves the complete curated appearance used by Targeting and Magic Shooter, including colors, addons, mounts, and lookTypeEx object fallbacks. `LoadOutfitById(outfitId, options?)` starts from a raw client outfit. Pass either result to `SetSource`, or use `SetOutfitIcon`/`SetOutfitById` directly. Options accept `head`, `body`, `legs`, `feet`, a nested `colors` table, `addons = 0..255`, `addons = {1, 2}`, `addons = "all"`, `direction = "north"|"east"|"south"|"west"` (or `0..3`), and `mount_id`.
- Screen outfit images are draggable with `:SetDraggable(true)`; low-level `HUD.AddScreenImage` uses `is_draggable = true`. World outfit images remain anchored to their world position.
- Outfit walking frames animate for both screen and world images. Named outfit sources are demand-loaded and become visible after the shared Targeting/Magic Shooter appearance registry is ready.
- Animated GIF frames and their frame delays are preserved for both screen and world images. GIF delays are bounded to 33-1000 ms per frame.
- PNG/GIF file reads, Base64 parsing, and image decompression run asynchronously. `Create()` returns after queuing the HUD element; the image becomes visible when decoding finishes. Removing the element or stopping its script safely cancels or discards unfinished work.
- Embedded image data is limited to 4 MiB. A Lua byte-array table is limited to 128 KiB so copying it cannot monopolize the client thread; use Base64 or a binary string for larger embedded images. PNG dimensions are limited to 2048x2048. GIFs are limited to 512x512, 64 frames, and a bounded total decoded size.
- Choose exactly one source setter per image; calling another source setter replaces the prior selection.

Text font notes:
- `ScreenText` and `WorldText` use Tibia's HUD font and size by default.
- `SetFont(family, pixelSize)` selects an installed system-font family and a pixel size from 1 to 256. It works both before and after `Create()`.
- `SetFontFamily(family)` and `SetFontSize(pixelSize)` update one property while preserving the other.
- Pass `nil` for a property to inherit that property from Tibia again; `SetFont(nil, nil)` fully resets the element.
- A missing system font may be replaced by a platform fallback, so scripts should prefer common Windows font-family names.

```lua
local title = ScreenText:New("status_title")
    :SetText("Validus status")
    :SetFont("Segoe UI", 20)
    :SetScreenPosition(120, 80)
    :Create()

title:SetFontSize(26)       -- runtime resize
title:SetFontFamily("Arial") -- runtime family change
title:SetFont(nil, nil)     -- restore Tibia font and size
```

### lua_consts.lua
Shared gameplay constants/enums for movement, pathfinding, effects, equipment, chat, creature state, combat modes, skills, VIP flags, vocation, and walker events.

Use constants from this file instead of magic numbers.

## Practical Script Examples

These examples are intentionally small, defensive, and written in the public PascalCase API style. They are good patterns for LLM-generated scripts to copy.

### 1. Repeating Low HP Sound Alert

Use `Module.Every` for repeating work and check that the player is available.
Leave unexpected failures uncaught so the runtime can log and isolate the
module. Add a narrow `pcall` only around an operation that has a defined
recoverable fallback.

```lua
local SCRIPT_ID = "low_hp_sound_example"

local function tick()
    if not Self.IsAvailable() then
        return
    end

    local hp = Self.GetHealthPercentage()
    if type(hp) == "number" and hp <= 35 then
        Sound.PlayByIdSmart(BotSoundId.LOW_HEALTH)
    end
end

function init()
    Module.Every(SCRIPT_ID .. "_tick", tick, 1000)
    print("[" .. SCRIPT_ID .. "] PASS: initialized")
end
```

### 2. Scan Visible Monsters

Use `Creatures.GetVisibleMonsters()` and creature wrapper methods instead of direct engine globals.

```lua
local SCRIPT_ID = "monster_scan_example"

Module.Every(SCRIPT_ID .. "_scan", function()
    local player = Creature.GetLocalPlayer()
    if not player then
        return
    end

    local monsters = Creatures.GetVisibleMonsters(true)
    local closestName = nil
    local closestDistance = nil

    for _, monster in ipairs(monsters) do
        if monster:IsValid() then
            local distance = monster:DistanceToCreature(player)
            if closestDistance == nil or distance < closestDistance then
                closestDistance = distance
                closestName = monster:GetName()
            end
        end
    end

    if closestName then
        print("Closest monster: " .. closestName .. " at distance " .. tostring(closestDistance))
    end
end, 1000)
```

### 3. Screen Text Status HUD

Create a persistent HUD object once, then update its text from a scheduled module.

```lua
local SCRIPT_ID = "screen_status_example"

local statusText = ScreenText:New(SCRIPT_ID .. "_hud")
statusText
    :SetColor({ r = 255, g = 255, b = 255, a = 255 })
    :SetText("Starting...")
    :Create()
    :SetScreenPosition(25, 80)

Module.Every(SCRIPT_ID .. "_update", function()
    if not Self.IsAvailable() then
        statusText:SetText("Player unavailable")
        return
    end

    local hp = Self.GetHealthPercentage() or 0
    local mana = Self.GetManaPercentage() or 0
    statusText:SetText("HP " .. tostring(hp) .. "% | Mana " .. tostring(mana) .. "%")
end, 1000)
```

### 4. Temporary World Marker At Mouse Position

Use world HUD objects for short-lived visual debugging.

```lua
local SCRIPT_ID = "mouse_marker_example"

local pos = Self.GetMousePositionInWorld()
if pos then
    WorldBox:New(SCRIPT_ID .. "_box", pos.x, pos.y, pos.z)
        :SetSize(32, 32)
        :SetColor({ r = 255, g = 0, b = 0, a = 60 })
        :SetBorderColor({ r = 255, g = 0, b = 0, a = 255 })
        :SetLifetime(3000)
        :Create()
end
```

### 5. Find An Item In Open Containers

Container searches should tolerate missing items and closed backpacks.

```lua
local ITEM_ID = 268

local item = Container.FindItemInOpenContainers(ITEM_ID)
if item then
    print("Found item " .. tostring(ITEM_ID) .. " in an open container.")
else
    print("Item " .. tostring(ITEM_ID) .. " was not found in open containers.")
end
```

### 6. Cavebot Waypoint Script Label

Waypoint script labels should always resume the walker when done.

```lua
Self.Say("hi")

-- Keep this as the final line of normal waypoint scripts.
Cavebot.Walker.Resume()
```

### 7. Cavebot Custom Action Handler

Return structured action results so the cavebot can decide whether to continue or jump to another label.

```lua
Cavebot.Actions.Register("check_capacity_for_refill", function(context)
    local minCapacity = tonumber(context.minCapacity) or 100
    local capacity = Self.GetCapacity()

    if type(capacity) ~= "number" then
        return {
            ok = false,
            actionType = "check_capacity_for_refill",
            error = "capacity_unavailable",
            goToLabel = context.failureLabel
        }
    end

    local needsRefill = capacity < minCapacity
    return {
        ok = true,
        actionType = "check_capacity_for_refill",
        needsRefill = needsRefill,
        goToLabel = needsRefill and context.failureLabel or context.successLabel
    }
end)
```

### 8. Hotkey Callback

Register hotkeys with a stable id and keep the callback short.

```lua
Hotkeys.RegisterCombo({
    id = "say_hi_hotkey",
    combo = "ctrl+h",
    callback = function()
        if Self.IsAvailable() then
            Self.Say("hi")
        end
    end
})
```

### 9. Good API Style

Use the public PascalCase modules from the core libraries.

```lua
-- Good
local hp = Self.GetHealthPercentage()
local monsters = Creatures.GetVisibleMonsters(true)
local player = Creature.GetLocalPlayer()
```

Do not generate scripts that call old lower-camel names or undocumented low-level globals. Use the documented PascalCase modules instead.

## Script Safety Checklist
Before finalizing any script, verify:

1. All loops yield with `wait`, a module, or a scheduled callback and cannot freeze the game thread.
2. Every table access checks nil/type when live game or feature data can be absent.
3. Only documented PascalCase APIs and named enum constants are used.
4. Feature operations do not touch reserved/internal features and use idempotent Set/Enable/Disable calls.
5. Alt is not used for registered callback hotkeys; it is allowed only for synthetic `Hotkeys.SendCombo`.
6. Position/creature access handles invalid, despawned, or cross-floor objects safely.
7. IDs, indexes, percentages, delays, and ranges are validated before mutation.
8. Repeating modules are not hidden inside a blanket `pcall`; recoverable catches are narrow and logged.
9. Event/hook callbacks are constant-time and non-yielding: no wait, network,
   storage/files, profile operations, large JSON, scans, pathfinding, retry
   loops, or per-event scheduling. They only feed one stable worker through a
   coalesced slot or hard-bounded queue.
10. Detached getter snapshots are never edited as if they were live settings.
11. Every accepted Walker defer token is completed or intentionally allowed to time out.
12. `terminate()` is synchronous, non-yielding, fast, and restores any non-owner-scoped setting the script intentionally changed.
13. Stable IDs/names are used for modules, events, hotkeys, HUD elements, and storage namespaces.
14. Tests print explicit PASS and FAIL results so both UI output and per-script log files are useful.

## Recommended Script Template
Use this as baseline for generated scripts.

```lua
local SCRIPT_ID = "my_script_id"

local function log(msg)
    print("[" .. SCRIPT_ID .. "] " .. tostring(msg))
end

local function tick()
    if not Self.IsAvailable() then
        return
    end

    -- Keep work bounded. Let unexpected errors reach the runtime so this
    -- module is logged and isolated. Use pcall only for a known recoverable
    -- operation with a real fallback.
end

function init()
    Module.Every(SCRIPT_ID .. "_tick", tick, 200)
    log("PASS: initialized")
end

function terminate()
    -- Do not call wait() here. Native owner-scoped resources are cleaned
    -- automatically; only restore intentional shared/live setting changes.
    log("stopped")
end
```

## LLM Generation Rules (Copy Into Prompt)
When generating ValidusBot Lua scripts:

1. Use only exact APIs and enum names documented in this file and its generated appendix; never invent a getter, action, or generic update table.
2. Prefer canonical core wrappers and `Engine.*` namespaces over compatibility globals or underscore-prefixed bindings.
3. For every named input, result, or callback record, read its definition under
   `Canonical Types And Exact Table Contracts` and use only its case-sensitive
   fields. Never guess aliases such as `item.id`, `item.slot`, or
   `container.index`.
4. Include argument validation and nil/type checks for unavailable live state.
5. Use Alt only with `Hotkeys.SendCombo`, never with callback registration.
6. Never use or toggle the Objects Dumper or another internal/reserved feature.
7. For repeated logic, use `Module.Every`; for one-shot work, use `Module.After` or `Events.Schedule`; all loops must yield.
8. Make every event/hook callback constant-time and non-yielding. It may only
   copy minimal scalar fields into a coalesced slot or hard-bounded queue for one
   pre-existing `Module.Every` worker. Never wait, perform network/storage/file
   work, scan, pathfind, recursively traverse, or schedule one task per event.
9. Do not blanket-`pcall` repeating modules. Catch only expected recoverable failures, log them, and preserve a useful fallback.
10. Treat getter tables as snapshots and change bot state only through explicit setters/actions.
11. Use `Time.MonotonicMs()` for elapsed time and `Storage` for JSON-compatible persistent script state.
12. Put yieldable setup in `init()` and only fast, non-yielding cleanup in `terminate()`.
13. Use stable owner-scoped names and emit explicit PASS/FAIL lines in diagnostic scripts.
14. Return one complete runnable `.lua` file with concise comments and no TODO placeholders unless requested.

## LLM API-selection workflow

For each requested behavior, an LLM should:

1. Find the canonical signature in Appendix A and named constants in
   `lua_consts.lua` within that appendix.
2. Resolve every named record in `Canonical Types And Exact Table Contracts`
   before reading or constructing it; copy its field names exactly.
3. Prefer the highest-level matching wrapper (`Cavebot`, `Hotkeys`, `Cooldowns`, `Spells`, `Sound`, `Storage`, or a HUD class).
4. Use `Engine.<Feature>` only for explicit feature configuration or actions, and call a setter rather than editing a getter snapshot.
5. Decide whether the work is immediate, repeating, one-shot, or event-driven and choose `init`, `Module.Every`, `Module.After`/`Events.Schedule`, or an event proxy accordingly.
6. Add capability checks for architecture/client-version-specific functions and nil checks for unavailable game state.
7. Identify owner-scoped resources and any shared/live setting that must be restored in `terminate()`.
8. Ensure errors remain observable, callbacks stay bounded, and validation/test output states PASS or FAIL explicitly.

## Example Prompt for ChatGPT/Claude
Use this prompt with this file attached:

"Generate a production-safe ValidusBot Lua script using only exact APIs from
the attached spec and its generated appendix. Script goal: <describe goal>.
Resolve every named table record and copy its case-sensitive fields exactly.
Prefer canonical high-level wrappers, validate live values, and use managed
yielding/scheduling. Every event/hook callback must be constant-time and
non-yielding: never call Synchronize/main-thread helpers, lock, wait, perform
network/storage work, or start one deferred task per event. Let unexpected
module errors reach the runtime. Use Alt only for synthetic SendCombo calls.
Return one complete .lua file with explicit diagnostic logging and no invented
APIs."


---

## Engine feature-control API

Use `Engine` for bot feature configuration and actions. Configuration is exposed through explicit getters and setters; do not assume that feature objects or their internal fields are available. Getter tables are detached snapshots, so changing a returned table does not change the bot. Call the matching setter.

The main feature namespaces are `Engine.Healer`, `Engine.Conditions`, `Engine.HealFriend`, `Engine.MagicShooter`, `Engine.Targeting`, `Engine.AmmoRefill`, `Engine.EquipmentManager`, `Engine.Alarms`, `Engine.PVPTools`, `Engine.ComboBot`, `Engine.Extras`, `Engine.TankMode`, `Engine.Looter`, `Engine.TimerActions`, `Engine.SuppliesSorter`, `Engine.Channels`, `Engine.Walker`, `Engine.Lure`, `Engine.HUD`, `Engine.Delays`, and `Engine.Scripter`. Feature activation remains under `Engine.Features`.

Important behavior:

- Entry indices are 1-based. Most entry setters affect the active profile; profile-selection functions should be called first when a feature supports profiles.
- Explicit setters validate types and ranges and preserve feature side effects such as timer resets, parsed-name cache rebuilds, packet-event refreshes, and HUD state refreshes.
- `Engine.ComboBot.GetRoomState()` omits room passwords and transport details. Lua cannot connect, disconnect, or send raw Combo Bot room commands.
- `Engine.Scripter.Start(name)` accepts only an exact `.lua` filename returned by `Engine.Scripter.GetAvailableScripts()`. It cannot execute source strings or disable the sandbox.
- `Engine.Scripter.GetOutput(name)` returns the current or most recently archived Runtime Workspace output for that exact script filename. It is read-only and returns an empty string when no output is available.
- A running script must call `Engine.Scripter.StopSelf()` to unload itself. Self-restart is intentionally unavailable; another script may call `Restart(name)`.
- `Engine.HUD` element operations retain per-script ownership. A script cannot mutate or remove another script's HUD elements.
- `Engine.Walker.Defer(timeoutMs)` returns an owner-bound decision token. Finish it with `CompleteDeferred(token)`; invalid, expired, or cross-script tokens are rejected.
- `Engine.Lure.UpdateSetting(index, setting)` updates a 1-based setting through native parsing/validation; tables returned by `GetSettings()` are still detached copies.
- `Engine.Delays.SetServerPingCheckEnabled()` also applies the required game ping-check interval change.
- Internal services such as the object dumper, packet/event dispatcher, use-item queue, and Validus networking are not exposed through `Engine`.

The exact signatures below are authoritative. Avoid legacy generic update-table helpers; use the explicit field setter for the value being changed.

## Extended native Game API

These native Game functions supplement the existing core and Game functions.
Wrong argument types and out-of-range integers raise a Lua error. For void C++
methods, true means the call passed validation and was submitted to the game
method; it does not confirm a server response. False means unavailable state,
native rejection, or an unimplemented native party action.

GamePosition is {x: integer, y: integer, z: integer} with x/y 0..65535 and z
0..15. GameCreatureSnapshot is {id: integer, name: string, position:
GamePosition, healthPercent: integer}. GameTileFlags is {isWalkable: boolean,
hasAutoMap: boolean, hasTeleport: boolean, hasCreature: boolean,
isBlockingPath: boolean, hasGroundItem: boolean, hasBlockingItem: boolean,
hasNpc: boolean, speed: integer}. GameOffer is {name: string, itemId: integer,
buyPrice: integer, sellPrice: integer, capacity: integer}. GameMonsterScore
is {monsterCount: integer, tileScore: integer}. GameBestTile adds position:
GamePosition to GameMonsterScore. GameRotateResult is {monsterCount: integer,
rotateDirection: integer, playerInPattern: boolean, targetInPattern: boolean,
checkIfPositionIsPassable: boolean, hasValidDirection: boolean,
patternIndex: integer}. GameVTable is {vTable: hexadecimal string,
numberOfVirtualFunctions: integer}; this is diagnostic data, not a pointer.

GamePattern and GameRotatePattern each have damage:
GamePatternCoordinate[] and optional pvpSafe: GamePatternCoordinate[].
Each coordinate is {x: integer, y: integer, score?: integer}, with integer
fields in -127..127 and absent score 0. GameRotatePattern also accepts optional checkIfPositionIsPassable: boolean. Damage must be nonempty, and each
array can contain at most 128 coordinates. Monster names are at most 128
strings. Position callbacks receive GamePosition and return a boolean;
monsterEligible receives a creature ID. The four rotate patterns are ordered north, east, south, west. Each callback is limited to 4096
calls; errors propagate. The path filter receives (from: GamePosition,
to: GamePosition, isGoal: boolean) and returns true to block the step.

Path calls return TWO values: directions: integer[] and result: integer.
Directions are north 0, east 1, south 2, west 3, diagonals 5..8. Path
results are OK 0, same position 1, impossible 2, too far 3, no way 4,
goal blocked 5, none 6. Flags are a PathFindFlags bit mask. Game.FindPath
accepts an optional ninth boolean argument after flags;
Game.FindPathWithAllowedInitialDirections accepts it after allowNpcs; and
Game.FindPathWithPositionFilter accepts it after allowNpcs. This
allowSpeculativeUnknown argument defaults to false for all three calls, so
old scripts and omitted flags preserve their previous behavior.

### Items, status, and map

| Function | Arguments | Returns | Meaning |
| --- | --- | --- | --- |
| Game.HasUsedBloodRageBuff | meleeSkillId: integer (0..255) | integer or nil | Buff status; nil without object dumper. |
| Game.HasUsedSharpShooterBuff | none | integer or nil | Buff status; nil without object dumper. |
| Game.GetManaShieldCapacityValues | none | {capacity: integer, maxCapacity: integer} or nil | Both shield values, nil if unavailable. |
| Game.GetRopeItemId | none | integer | Configured rope ID; 0 if absent. |
| Game.GetShovelItemId | none | integer | Configured shovel ID; 0 if absent. |
| Game.GetMacheteItemId | none | integer | Configured machete ID; 0 if absent. |
| Game.GetScytheItemId | none | integer | Configured scythe ID; 0 if absent. |
| Game.GetCurrentSpeed | none | integer or nil | Current speed; nil without object dumper. |
| Game.GetBaseSpeed | none | integer or nil | Base speed; nil without object dumper. |
| Game.GetClientVersion | none | string | Active client version. |
| Game.GetClientVersionRef | none | string | Copy of active client version. |
| Game.GetSpellManaCostByWords | spellWords: string | integer or nil | Known spell mana cost, or nil. |
| Game.GetMapTilesOnScreen | relativeX?: integer (0..7), relativeY?: integer (0..5), multifloor?: boolean | GamePosition[] | Available screen tile positions; defaults 7, 5, false. |
| Game.IsTilePassable | position: GamePosition | boolean | Cached tile walkability. |
| Game.GetTileFlags | position: GamePosition | GameTileFlags or nil | Cached tile flags, or nil. |
| Game.IsFoodItem | itemId: integer (1..65535) | boolean | Food catalogue membership. |
| Game.IsShovelItem | itemId: integer (1..65535) | boolean | Shovel catalogue membership. |
| Game.IsRopeItem | itemId: integer (1..65535) | boolean | Rope catalogue membership. |
| Game.IsMacheteItem | itemId: integer (1..65535) | boolean | Machete catalogue membership. |
| Game.IsBlockingItem | itemId: integer (1..65535) | boolean | Blocking catalogue membership. |
| Game.IsAutoExploreBlockingItem | itemId: integer (1..65535) | boolean | Auto exploration blocker membership. |
| Game.IsAutoExploreOpenHoleItem | itemId: integer (1..65535) | boolean | Auto exploration open hole membership. |
| Game.IsLadderItem | itemId: integer (1..65535) | boolean | Ladder catalogue membership. |
| Game.IsRopeSpotItem | itemId: integer (1..65535) | boolean | Rope spot catalogue membership. |
| Game.IsClosedHoleItem | itemId: integer (1..65535) | boolean | Closed hole catalogue membership. |
| Game.IsOpenedHoleItem | itemId: integer (1..65535) | boolean | Open hole catalogue membership. |
| Game.IsClosedDoorItem | itemId: integer (1..65535) | boolean | Closed door catalogue membership. |
| Game.IsOpenDoorItem | itemId: integer (1..65535) | boolean | Open door catalogue membership. |
| Game.IsDepotLockerItem | itemId: integer (1..65535) | boolean | Depot locker catalogue membership. |
| Game.IsScytheItem | itemId: integer (1..65535) | boolean | Scythe catalogue membership. |
| Game.IsTeleportItem | itemId: integer (1..65535) | boolean | Teleport catalogue membership. |
| Game.GetFoodIds | none | integer[] | Food item IDs. |
| Game.GetRopeIds | none | integer[] | Rope item IDs. |
| Game.GetShovelIds | none | integer[] | Shovel item IDs. |
| Game.GetMacheteIds | none | integer[] | Machete item IDs. |
| Game.GetScytheIds | none | integer[] | Scythe item IDs. |
| Game.GetBlockingIds | none | integer[] | Blocking item IDs. |
| Game.GetAutoExploreOpenHoleIds | none | integer[] | Auto exploration open hole IDs. |
| Game.GetClosedHolesIds | none | integer[] | Closed hole IDs. |
| Game.GetClosedDoorsIds | none | integer[] | Closed door IDs. |
| Game.GetOpenDoorsIds | none | integer[] | Open door IDs. |
| Game.GetOpenHolesIds | none | integer[] | Open hole IDs. |
| Game.GetDepotLockerIds | none | integer[] | Depot locker IDs. |
| Game.GetRopeSpotsIds | none | integer[] | Rope spot IDs. |
| Game.GetLadderIds | none | integer[] | Ladder IDs. |
| Game.GetTeleportIds | none | integer[] | Teleport IDs. |
| Game.GetSkinningCorpseIdsByTool | none | table of integer[] by tool ID | Skinning corpse catalogue. |
| Game.IsConfiguredSkinningTarget | toolItemId: integer (1..65535), corpseItemId: integer (1..65535) | boolean | Whether this tool handles the corpse. |

### NPC, containers, and game actions

| Function | Arguments | Returns | Meaning |
| --- | --- | --- | --- |
| Game.IsNPCTradeWindowOpened | none | boolean | NPC trade window state. |
| Game.GetNPCSellOfferItemIds | none | integer[] | Sellable item IDs, empty if unavailable. |
| Game.GetNPCName | none | string or nil | Trade partner name, nil if closed. |
| Game.GetNPCOffers | none | GameOffer[] | Offers, empty if closed. |
| Game.GetNPCTradeOffers | none | GameOffer[] | Alias used by NpcTradeStorage. |
| Game.SetAttackingCreature | creatureId: integer (1..4294967295) | boolean | Select attack target. |
| Game.SetFollowingCreature | creatureId: integer (1..4294967295) | boolean | Select follow target. |
| Game.InviteToParty | creatureId: integer (1..4294967295) | boolean | Always false; C++ action is a stub. |
| Game.RevokeInvitationToParty | creatureId: integer (1..4294967295) | boolean | Always false; C++ action is a stub. |
| Game.JoinParty | creatureId: integer (1..4294967295) | boolean | Always false; C++ action is a stub. |
| Game.LeaveParty | none | boolean | Always false; C++ action is a stub. |
| Game.PassPartyLeadership | creatureId: integer (1..4294967295) | boolean | Always false; C++ action is a stub. |
| Game.SetPartySharedExperience | active: boolean | boolean | Always false; C++ action is a stub. |
| Game.MaximizeContainer | containerIndex: integer (0..255), maximized: boolean | boolean | Set container window state. |
| Game.CloseContainer | containerNumber: integer (0..255) | boolean | Close a container. |
| Game.RequestUpContainer | containerNumber: integer (0..255) | boolean | Request parent container. |
| Game.RequestNextPageForContainer | containerNumber: integer (0..255) | boolean | Request next page. |
| Game.RequestPreviousPageForContainer | containerNumber: integer (0..255) | boolean | Request previous page. |
| Game.RetrieveItemFromStash | itemId: integer (1..65535), itemCount: integer (1..4294967295) | boolean | Request stash items. |
| Game.CloseStashWindowDialog | none | boolean | Close stash dialog. |
| Game.IsStashWindowDialogOpened | none | boolean or nil | Dialog state, nil without object dumper. |
| Game.OpenBrowseField | position: GamePosition | boolean | Browse a field at this position. |
| Game.QuickLootTile | position: GamePosition, topItemId: integer (1..65535), stackPosition: integer (0..255) | boolean | Quick loot a tile. |
| Game.AnswerServerModalDialog | buttonOptionId: integer (0..4294967295), dialogOptionId: integer (0..4294967295) | boolean | Answer a server modal dialog. |
| Game.OpenPrivateChannel | receiverName: string | boolean | Open private chat. |
| Game.Talk | message: string, messageMode: integer (0, 1, 2, or 9) | boolean | Send Whisper, Say, Yell, or Spell speech. |
| Game.TryMoveItemFromContainerToContainer | fromContainerNumber: integer (0..255), fromSlotNumber: integer (0..255), itemId: integer (1..65535), toContainerNumber: integer (0..255), toSlotNumber: integer (0..255), itemCount: integer (1..255) | boolean | Try moving between containers. |
| Game.TryMoveItemFromFloorToFloor | fromPosition: GamePosition, itemId: integer (1..65535), toPosition: GamePosition, itemCount: integer (1..255) | boolean | Try moving between floor tiles. |
| Game.UseWithItemFromFloorToContainer | floorPosition: GamePosition, fromItemId: integer (1..65535), toContainerNumber: integer (0..255), toSlotPosition: integer (0..255), toItemId: integer (1..65535) | boolean | Use floor item on container item. |
| Game.UseItemWithFromHotkeyOnContainerItem | fromItemId: integer (1..65535), toContainerNumber: integer (0..255), toSlotPosition: integer (0..255), toItemId: integer (1..65535) | boolean | Use hotkey item on container item. |
| Game.UseItemWithFromHotkeyOnEquipmentItem | fromItemId: integer (1..65535), toEquipmentSlot: integer (1..10), toItemId: integer (1..65535) | boolean | Use hotkey item on equipment. |
| Game.CancelWalkForSafetyBoundary | none | boolean | Cancel walking for safety; native result. |
| Game.QuickLootNearbyTiles | none | boolean | Quick loot nearby; x64 only. |
| Game.CastCrossHairSpell | spellWords: string, toPosition: GamePosition, creatureId: integer (0..4294967295) | boolean | Cast crosshair spell; x64 only. |

### Tactics and combat patterns

Fight mode values are 1 offensive, 2 balanced, 3 defensive; getter value
0 means unknown. Chase mode is 0 stand, 1 chase, 2 unknown. PVP mode is
0 white dove, 1 white hand, 2 yellow hand, 3 red fist, 4 unknown.
GetMonstersInArea flags are 1 PvP safe, 2 shoot over allies, 4 compare
monster names. Rotate direction is 0 north, 1 east, 2 south, 3 west.

| Function | Arguments | Returns | Meaning |
| --- | --- | --- | --- |
| Game.SetFightModeTactics | fightMode: integer (1..3) | boolean | Set fight mode. |
| Game.SetChaseModeTactics | chaseMode: integer (0..1) | boolean | Set chase mode. |
| Game.SetPVPEnabledTactics | enabled: boolean | boolean | Enable PvP. |
| Game.SetPVPModeTactics | pvpMode: integer (0..3) | boolean | Set PvP mode. |
| Game.SetFightingTactics | fightMode: integer (1..3), chaseMode: integer (0..1), pvpEnabled: boolean, expertPVPModeEnabled: boolean, pvpMode: integer (0..3) | boolean | Set all tactics. |
| Game.SetFightingTacticsWithoutFightMode | chaseMode: integer (0..1), pvpEnabled: boolean, expertPVPModeEnabled: boolean, pvpMode: integer (0..3) | boolean | Set newer client tactics; x64 only. |
| Game.GetFightModeTactics | none | integer or nil | Fight mode; nil without dumper. |
| Game.GetChaseModeTactics | none | integer or nil | Chase mode; nil without dumper. |
| Game.IsPVPEnabledTactics | none | boolean or nil | PvP state; nil without dumper. |
| Game.IsExpertPVPModeEnabled | none | boolean or nil | Expert PvP state; nil without dumper. |
| Game.GetPVPModeTactics | none | integer or nil | PvP mode; nil without dumper. |
| Game.GetMonstersInArea | center: GamePosition, pattern: GamePattern, flags: integer (0..7), minHealthPercentage: integer (0..100), maxHealthPercentage: integer (min..100), monsterNames: string[], monsterAllowed?: function | GameMonsterScore | Count monsters around a position. |
| Game.GetMonstersInAreaFromTile | center: GamePosition, pattern: GamePattern, minHealthPercentage: integer (0..100), maxHealthPercentage: integer (min..100), pvpSafe: boolean, shootOverAllies: boolean, monsterNames: string[], compareAgainstNames: boolean, monsterAllowed?: function | GameMonsterScore or nil | Tile overload; nil if tile unavailable. |
| Game.IsUnsafePlayerForPVPSafe | creatureId: integer (1..4294967295), allowAllies: boolean | boolean or nil | Unsafe player test, nil if creature absent. |
| Game.HasUnsafePlayerOnScreen | allowAllies: boolean | boolean | Visible unsafe player test. |
| Game.IsPatternPVPSafe | center: GamePosition, pattern: GamePattern, allowAllies: boolean, requireLineOfSight?: boolean | boolean | Pattern PvP safety; line of sight defaults true. |
| Game.IsRotatePatternPVPSafe | center: GamePosition, pattern: GameRotatePattern, allowAllies: boolean, requireLineOfSight?: boolean | boolean | Rotate pattern PvP safety. |
| Game.GetMonstersInPatternFromSelfForRotateSpells | patterns: GameRotatePattern[4], minHealthPercentage: integer (0..100), maxHealthPercentage: integer (min..100), pvpSafe: boolean, shootOverAllies: boolean, monsterNames: string[], compareAgainstNames: boolean, monsterAllowed?: function, monsterEligible?: function | GameRotateResult | Choose rotate direction around self. |
| Game.GetMonstersInPatternFromPosition | position: GamePosition, pattern: GamePattern, minHealthPercentage: integer (0..100), maxHealthPercentage: integer (min..100), pvpSafe: boolean, shootOverAllies: boolean, monsterNames: string[], compareAgainstNames: boolean, monsterAllowed?: function | integer | Count pattern hits. |
| Game.GetMonstersInPatternFromPositionExtraIconsCheck | position: GamePosition, pattern: GamePattern, minHealthPercentage: integer (0..100), maxHealthPercentage: integer (min..100), pvpSafe: boolean, shootOverAllies: boolean, monsterNames: string[], compareAgainstNames: boolean, creatureIconModificationsFlags: integer (0..6), monsterAllowed?: function | integer | Count hits with icon filter. |
| Game.GetBestTileByPattern | pattern: GamePattern, minHealthPercentage: integer (0..100), maxHealthPercentage: integer (min..100), pvpSafe: boolean, shootOverAllies: boolean, monsterNames: string[], compareAgainstNames: boolean, tileAllowed?: function, monsterAllowed?: function | GameBestTile or nil | Select best tile, nil if none. |
| Game.FindPathWithAllowedInitialDirections | start: GamePosition, goal: GamePosition, maxComplexity: integer (1..5000), flags: integer (0..255), allowedInitialDirectionsMask: integer (0..65535), allowNpcs?: boolean, allowSpeculativeUnknown?: boolean | integer[], integer | Path and result; both booleans default false. |
| Game.FindPathWithPositionFilter | start: GamePosition, goal: GamePosition, maxComplexity: integer (1..5000), flags: integer (0..255), isPositionBlocked: function, allowedInitialDirectionsMask?: integer (0..65535), allowNpcs?: boolean, allowSpeculativeUnknown?: boolean | integer[], integer | Path with filter; mask defaults 65535, booleans default false. |

### Client state and diagnostics

Creature and map results are detached snapshots. SetClientVersion accepts
only digits and dots, at most 32 characters, while offline.

| Function | Arguments | Returns | Meaning |
| --- | --- | --- | --- |
| Game.GetSelfCreature | none | GameCreatureSnapshot or nil | Local player snapshot. |
| Game.GetAttackingCreature | none | GameCreatureSnapshot or nil | Attack target snapshot. |
| Game.GetFollowingCreature | none | GameCreatureSnapshot or nil | Follow target snapshot. |
| Game.GetRTTIVTableBuffer | none | GameVTable[] | RTTI diagnostic snapshot. |
| Game.GetRTTI | rttiIndex: integer (0..44) | GameVTable | One RTTI entry. |
| Game.GetBitwiseAndKey | none | string | Bitwise key as hex string. |
| Game.SetClientVersion | clientVersionNumber: string | boolean | Set version and bitwise key offline. |
| Game.ReadItemsJsonFile | failOnUserJsonErrors?: boolean | boolean | Reload item definitions; default false. |
| Game.ReadSpellsAndRunesJsonFile | failOnUserJsonErrors?: boolean | boolean | Reload spells and runes; default false. |
| Game.InitilizeSpellAndRunePatterns | none | boolean | Rebuild spell and rune patterns. |
| Game.DumpSpellsInfoToJsonFile | none | boolean | Write spell information to product data. |
| Game.UpdateMapTilesCached | none | boolean | Refresh cached map tiles. |
| Game.PrintMessageInChannel | channelId: integer (0..255), messagePosition: integer (0..2), message: string, messageSender: string, textColor: table, senderLevel?: integer (0..65535) | boolean | Add local channel message. Color is {r?: integer (0..255), g?: integer (0..255), b?: integer (0..255), a?: integer (0..255)}; missing channels ignore the call. |
### Extended Walker Lua API

All route indices are one-based. Auto Explore and combat position arguments use x and y from 1 to 65535 and z from 0 to 15. A boolean mutation result is false when Walker is unavailable or the underlying C++ operation refuses the change. Invalid argument types and ranges raise a Lua error. The same methods are available through Engine.Walker.

AutoExploreSpan = {z: integer (0..15), y: integer (1..65535), xBegin: integer (1..65535), xEnd: integer (xBegin..65535)}. Mask mutations require Walker to be stopped. GetAutoExploreMaskSnapshot returns detached spans and revision; GetAutoExploreSpans returns detached spans alone. ReplaceAutoExploreMask validates the whole list before replacing it. SetAutoExploreTile and SetAutoExploreSpan use painted=true to add and false to erase.

SuppliesChecker = {minCapacity: integer (0..65535), minStaminaMinutes: integer (0..65535), checkLowCapacity: boolean, checkLowStamina: boolean, items: SupplyItem[]}. Each SupplyItem is {itemId: integer (1..65535), minCount: integer (0..65535), maxCount: integer (minCount..65535), enabled: boolean}. SetSuppliesChecker accepts a partial settings table; if items is supplied, it replaces the item array, capped at 1000 entries. GetSuppliesChecker returns a detached copy. Existing imbuement checks and labels are preserved.

CombatEgressDirectionGuidance = {physicalTraversableDirectionMask: integer, unsafeDirectionMask: integer, liveDiscouragedDirectionMask: integer, preferredDirectionMask: integer}. Direction bits follow Walker's native MoveDirection enum. FindAutoExploreConstrainedPath returns a direction array and a PathFindResult code (0 ok, 1 same position, 2 impossible, 3 too far, 4 no way, 5 blocked goal, 6 none). maxComplexity is 1..100000, flags is 0..255, and allowedInitialDirectionsMask defaults to 65535. PrepareAutoExploreStart requires Walker to be stopped, changes runtime preparation state, and returns an error message on failure.

| Function | Arguments | Returns | Meaning |
| --- | --- | --- | --- |
| Walker.AddWaypointAtPosition / Cavebot.Walker.AddWaypointAtPosition | waypoint: WalkerWaypointInput | integer|false | Adds a waypoint at its exact coordinates using the same input table as AddWaypoint. |
| Walker.IncrementWaypointIndex / Cavebot.Walker.IncrementWaypointIndex | resetLureRuntimeOnSelectionChange?: boolean | boolean | Advances route selection or completes the current Auto Explore target; false if no route is available. |
| Walker.ReorderWaypoint / Cavebot.Walker.ReorderWaypoint | sourceIndex: integer, targetIndex: integer, dropAfterTarget: boolean | boolean | Reorder route entries by one-based indices. |
| Walker.GetSelectedWaypoint / Cavebot.Walker.GetSelectedWaypoint | none | WalkerWaypoint|nil | Return a detached waypoint table or nil. |
| Walker.LabelExists / Cavebot.Walker.LabelExists | labelName: string | boolean | Check whether a route label exists. |
| Walker.GetSpecialAreaRevision / Cavebot.Walker.GetSpecialAreaRevision | none | integer | Get the special area revision. |
| Walker.GetWaypointTypeName / Cavebot.Walker.GetWaypointTypeName | waypointType: integer | string | Get a waypoint type name. |
| Walker.WaypointTypeUsesPosition / Cavebot.Walker.WaypointTypeUsesPosition | waypointType: integer | boolean | Check whether a waypoint type uses a coordinate. |
| Walker.GetActionWaypointKindName / Cavebot.Walker.GetActionWaypointKindName | actionKind: integer | string | Get an action waypoint kind name. |
| Walker.SetLeaveLurePlayerNames / Cavebot.Walker.SetLeaveLurePlayerNames | playerNames: string | boolean | Set comma-separated player names used by list modes. |
| Walker.GetLeaveLurePlayerNames / Cavebot.Walker.GetLeaveLurePlayerNames | none | string | Get the configured player names. |
| Walker.MatchesLeaveLurePlayerName / Cavebot.Walker.MatchesLeaveLurePlayerName | playerName: string | boolean | Check a name against the configured list. |
| Walker.SetAntiWallReposition / Cavebot.Walker.SetAntiWallReposition | enabled: boolean | boolean | Enable or disable anti-wall repositioning. |
| Walker.GetAntiWallReposition / Cavebot.Walker.GetAntiWallReposition | none | boolean | Read anti-wall repositioning. |
| Walker.SetAvoidNearbyCombatDeadEnds / Cavebot.Walker.SetAvoidNearbyCombatDeadEnds | enabled: boolean | boolean | Enable or disable nearby combat dead-end avoidance. |
| Walker.GetAvoidNearbyCombatDeadEnds / Cavebot.Walker.GetAvoidNearbyCombatDeadEnds | none | boolean | Read nearby combat dead-end avoidance. |
| Walker.SetCombatDeadEndCheckDistance / Cavebot.Walker.SetCombatDeadEndCheckDistance | distance: integer | boolean | Set combat dead-end check distance from 1 to 5. |
| Walker.GetCombatDeadEndCheckDistance / Cavebot.Walker.GetCombatDeadEndCheckDistance | none | integer | Read combat dead-end check distance. |
| Walker.SetAutoScrollWaypoints / Cavebot.Walker.SetAutoScrollWaypoints | enabled: boolean | boolean | Set route list auto-scroll. |
| Walker.GetAutoScrollWaypoints / Cavebot.Walker.GetAutoScrollWaypoints | none | boolean | Read route list auto-scroll. |
| Walker.SetWaypointMode / Cavebot.Walker.SetWaypointMode | mode: integer | boolean | Set recorder mode: 0 add, 1 replace, 2 insert. |
| Walker.GetWaypointMode / Cavebot.Walker.GetWaypointMode | none | integer | Read recorder mode. |
| Walker.SetWaypointDirection / Cavebot.Walker.SetWaypointDirection | direction: integer | boolean | Set recorder position offset direction from 0 to 8. |
| Walker.GetWaypointDirection / Cavebot.Walker.GetWaypointDirection | none | integer | Read recorder position offset direction. |
| Walker.GetMovementStatus / Cavebot.Walker.GetMovementStatus | none | table | Get current movement confirmation and reconciliation status. |
| Walker.HasLuaPauseOwnedByScript / Cavebot.Walker.HasLuaPauseOwnedByScript | none | boolean | Check whether this script owns a Walker pause. |
| Walker.GetAutoExploreSpans / Cavebot.Walker.GetAutoExploreSpans | none | AutoExploreSpan[] | Get painted area spans. |
| Walker.GetAutoExploreMaskRevision / Cavebot.Walker.GetAutoExploreMaskRevision | none | integer | Get painted area revision. |
| Walker.GetAutoExploreMaskSnapshot / Cavebot.Walker.GetAutoExploreMaskSnapshot | none | {revision: integer, spans: AutoExploreSpan[]} | Get painted area spans and revision. |
| Walker.SetAutoExploreTile / Cavebot.Walker.SetAutoExploreTile | x: integer, y: integer, z: integer, painted: boolean | boolean | Paint or erase a tile while Walker is stopped. |
| Walker.SetAutoExploreSpan / Cavebot.Walker.SetAutoExploreSpan | z: integer, y: integer, xBegin: integer, xEnd: integer, painted: boolean | boolean | Paint or erase a horizontal span while Walker is stopped. |
| Walker.ReplaceAutoExploreMask / Cavebot.Walker.ReplaceAutoExploreMask | spans: AutoExploreSpan[] | boolean | Replace painted spans while Walker is stopped. |
| Walker.ClearAutoExploreFloor / Cavebot.Walker.ClearAutoExploreFloor | z: integer | boolean | Clear painted tiles on one floor while Walker is stopped. |
| Walker.ClearAutoExploreMask / Cavebot.Walker.ClearAutoExploreMask | none | boolean | Clear all painted tiles while Walker is stopped. |
| Walker.IsAutoExploreMovementAllowed / Cavebot.Walker.IsAutoExploreMovementAllowed | fromX: integer, fromY: integer, fromZ: integer, toX: integer, toY: integer, toZ: integer, allowOutsideRecovery: boolean | boolean | Check an Auto Explore movement candidate. |
| Walker.FindAutoExploreConstrainedPath / Cavebot.Walker.FindAutoExploreConstrainedPath | fromX: integer, fromY: integer, fromZ: integer, toX: integer, toY: integer, toZ: integer, maxComplexity: integer, flags: integer, allowedInitialDirectionsMask: integer | integer[]|nil, integer | Find a route under Auto Explore traversal rules. |
| Walker.HasActiveAutoExploreTransition / Cavebot.Walker.HasActiveAutoExploreTransition | none | boolean | Check whether an Auto Explore connector transition is active. |
| Walker.PrepareAutoExploreStart / Cavebot.Walker.PrepareAutoExploreStart | none | boolean, string|nil | Prepare Auto Explore start and return success plus an optional error. |
| Walker.GetCombatEgressDirectionGuidance / Cavebot.Walker.GetCombatEgressDirectionGuidance | x: integer, y: integer, z: integer | CombatEgressDirectionGuidance|nil | Get movement direction masks for combat escape. |
| Walker.GetCombatEgressUnsafeDirectionMask / Cavebot.Walker.GetCombatEgressUnsafeDirectionMask | x: integer, y: integer, z: integer | integer | Get the unsafe combat escape direction mask. |
| Walker.GetCombatEgressProgressReturnDirectionMask / Cavebot.Walker.GetCombatEgressProgressReturnDirectionMask | x: integer, y: integer, z: integer | integer | Get the recent return direction mask. |
| Walker.IsCombatEgressHoldActive / Cavebot.Walker.IsCombatEgressHoldActive | x: integer, y: integer, z: integer | boolean | Check whether combat escape is holding this position. |
| Walker.GetSuppliesChecker / Cavebot.Walker.GetSuppliesChecker | none | SuppliesChecker | Get global supply thresholds and item entries. |
| Walker.SetSuppliesChecker / Cavebot.Walker.SetSuppliesChecker | settings: SuppliesChecker update | boolean | Update global supply thresholds and item entries. |

Walker movement observers are nonblocking: WalkerEvent.MOVEMENT_CONFIRMED=7 receives (oldX, oldY, oldZ, newX, newY, newZ, fullMapUpdate), and WalkerEvent.MOVEMENT_REJECTED=8 receives no arguments. Cavebot.OnMovementConfirmed(callback) and Cavebot.OnMovementRejected(callback) register these callbacks and return a registration handle. Rejection reports the incoming CancelWalk packet; it can include a stale rejection that Walker ignores internally.

Recorder modes are Add=0, Replace=1, Insert=2. Recorder directions are NorthWest=0, North=1, NorthEast=2, West=3, Center=4, East=5, SouthWest=6, South=7, SouthEast=8. Combat dead-end check distance is 1..5. Leave-lure modes PlayersInList=2 and PlayersNotInList=3 use SetLeaveLurePlayerNames; an asterisk matches every name. SetAutoRecorderEnabled returns false when the requested state cannot be applied while Walker is running. SetAutoRecorderOptions rejects wrong-typed supplied fields without applying any option changes. GetAutoRecorderOptions always returns all three boolean fields, using true defaults when Walker is unavailable.

## Appendix A: Public Core Lua API Index (Auto-Generated)
Generated from `docs/Scripts/core`. It intentionally excludes local helpers, compatibility aliases, raw bindings, protocol details, and API-surface bootstrap internals. Use the canonical names below; if a function is absent, treat it as unavailable.

### cavebot.lua
- `Cavebot.Defer(timeoutMs: integer) -> CavebotDeferredHandle`
- `Cavebot.Disable() -> nil`
- `Cavebot.DisableLure() -> nil`
- `Cavebot.Enable() -> nil`
- `Cavebot.EnableLure() -> nil`
- `Cavebot.GetStatus() -> CavebotStatus`
- `Cavebot.GoTo(labelName: string) -> nil`
- `Cavebot.GoToLabel(labelName: string) -> nil`
- `Cavebot.InterceptAction(callback: function(actionName: string)) -> integer|nil`
- `Cavebot.InterceptLabel(callback: function(labelName: string)) -> integer|nil`
- `Cavebot.IsEnabled() -> boolean`
- `Cavebot.IsLureEnabled() -> boolean`
- `Cavebot.Load(path: string, features?: CavebotBundleFeatureSelection) -> boolean, string`
- `Cavebot.Lure.AddSetting(setting: LureSettingInput) -> integer|false`
- `Cavebot.Lure.ClearSettings() -> boolean`
- `Cavebot.Lure.EndForceLure() -> boolean`
- `Cavebot.Lure.GetAttackWhileLuring() -> boolean`
- `Cavebot.Lure.GetConsiderOnlyReachable() -> boolean`
- `Cavebot.Lure.GetIgnoringMonsters() -> boolean`
- `Cavebot.Lure.GetKitingCloseMonsterCount() -> integer`
- `Cavebot.Lure.GetKitingCloseMonsterDistance() -> integer`
- `Cavebot.Lure.GetKitingMaximumFarthestDistance() -> integer`
- `Cavebot.Lure.GetKitingPreferredFarthestDistance() -> integer`
- `Cavebot.Lure.GetLuredCreaturesCount() -> integer`
- `Cavebot.Lure.GetNearRange() -> integer`
- `Cavebot.Lure.GetOption() -> integer`
- `Cavebot.Lure.GetSettingCount() -> integer`
- `Cavebot.Lure.GetSettings() -> LureSetting[]`
- `Cavebot.Lure.GetSlowWalkBurstSteps() -> integer`
- `Cavebot.Lure.GetSlowWalkDelayMs() -> integer`
- `Cavebot.Lure.GetSlowWalkingCreaturesCount() -> integer`
- `Cavebot.Lure.GetStartEndLureActive() -> boolean`
- `Cavebot.Lure.GetState() -> integer`
- `Cavebot.Lure.GetUnblocking() -> boolean`
- `Cavebot.Lure.GetWaypointDynamicLureActive() -> boolean`
- `Cavebot.Lure.HasActiveSettings() -> boolean`
- `Cavebot.Lure.IsEnabled() -> boolean`
- `Cavebot.Lure.IsFighting() -> boolean`
- `Cavebot.Lure.IsForceLure() -> boolean`
- `Cavebot.Lure.IsLuring() -> boolean`
- `Cavebot.Lure.IsOtherPlayerOnScreen() -> boolean`
- `Cavebot.Lure.RemoveSetting(index: integer) -> boolean`
- `Cavebot.Lure.SetAttackWhileLuring(enabled: boolean) -> boolean`
- `Cavebot.Lure.SetConsiderOnlyReachable(enabled: boolean) -> boolean`
- `Cavebot.Lure.SetEnabled(enabled: boolean) -> nil`
- `Cavebot.Lure.SetForceLure(enabled: boolean) -> boolean`
- `Cavebot.Lure.SetIgnoringMonsters(enabled: boolean) -> boolean`
- `Cavebot.Lure.SetKitingCloseMonsterCount(count: integer) -> boolean`
- `Cavebot.Lure.SetKitingCloseMonsterDistance(distance: integer) -> boolean`
- `Cavebot.Lure.SetKitingMaximumFarthestDistance(distance: integer) -> boolean`
- `Cavebot.Lure.SetKitingPreferredFarthestDistance(distance: integer) -> boolean`
- `Cavebot.Lure.SetNearRange(range: integer) -> boolean`
- `Cavebot.Lure.SetOption(option: integer) -> boolean`
- `Cavebot.Lure.SetSlowWalkBurstSteps(steps: integer) -> boolean`
- `Cavebot.Lure.SetSlowWalkDelayMs(delayMs: integer) -> boolean`
- `Cavebot.Lure.SetSlowWalkingCreaturesCount(count: integer) -> boolean`
- `Cavebot.Lure.SetStartEndLureActive(enabled: boolean) -> boolean`
- `Cavebot.Lure.SetUnblocking(enabled: boolean) -> boolean`
- `Cavebot.Lure.SetWaypointDynamicLureActive(enabled: boolean) -> boolean`
- `Cavebot.Lure.UpdateSetting(index: integer, updateData: LureSettingInput) -> boolean`
- `Cavebot.ObserveAction(callback: function(actionName: string)) -> integer|nil`
- `Cavebot.ObserveLabel(callback: function(labelName: string)) -> integer|nil`
- `Cavebot.ObserveWaypointChange(callback: function(event: CavebotWaypointChangeEvent)) -> integer|nil`
- `Cavebot.OnAction(callback: function(actionName: string)) -> integer|nil`
- `Cavebot.OnActionCompleted(callback: function(event: CavebotActionCompletedEvent)) -> integer|nil`
- `Cavebot.OnActionStarted(callback: function(event: CavebotActionStartedEvent)) -> integer|nil`
- `Cavebot.OnLabel(callback: function(labelName: string)) -> integer|nil`
- `Cavebot.OnMovementConfirmed(callback: function(oldX:integer,oldY:integer,oldZ:integer,newX:integer,newY:integer,newZ:integer,fullMapUpdate:boolean)) -> integer|nil`
- `Cavebot.OnMovementRejected(callback: function()) -> integer|nil`
- `Cavebot.OnWaypointChange(callback: function(event: CavebotWaypointChangeEvent)) -> integer|nil`
- `Cavebot.Pause(milliseconds: integer, autoResume: boolean) -> string|nil`
- `Cavebot.PrintStatus() -> nil`
- `Cavebot.RegisterEvent(eventId: integer, callback: WalkerEventCallback) -> integer|nil`
- `Cavebot.Resume() -> nil`
- `Cavebot.Save(path: string, features?: CavebotBundleFeatureSelection) -> boolean, string`
- `Cavebot.SetEnabled(enabled: boolean) -> nil`
- `Cavebot.SetEnginesEnabled(walkerEnabled: boolean, lureEnabled: boolean) -> CavebotStatus`
- `Cavebot.SetLureEnabled(enabled: boolean) -> nil`
- `Cavebot.UnregisterAllEvents() -> boolean`
- `Cavebot.Walker.AddAutoExploreConnector(connector?: WalkerAutoExploreConnectorInput) -> integer|string|false`
- `Cavebot.Walker.AddSpecialArea(area: WalkerSpecialAreaInput) -> integer|string|false`
- `Cavebot.Walker.AddWaypoint(waypoint: WalkerWaypointInput) -> integer|false`
- `Cavebot.Walker.AddWaypointAtPosition(waypoint: WalkerWaypointInput) -> integer|false`
- `Cavebot.Walker.ClearAutoExploreConnectors() -> boolean`
- `Cavebot.Walker.ClearAutoExploreFloor(z: integer) -> boolean`
- `Cavebot.Walker.ClearAutoExploreMask() -> boolean`
- `Cavebot.Walker.ClearSpecialAreas() -> boolean`
- `Cavebot.Walker.ClearWaypoints() -> boolean`
- `Cavebot.Walker.CompleteDeferred(token: integer) -> boolean`
- `Cavebot.Walker.Defer(timeoutMs: integer) -> integer`
- `Cavebot.Walker.DeleteAutoExploreConnector(id: integer|string) -> boolean`
- `Cavebot.Walker.DeleteSpecialArea(id: integer|string) -> boolean`
- `Cavebot.Walker.DeleteWaypoint(index: integer) -> boolean`
- `Cavebot.Walker.FindAutoExploreConstrainedPath(fromX: integer, fromY: integer, fromZ: integer, toX: integer, toY: integer, toZ: integer, maxComplexity: integer, flags: integer, allowedInitialDirectionsMask?: integer) -> integer[],integer`
- `Cavebot.Walker.GetActionWaypointKindName(actionKind: integer) -> string`
- `Cavebot.Walker.GetAntiWallReposition() -> boolean`
- `Cavebot.Walker.GetAutoExploreConnectorRecording() -> boolean`
- `Cavebot.Walker.GetAutoExploreConnectors() -> WalkerAutoExploreConnector[]`
- `Cavebot.Walker.GetAutoExploreMaskRevision() -> integer`
- `Cavebot.Walker.GetAutoExploreMaskSnapshot() -> WalkerAutoExploreMaskSnapshot`
- `Cavebot.Walker.GetAutoExploreSettings() -> WalkerAutoExploreSettings`
- `Cavebot.Walker.GetAutoExploreSpans() -> AutoExploreSpan[]`
- `Cavebot.Walker.GetAutoExploreStatus() -> WalkerAutoExploreStatus`
- `Cavebot.Walker.GetAutoRecorderEnabled() -> boolean`
- `Cavebot.Walker.GetAutoRecorderOptions() -> WalkerAutoRecorderOptions`
- `Cavebot.Walker.GetAutoScrollWaypoints() -> boolean`
- `Cavebot.Walker.GetAvoidNearbyCombatDeadEnds() -> boolean`
- `Cavebot.Walker.GetCombatDeadEndCheckDistance() -> integer`
- `Cavebot.Walker.GetCombatEgressDirectionGuidance(x: integer, y: integer, z: integer) -> WalkerCombatEgressDirectionGuidance|nil`
- `Cavebot.Walker.GetCombatEgressProgressReturnDirectionMask(x: integer, y: integer, z: integer) -> integer`
- `Cavebot.Walker.GetCombatEgressUnsafeDirectionMask(x: integer, y: integer, z: integer) -> integer`
- `Cavebot.Walker.GetDebugHud() -> boolean`
- `Cavebot.Walker.GetDistanceBetweenWaypoints() -> integer`
- `Cavebot.Walker.GetLeaveLureOnPlayer() -> boolean`
- `Cavebot.Walker.GetLeaveLurePlayerMode() -> integer`
- `Cavebot.Walker.GetLeaveLurePlayerNames() -> string`
- `Cavebot.Walker.GetMovementStatus() -> WalkerMovementStatus`
- `Cavebot.Walker.GetNavigationMode() -> WalkerNavigationMode`
- `Cavebot.Walker.GetNodeDistance() -> integer`
- `Cavebot.Walker.GetSelectedWaypoint() -> WalkerWaypoint|nil`
- `Cavebot.Walker.GetSelectedWaypointIndex() -> integer|nil`
- `Cavebot.Walker.GetSpecialAreaCount() -> integer`
- `Cavebot.Walker.GetSpecialAreaRevision() -> integer`
- `Cavebot.Walker.GetSpecialAreas() -> WalkerSpecialArea[]`
- `Cavebot.Walker.GetStartFromNearestWaypoint() -> boolean`
- `Cavebot.Walker.GetSuppliesChecker() -> WalkerSuppliesChecker|nil`
- `Cavebot.Walker.GetWalkToLureCenter() -> boolean`
- `Cavebot.Walker.GetWaypointCount() -> integer`
- `Cavebot.Walker.GetWaypointDirection() -> integer`
- `Cavebot.Walker.GetWaypointMode() -> integer`
- `Cavebot.Walker.GetWaypoints() -> WalkerWaypoint[]`
- `Cavebot.Walker.GetWaypointSuppliesChecker(index: number) -> WalkerSuppliesChecker|nil`
- `Cavebot.Walker.GetWaypointTypeName(waypointType: integer) -> string`
- `Cavebot.Walker.GoTo(labelName: string) -> nil`
- `Cavebot.Walker.HasActiveAutoExploreTransition() -> boolean`
- `Cavebot.Walker.HasLuaPauseOwnedByScript() -> boolean`
- `Cavebot.Walker.IncrementWaypointIndex(resetLureRuntimeOnSelectionChange?: boolean) -> boolean`
- `Cavebot.Walker.InsertWaypoint(index: integer, waypoint: WalkerWaypointInput) -> boolean`
- `Cavebot.Walker.IsAutoExploreMovementAllowed(fromX: integer, fromY: integer, fromZ: integer, toX: integer, toY: integer, toZ: integer, allowOutsideRecovery?: boolean) -> boolean`
- `Cavebot.Walker.IsAutoExplorePositionPainted(x: integer, y: integer, z: integer) -> boolean`
- `Cavebot.Walker.IsCombatEgressHoldActive(x: integer, y: integer, z: integer) -> boolean`
- `Cavebot.Walker.IsEnabled() -> boolean`
- `Cavebot.Walker.IsPausedByLua() -> boolean`
- `Cavebot.Walker.IsPositionInsideSpecialArea(x: integer, y: integer, z: integer, featureMask: integer) -> boolean`
- `Cavebot.Walker.IsStuck() -> boolean`
- `Cavebot.Walker.LabelExists(labelName: string) -> boolean`
- `Cavebot.Walker.MatchesLeaveLurePlayerName(playerName: string) -> boolean`
- `Cavebot.Walker.MoveWaypointDown(index: integer) -> boolean`
- `Cavebot.Walker.MoveWaypointUp(index: integer) -> boolean`
- `Cavebot.Walker.PrepareAutoExploreStart() -> boolean,string|nil`
- `Cavebot.Walker.ReorderSpecialArea(sourceIndex: integer, targetIndex: integer, dropAfterTarget?: boolean) -> boolean`
- `Cavebot.Walker.ReorderWaypoint(sourceIndex: integer, targetIndex: integer, dropAfterTarget: boolean) -> boolean`
- `Cavebot.Walker.ReplaceAutoExploreMask(spans: AutoExploreSpan[]) -> boolean`
- `Cavebot.Walker.ReplaceWaypoint(index: integer, waypoint: WalkerWaypointInput) -> boolean`
- `Cavebot.Walker.ResetAutoExploreCoverage() -> boolean`
- `Cavebot.Walker.Resume() -> nil`
- `Cavebot.Walker.SelectClosestWaypoint() -> boolean`
- `Cavebot.Walker.SetAntiWallReposition(enabled: boolean) -> boolean`
- `Cavebot.Walker.SetAutoExploreConnectorRecording(enabled: boolean) -> boolean`
- `Cavebot.Walker.SetAutoExploreSettings(settings: WalkerAutoExploreSettingsPatch) -> boolean`
- `Cavebot.Walker.SetAutoExploreSpan(z: integer, y: integer, xBegin: integer, xEnd: integer, painted: boolean) -> boolean`
- `Cavebot.Walker.SetAutoExploreTile(x: integer, y: integer, z: integer, painted: boolean) -> boolean`
- `Cavebot.Walker.SetAutoRecorderEnabled(enabled: boolean) -> boolean`
- `Cavebot.Walker.SetAutoRecorderOptions(options: WalkerAutoRecorderOptionsPatch) -> boolean`
- `Cavebot.Walker.SetAutoScrollWaypoints(enabled: boolean) -> boolean`
- `Cavebot.Walker.SetAvoidNearbyCombatDeadEnds(enabled: boolean) -> boolean`
- `Cavebot.Walker.SetCombatDeadEndCheckDistance(distance: integer) -> boolean`
- `Cavebot.Walker.SetDebugHud(enabled: boolean) -> boolean`
- `Cavebot.Walker.SetDistanceBetweenWaypoints(distance: integer) -> boolean`
- `Cavebot.Walker.SetEnabled(enabled: boolean) -> nil`
- `Cavebot.Walker.SetLeaveLureOnPlayer(enabled: boolean) -> boolean`
- `Cavebot.Walker.SetLeaveLurePlayerMode(mode: integer) -> boolean`
- `Cavebot.Walker.SetLeaveLurePlayerNames(playerNames: string) -> boolean`
- `Cavebot.Walker.SetNavigationMode(mode: "waypoints"|"auto_explore") -> boolean`
- `Cavebot.Walker.SetNodeDistance(distance: integer) -> boolean`
- `Cavebot.Walker.SetPausedByLua(paused: boolean) -> boolean`
- `Cavebot.Walker.SetSelectedWaypointIndex(index: integer) -> boolean`
- `Cavebot.Walker.SetStartFromNearestWaypoint(enabled: boolean) -> boolean`
- `Cavebot.Walker.SetSuppliesChecker(settings: WalkerSuppliesCheckerUpdate) -> boolean`
- `Cavebot.Walker.SetWalkToLureCenter(enabled: boolean) -> boolean`
- `Cavebot.Walker.SetWaypointDirection(direction: integer) -> boolean`
- `Cavebot.Walker.SetWaypointMode(mode: integer) -> boolean`
- `Cavebot.Walker.SetWaypointPosition(index: integer, x: integer, y: integer, z: integer) -> boolean`
- `Cavebot.Walker.UpdateAutoExploreConnector(id: integer|string, updateData: WalkerAutoExploreConnectorPatch) -> boolean`
- `Cavebot.Walker.UpdateSpecialArea(id: integer|string, updateData: WalkerSpecialAreaPatch) -> boolean`
- `Cavebot.Walker.WaypointTypeUsesPosition(waypointType: integer) -> boolean`

### cavebot_actions.lua
- `Cavebot.Actions.GetLastResult() -> CavebotActionResult|nil`
- `Cavebot.Actions.Register(actionType: string, handler: function(context: CavebotActionContext) -> CavebotActionResult) -> boolean`
- `Cavebot.Actions.Run(context: CavebotActionContext) -> CavebotActionResult`

### chat_channel.lua
- `ChatChannel.FromIdentifier(identifier: ChatChannelIdentifier) -> ChatChannel|nil`
- `ChatChannel.GetById(channelId: integer) -> ChatChannel|nil`
- `ChatChannel.GetByName(channelName: string) -> ChatChannel|nil`
- `ChatChannel.New(channelOrId: ChatChannelInput|integer, channelName?: string) -> ChatChannel`
- `ChatChannel:CanSend() -> boolean`
- `ChatChannel:GetId() -> integer`
- `ChatChannel:GetName() -> string`
- `ChatChannel:IsLocal() -> boolean`
- `ChatChannel:IsOpened() -> boolean`
- `ChatChannel:IsServerLog() -> boolean`
- `ChatChannel:IsValid() -> boolean`
- `ChatChannel:Refresh() -> boolean`
- `ChatChannel:Send(message: string) -> boolean`
- `ChatChannel:ToString() -> string`
- `ChatChannel:ToTable() -> ChatChannelRecord`

### chat_channel_storage.lua
- `ChatChannelStorage.CanSend(channelIdentifier: ChatChannelIdentifier) -> boolean`
- `ChatChannelStorage.FormatChannel(channel: ChatChannelRecord) -> string`
- `ChatChannelStorage.GetChannelNames(onlySendable?: boolean) -> string[]`
- `ChatChannelStorage.GetChatChannelById(channelId: integer) -> ChatChannelRecord|nil`
- `ChatChannelStorage.GetChatChannelByName(channelName: string) -> ChatChannelRecord|nil`
- `ChatChannelStorage.GetChatChannelCount() -> integer`
- `ChatChannelStorage.GetChatChannels() -> ChatChannelRecord[]`
- `ChatChannelStorage.GetLocalChatChannel() -> ChatChannelRecord|nil`
- `ChatChannelStorage.GetOpenedChannelCount() -> integer`
- `ChatChannelStorage.GetOpenedChannels() -> ChatChannelRecord[]`
- `ChatChannelStorage.GetServerLogChannel() -> ChatChannelRecord|nil`
- `ChatChannelStorage.GetSnapshot() -> ChatChannelStorageSnapshot`
- `ChatChannelStorage.HasChannelById(channelId: integer) -> boolean`
- `ChatChannelStorage.HasChannelByName(channelName: string) -> boolean`
- `ChatChannelStorage.IsAvailable() -> boolean`
- `ChatChannelStorage.ResolveChannel(channelIdentifier: ChatChannelIdentifier) -> ChatChannelRecord|nil`
- `ChatChannelStorage.Send(message: string, channelIdentifier: ChatChannelIdentifier) -> boolean`
- `ChatChannelStorage.ToIdLookupTable() -> table<integer, ChatChannelRecord>`
- `ChatChannelStorage.ToNameLookupTable() -> table<string, ChatChannelRecord>`

### client.lua
- `Client.GetPing() -> integer|nil`
- `Client.IsConnectionStable() -> boolean`

### container.lua
- `Container.FindItem(containerNumber: integer, itemId: integer, tierLevel?: integer) -> ContainerFindResult|nil`
- `Container.FindItemInOpenContainers(itemId: integer, tierLevel?: integer) -> ContainerFindResult|nil`
- `Container.GetById(containerId: integer) -> ContainerSnapshot|nil`
- `Container.GetByName(containerName: string) -> ContainerSnapshot|nil`
- `Container.GetByNumber(containerNumber: integer) -> ContainerSnapshot|nil`
- `Container.GetFreeSlots(containerNumber: integer) -> integer|nil`
- `Container.GetId(containerNumber: integer) -> integer|nil`
- `Container.GetItem(containerNumber: integer, slotIndex: integer) -> ContainerItem|nil`
- `Container.GetItems(containerNumber: integer) -> ContainerItem[]`
- `Container.GetItemsCount(containerNumber: integer) -> integer|nil`
- `Container.GetName(containerNumber: integer) -> string|nil`
- `Container.GetOpenContainers() -> ContainerSummary[]`
- `Container.GetSize(containerNumber: integer) -> integer|nil`
- `Container.LookItem(itemId: integer, itemPos: integer, containerIndex: integer) -> boolean`
- `Container.MoveItemFromEquipmentToContainer(equipmentSlot: integer, containerIndex: integer, slotIndex: integer, itemId: integer, itemCount: integer) -> boolean`
- `Container.MoveItemToContainer(fromContainerIndex: integer, fromSlotIndex: integer, itemId: integer, toContainerIndex: integer, toSlotIndex: integer, itemCount: integer) -> boolean`
- `Container.MoveItemToEquipment(containerIndex: integer, slotIndex: integer, itemId: integer, equipmentSlot: integer, itemCount: integer) -> boolean`
- `Container.MoveItemToFloor(containerIndex: integer, slotIndex: integer, itemId: integer, toPosition: PositionLike, itemCount: integer) -> boolean`
- `Container.OpenBackpack() -> boolean`
- `Container.OpenInNewWindow(equipmentSlotOrContainerId: integer, fromContainerNumber?: integer, fromContainerSlot?: integer) -> boolean`
- `Container.UseItem(itemId: integer, containerIndex: integer, itemPos: integer, useItemWithHotkey?: boolean) -> boolean`

### cooldowns.lua
- `Cooldowns.Group.GetTimeLeft(groupId: integer) -> integer`
- `Cooldowns.Group.IsInCooldown(groupId: integer) -> boolean`
- `Cooldowns.Group.IsReady(groupId: integer) -> boolean`
- `Cooldowns.Group.WillBeReady(groupId: integer, timeMs?: integer) -> boolean`
- `Cooldowns.Item.GetTimeLeft(itemId: integer) -> integer`
- `Cooldowns.Item.IsInCooldown(itemId: integer) -> boolean`
- `Cooldowns.Item.IsReady(itemId: integer) -> boolean`
- `Cooldowns.Item.WillBeReady(itemId: integer, timeMs?: integer) -> boolean`
- `Cooldowns.Spell.GetTimeLeft(spellWords: string) -> integer`
- `Cooldowns.Spell.IsInCooldown(spellWords: string) -> boolean`
- `Cooldowns.Spell.IsReady(spellWords: string) -> boolean`
- `Cooldowns.Spell.WillBeReady(spellWords: string, timeMs?: integer) -> boolean`
- `Cooldowns.UseWith.IsExhausted() -> boolean`
- `Cooldowns.UseWith.IsReady() -> boolean`
- `Cooldowns.Utils.FormatTime(ms: number) -> string`
- `Cooldowns.Utils.GetStatus(spells?: string[]) -> CooldownStatus`
- `Cooldowns.Utils.PrintStatus(spells?: string[]) -> nil`

### creature.lua
- `Creature.GetFollowed() -> Creature|nil`
- `Creature.GetLocalPlayer() -> Creature|nil`
- `Creature.GetTarget() -> Creature|nil`
- `Creature:ClearCache() -> nil`
- `Creature:DistanceTo(targetPos: PositionLike) -> integer`
- `Creature:DistanceToCreature(otherCreature: Creature) -> integer`
- `Creature:Equals(other: Creature) -> boolean`
- `Creature:GetDirection() -> integer`
- `Creature:GetGuildShield() -> integer`
- `Creature:GetHealthPercent() -> integer`
- `Creature:GetId() -> integer`
- `Creature:GetLowercaseName() -> string`
- `Creature:GetMasterId() -> integer`
- `Creature:GetName() -> string`
- `Creature:GetOutfit() -> CreatureOutfit`
- `Creature:GetPartyShield() -> integer`
- `Creature:GetPosition() -> Position`
- `Creature:GetSkull() -> integer`
- `Creature:GetSpeed() -> integer`
- `Creature:GetVocation() -> integer`
- `Creature:IsAdjacentTo(targetPos: PositionLike) -> boolean`
- `Creature:IsGameMaster() -> boolean`
- `Creature:IsInGuild() -> boolean`
- `Creature:IsInParty() -> boolean`
- `Creature:IsMonster() -> boolean`
- `Creature:IsMounted() -> boolean`
- `Creature:IsNPC() -> boolean`
- `Creature:IsPartyLeader() -> boolean`
- `Creature:IsPlayer() -> boolean`
- `Creature:IsReachable() -> boolean`
- `Creature:IsSameFloor(targetPos: PositionLike) -> boolean`
- `Creature:IsShootable() -> boolean`
- `Creature:IsSkulled() -> boolean`
- `Creature:IsSummon() -> boolean`
- `Creature:IsValid() -> boolean`
- `Creature:IsVisible() -> boolean`
- `Creature:IsWarAlly() -> boolean`
- `Creature:IsWarEnemy() -> boolean`
- `Creature:New(creatureId: integer) -> Creature`
- `Creature:ToString() -> string`

### creature_iterators.lua
- `Creature.ICreatures() -> CreatureIterator`
- `Creature.IMonsters() -> CreatureIterator`
- `Creature.INpcs() -> CreatureIterator`
- `Creature.IPlayers() -> CreatureIterator`
- `Creatures.GetAttackingCreatureId() -> integer|nil`
- `Creatures.GetCreatureByName(creatureName: string) -> Creature|nil`
- `Creatures.GetCreatureIdsByScan(typeFlags?: integer, xRelativeDistance?: integer, yRelativeDistance?: integer, multifloor?: boolean, ignoreSummons?: boolean) -> integer[]`
- `Creatures.GetCreaturesByScan(typeFlags?: integer, xRelativeDistance?: integer, yRelativeDistance?: integer, multifloor?: boolean, ignoreSummons?: boolean) -> Creature[]`
- `Creatures.GetFollowingCreatureId() -> integer|nil`
- `Creatures.GetLocalPlayerId() -> integer|nil`
- `Creatures.GetPlayerIdUnderMouse() -> integer|nil`
- `Creatures.GetVisibleCreatureIds() -> integer[]`
- `Creatures.GetVisibleCreatures() -> Creature[]`
- `Creatures.GetVisibleMonsters(ignoreSummons?: boolean) -> Creature[]`
- `Creatures.GetVisibleNpcs() -> Creature[]`
- `Creatures.GetVisiblePlayers() -> Creature[]`
- `Creatures.IsCreatureOnScreen(creatureId: integer, xRelativeDistance?: integer, yRelativeDistance?: integer, multifloor?: boolean) -> boolean`

### engine.lua
- `Engine.Alarms.Disable(alarmId: integer) -> boolean`
- `Engine.Alarms.DisableAll() -> nil`
- `Engine.Alarms.Enable(alarmId: integer) -> boolean`
- `Engine.Alarms.EnableAll() -> nil`
- `Engine.Alarms.EnableOnly(alarmIdsList: integer[]) -> nil`
- `Engine.Alarms.GetConfig() -> AlarmsConfig`
- `Engine.Alarms.GetCreatureFilter() -> string`
- `Engine.Alarms.GetLowHealthThreshold() -> integer`
- `Engine.Alarms.GetLowManaThreshold() -> integer`
- `Engine.Alarms.GetMessageFilter() -> string`
- `Engine.Alarms.IsBringToFocusEnabled() -> boolean`
- `Engine.Alarms.IsEnabled(alarmId: integer) -> boolean`
- `Engine.Alarms.IsFlashWindowEnabled() -> boolean`
- `Engine.Alarms.IsIgnoringAllyPlayers() -> boolean`
- `Engine.Alarms.PrintStatus() -> nil`
- `Engine.Alarms.SetAlarmMessages(messages: string) -> boolean`
- `Engine.Alarms.SetBringToFocus(enabled: boolean) -> boolean`
- `Engine.Alarms.SetBringToFocusEnabled(value: boolean) -> boolean`
- `Engine.Alarms.SetCreatureDetectedNames(names: string) -> boolean`
- `Engine.Alarms.SetCreatureFilter(namesString: string) -> boolean`
- `Engine.Alarms.SetDamageTakenRange(minimumDamage: integer, maximumDamage: integer) -> boolean`
- `Engine.Alarms.SetEnemyNames(value: string) -> boolean`
- `Engine.Alarms.SetFlashWindow(enabled: boolean) -> boolean`
- `Engine.Alarms.SetFlashWindowEnabled(value: boolean) -> boolean`
- `Engine.Alarms.SetGmChatCheckEnabled(value: boolean) -> boolean`
- `Engine.Alarms.SetGmNames(value: string) -> boolean`
- `Engine.Alarms.SetIgnoreAllyPlayers(ignore: boolean) -> boolean`
- `Engine.Alarms.SetLowHealthPercentage(arg1: integer) -> boolean`
- `Engine.Alarms.SetLowHealthThreshold(percentage: integer) -> boolean`
- `Engine.Alarms.SetLowManaPercentage(arg1: integer) -> boolean`
- `Engine.Alarms.SetLowManaThreshold(percentage: integer) -> boolean`
- `Engine.Alarms.SetMessageFilter(messagesString: string) -> boolean`
- `Engine.Alarms.SetPlayerAttackFilterMode(value: integer) -> boolean`
- `Engine.Alarms.SetPlayerAttackNames(value: string) -> boolean`
- `Engine.Alarms.SetPlayerDetectedFilterMode(value: integer) -> boolean`
- `Engine.Alarms.SetPlayerDetectedNames(value: string) -> boolean`
- `Engine.Alarms.SetSkullFilterMode(value: integer) -> boolean`
- `Engine.Alarms.SetSkullNames(value: string) -> boolean`
- `Engine.Alarms.Toggle(alarmId: integer) -> boolean`
- `Engine.AmmoRefill.Add(ammoData: AmmoRefillInput) -> integer|false`
- `Engine.AmmoRefill.AddProfile(profileName?: string) -> integer|false, string|nil`
- `Engine.AmmoRefill.ClearAll() -> boolean`
- `Engine.AmmoRefill.Disable(index: integer) -> boolean`
- `Engine.AmmoRefill.DisableAll() -> nil`
- `Engine.AmmoRefill.Enable(index: integer) -> boolean`
- `Engine.AmmoRefill.EnableAll() -> nil`
- `Engine.AmmoRefill.EnableOnly(itemIdsList: integer[]) -> nil`
- `Engine.AmmoRefill.FindByItemId(itemId: integer) -> IndexedAmmoRefillEntry|nil`
- `Engine.AmmoRefill.FindProfileByName(profileName: string) -> integer|nil`
- `Engine.AmmoRefill.Get(index: integer) -> AmmoRefillEntry|nil`
- `Engine.AmmoRefill.GetActiveProfile() -> ProfileSummary|nil`
- `Engine.AmmoRefill.GetAll() -> AmmoRefillEntry[]`
- `Engine.AmmoRefill.GetCurrentProfile() -> ProfileSummary|nil`
- `Engine.AmmoRefill.GetProfileCount() -> integer`
- `Engine.AmmoRefill.GetProfileNames() -> string[]`
- `Engine.AmmoRefill.GetProfiles() -> FeatureProfile[]`
- `Engine.AmmoRefill.NextProfile() -> ProfileSummary|nil`
- `Engine.AmmoRefill.PrintProfiles() -> nil`
- `Engine.AmmoRefill.PrintStatus() -> nil`
- `Engine.AmmoRefill.Remove(index: integer) -> boolean`
- `Engine.AmmoRefill.RemoveProfile(profile: integer|string) -> boolean`
- `Engine.AmmoRefill.RenameProfile(profile: integer|string, newName: string) -> boolean, string|nil`
- `Engine.AmmoRefill.SetActiveProfile(profile: integer|string) -> boolean`
- `Engine.AmmoRefill.SetCurrentProfile(profile: integer|string) -> boolean`
- `Engine.AmmoRefill.SetEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.AmmoRefill.SetEntryEquipFromHotkey(entryIndex: integer, value: boolean) -> boolean`
- `Engine.AmmoRefill.SetEntryItemId(entryIndex: integer, value: integer) -> boolean`
- `Engine.AmmoRefill.SetEntryRefillLeftHand(entryIndex: integer, value: boolean) -> boolean`
- `Engine.AmmoRefill.SetEntryThreshold(entryIndex: integer, value: integer) -> boolean`
- `Engine.AmmoRefill.SetProfile(indexOrName: integer|string) -> boolean`
- `Engine.AmmoRefill.Toggle(index: integer) -> boolean|nil`
- `Engine.Channels.AddEntry(name: string, message: string, intervalSeconds: integer, channelId: integer, talkAction: integer, enabled: boolean|nil) -> integer`
- `Engine.Channels.ClearEntries() -> boolean`
- `Engine.Channels.GetEntries() -> ChannelManagerEntry[]`
- `Engine.Channels.GetGlobalDelay() -> integer`
- `Engine.Channels.RemoveEntry(index: integer) -> boolean`
- `Engine.Channels.SetEntryChannelId(entryIndex: integer, value: integer) -> boolean`
- `Engine.Channels.SetEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Channels.SetEntryIntervalSeconds(entryIndex: integer, value: integer) -> boolean`
- `Engine.Channels.SetEntryMessage(entryIndex: integer, message: string) -> boolean`
- `Engine.Channels.SetEntryName(entryIndex: integer, value: string) -> boolean`
- `Engine.Channels.SetEntryTalkAction(entryIndex: integer, value: integer) -> boolean`
- `Engine.Channels.SetGlobalDelay(value: integer) -> boolean`
- `Engine.ComboBot.GetClientEntries() -> ComboClientEntry[]`
- `Engine.ComboBot.GetMode() -> integer`
- `Engine.ComboBot.GetRoomEntries() -> ComboRoomEntry[]`
- `Engine.ComboBot.GetRoomState() -> ComboRoomState`
- `Engine.ComboBot.SetClientEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.ComboBot.SetClientEntryFocusOption(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetClientEntryLeaderAction(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetClientEntryLeaderName(entryIndex: integer, value: string) -> boolean`
- `Engine.ComboBot.SetClientEntryLeaderSpellWords(entryIndex: integer, value: string) -> boolean`
- `Engine.ComboBot.SetClientEntryMyAction(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetClientEntryMyRuneId(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetClientEntryMySpellWords(entryIndex: integer, value: string) -> boolean`
- `Engine.ComboBot.SetClientEntryRange(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetClientEntryRequiresTarget(entryIndex: integer, value: boolean) -> boolean`
- `Engine.ComboBot.SetClientEntryShootType(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetMode(value: integer) -> boolean`
- `Engine.ComboBot.SetRoomEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.ComboBot.SetRoomEntryEquipMode(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetRoomEntryLeaderAction(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetRoomEntryLeaderRuneId(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetRoomEntryLeaderSpellWords(entryIndex: integer, value: string) -> boolean`
- `Engine.ComboBot.SetRoomEntryMyAction(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetRoomEntryMyRuneId(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetRoomEntryMySpellWords(entryIndex: integer, value: string) -> boolean`
- `Engine.ComboBot.SetRoomEntryRange(entryIndex: integer, value: integer) -> boolean`
- `Engine.ComboBot.SetRoomEntryRequiresTarget(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Conditions.GetCastInProtectionZoneEnabled() -> boolean`
- `Engine.Conditions.GetHoldSpells() -> ConditionSpellEntry[]`
- `Engine.Conditions.GetManaShieldDelay() -> integer`
- `Engine.Conditions.GetManaShieldTimerBased() -> integer`
- `Engine.Conditions.GetRecoverySpellDelay() -> integer`
- `Engine.Conditions.GetRecoverySpellTimerBased() -> integer`
- `Engine.Conditions.GetSpells() -> ConditionSpellEntry[]`
- `Engine.Conditions.GetUseHasteWithSharpShooterEnabled() -> boolean`
- `Engine.Conditions.SetCastInProtectionZoneEnabled(value: boolean) -> boolean`
- `Engine.Conditions.SetHoldSpellEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Conditions.SetHoldSpellFlag(entryIndex: integer, value: integer) -> boolean`
- `Engine.Conditions.SetHoldSpellManaCost(entryIndex: integer, value: integer) -> boolean`
- `Engine.Conditions.SetHoldSpellWords(entryIndex: integer, value: string) -> boolean`
- `Engine.Conditions.SetManaShieldDelay(value: integer) -> boolean`
- `Engine.Conditions.SetManaShieldTimerBased(value: integer) -> boolean`
- `Engine.Conditions.SetRecoverySpellDelay(value: integer) -> boolean`
- `Engine.Conditions.SetRecoverySpellTimerBased(value: integer) -> boolean`
- `Engine.Conditions.SetSpellEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Conditions.SetSpellFlag(entryIndex: integer, value: integer) -> boolean`
- `Engine.Conditions.SetSpellManaCost(entryIndex: integer, value: integer) -> boolean`
- `Engine.Conditions.SetSpellWords(entryIndex: integer, value: string) -> boolean`
- `Engine.Conditions.SetUseHasteWithSharpShooterEnabled(value: boolean) -> boolean`
- `Engine.Delays.GetAlarmDelay() -> integer`
- `Engine.Delays.GetAntiIdleDelay() -> integer`
- `Engine.Delays.GetAttackCreatureDelay() -> integer`
- `Engine.Delays.GetAttackItemDelay() -> integer`
- `Engine.Delays.GetAttackSpellDelay() -> integer`
- `Engine.Delays.GetConnectionStabilityCheckEnabled() -> boolean`
- `Engine.Delays.GetDashDelay() -> integer`
- `Engine.Delays.GetDropItemDelay() -> integer`
- `Engine.Delays.GetEatFoodDelay() -> integer`
- `Engine.Delays.GetEquipItemDelay() -> integer`
- `Engine.Delays.GetGlobalQueueSystemEnabled() -> boolean`
- `Engine.Delays.GetHealFriendItemDelay() -> integer`
- `Engine.Delays.GetHealFriendSpellDelay() -> integer`
- `Engine.Delays.GetHealItemDelay() -> integer`
- `Engine.Delays.GetHealSpellDelay() -> integer`
- `Engine.Delays.GetItemCooldownSystemEnabled() -> boolean`
- `Engine.Delays.GetItemPredictionSystemEnabled() -> boolean`
- `Engine.Delays.GetLootDelay() -> integer`
- `Engine.Delays.GetMoveDelay() -> integer`
- `Engine.Delays.GetReconnectDelay() -> integer`
- `Engine.Delays.GetServerPingCheckEnabled() -> boolean`
- `Engine.Delays.GetSpellCooldownSystemEnabled() -> boolean`
- `Engine.Delays.GetSpellPredictionSystemEnabled() -> boolean`
- `Engine.Delays.GetSupportSpellDelay() -> integer`
- `Engine.Delays.GetTargetingWalkDelay() -> integer`
- `Engine.Delays.GetUseItemInContainerDelay() -> integer`
- `Engine.Delays.GetUseWithCooldownSystemEnabled() -> boolean`
- `Engine.Delays.GetWalkerUseItemDelay() -> integer`
- `Engine.Delays.GetWalkerUseWithItemDelay() -> integer`
- `Engine.Delays.GetWalkerWalkDelay() -> integer`
- `Engine.Delays.SetAlarmDelay(value: integer) -> boolean`
- `Engine.Delays.SetAntiIdleDelay(value: integer) -> boolean`
- `Engine.Delays.SetAttackCreatureDelay(value: integer) -> boolean`
- `Engine.Delays.SetAttackItemDelay(value: integer) -> boolean`
- `Engine.Delays.SetAttackSpellDelay(value: integer) -> boolean`
- `Engine.Delays.SetConnectionStabilityCheckEnabled(value: boolean) -> boolean`
- `Engine.Delays.SetDashDelay(value: integer) -> boolean`
- `Engine.Delays.SetDropItemDelay(value: integer) -> boolean`
- `Engine.Delays.SetEatFoodDelay(value: integer) -> boolean`
- `Engine.Delays.SetEquipItemDelay(value: integer) -> boolean`
- `Engine.Delays.SetGlobalQueueSystemEnabled(value: boolean) -> boolean`
- `Engine.Delays.SetHealFriendItemDelay(value: integer) -> boolean`
- `Engine.Delays.SetHealFriendSpellDelay(value: integer) -> boolean`
- `Engine.Delays.SetHealItemDelay(value: integer) -> boolean`
- `Engine.Delays.SetHealSpellDelay(value: integer) -> boolean`
- `Engine.Delays.SetItemCooldownSystemEnabled(value: boolean) -> boolean`
- `Engine.Delays.SetItemPredictionSystemEnabled(value: boolean) -> boolean`
- `Engine.Delays.SetLootDelay(value: integer) -> boolean`
- `Engine.Delays.SetMoveDelay(value: integer) -> boolean`
- `Engine.Delays.SetReconnectDelay(value: integer) -> boolean`
- `Engine.Delays.SetServerPingCheckEnabled(value: boolean) -> boolean`
- `Engine.Delays.SetSpellCooldownSystemEnabled(value: boolean) -> boolean`
- `Engine.Delays.SetSpellPredictionSystemEnabled(value: boolean) -> boolean`
- `Engine.Delays.SetSupportSpellDelay(value: integer) -> boolean`
- `Engine.Delays.SetTargetingWalkDelay(value: integer) -> boolean`
- `Engine.Delays.SetUseItemInContainerDelay(value: integer) -> boolean`
- `Engine.Delays.SetUseWithCooldownSystemEnabled(value: boolean) -> boolean`
- `Engine.Delays.SetWalkerUseItemDelay(value: integer) -> boolean`
- `Engine.Delays.SetWalkerUseWithItemDelay(value: integer) -> boolean`
- `Engine.Delays.SetWalkerWalkDelay(value: integer) -> boolean`
- `Engine.Equipment.CanMove() -> boolean`
- `Engine.Equipment.CanRead() -> boolean`
- `Engine.Equipment.Equip(itemId: integer, tierLevel?: integer) -> boolean`
- `Engine.Equipment.GetAllSlotItems() -> table<integer, EquipmentItem>`
- `Engine.Equipment.GetSlotConstants() -> EquipmentSlotConstants`
- `Engine.Equipment.GetSlotIds() -> integer[]`
- `Engine.Equipment.GetSlotItem(equipmentSlot: integer) -> EquipmentItem|nil`
- `Engine.Equipment.GetSlotItemId(equipmentSlot: integer) -> integer|nil`
- `Engine.Equipment.GetSnapshot() -> InventorySnapshot`
- `Engine.Equipment.HasItemInSlot(equipmentSlot: integer) -> boolean|nil`
- `Engine.Equipment.LookSlotItem(itemId: integer, equipmentSlot: integer) -> boolean`
- `Engine.Equipment.MoveFromContainerToSlot(containerIndex: integer, slotIndex: integer, itemId: integer, equipmentSlot: integer, itemCount: integer) -> boolean`
- `Engine.Equipment.MoveFromSlotToContainer(equipmentSlot: integer, containerIndex: integer, slotIndex: integer, itemId: integer, itemCount: integer) -> boolean`
- `Engine.EquipmentManager.AddProfile(profileName?: string) -> integer|false, string|nil`
- `Engine.EquipmentManager.FindProfileByName(profileName: string) -> integer|nil`
- `Engine.EquipmentManager.GetActiveProfile() -> ProfileSummary|nil`
- `Engine.EquipmentManager.GetCurrentProfile() -> ProfileSummary|nil`
- `Engine.EquipmentManager.GetEntries() -> EquipmentManagerEntry[]`
- `Engine.EquipmentManager.GetProfileCount() -> integer`
- `Engine.EquipmentManager.GetProfileNames() -> string[]`
- `Engine.EquipmentManager.GetProfiles() -> FeatureProfile[]`
- `Engine.EquipmentManager.NextProfile() -> ProfileSummary|nil`
- `Engine.EquipmentManager.RemoveProfile(profile: integer|string) -> boolean`
- `Engine.EquipmentManager.RenameProfile(profile: integer|string, newName: string) -> boolean, string|nil`
- `Engine.EquipmentManager.SetActiveProfile(profile: integer|string) -> boolean`
- `Engine.EquipmentManager.SetConditionCreatureNames(entryIndex: integer, conditionIndex: integer, creatureNames: string) -> boolean`
- `Engine.EquipmentManager.SetConditionCreaturesCount(entryIndex: integer, conditionIndex: integer, count: integer) -> boolean`
- `Engine.EquipmentManager.SetConditionMonstersAround(entryIndex: integer, conditionIndex: integer, count: integer) -> boolean`
- `Engine.EquipmentManager.SetConditionPlayersAround(entryIndex: integer, conditionIndex: integer, count: integer) -> boolean`
- `Engine.EquipmentManager.SetConditionTargetName(entryIndex: integer, conditionIndex: integer, targetName: string) -> boolean`
- `Engine.EquipmentManager.SetConditionType(entryIndex: integer, conditionIndex: integer, conditionType: integer) -> boolean`
- `Engine.EquipmentManager.SetCurrentProfile(profile: integer|string) -> boolean`
- `Engine.EquipmentManager.SetEntryCheckHealthRange(entryIndex: integer, value: boolean) -> boolean`
- `Engine.EquipmentManager.SetEntryCheckManaRange(entryIndex: integer, value: boolean) -> boolean`
- `Engine.EquipmentManager.SetEntryConditionOperator(entryIndex: integer, value: integer) -> boolean`
- `Engine.EquipmentManager.SetEntryDelay(entryIndex: integer, delayMs: integer) -> boolean`
- `Engine.EquipmentManager.SetEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.EquipmentManager.SetEntryEquipAction(entryIndex: integer, value: integer) -> boolean`
- `Engine.EquipmentManager.SetEntryEquipFromHotkey(entryIndex: integer, value: boolean) -> boolean`
- `Engine.EquipmentManager.SetEntryExcludedItemIds(entryIndex: integer, value: integer[]) -> boolean`
- `Engine.EquipmentManager.SetEntryExcludedItemIdsEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.EquipmentManager.SetEntryHasDelay(entryIndex: integer, value: boolean) -> boolean`
- `Engine.EquipmentManager.SetEntryHealthManaOperator(entryIndex: integer, value: integer) -> boolean`
- `Engine.EquipmentManager.SetEntryHealthRange(entryIndex: integer, minimumPercentage: integer, maximumPercentage: integer) -> boolean`
- `Engine.EquipmentManager.SetEntryItemId(entryIndex: integer, value: integer) -> boolean`
- `Engine.EquipmentManager.SetEntryKeepEquipped(entryIndex: integer, value: boolean) -> boolean`
- `Engine.EquipmentManager.SetEntryKeepEquippedDuration(entryIndex: integer, value: boolean) -> boolean`
- `Engine.EquipmentManager.SetEntryManaRange(entryIndex: integer, minimumPercentage: integer, maximumPercentage: integer) -> boolean`
- `Engine.EquipmentManager.SetEntrySecondaryItemId(entryIndex: integer, value: integer) -> boolean`
- `Engine.EquipmentManager.SetEntrySlot(entryIndex: integer, value: integer) -> boolean`
- `Engine.EquipmentManager.SetEntryTier(entryIndex: integer, value: integer) -> boolean`
- `Engine.EquipmentManager.SetEntryUseExtraConditions(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Extras.GetAntiIdleEnabled() -> boolean`
- `Engine.Extras.GetAutoMountEnabled() -> boolean`
- `Engine.Extras.GetChangeGoldEnabled() -> boolean`
- `Engine.Extras.GetDashEnabled() -> boolean`
- `Engine.Extras.GetDisableMagicEffectsEnabled() -> boolean`
- `Engine.Extras.GetDisplayItemIdEnabled() -> boolean`
- `Engine.Extras.GetDodgeEnabled() -> boolean`
- `Engine.Extras.GetEatFoodEnabled() -> boolean`
- `Engine.Extras.GetEatFoodIds() -> integer[]`
- `Engine.Extras.GetExerciseDummyIds() -> integer[]`
- `Engine.Extras.GetExerciseWeaponIds() -> integer[]`
- `Engine.Extras.GetFakeXlogEnabled() -> boolean`
- `Engine.Extras.GetFollowDistance() -> integer`
- `Engine.Extras.GetFollowMode() -> integer`
- `Engine.Extras.GetFollowPlayerEnabled() -> boolean`
- `Engine.Extras.GetFollowPlayerName() -> string`
- `Engine.Extras.GetGoldChangeIds() -> integer[]`
- `Engine.Extras.GetOpenPrivateChannelOnPMEnabled() -> boolean`
- `Engine.Extras.GetReconnectEnabled() -> boolean`
- `Engine.Extras.GetReconnectWhenDeadEnabled() -> boolean`
- `Engine.Extras.GetShowShootEffectsEnabled() -> boolean`
- `Engine.Extras.GetTrainingDelay() -> integer`
- `Engine.Extras.GetTrainingEnabled() -> boolean`
- `Engine.Extras.SetAntiIdleEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetAutoMountEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetChangeGoldEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetDashEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetDisableMagicEffectsEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetDisplayItemIdEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetDodgeEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetEatFoodEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetEatFoodIds(itemIds: integer[]) -> boolean`
- `Engine.Extras.SetExerciseDummyIds(value: integer[]) -> boolean`
- `Engine.Extras.SetExerciseWeaponIds(value: integer[]) -> boolean`
- `Engine.Extras.SetFakeXlogEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetFollowDistance(value: integer) -> boolean`
- `Engine.Extras.SetFollowMode(value: integer) -> boolean`
- `Engine.Extras.SetFollowPlayerEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetFollowPlayerName(name: string) -> boolean`
- `Engine.Extras.SetGoldChangeIds(value: integer[]) -> boolean`
- `Engine.Extras.SetOpenPrivateChannelOnPMEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetReconnectEnabled(enabled: boolean) -> boolean`
- `Engine.Extras.SetReconnectWhenDeadEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetShowShootEffectsEnabled(value: boolean) -> boolean`
- `Engine.Extras.SetTrainingDelay(value: integer) -> boolean`
- `Engine.Extras.SetTrainingEnabled(value: boolean) -> boolean`
- `Engine.Extras.StartTraining() -> boolean`
- `Engine.Extras.StopTraining() -> boolean`
- `Engine.Features.Disable(featureIdentifier: integer|string) -> boolean`
- `Engine.Features.DisableAllExcept(excludeList?: FeatureIdentifier[]) -> nil`
- `Engine.Features.DisableMultiple(featureList: FeatureIdentifier[]) -> nil`
- `Engine.Features.Enable(featureIdentifier: integer|string) -> boolean`
- `Engine.Features.EnableMultiple(featureList: FeatureIdentifier[]) -> nil`
- `Engine.Features.GetActiveFeatures() -> integer[]`
- `Engine.Features.GetAllFeatureIds() -> integer[]`
- `Engine.Features.GetName(featureIdentifier: integer|string) -> string`
- `Engine.Features.IsActive(featureIdentifier: integer|string) -> boolean`
- `Engine.Features.PrintStatus() -> nil`
- `Engine.Features.SetActive(featureIdentifier: integer|string, activeStatus: boolean) -> boolean`
- `Engine.Features.Toggle(featureIdentifier: integer|string) -> boolean`
- `Engine.Healer.AddItem(itemData: HealerItemInput) -> integer|false`
- `Engine.Healer.AddSpell(spellData: HealerSpellInput) -> integer|false`
- `Engine.Healer.ClearAllItems() -> boolean`
- `Engine.Healer.ClearAllSpells() -> boolean`
- `Engine.Healer.DisableAllItems() -> nil`
- `Engine.Healer.DisableAllSpells() -> nil`
- `Engine.Healer.DisableItem(index: integer) -> boolean`
- `Engine.Healer.DisableSpell(index: integer) -> boolean`
- `Engine.Healer.EnableItem(index: integer) -> boolean`
- `Engine.Healer.EnableOnlyItems(itemIdsList: integer[]) -> nil`
- `Engine.Healer.EnableOnlySpells(spellWordsList: string[]) -> integer`
- `Engine.Healer.EnableSpell(index: integer) -> boolean`
- `Engine.Healer.FindItemById(itemId: integer) -> HealerItemEntry|nil`
- `Engine.Healer.FindSpellByWords(spellWords: string) -> IndexedHealerSpellEntry|nil`
- `Engine.Healer.GetItems() -> HealerItemEntry[]`
- `Engine.Healer.GetSpellByIndex(index: integer) -> HealerSpellEntry|nil`
- `Engine.Healer.GetSpells() -> HealerSpellEntry[]`
- `Engine.Healer.PrintItems() -> nil`
- `Engine.Healer.PrintSpells() -> nil`
- `Engine.Healer.RemoveItem(index: integer) -> boolean`
- `Engine.Healer.RemoveSpell(index: integer) -> boolean`
- `Engine.Healer.SetItemAction(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemAttribute(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemCastValue(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemCondition(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemDelay(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemEnabled(index: integer, enabled: boolean) -> boolean`
- `Engine.Healer.SetItemId(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemUseWhenFeared(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Healer.SetSpellAttribute(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetSpellCastValue(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetSpellCondition(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetSpellEnabled(index: integer, enabled: boolean) -> boolean`
- `Engine.Healer.SetSpellManaCost(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetSpellWords(entryIndex: integer, value: string) -> boolean`
- `Engine.Healer.ToggleItem(index: integer) -> boolean`
- `Engine.Healer.ToggleSpell(index: integer) -> boolean|nil`
- `Engine.HealFriend.GetArea() -> HealFriendArea`
- `Engine.HealFriend.GetMode() -> integer`
- `Engine.HealFriend.GetPlayerNames() -> string`
- `Engine.HealFriend.GetPrioritizeBeforeHealer() -> integer`
- `Engine.HealFriend.GetPriorityOverHealer() -> integer`
- `Engine.HealFriend.GetSafeHealthPercentage() -> integer`
- `Engine.HealFriend.GetVocations() -> HealFriendVocationEntry[]`
- `Engine.HealFriend.SetActionEnabled(vocationIndex: integer, actionIndex: integer, enabled: boolean) -> boolean`
- `Engine.HealFriend.SetActionHealthPercentage(vocationIndex: integer, actionIndex: integer, healthPercentage: integer) -> boolean`
- `Engine.HealFriend.SetActionItemId(vocationIndex: integer, actionIndex: integer, itemId: integer) -> boolean`
- `Engine.HealFriend.SetActionManaCost(vocationIndex: integer, actionIndex: integer, manaCost: integer) -> boolean`
- `Engine.HealFriend.SetActionMethod(vocationIndex: integer, actionIndex: integer, method: integer) -> boolean`
- `Engine.HealFriend.SetActionSpellWords(vocationIndex: integer, actionIndex: integer, spellWords: string) -> boolean`
- `Engine.HealFriend.SetAreaDruidRequired(value: boolean) -> boolean`
- `Engine.HealFriend.SetAreaEnabled(value: boolean) -> boolean`
- `Engine.HealFriend.SetAreaExtended(value: integer) -> boolean`
- `Engine.HealFriend.SetAreaHealthPercentage(value: integer) -> boolean`
- `Engine.HealFriend.SetAreaKnightRequired(value: boolean) -> boolean`
- `Engine.HealFriend.SetAreaManaCost(value: integer) -> boolean`
- `Engine.HealFriend.SetAreaMinimumHarmony(value: integer) -> boolean`
- `Engine.HealFriend.SetAreaMonkRequired(value: boolean) -> boolean`
- `Engine.HealFriend.SetAreaPaladinRequired(value: boolean) -> boolean`
- `Engine.HealFriend.SetAreaPlayersNeeded(count: integer) -> boolean`
- `Engine.HealFriend.SetAreaSorcererRequired(value: boolean) -> boolean`
- `Engine.HealFriend.SetAreaSpellWords(value: string) -> boolean`
- `Engine.HealFriend.SetAreaVocation(value: integer) -> boolean`
- `Engine.HealFriend.SetMode(value: integer) -> boolean`
- `Engine.HealFriend.SetPlayerNames(value: string) -> boolean`
- `Engine.HealFriend.SetPrioritizeBeforeHealer(value: integer) -> boolean`
- `Engine.HealFriend.SetPriorityOverHealer(value: integer) -> boolean`
- `Engine.HealFriend.SetSafeHealthPercentage(value: boolean) -> boolean`
- `Engine.HealFriend.SetVocationEnabled(vocationIndex: integer, enabled: boolean) -> boolean`
- `Engine.HealFriend.SetVocationPriority(vocationIndex: integer, priority: integer) -> boolean`
- `Engine.HUD.AddScreenImage(params: HudScreenImageParams) -> HudScreenImageParams`
- `Engine.HUD.AddScreenText(params: HudScreenTextParams) -> HudScreenTextParams`
- `Engine.HUD.AddWorldBox(params: HudWorldBoxParams) -> HudWorldBoxParams`
- `Engine.HUD.AddWorldImage(params: HudWorldImageParams) -> HudWorldImageParams`
- `Engine.HUD.AddWorldText(params: HudWorldTextParams) -> HudWorldTextParams`
- `Engine.HUD.ClearParent(child_id: string) -> nil`
- `Engine.HUD.GetConfig() -> HudFeatureConfig`
- `Engine.HUD.GetElementColor(id: string) -> ColorRGBA`
- `Engine.HUD.GetElementEnabled(id: string) -> boolean`
- `Engine.HUD.GetElementHeight(id: string) -> number`
- `Engine.HUD.GetElementText(id: string) -> string`
- `Engine.HUD.GetElementVisible(id: string) -> boolean`
- `Engine.HUD.GetElementWidth(id: string) -> number`
- `Engine.HUD.GetScreenElementPosition(id: string) -> HudScreenPosition`
- `Engine.HUD.GetSpecialFoodCounters() -> HudSpecialFoodCounter[]`
- `Engine.HUD.GetWorldElementPosition(id: string) -> Position`
- `Engine.HUD.RemoveElement(id: string) -> nil`
- `Engine.HUD.RemoveSpecialFoodCounter(itemId: integer) -> boolean`
- `Engine.HUD.SetAlignment(id: string, horizontal_align: integer, vertical_align: integer) -> nil`
- `Engine.HUD.SetClickable(id: string, clickable: boolean, callback: function()|nil) -> nil`
- `Engine.HUD.SetDraggable(id: string, draggable: boolean) -> nil`
- `Engine.HUD.SetDragTarget(id: string, targetId: string|nil) -> nil`
- `Engine.HUD.SetEnabled(elementId: string, enabled: boolean) -> nil`
- `Engine.HUD.SetLevelSpyEnabled(value: boolean) -> boolean`
- `Engine.HUD.SetMagicWallIds(value: integer[]) -> boolean`
- `Engine.HUD.SetMagicWallTimersEnabled(value: boolean) -> boolean`
- `Engine.HUD.SetOnDragEnd(id: string, callback: function(x: number, y: number)|nil) -> nil`
- `Engine.HUD.SetParent(child_id: string, parent_id: string) -> nil`
- `Engine.HUD.SetPosition(params: HudWorldPositionUpdate) -> nil`
- `Engine.HUD.SetScreenPosition(params: HudScreenPositionUpdate) -> nil`
- `Engine.HUD.SetSpecialFoodCounterDelay(itemId: integer, delaySeconds: integer) -> boolean`
- `Engine.HUD.SetTargetingAnchorEnabled(value: boolean) -> boolean`
- `Engine.HUD.SetTimerColor(red: number, green: number, blue: number, alpha: number) -> boolean`
- `Engine.HUD.SetWildGrowthIds(value: integer[]) -> boolean`
- `Engine.HUD.SetXRayEnabled(value: boolean) -> boolean`
- `Engine.HUD.SetZIndex(id: string, zIndex: integer) -> nil`
- `Engine.HUD.UpdateBorderColor(id: string, color: ColorRGBA) -> nil`
- `Engine.HUD.UpdateBorderWidth(id: string, border_width: number) -> nil`
- `Engine.HUD.UpdateColor(id: string, color: ColorRGBA) -> nil`
- `Engine.HUD.UpdateFont(id: string, fontFamily: string|nil, fontSize: integer|nil) -> nil`
- `Engine.HUD.UpdateHeight(id: string, height: number) -> nil`
- `Engine.HUD.UpdateImageLabel(params: HudImageLabelUpdate) -> nil`
- `Engine.HUD.UpdateLifetime(id: string, lifetime_ms: integer) -> nil`
- `Engine.HUD.UpdateOffset(id: string, offset_x: number, offset_y: number) -> nil`
- `Engine.HUD.UpdateText(id: string, text: string) -> nil`
- `Engine.HUD.UpdateWidth(id: string, width: number) -> nil`
- `Engine.Looter.GetActionType() -> integer`
- `Engine.Looter.GetMinimumCapacity() -> integer`
- `Engine.Looter.GetMode() -> integer`
- `Engine.Looter.LootAroundCharacter() -> boolean`
- `Engine.Looter.SetActionType(value: integer) -> boolean`
- `Engine.Looter.SetMinimumCapacity(value: integer) -> boolean`
- `Engine.Looter.SetMode(value: integer) -> boolean`
- `Engine.Lure.AddSetting(setting: LureSettingInput) -> integer|false`
- `Engine.Lure.ClearSettings() -> boolean`
- `Engine.Lure.EndForceLure() -> boolean`
- `Engine.Lure.GetAttackWhileLuring() -> boolean`
- `Engine.Lure.GetConsiderOnlyReachable() -> boolean`
- `Engine.Lure.GetIgnoringMonsters() -> boolean`
- `Engine.Lure.GetKitingCloseMonsterCount() -> integer`
- `Engine.Lure.GetKitingCloseMonsterDistance() -> integer`
- `Engine.Lure.GetKitingMaximumFarthestDistance() -> integer`
- `Engine.Lure.GetKitingPreferredFarthestDistance() -> integer`
- `Engine.Lure.GetLuredCreaturesCount() -> integer`
- `Engine.Lure.GetNearRange() -> integer`
- `Engine.Lure.GetOption() -> integer`
- `Engine.Lure.GetSettingCount() -> integer`
- `Engine.Lure.GetSettings() -> LureSetting[]`
- `Engine.Lure.GetSlowWalkBurstSteps() -> integer`
- `Engine.Lure.GetSlowWalkDelayMs() -> integer`
- `Engine.Lure.GetSlowWalkingCreaturesCount() -> integer`
- `Engine.Lure.GetStartEndLureActive() -> boolean`
- `Engine.Lure.GetState() -> integer`
- `Engine.Lure.GetUnblocking() -> boolean`
- `Engine.Lure.GetWaypointDynamicLureActive() -> boolean`
- `Engine.Lure.HasActiveSettings() -> boolean`
- `Engine.Lure.IsEnabled() -> boolean`
- `Engine.Lure.IsFighting() -> boolean`
- `Engine.Lure.IsForceLure() -> boolean`
- `Engine.Lure.IsLuring() -> boolean`
- `Engine.Lure.IsOtherPlayerOnScreen() -> boolean`
- `Engine.Lure.RemoveSetting(index: integer) -> boolean`
- `Engine.Lure.SetAttackWhileLuring(enabled: boolean) -> boolean`
- `Engine.Lure.SetConsiderOnlyReachable(enabled: boolean) -> boolean`
- `Engine.Lure.SetEnabled(enabled: boolean) -> nil`
- `Engine.Lure.SetForceLure(enabled: boolean) -> boolean`
- `Engine.Lure.SetIgnoringMonsters(enabled: boolean) -> boolean`
- `Engine.Lure.SetKitingCloseMonsterCount(count: integer) -> boolean`
- `Engine.Lure.SetKitingCloseMonsterDistance(distance: integer) -> boolean`
- `Engine.Lure.SetKitingMaximumFarthestDistance(distance: integer) -> boolean`
- `Engine.Lure.SetKitingPreferredFarthestDistance(distance: integer) -> boolean`
- `Engine.Lure.SetNearRange(range: integer) -> boolean`
- `Engine.Lure.SetOption(option: integer) -> boolean`
- `Engine.Lure.SetSlowWalkBurstSteps(steps: integer) -> boolean`
- `Engine.Lure.SetSlowWalkDelayMs(delayMs: integer) -> boolean`
- `Engine.Lure.SetSlowWalkingCreaturesCount(count: integer) -> boolean`
- `Engine.Lure.SetStartEndLureActive(enabled: boolean) -> boolean`
- `Engine.Lure.SetUnblocking(enabled: boolean) -> boolean`
- `Engine.Lure.SetWaypointDynamicLureActive(enabled: boolean) -> boolean`
- `Engine.Lure.UpdateSetting(index: integer, updateData: LureSettingInput) -> boolean`
- `Engine.MagicShooter.AddProfile(profileName?: string) -> integer|false, string|nil`
- `Engine.MagicShooter.FindProfileByName(profileName: string) -> integer|nil`
- `Engine.MagicShooter.GetActiveProfile() -> ProfileSummary|nil`
- `Engine.MagicShooter.GetCurrentProfile() -> ProfileSummary|nil`
- `Engine.MagicShooter.GetEntries(profile?: integer|string) -> MagicShooterEntry[]|nil, string|nil`
- `Engine.MagicShooter.GetProfileCount() -> integer`
- `Engine.MagicShooter.GetProfileNames() -> string[]`
- `Engine.MagicShooter.GetProfiles() -> FeatureProfile[]`
- `Engine.MagicShooter.NextProfile() -> ProfileSummary|nil`
- `Engine.MagicShooter.RemoveProfile(profile: integer|string) -> boolean`
- `Engine.MagicShooter.RenameProfile(profile: integer|string, newName: string) -> boolean, string|nil`
- `Engine.MagicShooter.SetActiveProfile(profile: integer|string) -> boolean`
- `Engine.MagicShooter.SetCurrentProfile(profile: integer|string) -> boolean`
- `Engine.MagicShooter.SetEntryAttackSkillBuffSpell(entryIndex: integer, value: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryCastMethod(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryChainJumpRange(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryChainMaxTargets(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryChainSelector(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryCondition(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryCustomDelay(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryCustomSpell(entryIndex: integer, value: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryDangerLevel(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryDistanceSkillIncreasePercentage(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryDontCastWhileWalking(entryIndex: integer, value: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryEffectType(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryEnabled(entryIndex: integer, enabled: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryEquipmentRequirement(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryForceUnknownStance(entryIndex: integer, value: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryHarmony(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryHarmonyCondition(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryHealthCondition(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryHealthPercentage(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryHitCountMode(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryManaPercentage(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryMaximumMonsterHealthPercentage(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryMeleeSkillIncreasePercentage(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryMinimumMonsterHealthPercentage(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryMomentumDelay(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryMonsterCount(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryMonsterCountCondition(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryMonsterNames(entryIndex: integer, names: string, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryOption(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryPatternAnchor(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryPatternId(entryIndex: integer, patternId: string, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryPatternSource(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryPatternVariant(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryPrioritizeWithMomentum(entryIndex: integer, value: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryPriorityLane(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryPVPSafe(entryIndex: integer, value: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryRange(entryIndex: integer, range: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryRequiresTarget(entryIndex: integer, value: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryRune(entryIndex: integer, runeId: integer, profile?: integer|string) -> boolean, string|nil`
- `Engine.MagicShooter.SetEntryShootAfterWalkDelay(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryShootOverAllies(entryIndex: integer, value: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntrySpell(entryIndex: integer, spellWords: string, profile?: integer|string) -> boolean, string|nil`
- `Engine.MagicShooter.SetEntryStanceGroup(entryIndex: integer, value: string, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryStanceId(entryIndex: integer, value: string, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryTargetHasToBeInPattern(entryIndex: integer, value: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryTargetPolicy(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryTrackedEffect(entryIndex: integer, value: integer, profile: integer|string|nil) -> boolean`
- `Engine.PVPTools.GetConfig() -> PVPConfig`
- `Engine.PVPTools.IsAntiPushEnabled() -> boolean`
- `Engine.PVPTools.IsHoldTargetEnabled() -> boolean`
- `Engine.PVPTools.ResetLastTarget() -> boolean`
- `Engine.PVPTools.SetAntiPushEnabled(enabled: boolean) -> boolean`
- `Engine.PVPTools.SetAntiPushTrashItem(entryIndex: integer, itemId: integer, quantity: integer) -> boolean`
- `Engine.PVPTools.SetDelayBetweenRuneAndPush(delayMs: integer) -> boolean`
- `Engine.PVPTools.SetHoldTargetEnabled(enabled: boolean) -> boolean`
- `Engine.PVPTools.SetKillTargetEnabled(value: boolean) -> boolean`
- `Engine.PVPTools.SetKillTargetHealthPercentage(value: integer) -> boolean`
- `Engine.PVPTools.SetKillTargetManaCost(value: integer) -> boolean`
- `Engine.PVPTools.SetKillTargetSpellWords(value: string) -> boolean`
- `Engine.PVPTools.SetMagicWallKeeperEnabled(value: boolean) -> boolean`
- `Engine.PVPTools.SetMouseTrashItem(entryIndex: integer, itemId: integer, quantity: integer) -> boolean`
- `Engine.PVPTools.SetPreviousSpotRuneIds(value: integer[]) -> boolean`
- `Engine.PVPTools.SetPreviousSpotWallEnabled(value: boolean) -> boolean`
- `Engine.PVPTools.SetPushAttackedPlayerEnabled(value: boolean) -> boolean`
- `Engine.PVPTools.SetPushmaxDisintegrateRuneId(value: integer) -> boolean`
- `Engine.PVPTools.SetPushmaxEnabled(value: boolean) -> boolean`
- `Engine.PVPTools.SetPushmaxNonDisintegrateRuneId(value: integer) -> boolean`
- `Engine.PVPTools.SetTrashOnMouseEnabled(value: boolean) -> boolean`
- `Engine.PVPTools.SetWallKeeperRuneIds(value: integer[]) -> boolean`
- `Engine.PVPTools.SetWildGrowthKeeperEnabled(value: boolean) -> boolean`
- `Engine.PVPTools.SetWildGrowthKeeperRuneIds(value: integer[]) -> boolean`
- `Engine.PVPTools.ToggleAntiPush() -> boolean`
- `Engine.PVPTools.ToggleHoldTarget() -> boolean`
- `Engine.Scripter.GetAutoStartEnabled() -> boolean`
- `Engine.Scripter.GetAvailableScripts() -> string[]`
- `Engine.Scripter.GetOutput(scriptName: string) -> string`
- `Engine.Scripter.GetRunningScripts() -> string[]`
- `Engine.Scripter.IsRunning(scriptName: string) -> boolean`
- `Engine.Scripter.Refresh() -> boolean`
- `Engine.Scripter.Restart(scriptName: string) -> boolean`
- `Engine.Scripter.SetAutoStartEnabled(value: boolean) -> boolean`
- `Engine.Scripter.Start(scriptName: string) -> boolean`
- `Engine.Scripter.Stop(scriptName: string) -> boolean`
- `Engine.Scripter.StopSelf() -> boolean`
- `Engine.Settings.Load(path: string, features: SettingsFeatureSelection|nil) -> boolean, string`
- `Engine.Settings.Save(path: string, features: SettingsFeatureSelection|nil) -> boolean, string`
- `Engine.SuppliesSorter.AddEntry(destinationContainerId: integer, itemIds: integer[], enabled: boolean|nil) -> integer`
- `Engine.SuppliesSorter.ClearEntries() -> boolean`
- `Engine.SuppliesSorter.GetEntries() -> SuppliesSorterEntry[]`
- `Engine.SuppliesSorter.RemoveEntry(index: integer) -> boolean`
- `Engine.SuppliesSorter.SetEntryDestinationContainerId(entryIndex: integer, value: integer) -> boolean`
- `Engine.SuppliesSorter.SetEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.SuppliesSorter.SetEntryItemIds(entryIndex: integer, value: integer[]) -> boolean`
- `Engine.TankMode.GetCancelManaShieldEnabled() -> boolean`
- `Engine.TankMode.GetCancelManaShieldHealthPercentage() -> integer`
- `Engine.TankMode.GetCancelManaShieldManaCost() -> integer`
- `Engine.TankMode.GetCancelManaShieldManaPercentage() -> integer`
- `Engine.TankMode.GetCancelManaShieldSpellWords() -> string`
- `Engine.TankMode.GetCancelWhileManaShieldReadyEnabled() -> boolean`
- `Engine.TankMode.GetManaShieldEnabled() -> boolean`
- `Engine.TankMode.GetManaShieldHealthPercentage() -> integer`
- `Engine.TankMode.GetManaShieldManaCost() -> integer`
- `Engine.TankMode.GetManaShieldManaPercentage() -> integer`
- `Engine.TankMode.GetManaShieldPotionEnabled() -> boolean`
- `Engine.TankMode.GetManaShieldPotionId() -> integer`
- `Engine.TankMode.GetManaShieldSpellWords() -> string`
- `Engine.TankMode.GetPotionOnSpellCooldownEnabled() -> boolean`
- `Engine.TankMode.GetPotionWhenFearedEnabled() -> boolean`
- `Engine.TankMode.SetCancelManaShieldEnabled(value: boolean) -> boolean`
- `Engine.TankMode.SetCancelManaShieldHealthPercentage(value: integer) -> boolean`
- `Engine.TankMode.SetCancelManaShieldManaCost(value: integer) -> boolean`
- `Engine.TankMode.SetCancelManaShieldManaPercentage(value: integer) -> boolean`
- `Engine.TankMode.SetCancelManaShieldSpellWords(value: string) -> boolean`
- `Engine.TankMode.SetCancelWhileManaShieldReadyEnabled(value: boolean) -> boolean`
- `Engine.TankMode.SetManaShieldEnabled(enabled: boolean) -> boolean`
- `Engine.TankMode.SetManaShieldHealthPercentage(percentage: integer) -> boolean`
- `Engine.TankMode.SetManaShieldManaCost(value: integer) -> boolean`
- `Engine.TankMode.SetManaShieldManaPercentage(value: integer) -> boolean`
- `Engine.TankMode.SetManaShieldPotionEnabled(value: boolean) -> boolean`
- `Engine.TankMode.SetManaShieldPotionId(value: integer) -> boolean`
- `Engine.TankMode.SetManaShieldSpellWords(value: string) -> boolean`
- `Engine.TankMode.SetPotionOnSpellCooldownEnabled(value: boolean) -> boolean`
- `Engine.TankMode.SetPotionWhenFearedEnabled(value: boolean) -> boolean`
- `Engine.Targeting.AddProfile(profileName?: string) -> integer|false, string|nil`
- `Engine.Targeting.FindProfileByName(profileName: string) -> integer|nil`
- `Engine.Targeting.GetActiveProfile() -> ProfileSummary|nil`
- `Engine.Targeting.GetCurrentProfile() -> ProfileSummary|nil`
- `Engine.Targeting.GetEntries(profile: integer|string|nil) -> TargetingEntry[]|nil`
- `Engine.Targeting.GetProfileCount() -> integer`
- `Engine.Targeting.GetProfileNames() -> string[]`
- `Engine.Targeting.GetProfiles() -> FeatureProfile[]`
- `Engine.Targeting.NextProfile() -> ProfileSummary|nil`
- `Engine.Targeting.RemoveProfile(profile: integer|string) -> boolean`
- `Engine.Targeting.RenameProfile(profile: integer|string, newName: string) -> boolean, string|nil`
- `Engine.Targeting.SetActiveProfile(profile: integer|string) -> boolean`
- `Engine.Targeting.SetCurrentProfile(profile: integer|string) -> boolean`
- `Engine.Targeting.SetEntryAnchoring(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryAnchoringRange(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryAttackOption(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryDangerLevel(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Targeting.SetEntryKeepDistanceOption(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryKeepDistanceRange(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryLootMonster(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryMaximumHealthPercentage(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryMinimumHealthPercentage(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryMonsterName(entryIndex: integer, name: string, profile: integer|string|nil) -> boolean`
- `Engine.Targeting.SetEntryMonstersIgnoreList(entryIndex: integer, names: string, profile: integer|string|nil) -> boolean`
- `Engine.Targeting.SetEntryMustBeReachable(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryMustBeShootable(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryPriority(entryIndex: integer, value: integer) -> boolean`
- `Engine.Targeting.SetEntryStayDiagonal(entryIndex: integer, value: integer) -> boolean`
- `Engine.TimerActions.AddEntry(type: integer, spellWords: string, itemId: integer, delay: integer, timeUnit: integer, useInProtectionZone: boolean, enabled: boolean|nil) -> integer`
- `Engine.TimerActions.ClearEntries() -> boolean`
- `Engine.TimerActions.GetEntries() -> TimerActionEntry[]`
- `Engine.TimerActions.RemoveEntry(index: integer) -> boolean`
- `Engine.TimerActions.SetEntryDelay(entryIndex: integer, delay: integer, timeUnit: integer) -> boolean`
- `Engine.TimerActions.SetEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.TimerActions.SetEntryItemId(entryIndex: integer, value: integer) -> boolean`
- `Engine.TimerActions.SetEntrySpellWords(entryIndex: integer, value: string) -> boolean`
- `Engine.TimerActions.SetEntryType(entryIndex: integer, value: integer) -> boolean`
- `Engine.TimerActions.SetEntryUseInProtectionZone(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Walker.AddAutoExploreConnector(connector: WalkerAutoExploreConnectorInput) -> integer|string|false`
- `Engine.Walker.AddSpecialArea(area: WalkerSpecialAreaInput) -> integer|string|false`
- `Engine.Walker.AddWaypoint(waypoint: RawWalkerWaypointInput) -> integer|false`
- `Engine.Walker.AddWaypointAtPosition(waypoint: WalkerWaypointInput) -> integer or false`
- `Engine.Walker.ClearAutoExploreConnectors() -> boolean`
- `Engine.Walker.ClearAutoExploreFloor(z: integer) -> boolean`
- `Engine.Walker.ClearAutoExploreMask() -> boolean`
- `Engine.Walker.ClearSpecialAreas() -> boolean`
- `Engine.Walker.ClearWaypoints() -> boolean`
- `Engine.Walker.CompleteDeferred(token: integer) -> boolean`
- `Engine.Walker.Defer(timeoutMs: integer) -> integer`
- `Engine.Walker.DeleteAutoExploreConnector(id: integer|string) -> boolean`
- `Engine.Walker.DeleteSpecialArea(id: integer|string) -> boolean`
- `Engine.Walker.DeleteWaypoint(index: integer) -> boolean`
- `Engine.Walker.FindAutoExploreConstrainedPath(fromX: integer, fromY: integer, fromZ: integer, toX: integer, toY: integer, toZ: integer, maxComplexity: integer, flags: integer, allowedInitialDirectionsMask: integer) -> integer[] or nil, integer`
- `Engine.Walker.GetActionWaypointKindName(actionKind: integer) -> string`
- `Engine.Walker.GetAntiWallReposition() -> boolean`
- `Engine.Walker.GetAutoExploreConnectorRecording() -> boolean`
- `Engine.Walker.GetAutoExploreConnectors() -> WalkerAutoExploreConnector[]`
- `Engine.Walker.GetAutoExploreMaskRevision() -> integer`
- `Engine.Walker.GetAutoExploreMaskSnapshot() -> WalkerAutoExploreMaskSnapshot`
- `Engine.Walker.GetAutoExploreSettings() -> WalkerAutoExploreSettings`
- `Engine.Walker.GetAutoExploreSpans() -> AutoExploreSpan[]`
- `Engine.Walker.GetAutoExploreStatus() -> WalkerAutoExploreStatus`
- `Engine.Walker.GetAutoRecorderEnabled() -> boolean`
- `Engine.Walker.GetAutoRecorderOptions() -> WalkerAutoRecorderOptions`
- `Engine.Walker.GetAutoScrollWaypoints() -> boolean`
- `Engine.Walker.GetAvoidNearbyCombatDeadEnds() -> boolean`
- `Engine.Walker.GetCombatDeadEndCheckDistance() -> integer`
- `Engine.Walker.GetCombatEgressDirectionGuidance(x: integer, y: integer, z: integer) -> WalkerCombatEgressDirectionGuidance or nil`
- `Engine.Walker.GetCombatEgressProgressReturnDirectionMask(x: integer, y: integer, z: integer) -> integer`
- `Engine.Walker.GetCombatEgressUnsafeDirectionMask(x: integer, y: integer, z: integer) -> integer`
- `Engine.Walker.GetDebugHud() -> boolean`
- `Engine.Walker.GetDistanceBetweenWaypoints() -> integer`
- `Engine.Walker.GetLeaveLureOnPlayer() -> boolean`
- `Engine.Walker.GetLeaveLurePlayerMode() -> integer`
- `Engine.Walker.GetLeaveLurePlayerNames() -> string`
- `Engine.Walker.GetMovementStatus() -> WalkerMovementStatus`
- `Engine.Walker.GetNavigationMode() -> WalkerNavigationMode`
- `Engine.Walker.GetNodeDistance() -> integer`
- `Engine.Walker.GetSelectedWaypoint() -> WalkerWaypoint or nil`
- `Engine.Walker.GetSelectedWaypointIndex() -> integer|nil`
- `Engine.Walker.GetSpecialAreaCount() -> integer`
- `Engine.Walker.GetSpecialAreaRevision() -> integer`
- `Engine.Walker.GetSpecialAreas() -> WalkerSpecialArea[]`
- `Engine.Walker.GetStartFromNearestWaypoint() -> boolean`
- `Engine.Walker.GetSuppliesChecker() -> WalkerSuppliesChecker`
- `Engine.Walker.GetWalkToLureCenter() -> boolean`
- `Engine.Walker.GetWaypointCount() -> integer`
- `Engine.Walker.GetWaypointDirection() -> integer`
- `Engine.Walker.GetWaypointMode() -> integer`
- `Engine.Walker.GetWaypoints() -> WalkerWaypoint[]`
- `Engine.Walker.GetWaypointTypeName(waypointType: integer) -> string`
- `Engine.Walker.GoTo(labelName: string) -> nil`
- `Engine.Walker.HasActiveAutoExploreTransition() -> boolean`
- `Engine.Walker.HasLuaPauseOwnedByScript() -> boolean`
- `Engine.Walker.IncrementWaypointIndex(resetLureRuntimeOnSelectionChange?: boolean) -> boolean`
- `Engine.Walker.InsertWaypoint(index: integer, waypoint: RawWalkerWaypointInput) -> boolean`
- `Engine.Walker.IsAutoExploreMovementAllowed(fromX: integer, fromY: integer, fromZ: integer, toX: integer, toY: integer, toZ: integer, allowOutsideRecovery: boolean) -> boolean`
- `Engine.Walker.IsAutoExplorePositionPainted(x: integer, y: integer, z: integer) -> boolean`
- `Engine.Walker.IsCombatEgressHoldActive(x: integer, y: integer, z: integer) -> boolean`
- `Engine.Walker.IsEnabled() -> boolean`
- `Engine.Walker.IsPausedByLua() -> boolean`
- `Engine.Walker.IsPositionInsideSpecialArea(x: integer, y: integer, z: integer, featureMask: integer) -> boolean`
- `Engine.Walker.IsStuck() -> boolean`
- `Engine.Walker.LabelExists(labelName: string) -> boolean`
- `Engine.Walker.MatchesLeaveLurePlayerName(playerName: string) -> boolean`
- `Engine.Walker.MoveWaypointDown(index?: integer) -> boolean`
- `Engine.Walker.MoveWaypointUp(index?: integer) -> boolean`
- `Engine.Walker.PrepareAutoExploreStart() -> boolean, string or nil`
- `Engine.Walker.ReorderSpecialArea(sourceIndex: integer, targetIndex: integer, dropAfterTarget?: boolean) -> boolean`
- `Engine.Walker.ReorderWaypoint(sourceIndex: integer, targetIndex: integer, dropAfterTarget: boolean) -> boolean`
- `Engine.Walker.ReplaceAutoExploreMask(spans: AutoExploreSpan[]) -> boolean`
- `Engine.Walker.ReplaceWaypoint(index: integer, waypoint: RawWalkerWaypointInput) -> boolean`
- `Engine.Walker.ResetAutoExploreCoverage() -> boolean`
- `Engine.Walker.Resume() -> nil`
- `Engine.Walker.SelectClosestWaypoint() -> boolean`
- `Engine.Walker.SetAntiWallReposition(enabled: boolean) -> boolean`
- `Engine.Walker.SetAutoExploreConnectorRecording(enabled: boolean) -> boolean`
- `Engine.Walker.SetAutoExploreSettings(settings: WalkerAutoExploreSettingsPatch) -> boolean`
- `Engine.Walker.SetAutoExploreSpan(z: integer, y: integer, xBegin: integer, xEnd: integer, painted: boolean) -> boolean`
- `Engine.Walker.SetAutoExploreTile(x: integer, y: integer, z: integer, painted: boolean) -> boolean`
- `Engine.Walker.SetAutoRecorderEnabled(enabled: boolean) -> boolean`
- `Engine.Walker.SetAutoRecorderOptions(options: WalkerAutoRecorderOptionsPatch) -> boolean`
- `Engine.Walker.SetAutoScrollWaypoints(enabled: boolean) -> boolean`
- `Engine.Walker.SetAvoidNearbyCombatDeadEnds(enabled: boolean) -> boolean`
- `Engine.Walker.SetCombatDeadEndCheckDistance(distance: integer) -> boolean`
- `Engine.Walker.SetDebugHud(enabled: boolean) -> boolean`
- `Engine.Walker.SetDistanceBetweenWaypoints(distance: integer) -> boolean`
- `Engine.Walker.SetEnabled(enabled: boolean) -> nil`
- `Engine.Walker.SetLeaveLureOnPlayer(enabled: boolean) -> boolean`
- `Engine.Walker.SetLeaveLurePlayerMode(mode: integer) -> boolean`
- `Engine.Walker.SetLeaveLurePlayerNames(playerNames: string) -> boolean`
- `Engine.Walker.SetNavigationMode(mode: string) -> boolean`
- `Engine.Walker.SetNodeDistance(distance: integer) -> boolean`
- `Engine.Walker.SetPausedByLua(paused: boolean) -> boolean`
- `Engine.Walker.SetSelectedWaypointIndex(index: integer) -> boolean`
- `Engine.Walker.SetStartFromNearestWaypoint(enabled: boolean) -> boolean`
- `Engine.Walker.SetSuppliesChecker(settings: WalkerSuppliesCheckerUpdate) -> boolean`
- `Engine.Walker.SetWalkToLureCenter(enabled: boolean) -> boolean`
- `Engine.Walker.SetWaypointDirection(direction: integer) -> boolean`
- `Engine.Walker.SetWaypointMode(mode: integer) -> boolean`
- `Engine.Walker.SetWaypointPosition(index: integer, x: integer, y: integer, z: integer) -> boolean`
- `Engine.Walker.UpdateAutoExploreConnector(id: integer|string, updateData: WalkerAutoExploreConnectorPatch) -> boolean`
- `Engine.Walker.UpdateSpecialArea(id: integer|string, updateData: WalkerSpecialAreaPatch) -> boolean`
- `Engine.Walker.WaypointTypeUsesPosition(waypointType: integer) -> boolean`
- `Settings.Load(path: string, features: SettingsFeatureSelection|nil) -> boolean, string`
- `Settings.Save(path: string, features: SettingsFeatureSelection|nil) -> boolean, string`

### event_proxies.lua
- `BattleMessageProxy:GetName() -> string`
- `BattleMessageProxy:New(name: string) -> BattleMessageProxy`
- `BattleMessageProxy:OnReceive(callback: function(proxy: BattleMessageProxy, message: string)) -> BattleMessageProxy`
- `ClientAttackPacketProxy:New(name: string) -> ClientAttackPacketProxy`
- `ClientEquipItemPacketProxy:New(name: string) -> ClientEquipItemPacketProxy`
- `ClientLookPacketProxy:New(name: string) -> ClientLookPacketProxy`
- `ClientMoveItemPacketProxy:New(name: string) -> ClientMoveItemPacketProxy`
- `ClientMovementPacketProxy:New(name: string) -> ClientMovementPacketProxy`
- `ClientTalkPacketProxy:New(name: string) -> ClientTalkPacketProxy`
- `ClientUseItemPacketProxy:New(name: string) -> ClientUseItemPacketProxy`
- `ClientUseItemWithPacketProxy:New(name: string) -> ClientUseItemWithPacketProxy`
- `ClientUseOnCreaturePacketProxy:New(name: string) -> ClientUseOnCreaturePacketProxy`
- `ContainerAddItemProxy:GetName() -> string`
- `ContainerAddItemProxy:New(name: string) -> ContainerAddItemProxy`
- `ContainerAddItemProxy:OnReceive(callback: function(proxy: ContainerAddItemProxy, containerIndex: nil, slot: nil, item: nil)) -> ContainerAddItemProxy`
- `ContainerCloseProxy:GetName() -> string`
- `ContainerCloseProxy:New(name: string) -> ContainerCloseProxy`
- `ContainerCloseProxy:OnReceive(callback: function(proxy: ContainerCloseProxy, containerIndex: nil)) -> ContainerCloseProxy`
- `ContainerOpenProxy:GetName() -> string`
- `ContainerOpenProxy:New(name: string) -> ContainerOpenProxy`
- `ContainerOpenProxy:OnReceive(callback: function(proxy: ContainerOpenProxy, containerIndex: nil, containerName: nil, containerID: nil)) -> ContainerOpenProxy`
- `ContainerRemoveItemProxy:GetName() -> string`
- `ContainerRemoveItemProxy:New(name: string) -> ContainerRemoveItemProxy`
- `ContainerRemoveItemProxy:OnReceive(callback: function(proxy: ContainerRemoveItemProxy, containerIndex: nil, slot: nil)) -> ContainerRemoveItemProxy`
- `ContainerUpdateItemProxy:GetName() -> string`
- `ContainerUpdateItemProxy:New(name: string) -> ContainerUpdateItemProxy`
- `ContainerUpdateItemProxy:OnReceive(callback: function(proxy: ContainerUpdateItemProxy, containerIndex: nil, slot: nil, item: nil)) -> ContainerUpdateItemProxy`
- `CreateOnMapPacketProxy:New(name: string) -> CreateOnMapPacketProxy`
- `CreatureAddProxy:GetName() -> string`
- `CreatureAddProxy:New(name: string) -> CreatureAddProxy`
- `CreatureAddProxy:OnReceive(callback: function(proxy: CreatureAddProxy, creatureId: integer, creatureName: string, position: Position)) -> CreatureAddProxy`
- `CreatureRemoveProxy:GetName() -> string`
- `CreatureRemoveProxy:New(name: string) -> CreatureRemoveProxy`
- `CreatureRemoveProxy:OnReceive(callback: function(proxy: CreatureRemoveProxy, creatureId: nil, position: Position, stackPosition: integer, packet: IncomingDeleteOnMapPacket)) -> CreatureRemoveProxy`
- `DeathProxy:GetName() -> string`
- `DeathProxy:New(name: string) -> DeathProxy`
- `DeathProxy:OnReceive(callback: function(proxy: DeathProxy)) -> DeathProxy`
- `DeleteOnMapPacketProxy:New(name: string) -> DeleteOnMapPacketProxy`
- `FullMapPacketProxy:New(name: string) -> FullMapPacketProxy`
- `GenericTextMessageProxy:GetName() -> string`
- `GenericTextMessageProxy:New(name: string) -> GenericTextMessageProxy`
- `GenericTextMessageProxy:OnReceive(callback: function(proxy: GenericTextMessageProxy, message: string)) -> GenericTextMessageProxy`
- `GraphicalEffectPacketProxy:New(name: string) -> GraphicalEffectPacketProxy`
- `IncomingPacketProxy:New(name: string, packetId: integer|integer[]) -> IncomingPacketProxy`
- `LootMessageProxy:GetName() -> string`
- `LootMessageProxy:New(name: string) -> LootMessageProxy`
- `LootMessageProxy:OnReceive(callback: function(proxy: LootMessageProxy, message: string)) -> LootMessageProxy`
- `MapRowPacketProxy:New(name: string) -> MapRowPacketProxy`
- `MoveCreaturePacketProxy:New(name: string) -> MoveCreaturePacketProxy`
- `OutgoingPacketProxy:New(name: string, packetId: integer|integer[]) -> OutgoingPacketProxy`
- `PacketEventProxy:GetName() -> string`
- `PacketEventProxy:GetRegistrationId() -> string|nil`
- `PacketEventProxy:IsIncoming() -> boolean`
- `PacketEventProxy:New(nameOrOptions: string|PacketEventProxyOptions, packetId?: integer|integer[], incoming?: boolean) -> PacketEventProxy`
- `PacketEventProxy:OnPacket(callback: function(proxy: PacketEventProxy, packet: PacketEventPayload)) -> PacketEventProxy`
- `PacketEventProxy:OnReceive(callback: function(proxy: PacketEventProxy, packet: IncomingPacketPayload)) -> PacketEventProxy`
- `PacketEventProxy:OnSend(callback: function(proxy: PacketEventProxy, packet: OutgoingPacketPayload)) -> PacketEventProxy`
- `PacketEventProxy:Unregister() -> boolean`
- `PlayerInventoryPacketProxy:New(name: string) -> PlayerInventoryPacketProxy`
- `SkillsChangeProxy:GetName() -> string`
- `SkillsChangeProxy:New(name: string) -> SkillsChangeProxy`
- `SkillsChangeProxy:OnReceive(callback: function(proxy: SkillsChangeProxy, packet: IncomingOpcodeOnlyPacket)) -> SkillsChangeProxy`
- `StatsChangeProxy:GetName() -> string`
- `StatsChangeProxy:New(name: string) -> StatsChangeProxy`
- `StatsChangeProxy:OnReceive(callback: function(proxy: StatsChangeProxy, packet: IncomingOpcodeOnlyPacket)) -> StatsChangeProxy`
- `TalkPacketProxy:New(name: string) -> TalkPacketProxy`
- `TextMessagePacketProxy:New(name: string) -> TextMessagePacketProxy`
- `UnjustifiedPointsPacketProxy:New(name: string) -> UnjustifiedPointsPacketProxy`

### features.lua
- `BotFeatureId.ALARMS = 8`
- `BotFeatureId.AMMO_REFILL = 16`
- `BotFeatureId.CHANNELS_MANAGER = 11`
- `BotFeatureId.COMBO_BOT = 14`
- `BotFeatureId.CONDITIONS_MANAGER = 2`
- `BotFeatureId.EQUIPMENT_MANAGER = 10`
- `BotFeatureId.EXTRAS = 9`
- `BotFeatureId.HEAL_FRIEND = 3`
- `BotFeatureId.HEALER = 1`
- `BotFeatureId.HUD = 15`
- `BotFeatureId.LOOTER = 13`
- `BotFeatureId.LURE_MANAGER = 4`
- `BotFeatureId.MAGIC_SHOOTER = 7`
- `BotFeatureId.PVP_TOOLS = 12`
- `BotFeatureId.SUPPLIES_SORTER = 19`
- `BotFeatureId.TANK_MODE = 17`
- `BotFeatureId.TARGETING = 6`
- `BotFeatureId.TIMER_ACTIONS = 18`
- `BotFeatureId.WALKER = 5`
- `Features.Disable(featureIdentifier: integer|string) -> boolean`
- `Features.DisableAllExcept(ExcludeList: FeatureIdentifier[]) -> nil`
- `Features.DisableMultiple(featureList: FeatureIdentifier[]) -> nil`
- `Features.Enable(featureIdentifier: integer|string) -> boolean`
- `Features.EnableMultiple(featureList: FeatureIdentifier[]) -> nil`
- `Features.GetActiveFeatures() -> integer[]`
- `Features.GetAllFeatureIds() -> integer[]`
- `Features.GetName(featureIdentifier: FeatureIdentifier) -> string`
- `Features.IsActive(featureIdentifier: FeatureIdentifier) -> boolean`
- `Features.PrintStatus() -> nil`
- `Features.SetActive(featureIdentifier: integer|string, activeStatus: boolean) -> boolean`
- `Features.Toggle(featureIdentifier: integer|string) -> boolean`

### game.lua
- `Game.EnterWorld() -> boolean`
- `Game.GetCharacterWorld(characterName: string) -> string`
- `Game.LoginToAccount(email: string, password: string) -> boolean`
- `Game.LoginToCharacter(characterName: string) -> boolean`
- `Game.LoginToPreviouslyLoggedCharacter() -> boolean`
- `Game.Logout() -> boolean`
- `Game.OpenContainerInNewWindow(equipmentSlotOrContainerId: integer, fromContainerNumber?: integer, fromContainerSlot?: integer) -> boolean`
- `Game.OpenStore() -> boolean`

### hotkeys.lua
- `Hotkeys.ParseCombo(combination: string) -> ParsedHotkey|nil, string|nil`
- `Hotkeys.RegisterCombo(params: HotkeyRegistrationOptions) -> boolean, string`
- `Hotkeys.SendCombo(combination: string, clientOnly?: boolean) -> boolean`
- `Hotkeys.SendKey(key: string|integer, clientOnly?: boolean) -> boolean`

### http.lua
- `Http.Get(url: string, options?: HttpConvenienceOptions) -> HttpResponse`
- `Http.GetJson(url: string, options?: HttpConvenienceOptions) -> JsonValue|nil, HttpResponse, string|nil`
- `Http.Post(url: string, body?: string, options?: HttpConvenienceOptions) -> HttpResponse`
- `Http.PostJson(url: string, value: JsonValue, options?: HttpConvenienceOptions) -> HttpResponse`
- `Http.Request(options: HttpRequestOptions) -> HttpResponse`

### hud_wrapper.lua
- `LoadOutfitById(outfitId: integer, options?: table<string,integer|string|integer[]|table<string,integer>>) -> table<string,integer|string>`
- `LoadOutfitIcon(monsterName: string, options?: table<string,integer|string|integer[]|table<string,integer>>) -> table<string,integer|string>`
- `ScreenImage:ClearParent() -> ScreenImage`
- `ScreenImage:Create() -> ScreenImage`
- `ScreenImage:GetEnabled() -> boolean`
- `ScreenImage:GetHeight() -> number`
- `ScreenImage:GetPosition() -> HudScreenPosition`
- `ScreenImage:GetVisible() -> boolean`
- `ScreenImage:GetWidth() -> number`
- `ScreenImage:IsCreated() -> boolean`
- `ScreenImage:New(id: string, renderLayer?: string) -> ScreenImage`
- `ScreenImage:Remove() -> nil`
- `ScreenImage:SetAlignment(h_align: integer, v_align: integer) -> ScreenImage`
- `ScreenImage:SetClickable(callback?: function()|false) -> ScreenImage`
- `ScreenImage:SetDraggable(draggable: boolean) -> ScreenImage`
- `ScreenImage:SetDragTarget(target: ScreenText|ScreenImage|string|nil) -> ScreenImage`
- `ScreenImage:SetEnabled(enabled: boolean) -> ScreenImage`
- `ScreenImage:SetItemId(itemId: integer) -> ScreenImage`
- `ScreenImage:SetItemName(itemName: string) -> ScreenImage`
- `ScreenImage:SetLabel(text: string|nil, color?: ColorRGBA, offsetX?: number, offsetY?: number) -> ScreenImage`
- `ScreenImage:SetOnDragEnd(callback?: function(x: number, y: number)|false) -> ScreenImage`
- `ScreenImage:SetOnMouseDown(callback?: function()|false) -> ScreenImage`
- `ScreenImage:SetOnMouseLeave(callback?: function()|false) -> ScreenImage`
- `ScreenImage:SetOnMouseUp(callback?: function()|false) -> ScreenImage`
- `ScreenImage:SetOutfitById(outfitId: integer, options?: table<string,integer|string|integer[]|table<string,integer>>) -> ScreenImage`
- `ScreenImage:SetOutfitIcon(monsterName: string, options?: table<string,integer|string|integer[]|table<string,integer>>) -> ScreenImage`
- `ScreenImage:SetParent(parent: ScreenText|ScreenImage|string) -> ScreenImage`
- `ScreenImage:SetPointerCallbacks(onMouseDown?: function()|false, onMouseUp?: function()|false, onMouseLeave?: function()|false) -> ScreenImage`
- `ScreenImage:SetRenderLayer(renderLayer: string) -> ScreenImage`
- `ScreenImage:SetScreenPosition(x: number, y: number) -> ScreenImage`
- `ScreenImage:SetSize(width: number, height: number) -> ScreenImage`
- `ScreenImage:SetSource(path: string) -> ScreenImage`
- `ScreenImage:SetSourceBase64(base64Image: string) -> ScreenImage`
- `ScreenImage:SetSourceBytes(imageBytes: HudImageBytes) -> ScreenImage`
- `ScreenImage:SetZIndex(zIndex: integer) -> ScreenImage`
- `ScreenText:ClearParent() -> ScreenText`
- `ScreenText:Create() -> ScreenText`
- `ScreenText:GetColor() -> ColorRGBA`
- `ScreenText:GetEnabled() -> boolean`
- `ScreenText:GetHeight() -> number`
- `ScreenText:GetPosition() -> HudScreenPosition`
- `ScreenText:GetText() -> string`
- `ScreenText:GetVisible() -> boolean`
- `ScreenText:GetWidth() -> number`
- `ScreenText:IsCreated() -> boolean`
- `ScreenText:New(id: string, renderLayer?: string) -> ScreenText`
- `ScreenText:Remove() -> nil`
- `ScreenText:SetAlignment(h_align: integer, v_align: integer) -> ScreenText`
- `ScreenText:SetClickable(callback?: function()|false) -> ScreenText`
- `ScreenText:SetColor(color: ColorRGBA) -> ScreenText`
- `ScreenText:SetDraggable(draggable: boolean) -> ScreenText`
- `ScreenText:SetDragTarget(target: ScreenText|ScreenImage|string|nil) -> ScreenText`
- `ScreenText:SetEnabled(enabled: boolean) -> ScreenText`
- `ScreenText:SetFont(family: string|nil, pixelSize: integer|nil) -> ScreenText`
- `ScreenText:SetFontFamily(family: string|nil) -> ScreenText`
- `ScreenText:SetFontSize(pixelSize: integer|nil) -> ScreenText`
- `ScreenText:SetOnDragEnd(callback?: function(x: number, y: number)|false) -> ScreenText`
- `ScreenText:SetOnMouseDown(callback?: function()|false) -> ScreenText`
- `ScreenText:SetOnMouseLeave(callback?: function()|false) -> ScreenText`
- `ScreenText:SetOnMouseUp(callback?: function()|false) -> ScreenText`
- `ScreenText:SetParent(parent: ScreenText|ScreenImage|string) -> ScreenText`
- `ScreenText:SetPointerCallbacks(onMouseDown?: function()|false, onMouseUp?: function()|false, onMouseLeave?: function()|false) -> ScreenText`
- `ScreenText:SetRenderLayer(renderLayer: string) -> ScreenText`
- `ScreenText:SetScreenPosition(x: number, y: number) -> ScreenText`
- `ScreenText:SetText(text: string) -> ScreenText`
- `ScreenText:SetZIndex(zIndex: integer) -> ScreenText`
- `WorldBox:ClearParent() -> WorldBox`
- `WorldBox:Create() -> WorldBox`
- `WorldBox:GetColor() -> ColorRGBA`
- `WorldBox:GetEnabled() -> boolean`
- `WorldBox:GetHeight() -> number`
- `WorldBox:GetPosition() -> Position`
- `WorldBox:GetVisible() -> boolean`
- `WorldBox:GetWidth() -> number`
- `WorldBox:IsCreated() -> boolean`
- `WorldBox:New(id: string, x: integer, y: integer, z: integer, renderLayer?: string) -> WorldBox`
- `WorldBox:Remove() -> nil`
- `WorldBox:SetBorderColor(border_color: ColorRGBA) -> WorldBox`
- `WorldBox:SetBorderWidth(border_width: number) -> WorldBox`
- `WorldBox:SetColor(color: ColorRGBA) -> WorldBox`
- `WorldBox:SetEnabled(enabled: boolean) -> WorldBox`
- `WorldBox:SetHeight(height: number) -> WorldBox`
- `WorldBox:SetLifetime(lifetime_ms: integer) -> WorldBox`
- `WorldBox:SetParent(parent_id: string) -> WorldBox`
- `WorldBox:SetPosition(x: integer, y: integer, z: integer) -> WorldBox`
- `WorldBox:SetRenderLayer(renderLayer: string) -> WorldBox`
- `WorldBox:SetSize(width: number, height: number) -> WorldBox`
- `WorldBox:SetWidth(width: number) -> WorldBox`
- `WorldBox:SetZIndex(zIndex: integer) -> WorldBox`
- `WorldImage:ClearParent() -> WorldImage`
- `WorldImage:Create() -> WorldImage`
- `WorldImage:GetEnabled() -> boolean`
- `WorldImage:GetHeight() -> number`
- `WorldImage:GetPosition() -> Position`
- `WorldImage:GetVisible() -> boolean`
- `WorldImage:GetWidth() -> number`
- `WorldImage:IsCreated() -> boolean`
- `WorldImage:New(id: string, x: integer, y: integer, z: integer, renderLayer?: string) -> WorldImage`
- `WorldImage:Remove() -> nil`
- `WorldImage:SetEnabled(enabled: boolean) -> WorldImage`
- `WorldImage:SetItemId(itemId: integer) -> WorldImage`
- `WorldImage:SetItemName(itemName: string) -> WorldImage`
- `WorldImage:SetLabel(text: string|nil, color?: ColorRGBA, offsetX?: number, offsetY?: number) -> WorldImage`
- `WorldImage:SetLifetime(lifetimeMs: integer) -> WorldImage`
- `WorldImage:SetOffset(offsetX: number, offsetY: number) -> WorldImage`
- `WorldImage:SetOutfitById(outfitId: integer, options?: table<string,integer|string|integer[]|table<string,integer>>) -> WorldImage`
- `WorldImage:SetOutfitIcon(monsterName: string, options?: table<string,integer|string|integer[]|table<string,integer>>) -> WorldImage`
- `WorldImage:SetParent(parent_id: string) -> WorldImage`
- `WorldImage:SetPosition(x: integer, y: integer, z: integer) -> WorldImage`
- `WorldImage:SetRenderLayer(renderLayer: string) -> WorldImage`
- `WorldImage:SetSize(width: number, height: number) -> WorldImage`
- `WorldImage:SetSource(path: string) -> WorldImage`
- `WorldImage:SetSourceBase64(base64Image: string) -> WorldImage`
- `WorldImage:SetSourceBytes(imageBytes: HudImageBytes) -> WorldImage`
- `WorldImage:SetZIndex(zIndex: integer) -> WorldImage`
- `WorldText:ClearParent() -> WorldText`
- `WorldText:Create() -> WorldText`
- `WorldText:GetColor() -> ColorRGBA`
- `WorldText:GetEnabled() -> boolean`
- `WorldText:GetHeight() -> number`
- `WorldText:GetPosition() -> Position`
- `WorldText:GetText() -> string`
- `WorldText:GetVisible() -> boolean`
- `WorldText:GetWidth() -> number`
- `WorldText:IsCreated() -> boolean`
- `WorldText:New(id: string, x: integer, y: integer, z: integer, renderLayer?: string) -> WorldText`
- `WorldText:Remove() -> nil`
- `WorldText:SetColor(color: ColorRGBA) -> WorldText`
- `WorldText:SetEnabled(enabled: boolean) -> WorldText`
- `WorldText:SetFont(family: string|nil, pixelSize: integer|nil) -> WorldText`
- `WorldText:SetFontFamily(family: string|nil) -> WorldText`
- `WorldText:SetFontSize(pixelSize: integer|nil) -> WorldText`
- `WorldText:SetLifetime(lifetime_ms: integer) -> WorldText`
- `WorldText:SetOffset(offset_x: number, offset_y: number) -> WorldText`
- `WorldText:SetParent(parent_id: string) -> WorldText`
- `WorldText:SetPosition(x: integer, y: integer, z: integer) -> WorldText`
- `WorldText:SetRenderLayer(renderLayer: string) -> WorldText`
- `WorldText:SetText(text: string) -> WorldText`
- `WorldText:SetZIndex(zIndex: integer) -> WorldText`

### inventory.lua
- `Inventory.CanMoveEquipment() -> boolean`
- `Inventory.CanReadEquipment() -> boolean`
- `Inventory.Equip(itemId: integer, tierLevel?: integer) -> boolean`
- `Inventory.GetAllSlotItems() -> table<integer, EquipmentItem>`
- `Inventory.GetEquipmentSlotConstants() -> EquipmentSlotConstants`
- `Inventory.GetSlotIds() -> integer[]`
- `Inventory.GetSlotItem(equipmentSlot: integer) -> EquipmentItem|nil`
- `Inventory.GetSlotItemId(equipmentSlot: integer) -> integer|nil`
- `Inventory.GetSnapshot() -> InventorySnapshot`
- `Inventory.HasItemInSlot(equipmentSlot: integer) -> boolean|nil`
- `Inventory.LookSlotItem(itemId: integer, equipmentSlot: integer) -> boolean`
- `Inventory.MoveFromContainerToSlot(containerIndex: integer, slotIndex: integer, itemId: integer, equipmentSlot: integer, itemCount: integer) -> boolean`
- `Inventory.MoveFromSlotToContainer(equipmentSlot: integer, containerIndex: integer, slotIndex: integer, itemId: integer, itemCount: integer) -> boolean`

### item.lua
- `Item.Buy(itemId: integer, itemCount: integer, ignoreCapacity?: boolean, buyInShoppingBags?: boolean) -> boolean`
- `Item.FindInContainer(containerNumber: integer, itemId: integer, tierLevel?: integer) -> ContainerFindResult|nil`
- `Item.GetDescription(itemId: integer) -> string|nil`
- `Item.GetFromContainer(containerNumber: integer, slotIndex: integer) -> ContainerItem|nil`
- `Item.GetInfo(itemId: integer) -> ObjectInfo|nil`
- `Item.GetName(itemId: integer) -> string|nil`
- `Item.HasFlag(itemId: integer, fieldName: string) -> boolean`
- `Item.IsContainer(itemId: integer) -> boolean`
- `Item.IsCreature(itemId: integer) -> boolean`
- `Item.IsCumulative(itemId: integer) -> boolean`
- `Item.IsGround(itemId: integer) -> boolean`
- `Item.IsLiquidContainer(itemId: integer) -> boolean`
- `Item.IsMovable(itemId: integer) -> boolean`
- `Item.IsMultiUsable(itemId: integer) -> boolean`
- `Item.IsTakable(itemId: integer) -> boolean`
- `Item.IsUsable(itemId: integer) -> boolean`
- `Item.Sell(itemId: integer, itemCount: integer, sellEquipped?: boolean) -> boolean`
- `Item.Use(itemId: integer) -> boolean`
- `Item.UseFromContainerOnFloor(floorPosition: PositionLike, fromItemId: integer, toItemId: integer, toStackPosition: integer) -> boolean`
- `Item.UseFromContainerToContainer(fromContainer: integer, fromSlot: integer, fromItemId: integer, toContainer: integer, toSlot: integer, toItemId: integer) -> boolean`
- `Item.UseFromFloorToContainer(floorPosition: PositionLike, fromItemId: integer, fromStackPosition: integer, toItemId: integer) -> boolean`
- `Item.UseOnCreature(itemId: integer, creatureId: integer) -> boolean`
- `Item.UseOnSelf(itemId: integer) -> boolean`

### json.lua
- Constant: `Json.Null` (JSON null sentinel)
- `Json.Array(value: JsonValue[]) -> JsonValue[]`
- `Json.Decode(text: string) -> JsonValue`
- `Json.Encode(value: JsonValue, pretty?: boolean|integer) -> string`
- `Json.Object(value: table<string, JsonValue>) -> table<string, JsonValue>`
- `Json.TryDecode(text: string) -> JsonValue|nil, string|nil`
- `Json.TryEncode(value: JsonValue, pretty?: boolean|integer) -> string|nil, string|nil`

### lua_consts.lua
- `BestiaryRace.AMPHIBIC = 1`
- `BestiaryRace.AQUATIC = 2`
- `BestiaryRace.BIRD = 3`
- `BestiaryRace.CONSTRUCT = 4`
- `BestiaryRace.DEMON = 5`
- `BestiaryRace.DRAGON = 6`
- `BestiaryRace.ELEMENTAL = 7`
- `BestiaryRace.EXTRA_DIMENSIONAL = 20`
- `BestiaryRace.FEY = 8`
- `BestiaryRace.GIANT = 9`
- `BestiaryRace.HUMAN = 10`
- `BestiaryRace.HUMANOID = 11`
- `BestiaryRace.INKBORN = 21`
- `BestiaryRace.LYCANTHROPE = 12`
- `BestiaryRace.MAGICAL = 13`
- `BestiaryRace.MAMMAL = 14`
- `BestiaryRace.NONE = 0`
- `BestiaryRace.PLANT = 15`
- `BestiaryRace.REPTILE = 16`
- `BestiaryRace.SLIME = 17`
- `BestiaryRace.UNDEAD = 18`
- `BestiaryRace.VERMIN = 19`
- `CharacterFlag.AGONY = 27`
- `CharacterFlag.BLEEDING = 15`
- `CharacterFlag.BURNING = 1`
- `CharacterFlag.COUNT = 30`
- `CharacterFlag.CURSED = 11`
- `CharacterFlag.DAZZLED = 10`
- `CharacterFlag.DROWNING = 8`
- `CharacterFlag.DRUNK = 3`
- `CharacterFlag.ELECTRIFIED = 2`
- `CharacterFlag.FEARED = 20`
- `CharacterFlag.FREEZING = 9`
- `CharacterFlag.GOSHNAR_TAINT_1 = 21`
- `CharacterFlag.GOSHNAR_TAINT_2 = 22`
- `CharacterFlag.GOSHNAR_TAINT_3 = 23`
- `CharacterFlag.GOSHNAR_TAINT_4 = 24`
- `CharacterFlag.GOSHNAR_TAINT_5 = 25`
- `CharacterFlag.GREATER_HEX = 18`
- `CharacterFlag.HASTED = 6`
- `CharacterFlag.IN_COMBAT = 7`
- `CharacterFlag.IN_PROTECTION_ZONE = 14`
- `CharacterFlag.INTENSE_HEX = 17`
- `CharacterFlag.LESSER_HEX = 16`
- `CharacterFlag.MANA_SHIELDED = 4`
- `CharacterFlag.MENTOR_OTHER = 29`
- `CharacterFlag.NEW_MANA_SHIELD = 26`
- `CharacterFlag.PARALYSED = 5`
- `CharacterFlag.POISONED = 0`
- `CharacterFlag.POWERLESS = 28`
- `CharacterFlag.PZ_BLOCK = 13`
- `CharacterFlag.ROOTED = 19`
- `CharacterFlag.STRENGTHENED = 12`
- `CharmRune.ADRENALINE_BURST = 9`
- `CharmRune.BLESS = 12`
- `CharmRune.CARNAGE = 22`
- `CharmRune.CLEANSE = 11`
- `CharmRune.CRIPPLE = 6`
- `CharmRune.CURSE = 5`
- `CharmRune.DIVINE_WRATH = 16`
- `CharmRune.DODGE = 8`
- `CharmRune.ENFLAME = 1`
- `CharmRune.FATAL_HOLD = 20`
- `CharmRune.FREEZE = 3`
- `CharmRune.GUT = 14`
- `CharmRune.LOW_BLOW = 15`
- `CharmRune.NONE = -1`
- `CharmRune.NUMB = 10`
- `CharmRune.OVERFLUX = 24`
- `CharmRune.OVERPOWER = 23`
- `CharmRune.PARRY = 7`
- `CharmRune.POISON = 2`
- `CharmRune.SAVAGE_BLOW = 19`
- `CharmRune.SCAVENGE = 13`
- `CharmRune.VAMPIRIC_EMBRACE = 17`
- `CharmRune.VOID_CALL = 18`
- `CharmRune.VOID_INVERSION = 21`
- `CharmRune.WOUND = 0`
- `CharmRune.ZAP = 4`
- `ChaseMode.CHASE = 1`
- `ChaseMode.STAND = 0`
- `ChaseMode.UNKNOWN = 2`
- `CooldownGroupId.ATTACK = 1`
- `CooldownGroupId.BURST_OF_NATURE = 10`
- `CooldownGroupId.CRIPPLING = 5`
- `CooldownGroupId.FOCUS = 7`
- `CooldownGroupId.GREAT_BEAMS = 9`
- `CooldownGroupId.HEALING = 2`
- `CooldownGroupId.SPECIAL = 4`
- `CooldownGroupId.SUPPORT = 3`
- `CooldownGroupId.ULTIMATE = 8`
- `CooldownGroupId.VIRTUE = 11`
- `CreatureIcon.FIENDISH = 5`
- `CreatureIcon.INFLUENCED = 4`
- `CreatureIcon.LOWER_DAMAGE = 2`
- `CreatureIcon.NONE = 0`
- `CreatureIcon.REDUCED_HEALTH = 6`
- `CreatureIcon.TURNED_MELEE = 3`
- `CreatureIcon.WEAKENED = 1`
- `CreatureType.CREATURETYPE_HIDDEN = 5`
- `CreatureType.CREATURETYPE_MONSTER = 1`
- `CreatureType.CREATURETYPE_NPC = 2`
- `CreatureType.CREATURETYPE_PLAYER = 0`
- `CreatureType.CREATURETYPE_SUMMON_OTHERS = 4`
- `CreatureType.CREATURETYPE_SUMMON_OWN = 3`
- `CreatureType.HIDDEN = 5`
- `CreatureType.MONSTER = 1`
- `CreatureType.NPC = 2`
- `CreatureType.PLAYER = 0`
- `CreatureType.SUMMON_OTHERS = 4`
- `CreatureType.SUMMON_OWN = 3`
- `EquipmentSlot.AMULET = 2`
- `EquipmentSlot.ARMOR = 4`
- `EquipmentSlot.ARROW = 10`
- `EquipmentSlot.BACKPACK = 3`
- `EquipmentSlot.BOOTS = 8`
- `EquipmentSlot.HELMET = 1`
- `EquipmentSlot.LEFT_HAND = 6`
- `EquipmentSlot.LEGS = 7`
- `EquipmentSlot.NONE = 0`
- `EquipmentSlot.RIGHT_HAND = 5`
- `EquipmentSlot.RING = 9`
- `EquipmentSlot.STORE = 11`
- `FightMode.BALANCED = 2`
- `FightMode.DEFENSIVE = 3`
- `FightMode.OFFENSIVE = 1`
- `FightMode.UNKNOWN = 0`
- `ImbuementType.CAPACITY = 17`
- `ImbuementType.CRITICAL_HIT = 3`
- `ImbuementType.DEATH_PROTECTION = 4`
- `ImbuementType.EARTH_PROTECTION = 5`
- `ImbuementType.ELEMENTAL_DAMAGE = 0`
- `ImbuementType.ENERGY_PROTECTION = 8`
- `ImbuementType.FIRE_PROTECTION = 6`
- `ImbuementType.HOLY_PROTECTION = 9`
- `ImbuementType.ICE_PROTECTION = 7`
- `ImbuementType.LIFE_LEECH = 1`
- `ImbuementType.MAGIC_LEVEL = 16`
- `ImbuementType.MANA_LEECH = 2`
- `ImbuementType.NONE = -1`
- `ImbuementType.PARALYSIS_DEFLECTION = 19`
- `ImbuementType.SKILL_AXE = 11`
- `ImbuementType.SKILL_CLUB = 13`
- `ImbuementType.SKILL_DISTANCE = 15`
- `ImbuementType.SKILL_FIST = 18`
- `ImbuementType.SKILL_SHIELDING = 14`
- `ImbuementType.SKILL_SWORD = 12`
- `ImbuementType.SPEED = 10`
- `MagicEffectsType.CREATE_DISTANCE_EFFECT = 4`
- `MagicEffectsType.CREATE_DISTANCE_EFFECT_REVERSED = 5`
- `MagicEffectsType.CREATE_EFFECT = 3`
- `MagicEffectsType.CREATE_SOUND_MAIN_EFFECT = 6`
- `MagicEffectsType.CREATE_SOUND_SECONDARY_EFFECT = 7`
- `MagicEffectsType.DELAY = 2`
- `MagicEffectsType.DELTA = 1`
- `MagicEffectsType.END_LOOP = 0`
- `MessageClasses.DAMAGE_DEALED = 21`
- `MessageClasses.DAMAGE_OTHERS = 25`
- `MessageClasses.DAMAGE_RECEIVED = 22`
- `MessageClasses.EXP = 24`
- `MessageClasses.EXP_OTHERS = 27`
- `MessageClasses.FAILURE = 19`
- `MessageClasses.GAME = 18`
- `MessageClasses.GAME_HIGHLIGHT = 50`
- `MessageClasses.GAME_MASTER_CONSOLE = 13`
- `MessageClasses.GUILD = 31`
- `MessageClasses.HEAL_OTHERS = 26`
- `MessageClasses.HEALED = 23`
- `MessageClasses.HOTKEY_USE = 37`
- `MessageClasses.LOGIN = 17`
- `MessageClasses.LOOK = 20`
- `MessageClasses.LOOT = 29`
- `MessageClasses.MANA = 41`
- `MessageClasses.MONSTER_SAY = 44`
- `MessageClasses.MONSTER_YELL = 43`
- `MessageClasses.NONE = 0`
- `MessageClasses.PARTY = 33`
- `MessageClasses.PARTY_MANAGEMENT = 32`
- `MessageClasses.REPORT = 36`
- `MessageClasses.STATUS = 28`
- `MessageClasses.STATUS_WARNING = 9`
- `MessageClasses.TRADE_NPC = 30`
- `MessageMode.BARK_LOUD = 35`
- `MessageMode.BARK_LOW = 34`
- `MessageMode.BEYOND_LAST = 42`
- `MessageMode.BLUE = 46`
- `MessageMode.CHANNEL = 7`
- `MessageMode.CHANNEL_HIGHLIGHT = 8`
- `MessageMode.CHANNEL_MANAGEMENT = 6`
- `MessageMode.DAMAGE_DEALED = 21`
- `MessageMode.DAMAGE_OTHERS = 25`
- `MessageMode.DAMAGE_RECEIVED = 22`
- `MessageMode.EXP = 24`
- `MessageMode.EXP_OTHERS = 27`
- `MessageMode.FAILURE = 19`
- `MessageMode.GAME = 18`
- `MessageMode.GAME_HIGHLIGHT = 50`
- `MessageMode.GAMEMASTER_BROADCAST = 12`
- `MessageMode.GAMEMASTER_CHANNEL = 13`
- `MessageMode.GAMEMASTER_PRIVATE_FROM = 14`
- `MessageMode.GAMEMASTER_PRIVATE_TO = 15`
- `MessageMode.GUILD = 31`
- `MessageMode.HEAL = 23`
- `MessageMode.HEAL_OTHERS = 26`
- `MessageMode.HOTKEY_USE = 37`
- `MessageMode.INVALID = 255`
- `MessageMode.LAST = 52`
- `MessageMode.LOGIN = 16`
- `MessageMode.LOOK = 20`
- `MessageMode.LOOT = 29`
- `MessageMode.MANA = 41`
- `MessageMode.MARKET = 40`
- `MessageMode.MONSTER_SAY = 44`
- `MessageMode.MONSTER_YELL = 43`
- `MessageMode.NPC_FROM = 10`
- `MessageMode.NPC_FROM_START_BLOCK = 51`
- `MessageMode.NPC_TO = 11`
- `MessageMode.PARTY = 33`
- `MessageMode.PARTY_MANAGEMENT = 32`
- `MessageMode.PRIVATE_FROM = 4`
- `MessageMode.PRIVATE_TO = 5`
- `MessageMode.RED = 45`
- `MessageMode.REPORT = 36`
- `MessageMode.RVR_ANSWER = 48`
- `MessageMode.RVR_CHANNEL = 47`
- `MessageMode.RVR_CONTINUE = 49`
- `MessageMode.SAY = 1`
- `MessageMode.SPELL = 9`
- `MessageMode.STATUS = 28`
- `MessageMode.THANKYOU = 39`
- `MessageMode.TRADE_NPC = 30`
- `MessageMode.TUTORIAL_HINT = 38`
- `MessageMode.WARNING = 17`
- `MessageMode.WHISPER = 0`
- `MessageMode.YELL = 2`
- `PrintMessagePosition.BOTTOM = 1`
- `PrintMessagePosition.LOOT = 2`
- `PrintMessagePosition.MIDDLE = 0`
- `PVPMode.RED_FIST = 3`
- `PVPMode.UNKNOWN = 4`
- `PVPMode.WHITE_DOVE = 0`
- `PVPMode.WHITE_HAND = 1`
- `PVPMode.YELLOW_HAND = 2`
- `Skill.AXE = 14`
- `Skill.CAPACITY = 9`
- `Skill.CLEAVE_PERCENTAGE = 30`
- `Skill.CLUB = 12`
- `Skill.CRITICAL_CHANCE = 21`
- `Skill.CRITICAL_EXTRA_DAMAGE = 22`
- `Skill.DAMAGE_REFLECTION = 36`
- `Skill.DISTANCE = 11`
- `Skill.EXPERIENCE = 1`
- `Skill.EXPERIENCE_GAIN = 3`
- `Skill.FISHING = 16`
- `Skill.FIST = 15`
- `Skill.FOOD = 17`
- `Skill.HIT_POINTS = 6`
- `Skill.LEVEL = 2`
- `Skill.LIFE_LEECH_AMOUNT = 24`
- `Skill.LIFE_LEECH_CHANCE = 23`
- `Skill.MAGIC_LEVEL = 4`
- `Skill.MAGIC_SHIELD_FLAT = 31`
- `Skill.MAGIC_SHIELD_PERCENT = 32`
- `Skill.MANA = 7`
- `Skill.MANA_LEECH_AMOUNT = 26`
- `Skill.MANA_LEECH_CHANCE = 25`
- `Skill.MOMENTUM_LEVEL = 29`
- `Skill.NONE = 0`
- `Skill.OFFLINE_TRAINING = 20`
- `Skill.ONSLAUGHT_LEVEL = 27`
- `Skill.PERFECT_SHOT_DAMAGE = 33`
- `Skill.RUSE_LEVEL = 28`
- `Skill.SHIELDING = 10`
- `Skill.SOUL = 18`
- `Skill.SPEED = 8`
- `Skill.STAMINA = 19`
- `Skill.SWORD = 13`
- `Skull.BLACK = 5`
- `Skull.GREEN = 2`
- `Skull.NO_SKULL = 0`
- `Skull.RED = 4`
- `Skull.REVENGE = 6`
- `Skull.WHITE = 3`
- `Skull.YELLOW = 1`
- `SpeakClasses.TALKTYPE_BROADCAST = 13`
- `SpeakClasses.TALKTYPE_CHANNEL_MANAGER = 6`
- `SpeakClasses.TALKTYPE_CHANNEL_O = 8`
- `SpeakClasses.TALKTYPE_CHANNEL_R1 = 14`
- `SpeakClasses.TALKTYPE_CHANNEL_R2 = 0xFF`
- `SpeakClasses.TALKTYPE_CHANNEL_Y = 7`
- `SpeakClasses.TALKTYPE_MONSTER_LAST_OLDPROTOCOL = 38`
- `SpeakClasses.TALKTYPE_MONSTER_SAY = 36`
- `SpeakClasses.TALKTYPE_MONSTER_YELL = 37`
- `SpeakClasses.TALKTYPE_NPC_UNKOWN = 11`
- `SpeakClasses.TALKTYPE_PRIVATE_FROM = 4`
- `SpeakClasses.TALKTYPE_PRIVATE_NP = 10`
- `SpeakClasses.TALKTYPE_PRIVATE_PN = 12`
- `SpeakClasses.TALKTYPE_PRIVATE_RED_FROM = 15`
- `SpeakClasses.TALKTYPE_PRIVATE_RED_TO = 16`
- `SpeakClasses.TALKTYPE_PRIVATE_TO = 5`
- `SpeakClasses.TALKTYPE_SAY = 1`
- `SpeakClasses.TALKTYPE_SPELL_USE = 9`
- `SpeakClasses.TALKTYPE_WHISPER = 2`
- `SpeakClasses.TALKTYPE_YELL = 3`
- `SpecialAreaFeature.All = 15`
- `SpecialAreaFeature.Looter = 8`
- `SpecialAreaFeature.MagicShooter = 4`
- `SpecialAreaFeature.Targeting = 2`
- `SpecialAreaFeature.Walker = 1`
- `VipFlag.AIM_TARGET = 4`
- `VipFlag.CROSS = 8`
- `VipFlag.GREEN_TARGET = 10`
- `VipFlag.GREEN_TRIANGLE = 7`
- `VipFlag.HEART = 1`
- `VipFlag.MONEY_SIGN = 9`
- `VipFlag.NO_FLAG = 0`
- `VipFlag.SKULL_CROSSED = 2`
- `VipFlag.STAR = 5`
- `VipFlag.THUNDER = 3`
- `VipFlag.YING_YANG = 6`
- `Vocation.VOCATION_DRUID_CIP = 4`
- `Vocation.VOCATION_KNIGHT_CIP = 1`
- `Vocation.VOCATION_MONK_CIP = 5`
- `Vocation.VOCATION_PALADIN_CIP = 2`
- `Vocation.VOCATION_SORCERER_CIP = 3`
- `WalkerEvent.ACTION_COMPLETED = 6`
- `WalkerEvent.ACTION_STARTED = 5`
- `WalkerEvent.MOVEMENT_CONFIRMED = 7`
- `WalkerEvent.MOVEMENT_REJECTED = 8`
- `WalkerEvent.OBSERVE_ACTION = 4`
- `WalkerEvent.OBSERVE_LABEL = 3`
- `WalkerEvent.ON_ACTION = 2`
- `WalkerEvent.ON_LABEL = 0`
- `WalkerEvent.ON_WAYPOINT_CHANGE = 1`
- `WeaponProficiencyAction.APPLY_PERKS = 3`
- `WeaponProficiencyAction.ITEM_INFO = 0`
- `WeaponProficiencyAction.LIST_INFO = 1`
- `WeaponProficiencyAction.RESET_PERKS = 2`
- `WeaponProficiencyBonus.ALPHA_STRIKE_EXTRA_DAMAGE = 28`
- `WeaponProficiencyBonus.ARMOR_PENETRATION = 30`
- `WeaponProficiencyBonus.ATTACK_DAMAGE = 0`
- `WeaponProficiencyBonus.ATTACK_RANGE = 24`
- `WeaponProficiencyBonus.AUTO_ATTACK_CRITICAL_EXTRA_DAMAGE = 15`
- `WeaponProficiencyBonus.AUTO_ATTACK_CRITICAL_HIT_CHANCE = 11`
- `WeaponProficiencyBonus.BESTIARY = 6`
- `WeaponProficiencyBonus.CRITICAL_EXTRA_DAMAGE = 12`
- `WeaponProficiencyBonus.CRITICAL_HIT_CHANCE = 8`
- `WeaponProficiencyBonus.DEFENSE_BONUS = 1`
- `WeaponProficiencyBonus.ELEMENTAL_CRITICAL_EXTRA_DAMAGE = 13`
- `WeaponProficiencyBonus.ELEMENTAL_HIT_CHANCE = 9`
- `WeaponProficiencyBonus.ELEMENTAL_PIERCE = 31`
- `WeaponProficiencyBonus.LIFE_GAIN_ON_HIT = 19`
- `WeaponProficiencyBonus.LIFE_GAIN_ON_KILL = 21`
- `WeaponProficiencyBonus.LIFE_LEECH = 17`
- `WeaponProficiencyBonus.MANA_GAIN_ON_HIT = 18`
- `WeaponProficiencyBonus.MANA_GAIN_ON_KILL = 20`
- `WeaponProficiencyBonus.MANA_LEECH = 16`
- `WeaponProficiencyBonus.OMEGA_STRIKE_EXTRA_DAMAGE = 29`
- `WeaponProficiencyBonus.PERFECT_SHOT_DAMAGE = 22`
- `WeaponProficiencyBonus.POWERFUL_FOE_BONUS = 7`
- `WeaponProficiencyBonus.RANGED_HIT_CHANCE = 23`
- `WeaponProficiencyBonus.RUNE_CRITICAL_EXTRA_DAMAGE = 14`
- `WeaponProficiencyBonus.RUNE_CRITICAL_HIT_CHANCE = 10`
- `WeaponProficiencyBonus.SKILL_BONUS = 3`
- `WeaponProficiencyBonus.SKILL_PERCENTAGE_AUTO_ATTACK = 25`
- `WeaponProficiencyBonus.SKILL_PERCENTAGE_SPELL_DAMAGE = 26`
- `WeaponProficiencyBonus.SKILL_PERCENTAGE_SPELL_HEALING = 27`
- `WeaponProficiencyBonus.SPECIALIZED_MAGIC_LEVEL = 4`
- `WeaponProficiencyBonus.SPELL_AUGMENT = 5`
- `WeaponProficiencyBonus.WEAPON_SHIELD_MODIFIER = 2`
- `WeaponProficiencyExperience.EASY = 1`
- `WeaponProficiencyExperience.HARD = 3`
- `WeaponProficiencyExperience.MEDIUM = 2`
- `WeaponProficiencyGain.HIT = 0`
- `WeaponProficiencyGain.KILL = 1`
- `WeaponProficiencyHealth.LIFE = 0`
- `WeaponProficiencyHealth.MANA = 1`
- `WeaponProficiencySkillPercentage.AUTO_ATTACK = 0`
- `WeaponProficiencySkillPercentage.SPELL_DAMAGE = 1`
- `WeaponProficiencySkillPercentage.SPELL_HEALING = 2`
- `WeaponProficiencySpellBoost.COOLDOWN = 1`
- `WeaponProficiencySpellBoost.CRITICAL_CHANCE = 12`
- `WeaponProficiencySpellBoost.CRITICAL_DAMAGE = 11`
- `WeaponProficiencySpellBoost.DAMAGE = 8`
- `WeaponProficiencySpellBoost.DAMAGE_REDUCTION = 9`
- `WeaponProficiencySpellBoost.GROUP_COOLDOWN = 2`
- `WeaponProficiencySpellBoost.HEAL = 10`
- `WeaponProficiencySpellBoost.LIFE_LEECH = 6`
- `WeaponProficiencySpellBoost.LIFE_LEECH_CHANCE = 7`
- `WeaponProficiencySpellBoost.MANA = 0`
- `WeaponProficiencySpellBoost.MANA_LEECH = 4`
- `WeaponProficiencySpellBoost.MANA_LEECH_CHANCE = 5`
- `WeaponProficiencySpellBoost.SECONDARY_GROUP_COOLDOWN = 3`
- `WeaponProficiencySpellBoost.TOTAL_COUNT = 13`

### map.lua
- `Map.FindPath(fromPosition: PositionLike, toPosition: PositionLike, maxComplexity?: integer, flags?: integer, allowSpeculativeUnknown?: boolean) -> MapPathResult`
- `Map.GetObjectInfo(itemId: integer) -> ObjectInfo|nil`
- `Map.GetTileFlags(position: PositionLike) -> MapTileFlags|nil`
- `Map.GetTileItems(position: PositionLike, includeCreatures?: boolean) -> MapTileItem[]`
- `Map.Look(position: PositionLike) -> boolean`
- `Map.MoveItemFloorToContainer(itemId: integer, fromPosition: PositionLike, containerIndex: integer, slotIndex: integer, itemCount: integer) -> boolean`
- `Map.MoveItemFloorToFloor(fromPosition: PositionLike, itemId: integer, toPosition: PositionLike, itemCount: integer) -> boolean`
- `Map.OpenBrowseField(position: PositionLike) -> boolean`
- `Map.UseItemOnFloor(position: PositionLike, stackPosition: integer, itemId: integer) -> boolean`

### minimap.lua
- `Minimap.FindPath(fromPosition: PositionLike, toPosition: PositionLike, maxComplexity?: integer, flags?: integer, allowSpeculativeUnknown?: boolean) -> MapPathResult`
- `Minimap.GetTileFlags(position: PositionLike) -> MapTileFlags|nil`
- `Minimap.GetTileInfo(position: PositionLike, includeCreatures?: boolean) -> MinimapTileInfo`
- `Minimap.GetTileItems(position: PositionLike, includeCreatures?: boolean) -> MapTileItem[]`
- `Minimap.GetTilePixelColor(position: PositionLike) -> integer|nil`
- `Minimap.IsPathable(position: PositionLike) -> boolean|nil`
- `Minimap.IsPixelColorWalkable(pixelColorIndex: integer) -> boolean`
- `Minimap.IsWalkable(position: PositionLike) -> boolean|nil`
- `Minimap.IsWalkableByColor(position: PositionLike) -> boolean|nil`

### module.lua
- `Module.After(name: string, callback: function(), delayMs: integer) -> boolean`
- `Module.Cancel(name: string) -> boolean`
- `Module.Every(name: string, callback: function(), delayMs: integer) -> boolean`
- `Module.Exists(name: string) -> boolean`
- `Module.Get(name: string) -> ModuleRecord|nil`
- `Module.List() -> ModuleListRecord[]`
- `Module.New(name: string, callback: function(), delayMs?: integer) -> nil`
- `Module.Pause(name: string) -> nil`
- `Module.PauseManaged(name: string) -> boolean`
- `Module.Resume(name: string) -> nil`
- `Module.ResumeManaged(name: string) -> boolean`
- `Module.Stop(name: string) -> nil`

### npc_trade_storage.lua
- `NpcTradeStorage.Buy(itemId: integer, itemCount: integer, ignoreCapacity?: boolean, buyInShoppingBags?: boolean) -> boolean`
- `NpcTradeStorage.FormatOffers() -> string[]`
- `NpcTradeStorage.GetNpcName() -> string|nil`
- `NpcTradeStorage.GetOfferByItemId(itemId: integer) -> NpcTradeOffer|nil`
- `NpcTradeStorage.GetOfferByName(itemName: string) -> NpcTradeOffer|nil`
- `NpcTradeStorage.GetOffers() -> NpcTradeOffer[]`
- `NpcTradeStorage.GetSnapshot() -> NpcTradeSnapshot`
- `NpcTradeStorage.IsAvailable() -> boolean`
- `NpcTradeStorage.IsOpen() -> boolean|nil`
- `NpcTradeStorage.Sell(itemId: integer, itemCount: integer, sellEquipped?: boolean) -> boolean`

### position.lua
- `Position.IsReachable(fromOrTarget: PositionLike|nil, toOrFrom?: PositionLike) -> boolean`
- `Position.IsShootable(fromOrTarget: PositionLike|nil, toOrFrom?: PositionLike) -> boolean`
- `Position.New(x: integer|PositionLike, y?: integer, z?: integer) -> Position`
- `Position:DistanceTo(otherPos: PositionLike) -> integer`

### self.lua
- `Self.Attack(creatureId: integer) -> boolean`
- `Self.BuyItem(itemId: integer, itemCount: integer, ignoreCapacity?: boolean, buyInShoppingBags?: boolean) -> boolean`
- `Self.CancelWalk() -> boolean`
- `Self.Dismount() -> boolean`
- `Self.Equip(itemId: integer, tierLevel?: integer) -> boolean`
- `Self.Follow(creatureId: integer) -> boolean`
- `Self.FormatStatsSnapshot(stats?: SelfStatsSnapshot, prefix?: string) -> string`
- `Self.GetCapacity() -> number|nil`
- `Self.GetCapacityFloor() -> integer|nil`
- `Self.GetCharacterWorld(characterName: string) -> string|nil`
- `Self.GetFollowId() -> integer|nil`
- `Self.GetHealth() -> integer|nil`
- `Self.GetHealthPercentage() -> number|nil`
- `Self.GetItemCount(itemId: integer, tierLevel?: integer) -> integer`
- `Self.GetLevel() -> integer|nil`
- `Self.GetLevelPercentage() -> integer|nil`
- `Self.GetMana() -> integer|nil`
- `Self.GetManaPercentage() -> number|nil`
- `Self.GetManaShieldCapacity() -> integer|nil`
- `Self.GetMaxHealth() -> integer|nil`
- `Self.GetMaxMana() -> integer|nil`
- `Self.GetMaxManaShieldCapacity() -> integer|nil`
- `Self.GetMousePositionInWorld() -> Position|nil`
- `Self.GetMousePositionText() -> string`
- `Self.GetMouseWorldX() -> integer|nil`
- `Self.GetMouseWorldY() -> integer|nil`
- `Self.GetMouseWorldZ() -> integer|nil`
- `Self.GetSoul() -> integer|nil`
- `Self.GetStamina() -> number|nil`
- `Self.GetStaminaDays() -> integer|nil`
- `Self.GetStaminaHours() -> integer|nil`
- `Self.GetStatsSnapshot() -> SelfStatsSnapshot`
- `Self.GetStatusFlagsSnapshot() -> SelfStatusFlags`
- `Self.GetTargetId() -> integer|nil`
- `Self.HasAgony() -> boolean|nil`
- `Self.HasCharacterFlag(flag: integer) -> boolean|nil`
- `Self.HasFollow() -> boolean|nil`
- `Self.HasGoshnarTaint1() -> boolean|nil`
- `Self.HasGoshnarTaint2() -> boolean|nil`
- `Self.HasGoshnarTaint3() -> boolean|nil`
- `Self.HasGoshnarTaint4() -> boolean|nil`
- `Self.HasGoshnarTaint5() -> boolean|nil`
- `Self.HasGreaterHex() -> boolean|nil`
- `Self.HasIntenseHex() -> boolean|nil`
- `Self.HasLesserHex() -> boolean|nil`
- `Self.HasMentorOther() -> boolean|nil`
- `Self.HasNewManaShield() -> boolean|nil`
- `Self.HasTarget() -> boolean|nil`
- `Self.IsAlive() -> boolean|nil`
- `Self.IsAttacking() -> boolean|nil`
- `Self.IsAvailable() -> boolean`
- `Self.IsBleeding() -> boolean|nil`
- `Self.IsBurning() -> boolean|nil`
- `Self.IsCursed() -> boolean|nil`
- `Self.IsDazzled() -> boolean|nil`
- `Self.IsDrowning() -> boolean|nil`
- `Self.IsDrunk() -> boolean|nil`
- `Self.IsElectrified() -> boolean|nil`
- `Self.IsFeared() -> boolean|nil`
- `Self.IsFollowing() -> boolean|nil`
- `Self.IsFreezing() -> boolean|nil`
- `Self.IsHasted() -> boolean|nil`
- `Self.IsHungry() -> boolean|nil`
- `Self.IsInCombat() -> boolean|nil`
- `Self.IsInProtectionZone() -> boolean|nil`
- `Self.IsInRestingArea() -> boolean|nil`
- `Self.IsManaShielded() -> boolean|nil`
- `Self.IsOnline() -> boolean|nil`
- `Self.IsParalyzed() -> boolean|nil`
- `Self.IsPoisoned() -> boolean|nil`
- `Self.IsPowerless() -> boolean|nil`
- `Self.IsPzBlocked() -> boolean|nil`
- `Self.IsRooted() -> boolean|nil`
- `Self.IsStrengthened() -> boolean|nil`
- `Self.LookAtCreature(creatureId: integer) -> boolean`
- `Self.LookAtPosition(position: PositionLike) -> boolean`
- `Self.Mount() -> boolean`
- `Self.PrivateMessage(playerName: string, message: string) -> boolean`
- `Self.Rotate(direction: integer) -> boolean`
- `Self.Say(message: string) -> boolean`
- `Self.SayOnChannel(message: string, channelId: integer) -> boolean`
- `Self.SayToNpc(message: string) -> boolean`
- `Self.SellItem(itemId: integer, itemCount: integer, sellEquipped?: boolean) -> boolean`
- `Self.Step(direction: integer) -> boolean`
- `Self.StopAttackAndFollow() -> boolean`
- `Self.Turn(direction: integer) -> boolean`
- `Self.UseItemInContainer(itemId: integer, containerIndex: integer, itemPos: integer, useItemWithHotkey?: boolean) -> boolean`
- `Self.UseItemOnFloor(position: PositionLike, stackPosition: integer, itemId: integer) -> boolean`
- `Self.Whisper(message: string) -> boolean`
- `Self.Yell(message: string) -> boolean`

### sound.lua
- `BotSoundId.CREATURE_DETECTED = 5`
- `BotSoundId.DAMAGE_TAKEN = 1`
- `BotSoundId.DISCONNECTED = 0`
- `BotSoundId.ENEMY_ON_SCREEN = 9`
- `BotSoundId.GM_ON_SCREEN = 11`
- `BotSoundId.LOCAL_MESSAGE = 10`
- `BotSoundId.LOW_HEALTH = 2`
- `BotSoundId.LOW_MANA = 3`
- `BotSoundId.PLAYER_ATTACK = 6`
- `BotSoundId.PLAYER_DETECTED = 7`
- `BotSoundId.PRIVATE_MESSAGE = 4`
- `BotSoundId.SKULL_ON_SCREEN = 8`
- `BotSoundId.UNJUSTIFIED_KILL = 13`
- `BotSoundId.WALKER_STUCK = 12`
- `Sound.ClearQueue() -> nil`
- `Sound.GetCurrentDuration() -> integer`
- `Sound.GetFileDuration(filePath: string) -> integer`
- `Sound.GetQueueLength() -> integer`
- `Sound.GetQueueSize() -> integer`
- `Sound.IsPlaying() -> boolean`
- `Sound.IsQueued(options: SoundPlaybackOptions) -> boolean`
- `Sound.Play(options: SoundPlaybackOptions) -> nil`
- `Sound.PlayAndWait(options: SoundPlaybackOptions, maxWaitMs?: number) -> boolean`
- `Sound.PlayBotSound(filename: string, instant?: boolean) -> nil`
- `Sound.PlayById(soundId: integer, instant?: boolean) -> nil`
- `Sound.PlayByIdSmart(soundId: integer, instant?: boolean) -> boolean`
- `Sound.PlayByName(soundName: string, instant?: boolean) -> nil`
- `Sound.PlayByNameSmart(soundName: string, instant?: boolean) -> boolean`
- `Sound.PlayFile(filePath: string, instant?: boolean) -> nil`
- `Sound.PlayFileSmart(filePath: string, instant?: boolean) -> boolean`
- `Sound.SetMinDelay(delayMs: integer) -> nil`
- `Sound.Stop() -> nil`
- `Sound.StopAll() -> nil`
- `Sound.WaitForCompletion(maxWaitMs?: number) -> boolean`
- `Time.MonotonicMs() -> integer`

### spells.lua
- `Spells.GetGroupIds(spellOrWordsOrId: string|integer) -> integer[]`
- `Spells.GetIdByName(name: string) -> integer|nil`
- `Spells.GetIdByWords(words: string) -> integer|nil`
- `Spells.GetInfo(spellOrWordsOrId: string|integer) -> SpellInfo`
- `Spells.GetLeftCooldownTime(spellOrWordsOrId: string|integer) -> integer`
- `Spells.GetLeftGroupCooldownTime(groupId: integer) -> integer`
- `Spells.GetWordsById(spellId: integer) -> string|nil`
- `Spells.GroupIsInCooldown(groupId: integer) -> boolean`
- `Spells.IsInCooldown(spellOrWordsOrId: string|integer) -> boolean`
- `Spells.IsReady(spellOrWordsOrId: string|integer) -> boolean`
- `Spells.IsUseWithItemExhausted() -> boolean`
- `Spells.Item.GetCooldownId(itemId: integer) -> integer|nil`
- `Spells.Item.GetGroupIds(itemId: integer) -> integer[]`
- `Spells.Item.GetInfo(itemId: integer) -> ItemSpellInfo`
- `Spells.Item.GetLeftCooldownTime(itemId: integer) -> integer`
- `Spells.Item.IsInCooldown(itemId: integer) -> boolean`
- `Spells.Item.IsReady(itemId: integer) -> boolean`
- `Spells.Item.WillBeReady(itemId: integer, timeMs: integer) -> boolean`
- `Spells.WillBeReady(spellOrWordsOrId: string|integer, timeMs: integer) -> boolean`

### storage.lua
- `SharedStorageScope:Clear() -> boolean, string|nil`
- `SharedStorageScope:Get(key: string, default?: JsonValue) -> JsonValue, string|nil`
- `SharedStorageScope:OffChanged(subscriptionId: string) -> boolean, string|nil`
- `SharedStorageScope:OnChanged(callback: function(event: SharedStorageChangeEvent), key?: string, includeSelf?: boolean) -> string|nil, string|nil`
- `SharedStorageScope:Remove(key: string) -> boolean, string|nil`
- `SharedStorageScope:Set(key: string, value: JsonValue) -> boolean, string|nil`
- `SharedStorageScope:Update(key: string, updater: function(current: JsonValue) -> JsonValue, default?: JsonValue) -> boolean, JsonValue, string|nil`
- `Storage.Character.Clear() -> boolean`
- `Storage.Character.Get(key: string, default?: JsonValue) -> JsonValue`
- `Storage.Character.Remove(key: string) -> boolean`
- `Storage.Character.Set(key: string, value: JsonValue) -> boolean`
- `Storage.ForCharacter(namespace: string) -> StorageScope`
- `Storage.Global.Clear() -> boolean`
- `Storage.Global.Get(key: string, default?: JsonValue) -> JsonValue`
- `Storage.Global.Remove(key: string) -> boolean`
- `Storage.Global.Set(key: string, value: JsonValue) -> boolean`
- `Storage.Namespace(namespace: string, perCharacter?: boolean) -> StorageScope`
- `Storage.Shared(namespace: string) -> SharedStorageScope`
- `Storage.SharedForCharacter(namespace: string) -> SharedStorageScope`
- `StorageScope:Get(key: string, default?: JsonValue) -> JsonValue`
- `StorageScope:Remove(key: string) -> boolean`
- `StorageScope:Set(key: string, value: JsonValue) -> boolean`

### ui.lua
- `UI.ActionRow(id: string, label: string, enabled: boolean, selected: boolean, onSelect?: function(), onToggle?: function(value:boolean), onDelete?: function(), onEdit?: function(), options?: UIActionRowOptions) -> nil`
- `UI.AlignRight(builder: function()) -> nil`
- `UI.Badge(id: string, label: string, color: ColorRGBA, options?: UIWidgetOptions) -> nil`
- `UI.Baseline(builder: function()) -> nil`
- `UI.Breadcrumbs(id: string, entries: string[], onSelect?: UIBreadcrumbSelectCallback, options?: UIContainerOptions) -> nil`
- `UI.BulletText(text: string) -> nil`
- `UI.Button(id: string, label: string, onClick?: function(), options?: UIWidgetOptions) -> nil`
- `UI.Callout(id: string, message: string, kind?: 'success'|'warning'|'error'|'info', options?: UICalloutOptions) -> nil`
- `UI.Canvas(id: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.CanvasCircle(center: UIVector2, radius: number, color: ColorRGBA, options?: UICanvasShapeOptions) -> nil`
- `UI.CanvasLine(from: UIVector2, to: UIVector2, color: ColorRGBA, thickness?: number) -> nil`
- `UI.CanvasPath(points: UIVector2[], color: ColorRGBA, options?: UICanvasShapeOptions) -> nil`
- `UI.CanvasRect(position: UIVector2, size: UIVector2, color: ColorRGBA, options?: UICanvasShapeOptions) -> nil`
- `UI.CanvasText(position: UIVector2, text: string, color: ColorRGBA) -> nil`
- `UI.CanvasTexture(kind: 'item'|'sprite'|'monster', resource: integer|string, position: UIVector2, size: UIVector2, options?: UITextureOptions) -> nil`
- `UI.Center(builder: function()) -> nil`
- `UI.Checkbox(id: string, label: string, value: boolean, onChange?: function(value:boolean), options?: UIWidgetOptions) -> nil`
- `UI.Child(id: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.CollapsingHeader(id: string, label: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.ColorEdit(id: string, label: string, color: ColorRGBA, onChange?: function(value:ColorRGBA), options?: UIWidgetOptions) -> nil`
- `UI.Column(id: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.Combo(id: string, label: string, selectedIndex: integer, items: string[], onChange?: function(value:integer), options?: UIWidgetOptions) -> nil`
- `UI.ContextMenu(id: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.CreateWindow(id: string, title: string, buildCallback: function(), options?: UIWindowOptions) -> UIWindow`
- `UI.Disabled(disabled: boolean, builder: function()) -> nil`
- `UI.DragFloat(id: string, label: string, value: number, minimum: number, maximum: number, onChange?: function(value:number), options?: UIWidgetOptions) -> nil`
- `UI.DragInt(id: string, label: string, value: integer, minimum: integer, maximum: integer, onChange?: function(value:integer), options?: UIWidgetOptions) -> nil`
- `UI.DragSource(id: string, label: string, payload: string, options?: UIWidgetOptions) -> nil`
- `UI.DropTarget(id: string, label: string, onDrop?: function(payload:string), options?: UIWidgetOptions) -> nil`
- `UI.Dummy(width?: number|UIVector2, height?: number) -> nil`
- `UI.EmptyState(id: string, title: string, detail?: string, options?: UIEmptyStateOptions) -> nil`
- `UI.EquipmentSlot(id: string, itemId: integer, slotName: string, onClick?: function(), options?: UITextureOptions) -> nil`
- `UI.FilterChip(id: string, label: string, selected: boolean, onChange?: function(value:boolean), options?: UIWidgetOptions) -> nil`
- `UI.Group(builder: function()) -> nil`
- `UI.Hotkey(id: string, label: string, eventId: string, options?: UIWidgetOptions) -> nil`
- `UI.IconButton(id: string, icon: string, onClick?: function(), options?: UIWidgetOptions) -> nil`
- `UI.Indent(amount: number, builder: function()) -> nil`
- `UI.InputFloat(id: string, label: string, value: number, onChange?: function(value:number), options?: UIWidgetOptions) -> nil`
- `UI.InputInt(id: string, label: string, value: integer, onChange?: function(value:integer), options?: UIWidgetOptions) -> nil`
- `UI.InputText(id: string, label: string, value: string, onChange?: function(value:string), options?: UIWidgetOptions) -> nil`
- `UI.InputTextMultiline(id: string, label: string, value: string, onChange?: function(value:string), options?: UIWidgetOptions) -> nil`
- `UI.ItemButton(id: string, itemId: integer, onClick?: function(), options?: UITextureOptions) -> nil`
- `UI.ItemSlot(id: string, itemId: integer, label?: string, onClick?: function(), options?: UITextureOptions) -> nil`
- `UI.ItemTexture(id: string, itemId: integer, options?: UITextureOptions) -> nil`
- `UI.LoadingIndicator(id: string, label: string, options?: UIWidgetOptions) -> nil`
- `UI.MetricCard(id: string, label: string, value: string|number, options?: UIMetricCardOptions) -> nil`
- `UI.Modal(id: string, label: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.MonsterButton(id: string, monsterName: string, onClick?: function(), options?: UITextureOptions) -> nil`
- `UI.MonsterTexture(id: string, monsterName: string, options?: UITextureOptions) -> nil`
- `UI.NewLine() -> nil`
- `UI.Notify(title: string, message: string, notificationType?: UINotificationType, options?: UINotificationOptions) -> boolean`
- `UI.OutfitButton(id: string, source: table<string,integer|string>, onClick?: function(), options?: UITextureOptions) -> nil`
- `UI.OutfitTexture(id: string, source: table<string,integer|string>, options?: UITextureOptions) -> nil`
- `UI.Padding(padding: UIPadding, builder: function()) -> nil`
- `UI.Panel(id: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.Popup(id: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.ProgressBar(fraction: number, overlay?: string, options?: UIWidgetOptions) -> nil`
- `UI.PropertyRow(id: string, label: string, value: string|number|boolean, options?: UIContainerOptions) -> nil`
- `UI.RadioButton(id: string, label: string, value: boolean, onChange?: function(value:boolean), options?: UIWidgetOptions) -> nil`
- `UI.ReorderList(id: string, items: string[], onReorder?: UIReorderCallback, options?: UIWidgetOptions) -> nil`
- `UI.Row(id: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.SameLine(offset?: number, spacing?: number) -> nil`
- `UI.SearchField(id: string, value: string, onChange?: function(value:string), options?: UIWidgetOptions) -> nil`
- `UI.SectionHeader(id: string, title: string, subtitle?: string, options?: UISectionHeaderOptions) -> nil`
- `UI.Selectable(id: string, label: string, selected: boolean, onChange?: function(value:boolean), options?: UIWidgetOptions) -> nil`
- `UI.Separator() -> nil`
- `UI.SliderFloat(id: string, label: string, value: number, minimum: number, maximum: number, onChange?: function(value:number), options?: UIWidgetOptions) -> nil`
- `UI.SliderInt(id: string, label: string, value: integer, minimum: integer, maximum: integer, onChange?: function(value:integer), options?: UIWidgetOptions) -> nil`
- `UI.SmallButton(id: string, label: string, onClick?: function(), options?: UIWidgetOptions) -> nil`
- `UI.Spacer(width?: number|UIVector2, height?: number) -> nil`
- `UI.Spacing(height?: number) -> nil`
- `UI.SplitPane(id: string, leftOptions?: UIContainerOptions, rightOptions?: UIContainerOptions, leftBuilder: function(), rightBuilder: function(), options?: UIContainerOptions) -> nil`
- `UI.SpriteTexture(id: string, spriteId: integer, options?: UITextureOptions) -> nil`
- `UI.StatusDot(id: string, label: string, color: ColorRGBA, options?: UIWidgetOptions) -> nil`
- `UI.Style(style: UIStyle, builder: function()) -> nil`
- `UI.TabBar(id: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.TabItem(id: string, label: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.Table(id: string, columns: string[], options?: UIContainerOptions, builder: function()) -> nil`
- `UI.TableNextColumn() -> nil`
- `UI.TableNextRow() -> nil`
- `UI.Text(text: string) -> nil`
- `UI.TextColored(text: string, color: ColorRGBA) -> nil`
- `UI.TextureButton(id: string, spriteId: integer, onClick?: function(), options?: UITextureOptions) -> nil`
- `UI.TextWrapped(text: string) -> nil`
- `UI.Toolbar(id: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UI.Tooltip(text: string) -> nil`
- `UI.Tree(id: string, label: string, options?: UIContainerOptions, builder: function()) -> nil`
- `UIWindow:ClosePopup(popupId: string) -> nil`
- `UIWindow:Destroy() -> boolean`
- `UIWindow:Focus() -> nil`
- `UIWindow:GetHotkeyEventId() -> string|nil`
- `UIWindow:Hide() -> nil`
- `UIWindow:Invalidate() -> nil`
- `UIWindow:IsVisible() -> boolean`
- `UIWindow:OpenPopup(popupId: string) -> nil`
- `UIWindow:SetDecorations(enabled: boolean) -> nil`
- `UIWindow:SetHotkey(combo?: string, options?: UIWindowHotkeyOptions) -> string|nil`
- `UIWindow:SetOnClose(callback?: function()) -> nil`
- `UIWindow:SetOpacity(opacity: number) -> nil`
- `UIWindow:SetPosition(position: UIVector2) -> nil`
- `UIWindow:SetRefreshInterval(milliseconds: integer) -> nil`
- `UIWindow:SetResizable(resizable: boolean) -> nil`
- `UIWindow:SetSize(size: UIVector2) -> nil`
- `UIWindow:SetStyle(style: UIStyle) -> nil`
- `UIWindow:SetTitle(title: string) -> nil`
- `UIWindow:Show() -> nil`
- `UIWindow:Toggle() -> boolean`

### vip.lua
- `VIP.Count() -> integer`
- `VIP.CountOnline() -> integer`
- `VIP.Exists(vipName: string) -> boolean`
- `VIP.FindByPrefix(namePrefix: string, onlyOnline?: boolean) -> VIPEntry[]`
- `VIP.Get(vipName: string) -> VIPEntry|nil`
- `VIP.GetAll() -> VIPEntry[]`
- `VIP.GetByType(vipType: integer) -> VIPEntry[]`
- `VIP.GetDescription(vipName: string) -> string|nil`
- `VIP.GetHearts() -> VIPEntry[]`
- `VIP.GetNames(onlyOnline?: boolean) -> string[]`
- `VIP.GetNotifyOnLogin(vipName: string) -> boolean|nil`
- `VIP.GetSnapshot() -> VIPSnapshot`
- `VIP.GetType(vipName: string) -> integer|nil`
- `VIP.IsAvailable() -> boolean`
- `VIP.IsHeart(vipName: string) -> boolean`
- `VIP.IsOnline(vipName: string) -> boolean`
- `VIP.ToLookupTable() -> table<string, VIPEntry>`

### websocket.lua
- `WebSocket.Connect(url: string, options?: WebSocketConnectOptions) -> WebSocketConnection|nil, string|nil`
- `WebSocketConnection:Close(closeCode?: integer, reason?: string) -> boolean, string|nil`
- `WebSocketConnection:IsOpen() -> boolean`
- `WebSocketConnection:Receive(timeoutMs?: integer) -> WebSocketEvent`
- `WebSocketConnection:Send(data: string, binary?: boolean) -> boolean, string|nil`

