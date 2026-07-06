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

This file now includes an auto-generated exhaustive API appendix (function name, params, returns) from Scripts/core to improve reliability when generating scripts.

## Runtime Model
- Scripts run inside the bot Lua runtime and rely on C++ bindings exposed as global tables.
- Core helper libraries are loaded from Scripts/core.
- User script public API uses canonical PascalCase module access (for example Self, Map, Container, Module, ChatChannelStorage, VIP).
- Direct/raw C++ binding globals are internal implementation details unless explicitly documented in this file.
- Script execution is cooperative. Long loops must yield (for example via wait(...)).
- Some APIs may return nil when game state or required bindings are unavailable.

## Hard Rules for Generated Scripts
1. Do not use os.exit, os.execute, or any process termination behavior.
2. Never disable or query the internal Objects Dumper feature through Features API.
3. Avoid busy loops. Any repeating logic must yield with wait(ms) or use Module helpers (Module.Every / Module.After).
4. Validate all external input and table fields before use.
5. Assume runtime values can be nil and handle fallbacks safely.
6. Use only documented PascalCase core wrappers in Scripts/core (Self.*, Map.*, Module.*, etc.).
7. Avoid hard crashes: use guards, type checks, and pcall where appropriate.
8. Keep scripts idempotent when possible (safe to re-run).
9. Use explicit comments for non-obvious logic.
10. Keep feature toggling scoped to user features only.
11. Do not use removed compatibility aliases such as self.*, map.*, engine.ammoRefill.*, CaveBot, or _G.position.
12. Do not call raw/internal C++ globals directly unless they are documented here as public API; prefer core wrappers such as Self, Creature, Creatures, Cavebot, Engine, Game, and Module.

## Public Module Naming
- Module tables are exposed for user scripts in PascalCase form.
- Each module surfaces grouped helpers:
  - query functions in Module.Query.*
  - action functions in Module.Actions.*
- Direct module methods are available in PascalCase form only.
- Compatibility aliases were removed; generated scripts must use the exact names documented in this file.

## Known Constraints
- Hotkeys: alt modifier is not supported by current Events.RegisterKeyEvent API.
- Standard Lua `table.concat` is allowed in sandboxed user scripts and can be used for safe string assembly.
- Extended keys: insert/delete/home/end/pageup/pagedown/arrows default to extended=true in Hotkeys.ParseCombo.
- Features API excludes Objects Dumper from public get/set/list/status paths.

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
- Position:IsReachable(fromPos)
- Position:IsShootable(fromPos)

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
- relation/type: IsPlayer, IsMonster, IsNPC, IsSummon, IsGameMaster
- stats/meta: GetHealthPercent, GetSpeed, GetVocation, GetSkull, GetOutfit
- spatial: GetPosition, DistanceTo, IsAdjacentTo, IsSameFloor, IsReachable, IsShootable
- utilities: Equals, ToString

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
Account/session helpers plus raw game-action wrappers.

Login/session API:
- Game.GetCharacterWorld(characterName)
- Game.LoginToPreviouslyLoggedCharacter()
- Game.LoginToAccount(email, password)
- Game.LoginToCharacter(characterName)
- Game.Logout()
- Game.EnterWorld()

Notes:
- Login functions validate required string arguments and return boolean success from the Lua wrapper.
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

### cooldowns.lua
Cooldown abstraction helpers.

Namespaces:
- Cooldowns.Spell: IsInCooldown, GetTimeLeft, WillBeReady, IsReady
- Cooldowns.Item: IsInCooldown, GetTimeLeft, WillBeReady, IsReady
- Cooldowns.Group: IsInCooldown, GetTimeLeft, WillBeReady, IsReady
- Cooldowns.UseWith: IsExhausted, IsReady
- Cooldowns.Utils: FormatTime, GetStatus, PrintStatus

Runtime Cooldown metadata methods:
- Cooldown.GetSpellIdByWords(words)
- Cooldown.GetSpellIdByName(name)
- Cooldown.GetSpellWordsById(spellId)
- Cooldown.GetSpellGroupCooldownIdsByWords(words)
- Cooldown.GetItemCooldownId(itemId)
- Cooldown.GetItemGroupCooldownIds(itemId)

### spells.lua
ZeroBot-style spell cooldown facade.

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
Packet/event proxy wrappers for common event categories.

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

### engine.lua
High-level grouped wrappers around Healer, Alarms, AmmoRefill and related systems.

Primary namespaces:
- Engine.Healer
- Engine.Alarms
- Engine.AmmoRefill

Use this file as convenience layer when your script requires advanced feature configuration workflows.

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

Waypoint script note:
- Script waypoints should call Cavebot.Walker.Resume() or Walker.Resume() when they are done.
- The DLL has a safety path that automatically resumes the walker and shows a warning when a waypoint script exits without resuming.
- If the walker is disabled while a waypoint Lua script is still running, waypoint scripts are stopped so the walker can be re-enabled cleanly.

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
Sound helper and convenience wrappers.

Includes:
- BotSoundId enum-like table
- Sound.Play / Stop / ClearQueue / GetQueueSize / IsPlaying / IsQueued / duration helpers
- sound.lua helper globals such as soundPlayById, soundPlayByName, soundPlayFile, smart dedup variants, and notification shortcuts
- convenience notification shortcuts (low health, low mana, etc.)

### hud_wrapper.lua
UI drawing wrapper classes for screen/world overlays.

Core classes:
- ScreenText
- ScreenImage
- WorldText
- WorldBox
- WorldImage

Use when script needs visual diagnostics, labels, or map markers.

Screen image notes:
- ScreenImage:SetLabel(text, color, offsetX, offsetY) attaches or updates text on the image.
- ScreenImage:SetScreenPosition(x, y) positions image HUD elements in screen pixels.
- ScreenImage:SetParent(parent_id) can parent icons to a draggable ScreenText handle; children keep their current offset when the parent moves.

### lua_consts.lua
Shared constants/enums for opcodes, movement, effects, categories, etc.

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

Do not generate scripts that call old lower-camel names, raw internal creature globals, or raw internal game globals. Use the documented PascalCase modules instead.

## Script Safety Checklist
Before finalizing any script, verify:
1. All loops yield (wait/module schedule) and cannot freeze runtime.
2. Every table access checks nil/type when data can be absent.
3. Feature operations do not touch reserved internal features.
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

## Appendix A: Core Lua Library Index (Auto-Generated)
Generated strictly from files in docs/Scripts/core. This appendix does not include raw/native bindings unless a core Lua file defines or exports them. If a function/file is missing here, treat it as unavailable from the core Lua libraries.

### cavebot.lua
Exported tables/constants:
- Cavebot

Functions:
- Cavebot.Disable()
- Cavebot.DisableLure()
- Cavebot.Enable()
- Cavebot.EnableLure()
- Cavebot.GetStatus()
- Cavebot.GoTo(labelName)
- Cavebot.GoToLabel(labelName)
- Cavebot.IsEnabled()
- Cavebot.IsLureEnabled()
- Cavebot.Lure.AddSetting(setting)
- Cavebot.Lure.ClearSettings()
- Cavebot.Lure.EndForceLure()
- Cavebot.Lure.GetAttackWhileLuring()
- Cavebot.Lure.GetConsiderOnlyReachable()
- Cavebot.Lure.GetIgnoringMonsters()
- Cavebot.Lure.GetLuredCreaturesCount()
- Cavebot.Lure.GetNearRange()
- Cavebot.Lure.GetOption()
- Cavebot.Lure.GetSettingCount()
- Cavebot.Lure.GetSettings()
- Cavebot.Lure.GetSlowWalkBurstSteps()
- Cavebot.Lure.GetSlowWalkDelayMs()
- Cavebot.Lure.GetSlowWalkingCreaturesCount()
- Cavebot.Lure.GetStartEndLureActive()
- Cavebot.Lure.GetState()
- Cavebot.Lure.GetUnblocking()
- Cavebot.Lure.GetWaypointDynamicLureActive()
- Cavebot.Lure.HasActiveSettings()
- Cavebot.Lure.IsEnabled()
- Cavebot.Lure.IsFighting()
- Cavebot.Lure.IsForceLure()
- Cavebot.Lure.IsLuring()
- Cavebot.Lure.IsOtherPlayerOnScreen()
- Cavebot.Lure.RemoveSetting(index)
- Cavebot.Lure.SetAttackWhileLuring(enabled)
- Cavebot.Lure.SetConsiderOnlyReachable(enabled)
- Cavebot.Lure.SetEnabled(enabled)
- Cavebot.Lure.SetForceLure(enabled)
- Cavebot.Lure.SetIgnoringMonsters(enabled)
- Cavebot.Lure.SetNearRange(range)
- Cavebot.Lure.SetOption(option)
- Cavebot.Lure.SetSlowWalkBurstSteps(steps)
- Cavebot.Lure.SetSlowWalkDelayMs(delayMs)
- Cavebot.Lure.SetSlowWalkingCreaturesCount(count)
- Cavebot.Lure.SetStartEndLureActive(enabled)
- Cavebot.Lure.SetUnblocking(enabled)
- Cavebot.Lure.SetWaypointDynamicLureActive(enabled)
- Cavebot.Lure.UpdateSetting(index, updateData)
- Cavebot.OnAction(callback)
- Cavebot.OnLabel(callback)
- Cavebot.OnWaypointChange(callback)
- Cavebot.Pause(milliseconds, autoResume)
- Cavebot.PrintStatus()
- Cavebot.RegisterEvent(eventId, callback)
- Cavebot.Resume()
- Cavebot.SetEnabled(enabled)
- Cavebot.SetEnginesEnabled(walkerEnabled, lureEnabled)
- Cavebot.SetLureEnabled(enabled)
- Cavebot.UnregisterAllEvents()
- Cavebot.Walker.AddWaypoint(waypoint)
- Cavebot.Walker.ClearWaypoints()
- Cavebot.Walker.DeleteWaypoint(index)
- Cavebot.Walker.GetAutoRecorderEnabled()
- Cavebot.Walker.GetAutoRecorderOptions()
- Cavebot.Walker.GetDebugHud()
- Cavebot.Walker.GetDistanceBetweenWaypoints()
- Cavebot.Walker.GetLeaveLureOnPlayer()
- Cavebot.Walker.GetNodeDistance()
- Cavebot.Walker.GetSelectedWaypointIndex()
- Cavebot.Walker.GetStartFromNearestWaypoint()
- Cavebot.Walker.GetWalkToLureCenter()
- Cavebot.Walker.GetWaypointCount()
- Cavebot.Walker.GetWaypoints()
- Cavebot.Walker.GoTo(labelName)
- Cavebot.Walker.InsertWaypoint(index, waypoint)
- Cavebot.Walker.IsEnabled()
- Cavebot.Walker.IsPausedByLua()
- Cavebot.Walker.IsStuck()
- Cavebot.Walker.MoveWaypointDown(index)
- Cavebot.Walker.MoveWaypointUp(index)
- Cavebot.Walker.ReplaceWaypoint(index, waypoint)
- Cavebot.Walker.Resume()
- Cavebot.Walker.SelectClosestWaypoint()
- Cavebot.Walker.SetAutoRecorderEnabled(enabled)
- Cavebot.Walker.SetAutoRecorderOptions(options)
- Cavebot.Walker.SetDebugHud(enabled)
- Cavebot.Walker.SetDistanceBetweenWaypoints(distance)
- Cavebot.Walker.SetEnabled(enabled)
- Cavebot.Walker.SetLeaveLureOnPlayer(enabled)
- Cavebot.Walker.SetNodeDistance(distance)
- Cavebot.Walker.SetPausedByLua(paused)
- Cavebot.Walker.SetSelectedWaypointIndex(index)
- Cavebot.Walker.SetStartFromNearestWaypoint(enabled)
- Cavebot.Walker.SetWalkToLureCenter(enabled)

Internal local helpers, not public API:
- call_events
- call_features
- call_lure
- call_walker
- get_feature_id
- normalize_waypoint_table
- require_table

### cavebot_actions.lua
Exported tables/constants:
- Cavebot

Functions:
- Cavebot.Actions.GetLastResult()
- Cavebot.Actions.Register(actionType, handler)
- Cavebot.Actions.Run(context)

Internal local helpers, not public API:
- as_number
- buy_item_from_npc
- buy_supplies_handler
- call_no_throw
- check_supplies_handler
- count_item_in_open_containers
- custom_script_handler
- finish
- get_action_config
- get_action_params
- get_supply_profile
- get_vendor_profile
- log_action_result
- non_empty_string
- normalize_context
- npc_say_handler
- pending_handler
- require_table
- route_to_label
- run_vendor_talk_sequence
- sell_item_to_npc
- sell_loot_handler
- talk_to_npc

### chat_channel.lua
Exported tables/constants:
- ChatChannel

Functions:
- ChatChannel.FromIdentifier(identifier)
- ChatChannel.GetById(channelId)
- ChatChannel.GetByName(channelName)
- ChatChannel.New(channelOrId, channelName)
- ChatChannel:CanSend()
- ChatChannel:GetId()
- ChatChannel:GetName()
- ChatChannel:IsLocal()
- ChatChannel:IsOpened()
- ChatChannel:IsServerLog()
- ChatChannel:IsValid()
- ChatChannel:Refresh()
- ChatChannel:Send(message)
- ChatChannel:ToString()
- ChatChannel:ToTable()

Internal local helpers, not public API:
- normalize_from_table
- resolve_from_storage
- validate_non_empty_string
- validate_non_negative_integer

### chat_channel_storage.lua
Exported tables/constants:
- ChatChannels
- ChatChannelStorage

Functions:
- ChatChannelStorage.CanSend(channelIdentifier)
- ChatChannelStorage.FormatChannel(channel)
- ChatChannelStorage.GetChannelNames(onlySendable)
- ChatChannelStorage.GetChatChannelById(channelId)
- ChatChannelStorage.GetChatChannelByName(channelName)
- ChatChannelStorage.GetChatChannelCount()
- ChatChannelStorage.GetChatChannels()
- ChatChannelStorage.GetLocalChatChannel()
- ChatChannelStorage.GetOpenedChannelCount()
- ChatChannelStorage.GetOpenedChannels()
- ChatChannelStorage.GetServerLogChannel()
- ChatChannelStorage.GetSnapshot()
- ChatChannelStorage.HasChannelById(channelId)
- ChatChannelStorage.HasChannelByName(channelName)
- ChatChannelStorage.IsAvailable()
- ChatChannelStorage.ResolveChannel(channelIdentifier)
- ChatChannelStorage.Send(message, channelIdentifier)
- ChatChannelStorage.ToIdLookupTable()
- ChatChannelStorage.ToNameLookupTable()

Internal local helpers, not public API:
- build_opened_channels_snapshot
- build_sendable_channels_snapshot
- call_backend_method
- dedupe_channels
- find_channel_by_id
- find_channel_by_name
- get_any_field_value
- get_any_method_value
- get_raw_channel_by_id
- get_raw_channel_by_name
- get_raw_chat_channels
- get_raw_local_chat_channel
- get_raw_opened_channels
- get_raw_server_log_channel
- normalize_channel_entry
- normalize_channel_id
- normalize_channel_name
- resolve_backend_table
- sort_channels
- to_channel_identifier_string
- to_lower_safe
- validate_non_empty_string
- validate_non_negative_integer

### container.lua
Exported tables/constants:
- Container

Functions:
- Container.FindItem(containerNumber, itemId, tierLevel)
- Container.FindItemInOpenContainers(itemId, tierLevel)
- Container.GetById(containerId)
- Container.GetByName(containerName)
- Container.GetByNumber(containerNumber)
- Container.GetFreeSlots(containerNumber)
- Container.GetId(containerNumber)
- Container.GetItem(containerNumber, slotIndex)
- Container.GetItems(containerNumber)
- Container.GetItemsCount(containerNumber)
- Container.GetName(containerNumber)
- Container.GetOpenContainers()
- Container.GetSize(containerNumber)
- Container.LookItem(itemId, itemPos, containerIndex)
- Container.MoveItemFromEquipmentToContainer(equipmentSlot, containerIndex, slotIndex, itemId, itemCount)
- Container.MoveItemToContainer(fromContainerIndex, fromSlotIndex, itemId, toContainerIndex, toSlotIndex, itemCount)
- Container.MoveItemToEquipment(containerIndex, slotIndex, itemId, equipmentSlot, itemCount)
- Container.MoveItemToFloor(containerIndex, slotIndex, itemId, toPosition, itemCount)
- Container.UseItem(itemId, containerIndex, itemPos, useItemWithHotkey)

Internal local helpers, not public API:
- validate_integer
- validate_non_negative_integer
- validate_position_table
- validate_positive_integer

### cooldowns.lua
Exported tables/constants:
- Cooldowns

Functions:
- Cooldowns.Group.GetTimeLeft(groupId)
- Cooldowns.Group.IsInCooldown(groupId)
- Cooldowns.Group.IsReady(groupId)
- Cooldowns.Group.WillBeReady(groupId, timeMs)
- Cooldowns.Item.GetTimeLeft(itemId)
- Cooldowns.Item.IsInCooldown(itemId)
- Cooldowns.Item.IsReady(itemId)
- Cooldowns.Item.WillBeReady(itemId, timeMs)
- Cooldowns.Spell.GetTimeLeft(spellWords)
- Cooldowns.Spell.IsInCooldown(spellWords)
- Cooldowns.Spell.IsReady(spellWords)
- Cooldowns.Spell.WillBeReady(spellWords, timeMs)
- Cooldowns.UseWith.IsExhausted()
- Cooldowns.UseWith.IsReady()
- Cooldowns.Utils.FormatTime(ms)
- Cooldowns.Utils.GetStatus(spells)
- Cooldowns.Utils.PrintStatus(spells)

### creature.lua
Exported tables/constants:
- Creature

Functions:
- Creature.GetFollowed()
- Creature.GetLocalPlayer()
- Creature.GetTarget()
- Creature:ClearCache()
- Creature:DistanceTo(targetPos)
- Creature:DistanceToCreature(otherCreature)
- Creature:Equals(other)
- Creature:GetDirection()
- Creature:GetGuildShield()
- Creature:GetHealthPercent()
- Creature:GetId()
- Creature:GetLowercaseName()
- Creature:GetMasterId()
- Creature:GetName()
- Creature:GetOutfit()
- Creature:GetPartyShield()
- Creature:GetPosition()
- Creature:GetSkull()
- Creature:GetSpeed()
- Creature:GetVocation()
- Creature:IsAdjacentTo(targetPos)
- Creature:IsGameMaster()
- Creature:IsInGuild()
- Creature:IsInParty()
- Creature:IsMonster()
- Creature:IsMounted()
- Creature:IsNPC()
- Creature:IsPartyLeader()
- Creature:IsPlayer()
- Creature:IsReachable()
- Creature:IsSameFloor(targetPos)
- Creature:IsShootable()
- Creature:IsSkulled()
- Creature:IsSummon()
- Creature:IsValid()
- Creature:IsVisible()
- Creature:IsWarAlly()
- Creature:IsWarEnemy()
- Creature:New(creatureId)
- Creature:ToString()

### creature_iterators.lua
Exported tables/constants:
- Creature
- Creatures

Functions:
- Creature.ICreatures()
- Creature.IMonsters()
- Creature.INpcs()
- Creature.IPlayers()
- Creatures.GetAttackingCreatureId()
- Creatures.GetCreatureByName(creatureName)
- Creatures.GetCreatureIdsByScan(typeFlags, xRelativeDistance, yRelativeDistance, multifloor, ignoreSummons)
- Creatures.GetCreaturesByScan(typeFlags, xRelativeDistance, yRelativeDistance, multifloor, ignoreSummons)
- Creatures.GetFollowingCreatureId()
- Creatures.GetLocalPlayerId()
- Creatures.GetPlayerIdUnderMouse()
- Creatures.GetVisibleCreatureIds()
- Creatures.GetVisibleCreatures()
- Creatures.GetVisibleMonsters(ignoreSummons)
- Creatures.GetVisibleNpcs()
- Creatures.GetVisiblePlayers()
- Creatures.IsCreatureOnScreen(creatureId, xRelativeDistance, yRelativeDistance, multifloor)

Internal local helpers, not public API:
- make_iterator

### engine.lua
Exported tables/constants:
- Engine

Functions:
- Engine.Alarms.Disable(alarmId)
- Engine.Alarms.DisableAll()
- Engine.Alarms.Enable(alarmId)
- Engine.Alarms.EnableAll()
- Engine.Alarms.EnableOnly(alarmIdsList)
- Engine.Alarms.GetCreatureFilter()
- Engine.Alarms.GetLowHealthThreshold()
- Engine.Alarms.GetLowManaThreshold()
- Engine.Alarms.GetMessageFilter()
- Engine.Alarms.IsBringToFocusEnabled()
- Engine.Alarms.IsEnabled(alarmId)
- Engine.Alarms.IsFlashWindowEnabled()
- Engine.Alarms.IsIgnoringAllyPlayers()
- Engine.Alarms.PrintStatus()
- Engine.Alarms.SetBringToFocus(enabled)
- Engine.Alarms.SetCreatureFilter(namesString)
- Engine.Alarms.SetFlashWindow(enabled)
- Engine.Alarms.SetIgnoreAllyPlayers(ignore)
- Engine.Alarms.SetLowHealthThreshold(percentage)
- Engine.Alarms.SetLowManaThreshold(percentage)
- Engine.Alarms.SetMessageFilter(messagesString)
- Engine.Alarms.Toggle(alarmId)
- Engine.AmmoRefill.Add(ammoData)
- Engine.AmmoRefill.AddProfile(profileName)
- Engine.AmmoRefill.ClearAll()
- Engine.AmmoRefill.Disable(index)
- Engine.AmmoRefill.DisableAll()
- Engine.AmmoRefill.Enable(index)
- Engine.AmmoRefill.EnableAll()
- Engine.AmmoRefill.EnableOnly(itemIdsList)
- Engine.AmmoRefill.FindByItemId(itemId)
- Engine.AmmoRefill.FindProfileByName(profileName)
- Engine.AmmoRefill.Get(index)
- Engine.AmmoRefill.GetAll()
- Engine.AmmoRefill.GetCurrentProfile()
- Engine.AmmoRefill.GetProfileNames()
- Engine.AmmoRefill.Modify(index, refillAtCount, refillInLeftHand, equipFromHotkey)
- Engine.AmmoRefill.PrintProfiles()
- Engine.AmmoRefill.PrintStatus()
- Engine.AmmoRefill.Remove(index)
- Engine.AmmoRefill.RemoveProfile(indexOrName)
- Engine.AmmoRefill.RenameProfile(indexOrName, newName)
- Engine.AmmoRefill.SetProfile(indexOrName)
- Engine.AmmoRefill.Toggle(index)
- Engine.AmmoRefill.Update(index, updateData)
- Engine.Healer.AddItem(itemData)
- Engine.Healer.AddSpell(spellData)
- Engine.Healer.ClearAllItems()
- Engine.Healer.ClearAllSpells()
- Engine.Healer.DisableAllItems()
- Engine.Healer.DisableAllSpells()
- Engine.Healer.DisableItem(index)
- Engine.Healer.DisableSpell(index)
- Engine.Healer.EnableItem(index)
- Engine.Healer.EnableOnlyItems(itemIdsList)
- Engine.Healer.EnableOnlySpells(spellWordsList)
- Engine.Healer.EnableSpell(index)
- Engine.Healer.FindItemById(itemId)
- Engine.Healer.FindSpellByWords(spellWords)
- Engine.Healer.GetItems()
- Engine.Healer.GetSpellByIndex(index)
- Engine.Healer.GetSpells()
- Engine.Healer.ModifySpell(index, castValue, manaCost)
- Engine.Healer.PrintItems()
- Engine.Healer.PrintSpells()
- Engine.Healer.RemoveItem(index)
- Engine.Healer.RemoveSpell(index)
- Engine.Healer.ToggleItem(index)
- Engine.Healer.ToggleSpell(index)

### event_proxies.lua
Exported tables/constants:
- BattleMessageProxy
- ContainerAddItemProxy
- ContainerCloseProxy
- ContainerOpenProxy
- ContainerRemoveItemProxy
- ContainerUpdateItemProxy
- CreatureAddProxy
- CreatureRemoveProxy
- DeathProxy
- GenericTextMessageProxy
- LootMessageProxy
- SkillsChangeProxy
- StatsChangeProxy

Functions:
- BattleMessageProxy:GetName()
- BattleMessageProxy:New(name)
- BattleMessageProxy:OnReceive(callback)
- ContainerAddItemProxy:GetName()
- ContainerAddItemProxy:New(name)
- ContainerAddItemProxy:OnReceive(callback)
- ContainerCloseProxy:GetName()
- ContainerCloseProxy:New(name)
- ContainerCloseProxy:OnReceive(callback)
- ContainerOpenProxy:GetName()
- ContainerOpenProxy:New(name)
- ContainerOpenProxy:OnReceive(callback)
- ContainerRemoveItemProxy:GetName()
- ContainerRemoveItemProxy:New(name)
- ContainerRemoveItemProxy:OnReceive(callback)
- ContainerUpdateItemProxy:GetName()
- ContainerUpdateItemProxy:New(name)
- ContainerUpdateItemProxy:OnReceive(callback)
- CreatureAddProxy:GetName()
- CreatureAddProxy:New(name)
- CreatureAddProxy:OnReceive(callback)
- CreatureRemoveProxy:GetName()
- CreatureRemoveProxy:New(name)
- CreatureRemoveProxy:OnReceive(callback)
- DeathProxy:GetName()
- DeathProxy:New(name)
- DeathProxy:OnReceive(callback)
- GenericTextMessageProxy:GetName()
- GenericTextMessageProxy:New(name)
- GenericTextMessageProxy:OnReceive(callback)
- LootMessageProxy:GetName()
- LootMessageProxy:New(name)
- LootMessageProxy:OnReceive(callback)
- SkillsChangeProxy:GetName()
- SkillsChangeProxy:New(name)
- SkillsChangeProxy:OnReceive(callback)
- StatsChangeProxy:GetName()
- StatsChangeProxy:New(name)
- StatsChangeProxy:OnReceive(callback)

Internal local helpers, not public API:
- attach_proxy_pascal_methods
- internalPacketHandler

### features.lua
Exported tables/constants:
- BotFeatureId
- BotFeatureName
- ExcludeList
- Features

Functions:
- Features.Disable(featureIdentifier)
- Features.DisableAllExcept(excludeList)
- Features.DisableMultiple(featureList)
- Features.Enable(featureIdentifier)
- Features.EnableMultiple(featureList)
- Features.GetActiveFeatures()
- Features.GetAllFeatureIds()
- Features.GetName(featureIdentifier)
- Features.IsActive(featureIdentifier)
- Features.PrintStatus()
- Features.SetActive(featureIdentifier, activeStatus)
- Features.Toggle(featureIdentifier)
- featuresDisable(featureIdentifier)
- featuresDisableAllExcept(ExcludeList)
- featuresDisableMultiple(featureList)
- featuresEnable(featureIdentifier)
- featuresEnableMultiple(featureList)
- featuresGetActiveFeatures()
- featuresGetAllFeatureIds()
- featuresGetName(featureIdentifier)
- featuresIsActive(featureIdentifier)
- featuresPrintStatus()
- featuresSetActive(featureIdentifier, activeStatus)
- featuresToggle(featureIdentifier)

Internal local helpers, not public API:
- extendFeaturesTable
- initializeCppFunctions
- isReservedInternalFeature
- resolveFeatureId
- validateFeatureId

### game.lua
Exported tables/constants:
- Game

Functions:
- Game.EnterWorld()
- Game.GetCharacterWorld(characterName)
- Game.LoginToAccount(email, password)
- Game.LoginToCharacter(characterName)
- Game.LoginToPreviouslyLoggedCharacter()
- Game.Logout()

Internal local helpers, not public API:
- require_native_function
- validate_non_empty_string

### hotkeys.lua
Exported tables/constants:
- Hotkeys

Functions:
- Hotkeys.ParseCombo(combination)
- Hotkeys.RegisterCombo(params)

Internal local helpers, not public API:
- normalize_part

### hud_wrapper.lua
Exported tables/constants:
- HorizontalAlignment
- ScreenImage
- ScreenText
- VerticalAlignment
- WorldBox
- WorldImage
- WorldText

Functions:
- ScreenImage:ClearParent()
- ScreenImage:Create()
- ScreenImage:New(id)
- ScreenImage:Remove()
- ScreenImage:SetAlignment(h_align, v_align)
- ScreenImage:SetClickable(callback)
- ScreenImage:SetDraggable(draggable)
- ScreenImage:SetEnabled(enabled)
- ScreenImage:SetItemId(itemId)
- ScreenImage:SetItemName(itemName)
- ScreenImage:SetLabel(text, color, offsetX, offsetY)
- ScreenImage:SetParent(parent_id)
- ScreenImage:SetScreenPosition(x, y)
- ScreenImage:SetSize(width, height)
- ScreenImage:SetSource(path)
- ScreenText:ClearParent()

Notes:
- ScreenImage:SetLabel(...) updates the label attached to an existing created image; use it after toggles to refresh ON/OFF text.
- ScreenImage:SetScreenPosition(x, y) can be called before or after Create(); if called before Create(), the saved screen position is applied immediately after creation.
- Screen HUD children follow parent screen movement after `SetParent(parent_id)`, including drag movement, while preserving the offset they had when parented. Position the child first, then call `SetParent(parent.id)`.
- Low-level HUD bindings for the same behavior are HUD.SetScreenPosition({ id = id, x = x, y = y }), HUD.SetParent(child_id, parent_id), HUD.ClearParent(child_id), and HUD.UpdateImageLabel({ id = id, label = text, label_color = color, label_offset_x = x, label_offset_y = y }).

- ScreenText:Create()
- ScreenText:GetColor()
- ScreenText:GetEnabled()
- ScreenText:GetHeight()
- ScreenText:GetPosition()
- ScreenText:GetText()
- ScreenText:GetVisible()
- ScreenText:GetWidth()
- ScreenText:IsCreated()
- ScreenText:New(id)
- ScreenText:Remove()
- ScreenText:SetAlignment(h_align, v_align)
- ScreenText:SetClickable(callback)
- ScreenText:SetColor(color)
- ScreenText:SetDraggable(draggable)
- ScreenText:SetEnabled(enabled)
- ScreenText:SetParent(parent_id)
- ScreenText:SetScreenPosition(x, y)
- ScreenText:SetText(text)
- WorldBox:ClearParent()
- WorldBox:Create()
- WorldBox:GetColor()
- WorldBox:GetEnabled()
- WorldBox:GetHeight()
- WorldBox:GetPosition()
- WorldBox:GetVisible()
- WorldBox:GetWidth()
- WorldBox:IsCreated()
- WorldBox:New(id, x, y, z)
- WorldBox:Remove()
- WorldBox:SetBorderColor(border_color)
- WorldBox:SetBorderWidth(border_width)
- WorldBox:SetColor(color)
- WorldBox:SetEnabled(enabled)
- WorldBox:SetHeight(height)
- WorldBox:SetLifetime(lifetime_ms)
- WorldBox:SetParent(parent_id)
- WorldBox:SetPosition(x, y, z)
- WorldBox:SetSize(width, height)
- WorldBox:SetWidth(width)
- WorldImage:ClearParent()
- WorldImage:Create()
- WorldImage:New(id, x, y, z)
- WorldImage:Remove()
- WorldImage:SetEnabled(enabled)
- WorldImage:SetItemId(itemId)
- WorldImage:SetItemName(itemName)
- WorldImage:SetLifetime(lifetimeMs)
- WorldImage:SetOffset(offsetX, offsetY)
- WorldImage:SetParent(parent_id)
- WorldImage:SetPosition(x, y, z)
- WorldImage:SetSize(width, height)
- WorldImage:SetSource(path)
- WorldText:ClearParent()
- WorldText:Create()
- WorldText:GetColor()
- WorldText:GetEnabled()
- WorldText:GetHeight()
- WorldText:GetPosition()
- WorldText:GetText()
- WorldText:GetVisible()
- WorldText:GetWidth()
- WorldText:IsCreated()
- WorldText:New(id, x, y, z)
- WorldText:Remove()
- WorldText:SetColor(color)
- WorldText:SetEnabled(enabled)
- WorldText:SetLifetime(lifetime_ms)
- WorldText:SetOffset(offset_x, offset_y)
- WorldText:SetParent(parent_id)
- WorldText:SetPosition(x, y, z)
- WorldText:SetText(text)

Internal local helpers, not public API:
- attach_method_aliases

### inventory.lua
Exported tables/constants:
- Inventory

Functions:
- Inventory.CanMoveEquipment()
- Inventory.CanReadEquipment()
- Inventory.Equip(itemId, tierLevel)
- Inventory.GetAllSlotItems()
- Inventory.GetEquipmentSlotConstants()
- Inventory.GetSlotIds()
- Inventory.GetSlotItem(equipmentSlot)
- Inventory.GetSlotItemId(equipmentSlot)
- Inventory.GetSnapshot()
- Inventory.HasItemInSlot(equipmentSlot)
- Inventory.LookSlotItem(itemId, equipmentSlot)
- Inventory.MoveFromContainerToSlot(containerIndex, slotIndex, itemId, equipmentSlot, itemCount)
- Inventory.MoveFromSlotToContainer(equipmentSlot, containerIndex, slotIndex, itemId, itemCount)

Internal local helpers, not public API:
- call_game_no_throw
- get_equipment_slot_constants
- validate_non_negative_integer
- validate_positive_integer

### item.lua
Exported tables/constants:
- Item

Functions:
- Item.Buy(itemId, itemCount, ignoreCapacity, buyInShoppingBags)
- Item.FindInContainer(containerNumber, itemId, tierLevel)
- Item.GetDescription(itemId)
- Item.GetFromContainer(containerNumber, slotIndex)
- Item.GetInfo(itemId)
- Item.GetName(itemId)
- Item.HasFlag(itemId, fieldName)
- Item.IsContainer(itemId)
- Item.IsCreature(itemId)
- Item.IsCumulative(itemId)
- Item.IsGround(itemId)
- Item.IsLiquidContainer(itemId)
- Item.IsMovable(itemId)
- Item.IsMultiUsable(itemId)
- Item.IsTakable(itemId)
- Item.IsUsable(itemId)
- Item.Sell(itemId, itemCount, sellEquipped)
- Item.UseFromContainerOnFloor(floorPosition, fromItemId, toItemId, toStackPosition)
- Item.UseFromContainerToContainer(fromContainer, fromSlot, fromItemId, toContainer, toSlot, toItemId)
- Item.UseFromFloorToContainer(floorPosition, fromItemId, fromStackPosition, toItemId)
- Item.UseOnCreature(itemId, creatureId)

Internal local helpers, not public API:
- validate_non_negative_integer
- validate_positive_integer

### lua_consts.lua
Exported tables/constants:
- CharacterFlag
- ChaseMode
- ClientOpcodes
- CombatParam
- CooldownGroupId
- CreatureIcon
- CreatureType
- EquipmentSlot
- FightMode
- GameServerOpcodes
- GuildShield
- ItemCategory
- MagicEffectClasses
- MessageClasses
- MessageMode
- MoveDirection
- PartyShield
- PathFindFlags
- PathFindResult
- PlayerAvatarOutfitIds
- PrintMessagePosition
- PVPMode
- RotateDirection
- ShootType
- Skill
- Skull
- SpeakClasses
- VipFlag
- Vocation
- WalkerEvent

Functions: none

### map.lua
Exported tables/constants:
- Directions
- Map

Functions:
- Map.FindPath(fromPosition, toPosition, maxComplexity, flags)
- Map.GetObjectInfo(itemId)
- Map.GetTileFlags(position)
- Map.GetTileItems(position, includeCreatures)
- Map.Look(position)
- Map.MoveItemFloorToContainer(itemId, fromPosition, containerIndex, slotIndex, itemCount)
- Map.MoveItemFloorToFloor(fromPosition, itemId, toPosition, itemCount)
- Map.UseItemOnFloor(position, stackPosition, itemId)

Internal local helpers, not public API:
- validate_non_negative_integer
- validate_position_table
- validate_positive_integer

### minimap.lua
Exported tables/constants:
- Directions
- Minimap

Functions:
- Minimap.FindPath(fromPosition, toPosition, maxComplexity, flags)
- Minimap.GetTileFlags(position)
- Minimap.GetTileInfo(position, includeCreatures)
- Minimap.GetTileItems(position, includeCreatures)
- Minimap.GetTilePixelColor(position)
- Minimap.IsPathable(position)
- Minimap.IsPixelColorWalkable(pixelColorIndex)
- Minimap.IsWalkable(position)
- Minimap.IsWalkableByColor(position)

Internal local helpers, not public API:
- validate_non_negative_integer
- validate_position_table

### module.lua
Exported tables/constants:
- Module

Functions:
- Module.After(name, callback, delayMs)
- Module.Cancel(name)
- Module.Every(name, callback, delayMs)
- Module.Exists(name)
- Module.Get(name)
- Module.List()
- Module.New(name, callback, delayMs)
- Module.Pause(name)
- Module.PauseManaged(name)
- Module.Resume(name)
- Module.ResumeManaged(name)
- Module.Stop(name)

Internal local helpers, not public API:
- ensure_module_api
- ensure_module_pause_resume_api
- normalize_delay

### npc_trade_storage.lua
Exported tables/constants:
- NpcTradeStorage

Functions:
- NpcTradeStorage.Buy(itemId, itemCount, ignoreCapacity, buyInShoppingBags)
- NpcTradeStorage.FormatOffers()
- NpcTradeStorage.GetNpcName()
- NpcTradeStorage.GetOfferByItemId(itemId)
- NpcTradeStorage.GetOfferByName(itemName)
- NpcTradeStorage.GetOffers()
- NpcTradeStorage.GetSnapshot()
- NpcTradeStorage.IsAvailable()
- NpcTradeStorage.IsOpen()
- NpcTradeStorage.Sell(itemId, itemCount, sellEquipped)

Internal local helpers, not public API:
- call_game
- normalize_offer
- to_lower_safe
- validate_non_empty_string
- validate_non_negative_integer
- validate_positive_integer

### position.lua
Functions:
- Position.IsReachable(fromPos, toPos)
- Position.IsShootable(fromPos, toPos)
- Position.New(x, y, z)
- Position:DistanceTo(otherPos)
- Position:IsReachable(fromPos)
- Position:IsShootable(fromPos)

Internal local helpers, not public API:
- get_game_check_fn
- get_local_player_position
- normalize_position

### self.lua
Exported tables/constants:
- Self

Functions:
- Self.Attack(creatureId)
- Self.BuyItem(itemId, itemCount, ignoreCapacity, buyInShoppingBags)
- Self.CancelWalk()
- Self.Dismount()
- Self.Equip(itemId, tierLevel)
- Self.Follow(creatureId)
- Self.FormatStatsSnapshot(stats, prefix)
- Self.GetCapacity()
- Self.GetCapacityFloor()
- Self.GetCharacterWorld(characterName)
- Self.GetFollowId()
- Self.GetHealth()
- Self.GetHealthPercentage()
- Self.GetLevel()
- Self.GetLevelPercentage()
- Self.GetMana()
- Self.GetManaPercentage()
- Self.GetManaShieldCapacity()
- Self.GetMaxHealth()
- Self.GetMaxMana()
- Self.GetMaxManaShieldCapacity()
- Self.GetMousePositionInWorld()
- Self.GetMousePositionText()
- Self.GetMouseWorldX()
- Self.GetMouseWorldY()
- Self.GetMouseWorldZ()
- Self.GetSoul()
- Self.GetStamina()
- Self.GetStaminaDays()
- Self.GetStaminaHours()
- Self.GetStatsSnapshot()
- Self.GetStatusFlagsSnapshot()
- Self.GetTargetId()
- Self.HasFollow()
- Self.HasTarget()
- Self.IsAlive()
- Self.IsAttacking()
- Self.IsAvailable()
- Self.IsBleeding()
- Self.IsBurning()
- Self.IsCursed()
- Self.IsDazzled()
- Self.IsDrowning()
- Self.IsDrunk()
- Self.IsElectrified()
- Self.IsFeared()
- Self.IsFollowing()
- Self.IsFreezing()
- Self.IsHasted()
- Self.IsHungry()
- Self.IsInCombat()
- Self.IsInProtectionZone()
- Self.IsInRestingArea()
- Self.IsManaShielded()
- Self.IsOnline()
- Self.IsParalyzed()
- Self.IsPoisoned()
- Self.IsRooted()
- Self.IsStrengthened()
- Self.LookAtCreature(creatureId)
- Self.LookAtPosition(position)
- Self.Mount()
- Self.PrivateMessage(playerName, message)
- Self.Say(message)
- Self.SayOnChannel(message, channelId)
- Self.SayToNpc(message)
- Self.SellItem(itemId, itemCount, sellEquipped)
- Self.Step(direction)
- Self.StopAttackAndFollow()
- Self.UseItemInContainer(itemId, containerIndex, itemPos, useItemWithHotkey)
- Self.UseItemOnFloor(position, stackPosition, itemId)
- Self.Whisper(message)
- Self.Yell(message)

Internal local helpers, not public API:
- call_status_boolean
- get_game_status_binding
- get_self_cpp_binding
- validate_position_table
- validate_positive_integer

### sound.lua
Exported tables/constants:
- BotSoundId
- Sound

Functions:
- Sound.ClearQueue()
- Sound.GetCurrentDuration()
- Sound.GetFileDuration(filePath)
- Sound.GetQueueSize()
- Sound.IsPlaying()
- Sound.IsQueued(options)
- Sound.Play(options)
- Sound.SetMinDelay(delayMs)
- Sound.Stop()
- soundGetCurrentDuration()
- soundGetFileDuration(filePath)
- soundGetQueueLength()
- soundGMDetected(instant)
- soundIsCurrentlyPlaying()
- soundIsQueued(options)
- soundLowHealth(instant)
- soundLowMana(instant)
- soundPlayAndWait(options, maxWaitMs)
- soundPlayBotSound(filename, instant)
- soundPlayById(soundId, instant)
- soundPlayByIdSmart(soundId, instant)
- soundPlayByName(soundName, instant)
- soundPlayByNameSmart(soundName, instant)
- soundPlayerDetected(instant)
- soundPlayFile(filePath, instant)
- soundPlayFileSmart(filePath, instant)
- soundSetQueueDelay(delayMs)
- soundStopAll()
- soundWaitForCompletion(maxWaitMs)

Internal local helpers, not public API:
- validateSoundOptions

### spells.lua
Exported tables/constants:
- Spells

Functions:
- Spells.GetGroupIds(spellOrWordsOrId)
- Spells.GetIdByName(name)
- Spells.GetIdByWords(words)
- Spells.GetInfo(spellOrWordsOrId)
- Spells.GetLeftCooldownTime(spellOrWordsOrId)
- Spells.GetLeftGroupCooldownTime(groupId)
- Spells.GetWordsById(spellId)
- Spells.GroupIsInCooldown(groupId)
- Spells.IsInCooldown(spellOrWordsOrId)
- Spells.IsReady(spellOrWordsOrId)
- Spells.IsUseWithItemExhausted()
- Spells.Item.GetCooldownId(itemId)
- Spells.Item.GetGroupIds(itemId)
- Spells.Item.GetInfo(itemId)
- Spells.Item.GetLeftCooldownTime(itemId)
- Spells.Item.IsInCooldown(itemId)
- Spells.Item.IsReady(itemId)
- Spells.Item.WillBeReady(itemId, timeMs)
- Spells.WillBeReady(spellOrWordsOrId, timeMs)

Internal local helpers, not public API:
- require_cooldown_table
- resolve_spell_words

### vip.lua
Exported tables/constants:
- Vip
- VipStorage
- VIPStorage

Functions:
- Vip.Count()
- Vip.CountOnline()
- Vip.Exists(vipName)
- Vip.FindByPrefix(namePrefix, onlyOnline)
- Vip.Get(vipName)
- Vip.GetAll()
- Vip.GetByType(vipType)
- Vip.GetDescription(vipName)
- Vip.GetHearts()
- Vip.GetNames(onlyOnline)
- Vip.GetNotifyOnLogin(vipName)
- Vip.GetSnapshot()
- Vip.GetType(vipName)
- Vip.IsAvailable()
- Vip.IsHeart(vipName)
- Vip.IsOnline(vipName)
- Vip.ToLookupTable()

Internal local helpers, not public API:
- call_method_no_args
- get_heart_flag_value
- get_raw_vip_by_name
- get_raw_vip_list
- normalize_vip_entry
- normalize_vip_list
- resolve_backend_table
- sort_vips_by_name
- to_lower_safe
- try_get_any_field_value
- try_get_any_method_value
- validate_non_empty_string
- validate_non_negative_integer

### zz_api_surface.lua
Exported tables/constants:
- Core
- Seen

Functions: none

Internal local helpers, not public API:
- add_function_bucket
- build_public_module
- classify_api_bucket
- export_public
- normalize_module
- starts_with
- to_pascal

