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

This file includes a curated behavior guide plus an auto-generated public API appendix derived from `docs/Scripts/core`. The appendix lists canonical function names, parameter types where annotated, and return types while deliberately omitting local helpers, compatibility aliases, raw bindings, protocol details, and bootstrap internals.

## Runtime Model
- Scripts run inside the bot Lua runtime and use supported APIs exposed as global tables.
- Core helper libraries are loaded from Scripts/core.
- User script public API uses canonical PascalCase module access (for example Self, Map, Container, Module, ChatChannelStorage, VIP).
- Low-level runtime globals are unsupported unless explicitly documented in this file.
- Script execution is cooperative. Long loops must yield (for example via wait(...)).
- Some APIs may return nil when game state or a required capability is unavailable.

## Hard Rules for Generated Scripts
1. Do not use os.execute. Use os.exit only for an explicit emergency client-shutdown policy (for example, preventing a red-skull equipment loss); it terminates the entire game client and is audit-logged.
2. Never disable or query the reserved Objects Dumper feature through Features API.
3. Avoid busy loops. Any repeating logic must yield with wait(ms) or use Module helpers (Module.Every / Module.After).
4. Validate all external input and table fields before use.
5. Assume runtime values can be nil and handle fallbacks safely.
6. Use only documented PascalCase core wrappers in Scripts/core (Self.*, Map.*, Module.*, etc.).
7. Avoid hard crashes: use guards, type checks, and pcall where appropriate.
8. Keep scripts idempotent when possible (safe to re-run).
9. Use explicit comments for non-obvious logic.
10. Keep feature toggling scoped to user features only.
11. Do not use removed compatibility aliases such as self.*, map.*, engine.ammoRefill.*, CaveBot, or _G.position.
12. Do not call low-level runtime globals directly unless they are documented here as public API; prefer core wrappers such as Self, Creature, Creatures, Cavebot, Engine, Game, and Module.

## Public Module Naming
- Module tables are exposed for user scripts in PascalCase form.
- Some modules also expose grouped namespaces such as `Cavebot.Walker`, `Cavebot.Actions`, `Cooldowns.Spell`, and `Engine.Healer`.
- Use the direct canonical methods documented here. Do not infer a `Query` or `Actions` namespace unless it is explicitly listed.
- Compatibility aliases are not part of the supported script-writing surface; generated scripts must use the exact names documented in this file.

## Known Constraints
- Hotkeys: alt modifier is not supported by current Events.RegisterKeyEvent API.
- Standard Lua `table.concat` is allowed in sandboxed user scripts and can be used for safe string assembly.
- Extended keys: insert/delete/home/end/pageup/pagedown/arrows default to extended=true in Hotkeys.ParseCombo.
- Features API excludes Objects Dumper from public get/set/list/status paths.
- HTTP and WebSocket operations must be started from a managed script coroutine. They yield cooperatively while waiting.
- HTTP has no callback lifecycle API. WebSockets use explicit `Receive`; there are no `onOpen`, `onClose`, `onError`, `onRedirect`, or automatic reconnect callbacks.
- Capability wrappers such as Inventory and NPC trade can return nil/false when the corresponding game state is unavailable. Check `IsAvailable`/capability methods where provided.

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

### Magic Shooter entries
Lua scripts can inspect and configure existing Magic Shooter entries without switching profiles or rebuilding them:

- `Engine.MagicShooter.GetEntries(profile?) -> entries|nil, error?`
- `Engine.MagicShooter.SetEntryRune(entryIndex, runeId, profile?) -> success, error?`
- `Engine.MagicShooter.SetEntrySpell(entryIndex, spellWords, profile?) -> success, error?`
- `Engine.MagicShooter.SetEntryEnabled(entryIndex, enabled, profile?) -> boolean`
- `Engine.MagicShooter.SetEntryMonsterNames(entryIndex, names, profile?) -> boolean`
- `Engine.MagicShooter.SetEntryRange(entryIndex, range, profile?) -> boolean`
- Explicit setters also exist for option, condition, mana/health/harmony thresholds, monster count and comparison, monster HP range, danger, PvP safety, ally shooting, target requirement, custom/walk/momentum delays, skill-buff percentages, movement/momentum flags, cast method, and pattern anchor/source/variant.

`profile` defaults to the active Magic Shooter profile. It may be a 1-based profile index or an exact profile name. Entry indexes are also 1-based and match the order shown in the selected profile. `GetEntries` returns a detached snapshot with all public condition, threshold, delay, safety, skill-buff, and pattern fields. Editing that returned table has no effect; call the matching setter.

Replacing an action preserves that entry's enabled state, position, monster names, creature-count/health/mana conditions, delays, and advanced settings. A recognized rune or spell also updates its action type, range, and built-in pattern using the same automatic detection as the Magic Shooter UI. An unknown custom action can replace an entry that is already the same kind, but cannot cross from rune to spell or spell to rune because the missing action metadata would be ambiguous. Changes affect the live in-memory profile and are included by a later normal settings save.

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
- Map.FindPath(fromPosition, toPosition, maxComplexity, flags)

### minimap.lua
Read-only minimap tile queries and pathfinding helpers.

Primary API:
- Minimap.GetTileFlags(position)
- Minimap.GetTileItems(position, includeCreatures?)
- Minimap.IsWalkable(position) / IsPathable(position)
- Minimap.GetTilePixelColor(position)
- Minimap.IsPixelColorWalkable(pixelColorIndex)
- Minimap.IsWalkableByColor(position)
- Minimap.FindPath(fromPosition, toPosition, maxComplexity?, flags?)
- Minimap.GetTileInfo(position, includeCreatures?)

Tile checks may return nil when minimap data is unavailable. `FindPath` returns `{ Directions = integer[], pathFindResult = PathFindResult.* }`; note the capital `D` in `Directions`. `GetTileInfo` returns `position`, `flags`, `items`, `pixelColor`, and `walkableByColor`.

### item.lua
Item use and trade wrappers.

Primary API:
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

### event_proxies.lua
Event proxy wrappers for common game event categories.

Available proxies:
- GenericTextMessageProxy
- BattleMessageProxy
- LootMessageProxy
- ContainerOpenProxy
- ContainerCloseProxy
- ContainerAddItemProxy
- ContainerUpdateItemProxy
- ContainerRemoveItemProxy
- StatsChangeProxy
- SkillsChangeProxy
- CreatureAddProxy
- CreatureRemoveProxy
- DeathProxy

Common pattern:
- local p = SomeProxy:New("name")
- p:OnReceive(function(proxy, ...) ... end)

Callback arguments after `proxy`:
- GenericTextMessageProxy, BattleMessageProxy, LootMessageProxy: `message`
- ContainerOpenProxy: `containerIndex, containerName, containerID`
- ContainerCloseProxy: `containerIndex`
- ContainerAddItemProxy, ContainerUpdateItemProxy: `containerIndex, slot, item`
- ContainerRemoveItemProxy: `containerIndex, slot`
- StatsChangeProxy, SkillsChangeProxy: `eventData`
- CreatureAddProxy: `creatureId, creatureName, position`
- CreatureRemoveProxy: `creatureId`
- DeathProxy: no additional arguments

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

Use `Engine.Features` to query, enable, disable, or toggle public bot features, including Supplies Sorter. IDs for internal object dumping, queue/event infrastructure, and the scripter are rejected by the native boundary. `Engine.Equipment` provides live equipped-slot data and equipment actions; it is distinct from Equipment Manager's saved configuration. Magic Shooter and Targeting expose profile and explicit entry control.

The older global `Features`, `Inventory`, `PVPTools`, `MagicShooter`, and `Targeting` tables remain available for compatibility. New scripts should prefer the corresponding `Engine.*` namespaces.

### cavebot.lua
High-level cavebot orchestration wrappers over Walker and Lure Manager feature toggles.

Primary API:
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
- Cavebot.GoTo(labelName)
- Cavebot.GoToLabel(labelName)
- Cavebot.Pause(milliseconds, autoResume)
- Cavebot.RegisterEvent(eventId, callback)
- Cavebot.OnLabel(callback)
- Cavebot.OnWaypointChange(callback)
- Cavebot.OnAction(callback)
- Cavebot.UnregisterAllEvents()
- Cavebot.GetStatus()
- Cavebot.PrintStatus()

Walker namespace (full runtime wrappers):
- Cavebot.Walker.SetEnabled(enabled)
- Cavebot.Walker.IsEnabled()
- Cavebot.Walker.Resume()
- Cavebot.Walker.GoTo(labelName)
- Cavebot.Walker.GetSelectedWaypointIndex()
- Cavebot.Walker.SetSelectedWaypointIndex(index)
- Cavebot.Walker.SelectClosestWaypoint()
- Cavebot.Walker.GetWaypointCount()
- Cavebot.Walker.GetWaypoints()
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
- Cavebot.Lure.SetOption(option)
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

`Sound.Play` and `Sound.IsQueued` options must identify exactly one source: `sound_id = BotSoundId.*`, `sound_name = string`, or `file_path = string`. `Sound.Play` also accepts `instant = boolean`. Use these canonical methods rather than legacy global sound helper functions.

### storage.lua
Persistent JSON-compatible values scoped to the current script. No file paths or manual serialization are needed.

Direct scopes:
- Storage.Global.Get(key, default?) / Set(key, value) / Remove(key) / Clear()
- Storage.Character.Get(key, default?) / Set(key, value) / Remove(key) / Clear()

Logical namespaces:
- Storage.Namespace(namespace, perCharacter?)
- Storage.ForCharacter(namespace)
- scope:Get(key, default?) / Set(key, value) / Remove(key)

Global values are shared by characters running the same script; character values are isolated by the logged-in character. Namespaces must be non-empty, at most 64 bytes, and contain only letters, numbers, `_`, `-`, and `.`. A namespace plus key may not exceed 256 bytes. Supported values are nil, booleans, finite numbers, strings, and nested tables with string keys or contiguous 1-based array indexes. Cyclic tables are rejected. Storage is limited to 2 MB per script, 12 nested levels, and 4,096 entries per table. Prefer a stable, script-specific namespace to avoid key collisions.

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

All classes support `New`, `Create`, `Remove`, `SetEnabled`, `SetZIndex`, `SetParent`, `ClearParent`, `IsCreated`, `GetEnabled`, `GetVisible`, `GetPosition`, `GetWidth`, and `GetHeight` as applicable. Setters return the same object for chaining.

Class-specific methods:
- ScreenText: SetText, SetColor, SetAlignment, SetDraggable, SetDragTarget, SetOnDragEnd, SetClickable, SetScreenPosition, GetText, GetColor
- ScreenImage: SetSource, SetItemId, SetItemName, SetSize, SetLabel, SetAlignment, SetDraggable, SetDragTarget, SetOnDragEnd, SetClickable, SetScreenPosition
- WorldText: SetText, SetColor, SetPosition, SetLifetime, SetOffset, GetText, GetColor
- WorldBox: SetSize, SetWidth, SetHeight, SetColor, SetBorderWidth, SetBorderColor, SetPosition, SetLifetime, GetColor
- WorldImage: SetSource, SetItemId, SetItemName, SetSize, SetPosition, SetOffset, SetLifetime

Use HUD objects for visual diagnostics, status displays, labels, and map markers. IDs should be stable and unique within the script.

Screen image notes:
- ScreenImage:SetLabel(text, color, offsetX, offsetY) attaches or updates text on the image.
- ScreenImage:SetScreenPosition(x, y) positions image HUD elements in screen pixels.
- ScreenImage:SetParent(parent_id) can parent icons to a draggable ScreenText handle; children keep their current offset when the parent moves.

### lua_consts.lua
Shared gameplay constants/enums for movement, pathfinding, effects, equipment, chat, creature state, combat modes, skills, VIP flags, vocation, and walker events.

Use constants from this file instead of magic numbers.

## Practical Script Examples

These examples are intentionally small, defensive, and written in the public PascalCase API style. They are good patterns for LLM-generated scripts to copy.

### 1. Repeating Low HP Sound Alert

Use `Module.Every` for repeating work, check that the player is available, and wrap runtime logic with `pcall` so one temporary failure does not kill the script.

```lua
local SCRIPT_ID = "low_hp_sound_example"

local function safe(fn, label)
    local ok, err = pcall(fn)
    if not ok then
        print("[" .. SCRIPT_ID .. "] " .. label .. " failed: " .. tostring(err))
    end
end

Module.Every(SCRIPT_ID .. "_tick", function()
    safe(function()
        if not Self.IsAvailable() then
            return
        end

        local hp = Self.GetHealthPercentage()
        if type(hp) == "number" and hp <= 35 then
            Sound.Play({ sound_id = BotSoundId.LOW_HEALTH })
        end
    end, "tick")
end, 1000)
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
1. All loops yield (wait/module schedule) and cannot freeze runtime.
2. Every table access checks nil/type when data can be absent.
3. Feature operations do not touch reserved features.
4. Hotkeys avoid unsupported alt modifier.
5. Position/creature access handles invalid objects safely.
6. Actions with IDs validate integers and positive ranges.
7. Script can be stopped cleanly (no destructive side effects).

## Recommended Script Template
Use this as baseline for generated scripts.

```lua
local SCRIPT_ID = "my_script_id"

local function log(msg)
    print("[" .. SCRIPT_ID .. "] " .. tostring(msg))
end

local function safe_run(fn, context)
    local ok, err = pcall(fn)
    if not ok then
        log("ERROR in " .. tostring(context) .. ": " .. tostring(err))
    end
end

local function tick()
    -- Put guarded logic here.
    -- Always assume runtime values can be nil.
end

Module.Every(SCRIPT_ID .. "_tick", function()
    safe_run(tick, "tick")
end, 200)

log("started")
```

## LLM Generation Rules (Copy Into Prompt)
When generating ValidusBot Lua scripts:
1. Use only APIs documented in this file.
2. Prefer core wrappers over raw low-level calls.
3. Include argument validation and nil checks.
4. Never use alt for hotkeys.
5. Never use or toggle Objects Dumper feature.
6. For repeated logic, use Module.Every/Module.After or wait-based yielding.
7. Return complete runnable script with concise comments.
8. Avoid placeholders like TODO unless explicitly requested.

## Example Prompt for ChatGPT/Claude
Use this prompt with this file attached:

"Generate a production-safe ValidusBot Lua script using only APIs from the attached spec. Script goal: <describe goal>. Include robust nil checks, validation, and non-blocking scheduling. Do not use unsupported modifiers/features. Return one complete .lua file."


---

## Engine feature-control API

Use `Engine` for bot feature configuration and actions. Configuration is exposed through explicit getters and setters; do not assume that feature objects or their internal fields are available. Getter tables are detached snapshots, so changing a returned table does not change the bot. Call the matching setter.

The main feature namespaces are `Engine.Healer`, `Engine.Conditions`, `Engine.HealFriend`, `Engine.MagicShooter`, `Engine.Targeting`, `Engine.AmmoRefill`, `Engine.EquipmentManager`, `Engine.Alarms`, `Engine.PVPTools`, `Engine.ComboBot`, `Engine.Extras`, `Engine.TankMode`, `Engine.Looter`, `Engine.TimerActions`, `Engine.SuppliesSorter`, `Engine.Channels`, `Engine.Walker`, `Engine.Lure`, `Engine.HUD`, `Engine.Delays`, and `Engine.Scripter`. Feature activation remains under `Engine.Features`.

Important behavior:

- Entry indices are 1-based. Most entry setters affect the active profile; profile-selection functions should be called first when a feature supports profiles.
- Explicit setters validate types and ranges and preserve feature side effects such as timer resets, parsed-name cache rebuilds, packet-event refreshes, and HUD state refreshes.
- `Engine.ComboBot.GetRoomState()` omits room passwords and transport details. Lua cannot connect, disconnect, or send raw Combo Bot room commands.
- `Engine.Scripter.Start(name)` accepts only an exact `.lua` filename returned by `Engine.Scripter.GetAvailableScripts()`. It cannot execute source strings or disable the sandbox.
- A running script must call `Engine.Scripter.StopSelf()` to unload itself. Self-restart is intentionally unavailable; another script may call `Restart(name)`.
- `Engine.HUD` element operations retain per-script ownership. A script cannot mutate or remove another script's HUD elements.
- `Engine.Delays.SetServerPingCheckEnabled()` also applies the required game ping-check interval change.
- Internal services such as the object dumper, packet/event dispatcher, use-item queue, and Validus networking are not exposed through `Engine`.

The exact signatures below are authoritative. Avoid legacy generic update-table helpers; use the explicit field setter for the value being changed.

## Appendix A: Public Core Lua API Index (Auto-Generated)
Generated from `docs/Scripts/core`. It intentionally excludes local helpers, compatibility aliases, raw bindings, protocol details, and API-surface bootstrap internals. Use the canonical names below; if a function is absent, treat it as unavailable.

### cavebot.lua
- `Cavebot.Disable()`
- `Cavebot.DisableLure()`
- `Cavebot.Enable()`
- `Cavebot.EnableLure()`
- `Cavebot.GetStatus()`
- `Cavebot.GoTo(labelName)`
- `Cavebot.GoToLabel(labelName)`
- `Cavebot.IsEnabled()`
- `Cavebot.IsLureEnabled()`
- `Cavebot.Lure.AddSetting(setting)`
- `Cavebot.Lure.ClearSettings()`
- `Cavebot.Lure.EndForceLure()`
- `Cavebot.Lure.GetAttackWhileLuring()`
- `Cavebot.Lure.GetConsiderOnlyReachable()`
- `Cavebot.Lure.GetIgnoringMonsters()`
- `Cavebot.Lure.GetLuredCreaturesCount()`
- `Cavebot.Lure.GetNearRange()`
- `Cavebot.Lure.GetOption()`
- `Cavebot.Lure.GetSettingCount()`
- `Cavebot.Lure.GetSettings()`
- `Cavebot.Lure.GetSlowWalkBurstSteps()`
- `Cavebot.Lure.GetSlowWalkDelayMs()`
- `Cavebot.Lure.GetSlowWalkingCreaturesCount()`
- `Cavebot.Lure.GetStartEndLureActive()`
- `Cavebot.Lure.GetState()`
- `Cavebot.Lure.GetUnblocking()`
- `Cavebot.Lure.GetWaypointDynamicLureActive()`
- `Cavebot.Lure.HasActiveSettings()`
- `Cavebot.Lure.IsEnabled()`
- `Cavebot.Lure.IsFighting()`
- `Cavebot.Lure.IsForceLure()`
- `Cavebot.Lure.IsLuring()`
- `Cavebot.Lure.IsOtherPlayerOnScreen()`
- `Cavebot.Lure.RemoveSetting(index)`
- `Cavebot.Lure.SetAttackWhileLuring(enabled)`
- `Cavebot.Lure.SetConsiderOnlyReachable(enabled)`
- `Cavebot.Lure.SetEnabled(enabled)`
- `Cavebot.Lure.SetForceLure(enabled)`
- `Cavebot.Lure.SetIgnoringMonsters(enabled)`
- `Cavebot.Lure.SetNearRange(range)`
- `Cavebot.Lure.SetOption(option)`
- `Cavebot.Lure.SetSlowWalkBurstSteps(steps)`
- `Cavebot.Lure.SetSlowWalkDelayMs(delayMs)`
- `Cavebot.Lure.SetSlowWalkingCreaturesCount(count)`
- `Cavebot.Lure.SetStartEndLureActive(enabled)`
- `Cavebot.Lure.SetUnblocking(enabled)`
- `Cavebot.Lure.SetWaypointDynamicLureActive(enabled)`
- `Cavebot.Lure.UpdateSetting(index, updateData)`
- `Cavebot.OnAction(callback)`
- `Cavebot.OnLabel(callback)`
- `Cavebot.OnWaypointChange(callback)`
- `Cavebot.Pause(milliseconds, autoResume)`
- `Cavebot.PrintStatus()`
- `Cavebot.RegisterEvent(eventId, callback)`
- `Cavebot.Resume()`
- `Cavebot.SetEnabled(enabled)`
- `Cavebot.SetEnginesEnabled(walkerEnabled, lureEnabled)`
- `Cavebot.SetLureEnabled(enabled)`
- `Cavebot.UnregisterAllEvents()`
- `Cavebot.Walker.AddWaypoint(waypoint)`
- `Cavebot.Walker.ClearWaypoints()`
- `Cavebot.Walker.DeleteWaypoint(index)`
- `Cavebot.Walker.GetAutoRecorderEnabled()`
- `Cavebot.Walker.GetAutoRecorderOptions()`
- `Cavebot.Walker.GetDebugHud()`
- `Cavebot.Walker.GetDistanceBetweenWaypoints()`
- `Cavebot.Walker.GetLeaveLureOnPlayer()`
- `Cavebot.Walker.GetLeaveLurePlayerMode() -> integer`
- `Cavebot.Walker.GetNodeDistance()`
- `Cavebot.Walker.GetSelectedWaypointIndex()`
- `Cavebot.Walker.GetStartFromNearestWaypoint()`
- `Cavebot.Walker.GetWalkToLureCenter()`
- `Cavebot.Walker.GetWaypointCount()`
- `Cavebot.Walker.GetWaypoints()`
- `Cavebot.Walker.GoTo(labelName)`
- `Cavebot.Walker.InsertWaypoint(index, waypoint)`
- `Cavebot.Walker.IsEnabled()`
- `Cavebot.Walker.IsPausedByLua()`
- `Cavebot.Walker.IsStuck()`
- `Cavebot.Walker.MoveWaypointDown(index)`
- `Cavebot.Walker.MoveWaypointUp(index)`
- `Cavebot.Walker.ReplaceWaypoint(index, waypoint)`
- `Cavebot.Walker.Resume()`
- `Cavebot.Walker.SelectClosestWaypoint()`
- `Cavebot.Walker.SetAutoRecorderEnabled(enabled)`
- `Cavebot.Walker.SetAutoRecorderOptions(options)`
- `Cavebot.Walker.SetDebugHud(enabled)`
- `Cavebot.Walker.SetDistanceBetweenWaypoints(distance)`
- `Cavebot.Walker.SetEnabled(enabled)`
- `Cavebot.Walker.SetLeaveLureOnPlayer(enabled)`
- `Cavebot.Walker.SetLeaveLurePlayerMode(mode: integer) -> boolean`
- `Cavebot.Walker.SetNodeDistance(distance)`
- `Cavebot.Walker.SetPausedByLua(paused)`
- `Cavebot.Walker.SetSelectedWaypointIndex(index)`
- `Cavebot.Walker.SetStartFromNearestWaypoint(enabled)`
- `Cavebot.Walker.SetWalkToLureCenter(enabled)`

### cavebot_actions.lua
- `Cavebot.Actions.GetLastResult()`
- `Cavebot.Actions.Register(actionType, handler)`
- `Cavebot.Actions.Run(context)`

### chat_channel.lua
- `ChatChannel.FromIdentifier(identifier: any) -> table|nil`
- `ChatChannel.GetById(channelId: integer) -> table|nil`
- `ChatChannel.GetByName(channelName: string) -> table|nil`
- `ChatChannel.New(channelOrId: table|integer, channelName?: string) -> table`
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
- `ChatChannel:ToTable() -> table`

### chat_channel_storage.lua
- `ChatChannelStorage.CanSend(channelIdentifier: any) -> boolean`
- `ChatChannelStorage.FormatChannel(channel: table) -> string`
- `ChatChannelStorage.GetChannelNames(onlySendable?: boolean) -> string[]`
- `ChatChannelStorage.GetChatChannelById(channelId: integer) -> table|nil`
- `ChatChannelStorage.GetChatChannelByName(channelName: string) -> table|nil`
- `ChatChannelStorage.GetChatChannelCount() -> integer`
- `ChatChannelStorage.GetChatChannels() -> table[]`
- `ChatChannelStorage.GetLocalChatChannel() -> table|nil`
- `ChatChannelStorage.GetOpenedChannelCount() -> integer`
- `ChatChannelStorage.GetOpenedChannels() -> table[]`
- `ChatChannelStorage.GetServerLogChannel() -> table|nil`
- `ChatChannelStorage.GetSnapshot() -> table`
- `ChatChannelStorage.HasChannelById(channelId: integer) -> boolean`
- `ChatChannelStorage.HasChannelByName(channelName: string) -> boolean`
- `ChatChannelStorage.IsAvailable() -> boolean`
- `ChatChannelStorage.ResolveChannel(channelIdentifier: any) -> table|nil`
- `ChatChannelStorage.Send(message: string, channelIdentifier: any) -> boolean`
- `ChatChannelStorage.ToIdLookupTable() -> table`
- `ChatChannelStorage.ToNameLookupTable() -> table`

### container.lua
- `Container.FindItem(containerNumber: integer, itemId: integer, tierLevel?: integer) -> table|nil`
- `Container.FindItemInOpenContainers(itemId: integer, tierLevel?: integer) -> table|nil`
- `Container.GetById(containerId: integer) -> table|nil`
- `Container.GetByName(containerName: string) -> table|nil`
- `Container.GetByNumber(containerNumber: integer) -> table|nil`
- `Container.GetFreeSlots(containerNumber: integer) -> integer|nil`
- `Container.GetId(containerNumber: integer) -> integer|nil`
- `Container.GetItem(containerNumber: integer, slotIndex: integer) -> table|nil`
- `Container.GetItems(containerNumber: integer) -> table[]`
- `Container.GetItemsCount(containerNumber: integer) -> integer|nil`
- `Container.GetName(containerNumber: integer) -> string|nil`
- `Container.GetOpenContainers() -> table[]`
- `Container.GetSize(containerNumber: integer) -> integer|nil`
- `Container.LookItem(itemId: integer, itemPos: integer, containerIndex: integer) -> any`
- `Container.MoveItemFromEquipmentToContainer(equipmentSlot: integer, containerIndex: integer, slotIndex: integer, itemId: integer, itemCount: integer) -> any`
- `Container.MoveItemToContainer(fromContainerIndex: integer, fromSlotIndex: integer, itemId: integer, toContainerIndex: integer, toSlotIndex: integer, itemCount: integer) -> any`
- `Container.MoveItemToEquipment(containerIndex: integer, slotIndex: integer, itemId: integer, equipmentSlot: integer, itemCount: integer) -> any`
- `Container.MoveItemToFloor(containerIndex: integer, slotIndex: integer, itemId: integer, toPosition: table, itemCount: integer) -> any`
- `Container.UseItem(itemId: integer, containerIndex: integer, itemPos: integer, useItemWithHotkey?: boolean) -> any`

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
- `Cooldowns.Utils.GetStatus(spells?: string[]) -> table`
- `Cooldowns.Utils.PrintStatus(spells?: string[])`

### creature.lua
- `Creature.GetFollowed() -> Creature|nil`
- `Creature.GetLocalPlayer() -> Creature|nil`
- `Creature.GetTarget() -> Creature|nil`
- `Creature:ClearCache()`
- `Creature:DistanceTo(targetPos: table) -> number`
- `Creature:DistanceToCreature(otherCreature: Creature) -> number`
- `Creature:Equals(other: Creature) -> boolean`
- `Creature:GetDirection() -> number`
- `Creature:GetGuildShield() -> number`
- `Creature:GetHealthPercent() -> number`
- `Creature:GetId() -> number`
- `Creature:GetLowercaseName() -> string`
- `Creature:GetMasterId() -> number`
- `Creature:GetName() -> string`
- `Creature:GetOutfit() -> table`
- `Creature:GetPartyShield() -> number`
- `Creature:GetPosition() -> table`
- `Creature:GetSkull() -> number`
- `Creature:GetSpeed() -> number`
- `Creature:GetVocation() -> number`
- `Creature:IsAdjacentTo(targetPos: table) -> boolean`
- `Creature:IsGameMaster() -> boolean`
- `Creature:IsInGuild() -> boolean`
- `Creature:IsInParty() -> boolean`
- `Creature:IsMonster() -> boolean`
- `Creature:IsMounted() -> boolean`
- `Creature:IsNPC() -> boolean`
- `Creature:IsPartyLeader() -> boolean`
- `Creature:IsPlayer() -> boolean`
- `Creature:IsReachable() -> boolean`
- `Creature:IsSameFloor(targetPos: table) -> boolean`
- `Creature:IsShootable() -> boolean`
- `Creature:IsSkulled() -> boolean`
- `Creature:IsSummon() -> boolean`
- `Creature:IsValid() -> boolean`
- `Creature:IsVisible() -> boolean`
- `Creature:IsWarAlly() -> boolean`
- `Creature:IsWarEnemy() -> boolean`
- `Creature:New(creatureId: number) -> Creature`
- `Creature:ToString() -> string`

### creature_iterators.lua
- `Creature.ICreatures() -> function`
- `Creature.IMonsters() -> function`
- `Creature.INpcs() -> function`
- `Creature.IPlayers() -> function`
- `Creatures.GetAttackingCreatureId() -> integer|nil`
- `Creatures.GetCreatureByName(creatureName: string) -> table|nil`
- `Creatures.GetCreatureIdsByScan(typeFlags?: integer, xRelativeDistance?: integer, yRelativeDistance?: integer, multifloor?: boolean, ignoreSummons?: boolean) -> integer[]`
- `Creatures.GetCreaturesByScan(typeFlags?: integer, xRelativeDistance?: integer, yRelativeDistance?: integer, multifloor?: boolean, ignoreSummons?: boolean) -> table[]`
- `Creatures.GetFollowingCreatureId() -> integer|nil`
- `Creatures.GetLocalPlayerId() -> integer|nil`
- `Creatures.GetPlayerIdUnderMouse() -> integer|nil`
- `Creatures.GetVisibleCreatureIds() -> integer[]`
- `Creatures.GetVisibleCreatures() -> table[]`
- `Creatures.GetVisibleMonsters(ignoreSummons?: boolean) -> table[]`
- `Creatures.GetVisibleNpcs() -> table[]`
- `Creatures.GetVisiblePlayers() -> table[]`
- `Creatures.IsCreatureOnScreen(creatureId: integer, xRelativeDistance?: integer, yRelativeDistance?: integer, multifloor?: boolean) -> boolean`

### engine.lua
- `Engine.Alarms.Disable(alarmId: number) -> boolean`
- `Engine.Alarms.DisableAll() -> nil`
- `Engine.Alarms.Enable(alarmId: number) -> boolean`
- `Engine.Alarms.EnableAll() -> nil`
- `Engine.Alarms.EnableOnly(alarmIdsList: table) -> nil`
- `Engine.Alarms.GetConfig() -> table`
- `Engine.Alarms.GetCreatureFilter() -> string`
- `Engine.Alarms.GetLowHealthThreshold() -> number`
- `Engine.Alarms.GetLowManaThreshold() -> number`
- `Engine.Alarms.GetMessageFilter() -> string`
- `Engine.Alarms.IsBringToFocusEnabled() -> boolean`
- `Engine.Alarms.IsEnabled(alarmId: number) -> boolean`
- `Engine.Alarms.IsFlashWindowEnabled() -> boolean`
- `Engine.Alarms.IsIgnoringAllyPlayers() -> boolean`
- `Engine.Alarms.PrintStatus() -> nil`
- `Engine.Alarms.SetAlarmMessages(messages: any) -> boolean`
- `Engine.Alarms.SetBringToFocus(enabled: boolean) -> boolean`
- `Engine.Alarms.SetBringToFocusEnabled(value: boolean) -> boolean`
- `Engine.Alarms.SetCreatureDetectedNames(names: any) -> boolean`
- `Engine.Alarms.SetCreatureFilter(namesString: string) -> boolean`
- `Engine.Alarms.SetDamageTakenRange(minimumDamage: integer, maximumDamage: integer) -> boolean`
- `Engine.Alarms.SetEnemyNames(value: string) -> boolean`
- `Engine.Alarms.SetFlashWindow(enabled: boolean) -> boolean`
- `Engine.Alarms.SetFlashWindowEnabled(value: boolean) -> boolean`
- `Engine.Alarms.SetGmChatCheckEnabled(value: boolean) -> boolean`
- `Engine.Alarms.SetGmNames(value: string) -> boolean`
- `Engine.Alarms.SetIgnoreAllyPlayers(ignore: boolean) -> boolean`
- `Engine.Alarms.SetLowHealthPercentage(arg1: any) -> boolean`
- `Engine.Alarms.SetLowHealthThreshold(percentage: number) -> boolean`
- `Engine.Alarms.SetLowManaPercentage(arg1: any) -> boolean`
- `Engine.Alarms.SetLowManaThreshold(percentage: number) -> boolean`
- `Engine.Alarms.SetMessageFilter(messagesString: string) -> boolean`
- `Engine.Alarms.SetPlayerAttackFilterMode(value: integer) -> boolean`
- `Engine.Alarms.SetPlayerAttackNames(value: string) -> boolean`
- `Engine.Alarms.SetPlayerDetectedFilterMode(value: integer) -> boolean`
- `Engine.Alarms.SetPlayerDetectedNames(value: string) -> boolean`
- `Engine.Alarms.SetSkullFilterMode(value: integer) -> boolean`
- `Engine.Alarms.SetSkullNames(value: string) -> boolean`
- `Engine.Alarms.Toggle(alarmId: number) -> boolean`
- `Engine.AmmoRefill.Add(ammoData: table) -> number`
- `Engine.AmmoRefill.AddProfile(profileName: string|nil) -> number|boolean`
- `Engine.AmmoRefill.ClearAll() -> nil`
- `Engine.AmmoRefill.Disable(index: number) -> boolean`
- `Engine.AmmoRefill.DisableAll() -> nil`
- `Engine.AmmoRefill.Enable(index: number) -> boolean`
- `Engine.AmmoRefill.EnableAll() -> nil`
- `Engine.AmmoRefill.EnableOnly(itemIdsList: table) -> nil`
- `Engine.AmmoRefill.FindByItemId(itemId: number) -> table|nil`
- `Engine.AmmoRefill.FindProfileByName(profileName: string) -> number|nil`
- `Engine.AmmoRefill.Get(index: number) -> table|nil`
- `Engine.AmmoRefill.GetAll() -> table`
- `Engine.AmmoRefill.GetCurrentProfile() -> table|nil`
- `Engine.AmmoRefill.GetProfileNames() -> table`
- `Engine.AmmoRefill.PrintProfiles() -> nil`
- `Engine.AmmoRefill.PrintStatus() -> nil`
- `Engine.AmmoRefill.Remove(index: number) -> boolean`
- `Engine.AmmoRefill.RemoveProfile(indexOrName: number|string) -> boolean`
- `Engine.AmmoRefill.RenameProfile(indexOrName: number|string, newName: string) -> boolean`
- `Engine.AmmoRefill.SetEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.AmmoRefill.SetEntryEquipFromHotkey(entryIndex: integer, value: boolean) -> boolean`
- `Engine.AmmoRefill.SetEntryItemId(entryIndex: integer, value: integer) -> boolean`
- `Engine.AmmoRefill.SetEntryRefillLeftHand(entryIndex: integer, value: boolean) -> boolean`
- `Engine.AmmoRefill.SetEntryThreshold(entryIndex: integer, value: integer) -> boolean`
- `Engine.AmmoRefill.SetProfile(indexOrName: number|string) -> boolean`
- `Engine.AmmoRefill.Toggle(index: number) -> boolean|nil`
- `Engine.Channels.AddEntry(name: string, message: string, intervalSeconds: integer, channelId: integer, talkAction: integer, enabled: boolean|nil) -> integer`
- `Engine.Channels.ClearEntries() -> boolean`
- `Engine.Channels.GetEntries() -> table[]`
- `Engine.Channels.GetGlobalDelay() -> integer`
- `Engine.Channels.RemoveEntry(index: integer) -> boolean`
- `Engine.Channels.SetEntryChannelId(entryIndex: integer, value: integer) -> boolean`
- `Engine.Channels.SetEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Channels.SetEntryIntervalSeconds(entryIndex: integer, value: integer) -> boolean`
- `Engine.Channels.SetEntryMessage(entryIndex: integer, message: string) -> boolean`
- `Engine.Channels.SetEntryName(entryIndex: integer, value: string) -> boolean`
- `Engine.Channels.SetEntryTalkAction(entryIndex: integer, value: integer) -> boolean`
- `Engine.Channels.SetGlobalDelay(value: integer) -> boolean`
- `Engine.ComboBot.GetClientEntries() -> table[]`
- `Engine.ComboBot.GetMode() -> integer`
- `Engine.ComboBot.GetRoomEntries() -> table[]`
- `Engine.ComboBot.GetRoomState() -> table`
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
- `Engine.Conditions.GetHoldSpells() -> table[]`
- `Engine.Conditions.GetManaShieldDelay() -> integer`
- `Engine.Conditions.GetManaShieldTimerBased() -> integer`
- `Engine.Conditions.GetRecoverySpellDelay() -> integer`
- `Engine.Conditions.GetRecoverySpellTimerBased() -> integer`
- `Engine.Conditions.GetSpells() -> table[]`
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
- `Engine.Equipment.Equip(itemId: integer, tierLevel: integer|nil) -> boolean`
- `Engine.Equipment.GetAllSlotItems() -> table`
- `Engine.Equipment.GetSlotConstants() -> table`
- `Engine.Equipment.GetSlotIds() -> integer[]`
- `Engine.Equipment.GetSlotItem(equipmentSlot: integer) -> table|nil`
- `Engine.Equipment.GetSlotItemId(equipmentSlot: integer) -> integer|nil`
- `Engine.Equipment.GetSnapshot() -> table`
- `Engine.Equipment.HasItemInSlot(equipmentSlot: integer) -> boolean|nil`
- `Engine.Equipment.LookSlotItem(itemId: integer, equipmentSlot: integer) -> boolean`
- `Engine.Equipment.MoveFromContainerToSlot(containerIndex: integer, slotIndex: integer, itemId: integer, equipmentSlot: integer, itemCount: integer) -> boolean`
- `Engine.Equipment.MoveFromSlotToContainer(equipmentSlot: integer, containerIndex: integer, slotIndex: integer, itemId: integer, itemCount: integer) -> boolean`
- `Engine.EquipmentManager.GetEntries() -> table[]`
- `Engine.EquipmentManager.GetProfiles() -> table[]`
- `Engine.EquipmentManager.SetActiveProfile(index: integer) -> boolean`
- `Engine.EquipmentManager.SetConditionCreatureNames(entryIndex: integer, conditionIndex: integer, creatureNames: string) -> boolean`
- `Engine.EquipmentManager.SetConditionCreaturesCount(entryIndex: integer, conditionIndex: integer, count: integer) -> boolean`
- `Engine.EquipmentManager.SetConditionMonstersAround(entryIndex: integer, conditionIndex: integer, count: integer) -> boolean`
- `Engine.EquipmentManager.SetConditionPlayersAround(entryIndex: integer, conditionIndex: integer, count: integer) -> boolean`
- `Engine.EquipmentManager.SetConditionTargetName(entryIndex: integer, conditionIndex: integer, targetName: string) -> boolean`
- `Engine.EquipmentManager.SetConditionType(entryIndex: integer, conditionIndex: integer, conditionType: integer) -> boolean`
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
- `Engine.Features.Disable(featureIdentifier: integer|string) -> nil`
- `Engine.Features.DisableAllExcept(excludeList: table|nil) -> nil`
- `Engine.Features.DisableMultiple(featureList: table) -> nil`
- `Engine.Features.Enable(featureIdentifier: integer|string) -> nil`
- `Engine.Features.EnableMultiple(featureList: table) -> nil`
- `Engine.Features.GetActiveFeatures() -> integer[]`
- `Engine.Features.GetAllFeatureIds() -> integer[]`
- `Engine.Features.GetName(featureIdentifier: integer|string) -> string`
- `Engine.Features.IsActive(featureIdentifier: integer|string) -> boolean`
- `Engine.Features.PrintStatus() -> nil`
- `Engine.Features.SetActive(featureIdentifier: integer|string, activeStatus: boolean) -> nil`
- `Engine.Features.Toggle(featureIdentifier: integer|string) -> nil`
- `Engine.Healer.AddItem(itemData: table) -> nil`
- `Engine.Healer.AddSpell(spellData: table) -> nil`
- `Engine.Healer.ClearAllItems() -> nil`
- `Engine.Healer.ClearAllSpells() -> nil`
- `Engine.Healer.DisableAllItems() -> nil`
- `Engine.Healer.DisableAllSpells() -> nil`
- `Engine.Healer.DisableItem(index: number) -> boolean`
- `Engine.Healer.DisableSpell(index: integer) -> nil`
- `Engine.Healer.EnableItem(index: number) -> boolean`
- `Engine.Healer.EnableOnlyItems(itemIdsList: table) -> nil`
- `Engine.Healer.EnableOnlySpells(spellWordsList: table) -> number`
- `Engine.Healer.EnableSpell(index: integer) -> nil`
- `Engine.Healer.FindItemById(itemId: integer) -> table|nil`
- `Engine.Healer.FindSpellByWords(spellWords: string) -> table|nil`
- `Engine.Healer.GetItems() -> table`
- `Engine.Healer.GetSpellByIndex(index: integer) -> table|nil`
- `Engine.Healer.GetSpells() -> table`
- `Engine.Healer.PrintItems() -> nil`
- `Engine.Healer.PrintSpells() -> nil`
- `Engine.Healer.RemoveItem(index: integer) -> nil`
- `Engine.Healer.RemoveSpell(index: integer) -> nil`
- `Engine.Healer.SetItemAction(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemAttribute(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemCastValue(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemCondition(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemDelay(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemEnabled(index: any, enabled: any) -> boolean`
- `Engine.Healer.SetItemId(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetItemUseWhenFeared(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Healer.SetSpellAttribute(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetSpellCastValue(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetSpellCondition(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetSpellEnabled(index: any, enabled: any) -> boolean`
- `Engine.Healer.SetSpellManaCost(entryIndex: integer, value: integer) -> boolean`
- `Engine.Healer.SetSpellWords(entryIndex: integer, value: string) -> boolean`
- `Engine.Healer.ToggleItem(index: number) -> boolean`
- `Engine.Healer.ToggleSpell(index: integer) -> nil`
- `Engine.HealFriend.GetArea() -> table`
- `Engine.HealFriend.GetMode() -> integer`
- `Engine.HealFriend.GetPlayerNames() -> string`
- `Engine.HealFriend.GetPrioritizeBeforeHealer() -> integer`
- `Engine.HealFriend.GetPriorityOverHealer() -> integer`
- `Engine.HealFriend.GetSafeHealthPercentage() -> integer`
- `Engine.HealFriend.GetVocations() -> table[]`
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
- `Engine.HUD.AddScreenImage(params: table) -> table`
- `Engine.HUD.AddScreenText(params: table) -> table`
- `Engine.HUD.AddWorldBox(params: table) -> table`
- `Engine.HUD.AddWorldImage(params: table) -> table`
- `Engine.HUD.AddWorldText(params: table) -> table`
- `Engine.HUD.ClearParent(child_id: string) -> nil`
- `Engine.HUD.GetConfig() -> table`
- `Engine.HUD.GetElementColor(id: string) -> table|nil`
- `Engine.HUD.GetElementEnabled(id: string) -> boolean`
- `Engine.HUD.GetElementHeight(id: string) -> number`
- `Engine.HUD.GetElementText(id: string) -> string|nil`
- `Engine.HUD.GetElementVisible(id: string) -> boolean`
- `Engine.HUD.GetElementWidth(id: string) -> number`
- `Engine.HUD.GetScreenElementPosition(id: string) -> table|nil`
- `Engine.HUD.GetSpecialFoodCounters() -> table[]`
- `Engine.HUD.GetWorldElementPosition(id: string) -> table|nil`
- `Engine.HUD.RemoveElement(id: string) -> nil`
- `Engine.HUD.RemoveSpecialFoodCounter(itemId: integer) -> boolean`
- `Engine.HUD.SetAlignment(id: string, horizontal_align: integer, vertical_align: integer) -> nil`
- `Engine.HUD.SetClickable(id: string, clickable: boolean, callback: function|nil) -> nil`
- `Engine.HUD.SetDraggable(id: string, draggable: boolean) -> nil`
- `Engine.HUD.SetDragTarget(id: string, targetId: string|nil) -> nil`
- `Engine.HUD.SetEnabled(value: boolean) -> boolean`
- `Engine.HUD.SetLevelSpyEnabled(value: boolean) -> boolean`
- `Engine.HUD.SetMagicWallIds(value: integer[]) -> boolean`
- `Engine.HUD.SetMagicWallTimersEnabled(value: boolean) -> boolean`
- `Engine.HUD.SetOnDragEnd(id: string, callback: function|nil) -> nil`
- `Engine.HUD.SetParent(child_id: string, parent_id: string) -> nil`
- `Engine.HUD.SetPosition(params: table) -> nil`
- `Engine.HUD.SetScreenPosition(params: table) -> nil`
- `Engine.HUD.SetSpecialFoodCounterDelay(itemId: integer, delaySeconds: integer) -> boolean`
- `Engine.HUD.SetTargetingAnchorEnabled(value: boolean) -> boolean`
- `Engine.HUD.SetTimerColor(red: number, green: number, blue: number, alpha: number) -> boolean`
- `Engine.HUD.SetWildGrowthIds(value: integer[]) -> boolean`
- `Engine.HUD.SetXRayEnabled(value: boolean) -> boolean`
- `Engine.HUD.SetZIndex(id: string, zIndex: integer) -> nil`
- `Engine.HUD.UpdateBorderColor(id: string, color: table) -> nil`
- `Engine.HUD.UpdateBorderWidth(id: string, border_width: number) -> nil`
- `Engine.HUD.UpdateColor(id: string, color: table) -> nil`
- `Engine.HUD.UpdateHeight(id: string, height: number) -> nil`
- `Engine.HUD.UpdateImageLabel(params: table) -> nil`
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
- `Engine.Lure.AddSetting() -> any`
- `Engine.Lure.ClearSettings() -> any`
- `Engine.Lure.EndForceLure() -> any`
- `Engine.Lure.GetAttackWhileLuring() -> any`
- `Engine.Lure.GetConsiderOnlyReachable() -> any`
- `Engine.Lure.GetIgnoringMonsters() -> any`
- `Engine.Lure.GetLuredCreaturesCount() -> any`
- `Engine.Lure.GetNearRange() -> any`
- `Engine.Lure.GetOption() -> any`
- `Engine.Lure.GetSettingCount() -> any`
- `Engine.Lure.GetSettings() -> any`
- `Engine.Lure.GetSlowWalkBurstSteps() -> any`
- `Engine.Lure.GetSlowWalkDelayMs() -> any`
- `Engine.Lure.GetSlowWalkingCreaturesCount() -> any`
- `Engine.Lure.GetStartEndLureActive() -> any`
- `Engine.Lure.GetState() -> any`
- `Engine.Lure.GetUnblocking() -> any`
- `Engine.Lure.GetWaypointDynamicLureActive() -> any`
- `Engine.Lure.HasActiveSettings() -> any`
- `Engine.Lure.IsEnabled() -> any`
- `Engine.Lure.IsFighting() -> any`
- `Engine.Lure.IsForceLure() -> any`
- `Engine.Lure.IsLuring() -> any`
- `Engine.Lure.IsOtherPlayerOnScreen() -> any`
- `Engine.Lure.RemoveSetting() -> any`
- `Engine.Lure.SetAttackWhileLuring() -> any`
- `Engine.Lure.SetConsiderOnlyReachable() -> any`
- `Engine.Lure.SetEnabled() -> any`
- `Engine.Lure.SetForceLure() -> any`
- `Engine.Lure.SetIgnoringMonsters() -> any`
- `Engine.Lure.SetNearRange() -> any`
- `Engine.Lure.SetOption() -> any`
- `Engine.Lure.SetSlowWalkBurstSteps() -> any`
- `Engine.Lure.SetSlowWalkDelayMs() -> any`
- `Engine.Lure.SetSlowWalkingCreaturesCount() -> any`
- `Engine.Lure.SetStartEndLureActive() -> any`
- `Engine.Lure.SetUnblocking() -> any`
- `Engine.Lure.SetWaypointDynamicLureActive() -> any`
- `Engine.MagicShooter.GetActiveProfile() -> table|nil`
- `Engine.MagicShooter.GetCurrentProfile() -> table|nil`
- `Engine.MagicShooter.GetEntries(profile: integer|string|nil) -> table[]|nil, string|nil`
- `Engine.MagicShooter.GetProfileCount() -> integer`
- `Engine.MagicShooter.GetProfileNames() -> string[]`
- `Engine.MagicShooter.NextProfile() -> table|nil`
- `Engine.MagicShooter.SetActiveProfile(profile: integer|string) -> boolean`
- `Engine.MagicShooter.SetCurrentProfile(profile: integer|string) -> boolean`
- `Engine.MagicShooter.SetEntryAttackSkillBuffSpell(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryCastMethod(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryCondition(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryCustomDelay(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryCustomSpell(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryDangerLevel(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryDistanceSkillIncreasePercentage(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryDontCastWhileWalking(entryIndex: integer, value: boolean) -> boolean`
- `Engine.MagicShooter.SetEntryEnabled(entryIndex: integer, enabled: boolean, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryHarmony(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryHarmonyCondition(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryHealthCondition(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryHealthPercentage(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryManaPercentage(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryMaximumMonsterHealthPercentage(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryMeleeSkillIncreasePercentage(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryMinimumMonsterHealthPercentage(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryMomentumDelay(entryIndex: integer, value: boolean) -> boolean`
- `Engine.MagicShooter.SetEntryMonsterCount(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryMonsterCountCondition(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryMonsterNames(entryIndex: integer, names: string, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryOption(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryPatternAnchor(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryPatternSource(entryIndex: integer, value: string) -> boolean`
- `Engine.MagicShooter.SetEntryPatternVariant(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryPrioritizeWithMomentum(entryIndex: integer, value: boolean) -> boolean`
- `Engine.MagicShooter.SetEntryPVPSafe(entryIndex: integer, value: boolean) -> boolean`
- `Engine.MagicShooter.SetEntryRange(entryIndex: integer, range: integer, profile: integer|string|nil) -> boolean`
- `Engine.MagicShooter.SetEntryRequiresTarget(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryRune(entryIndex: integer, runeId: integer, profile: integer|string|nil) -> boolean, string|nil`
- `Engine.MagicShooter.SetEntryShootAfterWalkDelay(entryIndex: integer, value: integer) -> boolean`
- `Engine.MagicShooter.SetEntryShootOverAllies(entryIndex: integer, value: boolean) -> boolean`
- `Engine.MagicShooter.SetEntrySpell(entryIndex: integer, spellWords: string, profile: integer|string|nil) -> boolean, string|nil`
- `Engine.PVPTools.GetConfig() -> table`
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
- `Engine.Scripter.GetAvailableScripts() -> table[]`
- `Engine.Scripter.GetRunningScripts() -> table[]`
- `Engine.Scripter.IsRunning(scriptName: string) -> boolean`
- `Engine.Scripter.Refresh() -> boolean`
- `Engine.Scripter.Restart(scriptName: string) -> boolean`
- `Engine.Scripter.SetAutoStartEnabled(value: boolean) -> boolean`
- `Engine.Scripter.Start(scriptName: string) -> boolean`
- `Engine.Scripter.Stop(scriptName: string) -> boolean`
- `Engine.Scripter.StopSelf() -> boolean`
- `Engine.SuppliesSorter.AddEntry(destinationContainerId: integer, itemIds: integer[], enabled: boolean|nil) -> integer`
- `Engine.SuppliesSorter.ClearEntries() -> boolean`
- `Engine.SuppliesSorter.GetEntries() -> table[]`
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
- `Engine.Targeting.GetActiveProfile() -> table|nil`
- `Engine.Targeting.GetCurrentProfile() -> table|nil`
- `Engine.Targeting.GetEntries(profile: integer|string|nil) -> table[]|nil`
- `Engine.Targeting.GetProfileCount() -> integer`
- `Engine.Targeting.GetProfileNames() -> string[]`
- `Engine.Targeting.NextProfile() -> table|nil`
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
- `Engine.TimerActions.GetEntries() -> table[]`
- `Engine.TimerActions.RemoveEntry(index: integer) -> boolean`
- `Engine.TimerActions.SetEntryDelay(entryIndex: integer, delay: integer, timeUnit: integer) -> boolean`
- `Engine.TimerActions.SetEntryEnabled(entryIndex: integer, value: boolean) -> boolean`
- `Engine.TimerActions.SetEntryItemId(entryIndex: integer, value: integer) -> boolean`
- `Engine.TimerActions.SetEntrySpellWords(entryIndex: integer, value: string) -> boolean`
- `Engine.TimerActions.SetEntryType(entryIndex: integer, value: integer) -> boolean`
- `Engine.TimerActions.SetEntryUseInProtectionZone(entryIndex: integer, value: boolean) -> boolean`
- `Engine.Walker.AddWaypoint() -> any`
- `Engine.Walker.ClearWaypoints() -> any`
- `Engine.Walker.DeleteWaypoint() -> any`
- `Engine.Walker.GetAutoRecorderEnabled() -> any`
- `Engine.Walker.GetAutoRecorderOptions() -> any`
- `Engine.Walker.GetDebugHud() -> any`
- `Engine.Walker.GetDistanceBetweenWaypoints() -> any`
- `Engine.Walker.GetLeaveLureOnPlayer() -> any`
- `Engine.Walker.GetLeaveLurePlayerMode() -> integer`
- `Engine.Walker.GetNodeDistance() -> any`
- `Engine.Walker.GetSelectedWaypointIndex() -> any`
- `Engine.Walker.GetStartFromNearestWaypoint() -> any`
- `Engine.Walker.GetWalkToLureCenter() -> any`
- `Engine.Walker.GetWaypointCount() -> any`
- `Engine.Walker.GetWaypoints() -> any`
- `Engine.Walker.GoTo() -> any`
- `Engine.Walker.InsertWaypoint() -> any`
- `Engine.Walker.IsEnabled() -> any`
- `Engine.Walker.IsPausedByLua() -> any`
- `Engine.Walker.IsStuck() -> any`
- `Engine.Walker.MoveWaypointDown() -> any`
- `Engine.Walker.MoveWaypointUp() -> any`
- `Engine.Walker.ReplaceWaypoint() -> any`
- `Engine.Walker.Resume() -> any`
- `Engine.Walker.SelectClosestWaypoint() -> any`
- `Engine.Walker.SetAutoRecorderEnabled() -> any`
- `Engine.Walker.SetAutoRecorderOptions() -> any`
- `Engine.Walker.SetDebugHud() -> any`
- `Engine.Walker.SetDistanceBetweenWaypoints() -> any`
- `Engine.Walker.SetEnabled() -> any`
- `Engine.Walker.SetLeaveLureOnPlayer() -> any`
- `Engine.Walker.SetLeaveLurePlayerMode(mode: integer) -> boolean`
- `Engine.Walker.SetNodeDistance() -> any`
- `Engine.Walker.SetPausedByLua() -> any`
- `Engine.Walker.SetSelectedWaypointIndex() -> any`
- `Engine.Walker.SetStartFromNearestWaypoint() -> any`
- `Engine.Walker.SetWalkToLureCenter() -> any`

### event_proxies.lua
- `BattleMessageProxy:GetName() -> string`
- `BattleMessageProxy:New(name: string) -> table`
- `BattleMessageProxy:OnReceive(callback: function) -> table`
- `ContainerAddItemProxy:GetName() -> string`
- `ContainerAddItemProxy:New(name: string) -> table`
- `ContainerAddItemProxy:OnReceive(callback: function) -> table`
- `ContainerCloseProxy:GetName() -> string`
- `ContainerCloseProxy:New(name: string) -> table`
- `ContainerCloseProxy:OnReceive(callback: function) -> table`
- `ContainerOpenProxy:GetName() -> string`
- `ContainerOpenProxy:New(name: string) -> table`
- `ContainerOpenProxy:OnReceive(callback: function) -> table`
- `ContainerRemoveItemProxy:GetName() -> string`
- `ContainerRemoveItemProxy:New(name: string) -> table`
- `ContainerRemoveItemProxy:OnReceive(callback: function) -> table`
- `ContainerUpdateItemProxy:GetName() -> string`
- `ContainerUpdateItemProxy:New(name: string) -> table`
- `ContainerUpdateItemProxy:OnReceive(callback: function) -> table`
- `CreatureAddProxy:GetName() -> string`
- `CreatureAddProxy:New(name: string) -> table`
- `CreatureAddProxy:OnReceive(callback: function) -> table`
- `CreatureRemoveProxy:GetName() -> string`
- `CreatureRemoveProxy:New(name: string) -> table`
- `CreatureRemoveProxy:OnReceive(callback: function) -> table`
- `DeathProxy:GetName() -> string`
- `DeathProxy:New(name: string) -> table`
- `DeathProxy:OnReceive(callback: function) -> table`
- `GenericTextMessageProxy:GetName() -> string`
- `GenericTextMessageProxy:New(name: string) -> table`
- `GenericTextMessageProxy:OnReceive(callback: function) -> table`
- `LootMessageProxy:GetName() -> string`
- `LootMessageProxy:New(name: string) -> table`
- `LootMessageProxy:OnReceive(callback: function) -> table`
- `SkillsChangeProxy:GetName() -> string`
- `SkillsChangeProxy:New(name: string) -> table`
- `SkillsChangeProxy:OnReceive(callback: function) -> table`
- `StatsChangeProxy:GetName() -> string`
- `StatsChangeProxy:New(name: string) -> table`
- `StatsChangeProxy:OnReceive(callback: function) -> table`

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
- `Features.Disable(featureIdentifier)`
- `Features.DisableAllExcept(ExcludeList)`
- `Features.DisableMultiple(featureList)`
- `Features.Enable(featureIdentifier)`
- `Features.EnableMultiple(featureList)`
- `Features.GetActiveFeatures()`
- `Features.GetAllFeatureIds()`
- `Features.GetName(featureIdentifier)`
- `Features.IsActive(featureIdentifier)`
- `Features.PrintStatus()`
- `Features.SetActive(featureIdentifier, activeStatus)`
- `Features.Toggle(featureIdentifier)`

### game.lua
- `Game.EnterWorld() -> boolean`
- `Game.GetCharacterWorld(characterName: string) -> string`
- `Game.LoginToAccount(email: string, password: string) -> boolean`
- `Game.LoginToCharacter(characterName: string) -> boolean`
- `Game.LoginToPreviouslyLoggedCharacter() -> boolean`
- `Game.Logout() -> boolean`
- `Game.OpenContainerInNewWindow(equipmentSlotOrContainerId: number, fromContainerNumber: number|nil, fromContainerSlot: number|nil) -> boolean`
- `Game.OpenStore() -> boolean`

### hotkeys.lua
- `Hotkeys.ParseCombo(combination: string) -> table|nil, string|nil`
- `Hotkeys.RegisterCombo(params: table) -> boolean`

### http.lua
- `Http.Get(url: string, options?: table) -> table`
- `Http.GetJson(url: string, options?: table) -> any|nil, table, string|nil`
- `Http.Post(url: string, body?: string, options?: table) -> table`
- `Http.PostJson(url: string, value: any, options?: table) -> table`
- `Http.Request(options: table) -> table`

### hud_wrapper.lua
- `ScreenImage:ClearParent() -> ScreenImage`
- `ScreenImage:Create()`
- `ScreenImage:GetEnabled() -> boolean`
- `ScreenImage:GetHeight() -> number`
- `ScreenImage:GetPosition() -> table`
- `ScreenImage:GetVisible() -> boolean`
- `ScreenImage:GetWidth() -> number`
- `ScreenImage:IsCreated() -> boolean`
- `ScreenImage:New(id)`
- `ScreenImage:Remove()`
- `ScreenImage:SetAlignment(h_align, v_align)`
- `ScreenImage:SetClickable(callback)`
- `ScreenImage:SetDraggable(draggable)`
- `ScreenImage:SetDragTarget(target: ScreenText|ScreenImage|string|nil) -> ScreenImage`
- `ScreenImage:SetEnabled(enabled)`
- `ScreenImage:SetItemId(itemId)`
- `ScreenImage:SetItemName(itemName)`
- `ScreenImage:SetLabel(text, color, offsetX, offsetY)`
- `ScreenImage:SetOnDragEnd(callback: function|nil) -> ScreenImage`
- `ScreenImage:SetParent(parent: ScreenText|ScreenImage|string) -> ScreenImage`
- `ScreenImage:SetScreenPosition(x: number, y: number) -> ScreenImage`
- `ScreenImage:SetSize(width, height)`
- `ScreenImage:SetSource(path)`
- `ScreenImage:SetZIndex(zIndex)`
- `ScreenText:ClearParent() -> ScreenText`
- `ScreenText:Create() -> ScreenText`
- `ScreenText:GetColor() -> table`
- `ScreenText:GetEnabled() -> boolean`
- `ScreenText:GetHeight() -> number`
- `ScreenText:GetPosition() -> table`
- `ScreenText:GetText() -> string`
- `ScreenText:GetVisible() -> boolean`
- `ScreenText:GetWidth() -> number`
- `ScreenText:IsCreated() -> boolean`
- `ScreenText:New(id: string) -> ScreenText`
- `ScreenText:Remove()`
- `ScreenText:SetAlignment(h_align: number, v_align: number) -> ScreenText`
- `ScreenText:SetClickable(callback: function) -> ScreenText`
- `ScreenText:SetColor(color: table) -> ScreenText`
- `ScreenText:SetDraggable(draggable: boolean) -> ScreenText`
- `ScreenText:SetDragTarget(target: ScreenText|ScreenImage|string|nil) -> ScreenText`
- `ScreenText:SetEnabled(enabled: boolean) -> ScreenText`
- `ScreenText:SetOnDragEnd(callback: function|nil) -> ScreenText`
- `ScreenText:SetParent(parent: ScreenText|ScreenImage|string) -> ScreenText`
- `ScreenText:SetScreenPosition(x: number, y: number) -> ScreenText`
- `ScreenText:SetText(text: string) -> ScreenText`
- `ScreenText:SetZIndex(zIndex: number) -> ScreenText`
- `WorldBox:ClearParent() -> WorldBox`
- `WorldBox:Create() -> WorldBox`
- `WorldBox:GetColor() -> table`
- `WorldBox:GetEnabled() -> boolean`
- `WorldBox:GetHeight() -> number`
- `WorldBox:GetPosition() -> table`
- `WorldBox:GetVisible() -> boolean`
- `WorldBox:GetWidth() -> number`
- `WorldBox:IsCreated() -> boolean`
- `WorldBox:New(id: string, x: number, y: number, z: number) -> WorldBox`
- `WorldBox:Remove()`
- `WorldBox:SetBorderColor(border_color: table) -> WorldBox`
- `WorldBox:SetBorderWidth(border_width: number) -> WorldBox`
- `WorldBox:SetColor(color: table) -> WorldBox`
- `WorldBox:SetEnabled(enabled: boolean) -> WorldBox`
- `WorldBox:SetHeight(height: number) -> WorldBox`
- `WorldBox:SetLifetime(lifetime_ms: number) -> WorldBox`
- `WorldBox:SetParent(parent_id: string) -> WorldBox`
- `WorldBox:SetPosition(x: number, y: number, z: number) -> WorldBox`
- `WorldBox:SetSize(width: number, height: number) -> WorldBox`
- `WorldBox:SetWidth(width: number) -> WorldBox`
- `WorldBox:SetZIndex(zIndex: number) -> WorldBox`
- `WorldImage:ClearParent() -> WorldImage`
- `WorldImage:Create()`
- `WorldImage:GetEnabled() -> boolean`
- `WorldImage:GetHeight() -> number`
- `WorldImage:GetPosition() -> table`
- `WorldImage:GetVisible() -> boolean`
- `WorldImage:GetWidth() -> number`
- `WorldImage:IsCreated() -> boolean`
- `WorldImage:New(id, x, y, z)`
- `WorldImage:Remove()`
- `WorldImage:SetEnabled(enabled: boolean) -> WorldImage`
- `WorldImage:SetItemId(itemId)`
- `WorldImage:SetItemName(itemName)`
- `WorldImage:SetLifetime(lifetimeMs)`
- `WorldImage:SetOffset(offsetX, offsetY)`
- `WorldImage:SetParent(parent_id: string) -> WorldImage`
- `WorldImage:SetPosition(x, y, z)`
- `WorldImage:SetSize(width, height)`
- `WorldImage:SetSource(path)`
- `WorldImage:SetZIndex(zIndex)`
- `WorldText:ClearParent() -> WorldText`
- `WorldText:Create() -> WorldText`
- `WorldText:GetColor() -> table`
- `WorldText:GetEnabled() -> boolean`
- `WorldText:GetHeight() -> number`
- `WorldText:GetPosition() -> table`
- `WorldText:GetText() -> string`
- `WorldText:GetVisible() -> boolean`
- `WorldText:GetWidth() -> number`
- `WorldText:IsCreated() -> boolean`
- `WorldText:New(id: string, x: number, y: number, z: number) -> WorldText`
- `WorldText:Remove()`
- `WorldText:SetColor(color: table) -> WorldText`
- `WorldText:SetEnabled(enabled: boolean) -> WorldText`
- `WorldText:SetLifetime(lifetime_ms: number) -> WorldText|WorldBox`
- `WorldText:SetOffset(offset_x: number, offset_y: number) -> WorldText|WorldBox`
- `WorldText:SetParent(parent_id: string) -> WorldText`
- `WorldText:SetPosition(x: number, y: number, z: number) -> WorldText`
- `WorldText:SetText(text: string) -> WorldText`
- `WorldText:SetZIndex(zIndex: number) -> WorldText`

### inventory.lua
- `Inventory.CanMoveEquipment() -> boolean`
- `Inventory.CanReadEquipment() -> boolean`
- `Inventory.Equip(itemId: integer, tierLevel?: integer) -> any`
- `Inventory.GetAllSlotItems() -> table`
- `Inventory.GetEquipmentSlotConstants() -> table`
- `Inventory.GetSlotIds() -> integer[]`
- `Inventory.GetSlotItem(equipmentSlot: integer) -> table|nil`
- `Inventory.GetSlotItemId(equipmentSlot: integer) -> integer|nil`
- `Inventory.GetSnapshot() -> table`
- `Inventory.HasItemInSlot(equipmentSlot: integer) -> boolean|nil`
- `Inventory.LookSlotItem(itemId: integer, equipmentSlot: integer) -> any`
- `Inventory.MoveFromContainerToSlot(containerIndex: integer, slotIndex: integer, itemId: integer, equipmentSlot: integer, itemCount: integer) -> any`
- `Inventory.MoveFromSlotToContainer(equipmentSlot: integer, containerIndex: integer, slotIndex: integer, itemId: integer, itemCount: integer) -> any`

### item.lua
- `Item.Buy(itemId: integer, itemCount: integer, ignoreCapacity?: boolean, buyInShoppingBags?: boolean) -> any`
- `Item.FindInContainer(containerNumber: integer, itemId: integer, tierLevel?: integer) -> table|nil`
- `Item.GetDescription(itemId: integer) -> string|nil`
- `Item.GetFromContainer(containerNumber: integer, slotIndex: integer) -> table|nil`
- `Item.GetInfo(itemId: integer) -> table|nil`
- `Item.GetName(itemId: integer) -> string|nil`
- `Item.HasFlag(itemId: integer, fieldName: string) -> boolean`
- `Item.IsContainer(itemId)`
- `Item.IsCreature(itemId)`
- `Item.IsCumulative(itemId)`
- `Item.IsGround(itemId)`
- `Item.IsLiquidContainer(itemId)`
- `Item.IsMovable(itemId)`
- `Item.IsMultiUsable(itemId)`
- `Item.IsTakable(itemId)`
- `Item.IsUsable(itemId)`
- `Item.Sell(itemId: integer, itemCount: integer, sellEquipped?: boolean) -> any`
- `Item.UseFromContainerOnFloor(floorPosition: table, fromItemId: integer, toItemId: integer, toStackPosition: integer) -> any`
- `Item.UseFromContainerToContainer(fromContainer: integer, fromSlot: integer, fromItemId: integer, toContainer: integer, toSlot: integer, toItemId: integer) -> any`
- `Item.UseFromFloorToContainer(floorPosition: table, fromItemId: integer, fromStackPosition: integer, toItemId: integer) -> any`
- `Item.UseOnCreature(itemId: integer, creatureId: integer) -> any`

### json.lua
- Constant: `Json.Null` (JSON null sentinel)
- `Json.Array(value: table) -> table`
- `Json.Decode(text: string) -> any`
- `Json.Encode(value: any, pretty?: boolean|integer) -> string`
- `Json.Object(value: table) -> table`
- `Json.TryDecode(text: string) -> any|nil, string|nil`
- `Json.TryEncode(value: any, pretty?: boolean|integer) -> string|nil, string|nil`

### lua_consts.lua
- `CharacterFlag.BLEEDING = 15`
- `CharacterFlag.BURNING = 1`
- `CharacterFlag.CURSED = 11`
- `CharacterFlag.DAZZLED = 10`
- `CharacterFlag.DROWNING = 8`
- `CharacterFlag.DRUNK = 3`
- `CharacterFlag.ELECTRIFIED = 2`
- `CharacterFlag.FEARED = 20`
- `CharacterFlag.FREEZING = 9`
- `CharacterFlag.HASTED = 6`
- `CharacterFlag.IN_COMBAT = 7`
- `CharacterFlag.IN_PROTECTION_ZONE = 14`
- `CharacterFlag.MANA_SHIELDED = 4`
- `CharacterFlag.PARALYSED = 5`
- `CharacterFlag.POISONED = 0`
- `CharacterFlag.ROOTED = 19`
- `CharacterFlag.STRENGTHENED = 12`
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
- `WalkerEvent.ON_ACTION = 2`
- `WalkerEvent.ON_LABEL = 0`
- `WalkerEvent.ON_WAYPOINT_CHANGE = 1`

### map.lua
- `Map.FindPath(fromPosition: table, toPosition: table, maxComplexity?: integer, flags?: integer) -> table`
- `Map.GetObjectInfo(itemId: integer) -> table|nil`
- `Map.GetTileFlags(position: table) -> table|nil`
- `Map.GetTileItems(position: table, includeCreatures?: boolean) -> table[]`
- `Map.Look(position: table) -> any`
- `Map.MoveItemFloorToContainer(itemId: integer, fromPosition: table, containerIndex: integer, slotIndex: integer, itemCount: integer) -> any`
- `Map.MoveItemFloorToFloor(fromPosition: table, itemId: integer, toPosition: table, itemCount: integer) -> any`
- `Map.UseItemOnFloor(position: table, stackPosition: integer, itemId: integer) -> any`

### minimap.lua
- `Minimap.FindPath(fromPosition: table, toPosition: table, maxComplexity?: integer, flags?: integer) -> table`
- `Minimap.GetTileFlags(position: table) -> table|nil`
- `Minimap.GetTileInfo(position: table, includeCreatures?: boolean) -> table`
- `Minimap.GetTileItems(position: table, includeCreatures?: boolean) -> table[]`
- `Minimap.GetTilePixelColor(position: table) -> integer|nil`
- `Minimap.IsPathable(position: table) -> boolean|nil`
- `Minimap.IsPixelColorWalkable(pixelColorIndex: integer) -> boolean`
- `Minimap.IsWalkable(position: table) -> boolean|nil`
- `Minimap.IsWalkableByColor(position: table) -> boolean|nil`

### module.lua
- `Module.After(name, callback, delayMs)`
- `Module.Cancel(name)`
- `Module.Every(name, callback, delayMs)`
- `Module.Exists(name)`
- `Module.Get(name)`
- `Module.List()`
- `Module.New(name, callback, delayMs)`
- `Module.Pause(name)`
- `Module.PauseManaged(name)`
- `Module.Resume(name)`
- `Module.ResumeManaged(name)`
- `Module.Stop(name)`

### npc_trade_storage.lua
- `NpcTradeStorage.Buy(itemId: integer, itemCount: integer, ignoreCapacity?: boolean, buyInShoppingBags?: boolean) -> any`
- `NpcTradeStorage.FormatOffers() -> string[]`
- `NpcTradeStorage.GetNpcName() -> string|nil`
- `NpcTradeStorage.GetOfferByItemId(itemId: integer) -> table|nil`
- `NpcTradeStorage.GetOfferByName(itemName: string) -> table|nil`
- `NpcTradeStorage.GetOffers() -> table[]`
- `NpcTradeStorage.GetSnapshot() -> table`
- `NpcTradeStorage.IsAvailable() -> boolean`
- `NpcTradeStorage.IsOpen() -> boolean|nil`
- `NpcTradeStorage.Sell(itemId: integer, itemCount: integer, sellEquipped?: boolean) -> any`

### position.lua
- `Position.IsReachable(fromOrTarget: table|Position|nil, toOrFrom?: table) -> boolean`
- `Position.IsShootable(fromOrTarget: table|Position|nil, toOrFrom?: table) -> boolean`
- `Position.New(x: number|table, y?: number, z?: number) -> Position`
- `Position:DistanceTo(otherPos: table|Position) -> integer`

### self.lua
- `Self.Attack(creatureId: integer) -> any`
- `Self.BuyItem(itemId: integer, itemCount: integer, ignoreCapacity?: boolean, buyInShoppingBags?: boolean) -> any`
- `Self.CancelWalk() -> any`
- `Self.Dismount() -> any`
- `Self.Equip(itemId: integer, tierLevel?: integer) -> any`
- `Self.Follow(creatureId: integer) -> any`
- `Self.FormatStatsSnapshot(stats?: table, prefix?: string) -> string`
- `Self.GetCapacity() -> number|nil`
- `Self.GetCapacityFloor() -> integer|nil`
- `Self.GetCharacterWorld(characterName: string) -> string|nil`
- `Self.GetFollowId() -> integer|nil`
- `Self.GetHealth() -> integer|nil`
- `Self.GetHealthPercentage() -> number|nil`
- `Self.GetLevel() -> integer|nil`
- `Self.GetLevelPercentage() -> number|nil`
- `Self.GetMana() -> integer|nil`
- `Self.GetManaPercentage() -> number|nil`
- `Self.GetManaShieldCapacity() -> integer|nil`
- `Self.GetMaxHealth() -> integer|nil`
- `Self.GetMaxMana() -> integer|nil`
- `Self.GetMaxManaShieldCapacity() -> integer|nil`
- `Self.GetMousePositionInWorld() -> table|nil`
- `Self.GetMousePositionText() -> string`
- `Self.GetMouseWorldX() -> number|nil`
- `Self.GetMouseWorldY() -> number|nil`
- `Self.GetMouseWorldZ() -> number|nil`
- `Self.GetSoul() -> integer|nil`
- `Self.GetStamina() -> integer|nil`
- `Self.GetStaminaDays() -> integer|nil`
- `Self.GetStaminaHours() -> integer|nil`
- `Self.GetStatsSnapshot() -> table`
- `Self.GetStatusFlagsSnapshot() -> table`
- `Self.GetTargetId() -> integer|nil`
- `Self.HasFollow() -> boolean|nil`
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
- `Self.IsRooted() -> boolean|nil`
- `Self.IsStrengthened() -> boolean|nil`
- `Self.LookAtCreature(creatureId: integer) -> any`
- `Self.LookAtPosition(position: table) -> any`
- `Self.Mount() -> any`
- `Self.PrivateMessage(playerName: string, message: string) -> boolean`
- `Self.Say(message: string) -> boolean`
- `Self.SayOnChannel(message: string, channelId: integer) -> any`
- `Self.SayToNpc(message: string) -> boolean`
- `Self.SellItem(itemId: integer, itemCount: integer, sellEquipped?: boolean) -> any`
- `Self.Step(direction: integer) -> any`
- `Self.StopAttackAndFollow() -> any`
- `Self.UseItemInContainer(itemId: integer, containerIndex: integer, itemPos: integer, useItemWithHotkey?: boolean) -> any`
- `Self.UseItemOnFloor(position: table, stackPosition: integer, itemId: integer) -> any`
- `Self.Whisper(message: string) -> boolean`
- `Self.Yell(message: string) -> boolean`

### sound.lua
- `BotSoundId.CREATURE_DETECTED = 5`
- `BotSoundId.CUSTOM_SOUND_1 = 14`
- `BotSoundId.CUSTOM_SOUND_2 = 15`
- `BotSoundId.CUSTOM_SOUND_3 = 16`
- `BotSoundId.CUSTOM_SOUND_4 = 17`
- `BotSoundId.CUSTOM_SOUND_5 = 18`
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
- `Sound.ClearQueue() -> boolean`
- `Sound.GetCurrentDuration() -> integer`
- `Sound.GetFileDuration(filePath: string) -> integer`
- `Sound.GetQueueSize() -> integer`
- `Sound.IsPlaying() -> boolean`
- `Sound.IsQueued(options: table) -> boolean`
- `Sound.Play(options: table) -> boolean`
- `Sound.SetMinDelay(delayMs: integer) -> boolean`
- `Sound.Stop() -> boolean`

### spells.lua
- `Spells.GetGroupIds(spellOrWordsOrId)`
- `Spells.GetIdByName(name)`
- `Spells.GetIdByWords(words)`
- `Spells.GetInfo(spellOrWordsOrId)`
- `Spells.GetLeftCooldownTime(spellOrWordsOrId)`
- `Spells.GetLeftGroupCooldownTime(groupId)`
- `Spells.GetWordsById(spellId)`
- `Spells.GroupIsInCooldown(groupId)`
- `Spells.IsInCooldown(spellOrWordsOrId)`
- `Spells.IsReady(spellOrWordsOrId)`
- `Spells.IsUseWithItemExhausted()`
- `Spells.Item.GetCooldownId(itemId)`
- `Spells.Item.GetGroupIds(itemId)`
- `Spells.Item.GetInfo(itemId)`
- `Spells.Item.GetLeftCooldownTime(itemId)`
- `Spells.Item.IsInCooldown(itemId)`
- `Spells.Item.IsReady(itemId)`
- `Spells.Item.WillBeReady(itemId, timeMs)`
- `Spells.WillBeReady(spellOrWordsOrId, timeMs)`

### storage.lua
- `Storage.Character.Clear() -> boolean`
- `Storage.Character.Get(key: string, default?: any) -> any`
- `Storage.Character.Remove(key: string) -> boolean`
- `Storage.Character.Set(key: string, value: any) -> boolean`
- `Storage.ForCharacter(namespace: string) -> StorageScope`
- `Storage.Global.Clear() -> boolean`
- `Storage.Global.Get(key: string, default?: any) -> any`
- `Storage.Global.Remove(key: string) -> boolean`
- `Storage.Global.Set(key: string, value: any) -> boolean`
- `Storage.Namespace(namespace: string, perCharacter?: boolean) -> StorageScope`
- `StorageScope:Get(key: string, default?: any) -> any`
- `StorageScope:Remove(key: string) -> boolean`
- `StorageScope:Set(key: string, value: any) -> boolean`

### vip.lua
- `VIP.Count() -> integer`
- `VIP.CountOnline() -> integer`
- `VIP.Exists(vipName: string) -> boolean`
- `VIP.FindByPrefix(namePrefix: string, onlyOnline?: boolean) -> table[]`
- `VIP.Get(vipName: string) -> table|nil`
- `VIP.GetAll() -> table[]`
- `VIP.GetByType(vipType: integer) -> table[]`
- `VIP.GetDescription(vipName: string) -> string|nil`
- `VIP.GetHearts() -> table[]`
- `VIP.GetNames(onlyOnline?: boolean) -> string[]`
- `VIP.GetNotifyOnLogin(vipName: string) -> boolean|nil`
- `VIP.GetSnapshot() -> table`
- `VIP.GetType(vipName: string) -> integer|nil`
- `VIP.IsAvailable() -> boolean`
- `VIP.IsHeart(vipName: string) -> boolean`
- `VIP.IsOnline(vipName: string) -> boolean`
- `VIP.ToLookupTable() -> table`

### websocket.lua
- `WebSocket.Connect(url: string, options?: table) -> table|nil, string|nil`
- `WebSocketConnection:Close(closeCode?: integer, reason?: string) -> boolean, string|nil`
- `WebSocketConnection:IsOpen() -> boolean`
- `WebSocketConnection:Receive(timeoutMs?: integer) -> table`
- `WebSocketConnection:Send(data: string, binary?: boolean) -> boolean, string|nil`
