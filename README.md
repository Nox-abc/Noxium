# Noxium Wiki — Luau-style Lua for Minecraft

> Noxium lets you write Minecraft plugins as `.nox` files using a **Luau/Roblox-style API**.  
> Drop `.nox` files into `plugins/noxium/noxiums/` and they load automatically.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [The `game` Object & Services](#the-game-object--services)
3. [Signals (Events)](#signals-events)
4. [Services Reference](#services-reference)
   - [Players](#players-service)
   - [RunService](#runservice)
   - [Commands](#commands-service)
   - [GuiService](#guiservice--inventory-guis)
   - [Chat](#chat-service)
   - [DataStore](#datastore-service)
   - [Workspace](#workspace-service)
5. [Object Reference](#object-reference)
   - [Player](#player-object)
   - [World](#world-object)
   - [Location](#location-object)
   - [Block](#block-object)
   - [Item](#item-object)
   - [Inventory](#inventory-object)
   - [Entity](#entity-object)
   - [Vector3](#vector3)
6. [Instance.new()](#instancenew)
7. [GUI System](#gui-system--full-guide)
8. [Event Reference](#event-reference)
9. [Color Codes](#color-codes)
10. [Full Examples](#full-examples)
11. [Management Commands](#management-commands)

---

## Getting Started

1. Put `noxium-1.0.0.jar` in your server's `plugins/` folder.
2. Start the server — `plugins/noxium/noxiums/` is created automatically with a bundled `welcome.nox`.
3. Create a `.nox` file in that folder:

```lua
-- myfirst.nox
local Players = game:GetService("Players")
local Commands = game:GetService("Commands")

Players.PlayerAdded:Connect(function(player)
    player:SendMessage("&aHello, " .. player.Name .. "!")
end)

Commands:Register("hi", function(player, args)
    player:SendMessage("&6Hi there!")
end)
```

4. Run `/noxium reload` in-game or restart.

---

## The `game` Object & Services

Every script has a global `game` object — just like in Roblox. Use it to get services:

```lua
local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local Commands   = game:GetService("Commands")
local GuiService = game:GetService("GuiService")
local Chat       = game:GetService("Chat")
local DataStore  = game:GetService("DataStore")
local Workspace  = game:GetService("Workspace")
```

**Shorthand** — services are also available as properties:

```lua
local Players = game.Players          -- same as GetService("Players")
local Workspace = game.Workspace
```

**Game-level methods:**

```lua
game:Broadcast("&aHello everyone!")          -- send to all players
game:ConsoleCommand("say Hello")             -- run as console

local version = game.Version                 -- Minecraft server version
local name = game.Name                       -- server software name
```

---

## Signals (Events)

Events work exactly like Roblox's `RBXScriptSignal`. Connect a function, get a connection back.

```lua
local Players = game:GetService("Players")

-- Connect returns a Connection object
local conn = Players.PlayerAdded:Connect(function(player)
    print("Joined: " .. player.Name)
end)

-- Disconnect when you no longer need it
conn:Disconnect()
```

**Multiple connections are supported:**
```lua
Players.PlayerAdded:Connect(fn1)
Players.PlayerAdded:Connect(fn2)
-- Both fire when a player joins
```

---

## Services Reference

### Players Service

```lua
local Players = game:GetService("Players")
```

| Signal | Args | Description |
|---|---|---|
| `Players.PlayerAdded` | `player` | Player joins |
| `Players.PlayerRemoving` | `player` | Player leaves |
| `Players.PlayerDied` | `player, killer, event` | Player dies |
| `Players.PlayerRespawned` | `player, event` | Player respawns |
| `Players.PlayerChatted` | `player, message, event` | Player sends chat |

| Method | Returns | Description |
|---|---|---|
| `Players:GetPlayer(name)` | Player\|nil | Get online player by name |
| `Players:GetPlayers()` | table | All online players |
| `Players:FindFirstChild(name)` | Player\|nil | Same as GetPlayer |
| `Players.NumPlayers` | number | Current online count |
| `Players.MaxPlayers` | number | Server max slots |

```lua
local Players = game:GetService("Players")

Players.PlayerAdded:Connect(function(player)
    -- player is a full Player object
    player:SendMessage("Welcome, " .. player.Name)
end)

Players.PlayerRemoving:Connect(function(player)
    print(player.Name .. " left")
end)

-- Get a specific player
local steve = Players:GetPlayer("Steve")
if steve then steve:SendMessage("Found you!") end
```

---

### RunService

```lua
local RunService = game:GetService("RunService")
```

| Method | Description |
|---|---|
| `RunService:Delay(ticks, fn)` | Run `fn` once after N ticks (20 = 1s) |
| `RunService:Every(ticks, fn)` | Run `fn` every N ticks |
| `RunService:Every(delay, interval, fn)` | Run after delay, then every interval |
| `RunService:DelayAsync(ticks, fn)` | Async version of Delay |
| `RunService:EveryAsync(ticks, fn)` | Async version of Every |
| `RunService:Cancel(taskId)` | Cancel a task by its ID |

All scheduling methods return a `taskId` (number) you can cancel.

```lua
local RunService = game:GetService("RunService")

-- Run once after 5 seconds (100 ticks)
RunService:Delay(100, function()
    game:Broadcast("&a5 seconds have passed!")
end)

-- Repeat every minute (1200 ticks)
local id = RunService:Every(1200, function()
    game:Broadcast("&6[Tip] Type /help for commands!")
end)

-- Cancel it later
RunService:Cancel(id)
```

---

### Commands Service

```lua
local Commands = game:GetService("Commands")
```

#### `Commands:Register(name, [description], [usage], fn)`

Register a custom `/command`. The `fn` receives `(player, args)`.

- `player` — the Player who ran it, or a ConsoleSender object
- `args` — 1-indexed table of arguments

```lua
local Commands = game:GetService("Commands")

-- Simple command
Commands:Register("hello", function(player, args)
    player:SendMessage("&aHello, " .. player.Name .. "!")
end)

-- With description
Commands:Register("fly", "Toggle fly mode", function(player, args)
    player.Flying = not player.Flying
    player:SendMessage(player.Flying and "&aFlying!" or "&cNot flying.")
end)

-- With arguments
Commands:Register("tp", "Teleport to coordinates", "/<name> <x> <y> <z>", function(player, args)
    if #args < 3 then
        player:SendMessage("&cUsage: /tp <x> <y> <z>")
        return
    end
    local loc = Instance.new("Location", {
        World = player.World.Name,
        X = tonumber(args[1]), Y = tonumber(args[2]), Z = tonumber(args[3])
    })
    player:Teleport(loc)
end)
```

**Console check:**
```lua
Commands:Register("fly", function(player, args)
    if player._type == "Console" then
        player:SendMessage("Players only!")
        return
    end
    -- safe to use player methods now
end)
```

---

### GuiService — Inventory GUIs

```lua
local GuiService = game:GetService("GuiService")
```

Create fully interactive inventory menus with clickable buttons.

#### Creating a GUI

```lua
local menu = GuiService:New("Chest", {
    Title = "&6My Menu",    -- supports & color codes
    Rows = 3,               -- 1–6 rows
    PreventClose = false,   -- if true, menu re-opens when closed
    OnClose = function(player)  -- optional close callback
        player:SendMessage("You closed the menu.")
    end,
})
```

#### Setting Slots

Slots are **0-indexed** (slot 0 = top-left, slot 8 = top-right).

```lua
menu:SetSlot(13, {              -- center of a 3-row menu
    Material = "DIAMOND",
    Name = "&b&lShiny Button",
    Lore = {"&7Click to do something", "&8Shift-click for more"},
    Amount = 1,
    Glow = true,                -- adds enchant glow effect
    OnClick = function(player, clickType)
        -- clickType: "LEFT", "RIGHT", "SHIFT_LEFT", "SHIFT_RIGHT", "MIDDLE", etc.
        player:SendMessage("Clicked with " .. clickType)
    end,
    OnRightClick = function(player, clickType)
        player:SendMessage("Right clicked!")
    end,
    OnShiftClick = function(player, clickType)
        player:SendMessage("Shift clicked!")
    end,
})
```

#### Filling Slots

```lua
-- Fill empty slots with glass panes
menu:Fill({
    Material = "GRAY_STAINED_GLASS_PANE",
    Name = " ",      -- one space = invisible name
})

-- Fill ALL slots (override existing)
menu:FillAll({ Material = "BLACK_STAINED_GLASS_PANE", Name = " " })

-- Put glass pane border around edges
menu:Border({ Material = "BLACK_STAINED_GLASS_PANE", Name = " " })
```

#### Opening & Closing

```lua
-- Open for a player
menu:Open(player)

-- Close for a player
menu:Close(player)

-- Refresh items while it's open (call after changing slots)
menu:Update(player)

-- Clear all slots
menu:Clear()

-- Remove one slot
menu:ClearSlot(13)

-- Check if open for a player
if menu:IsOpenFor(player) then
    print("Player has the menu open")
end
```

#### GUI Properties

```lua
menu.Title = "&6New Title"  -- change title (takes effect on next open)
menu.Rows                   -- read the row count
menu.Size                   -- total slots (rows * 9)
menu:SetOnClose(fn)         -- set or replace the OnClose callback
menu:SetPreventClose(true)  -- prevent players from closing
```

#### Multi-page GUI Example

```lua
local Players = game:GetService("Players")
local Commands = game:GetService("Commands")
local GuiService = game:GetService("GuiService")

local items = {}
for i = 1, 50 do
    table.insert(items, { name = "&fItem #" .. i, mat = "PAPER" })
end

local function buildShop(player, page)
    local gui = GuiService:New("Chest", { Title = "&6Shop — Page " .. page, Rows = 6 })
    gui:Border({ Material = "BLACK_STAINED_GLASS_PANE", Name = " " })

    local perPage = 28  -- 4 rows of 7
    local startSlots = {10, 11, 12, 13, 14, 15, 16, 19, 20, 21, 22, 23, 24, 25, 28, 29, 30, 31, 32, 33, 34, 37, 38, 39, 40, 41, 42, 43}
    local startIdx = (page - 1) * perPage + 1

    for i = 1, perPage do
        local itemIdx = startIdx + i - 1
        if items[itemIdx] then
            local item = items[itemIdx]
            local slot = startSlots[i]
            gui:SetSlot(slot, {
                Material = item.mat,
                Name = item.name,
                Lore = {"&7Click to buy", "&ePage " .. page},
                OnClick = function(p, click)
                    p:SendMessage("&aBought: " .. item.name)
                    gui:Close(p)
                end,
            })
        end
    end

    -- Previous page button
    if page > 1 then
        gui:SetSlot(45, {
            Material = "ARROW", Name = "&aPrevious Page",
            OnClick = function(p) gui:Close(p); buildShop(p, page - 1):Open(p) end,
        })
    end

    -- Next page button
    if startIdx + perPage - 1 <= #items then
        gui:SetSlot(53, {
            Material = "ARROW", Name = "&aNext Page",
            OnClick = function(p) gui:Close(p); buildShop(p, page + 1):Open(p) end,
        })
    end

    return gui
end

Commands:Register("shop", "Open the shop", function(player, args)
    buildShop(player, 1):Open(player)
end)
```

---

### Chat Service

```lua
local Chat = game:GetService("Chat")
```

| Signal | Args | Description |
|---|---|---|
| `Chat.MessageSent` | `player, message, event` | Player sends a message |
| `Chat.CommandFired` | `player, command, event` | Player runs a command |

| Method | Description |
|---|---|
| `Chat:Broadcast(msg)` | Broadcast to all players |
| `Chat:SendSystemMessage(player, msg)` | Send to a specific player |

```lua
local Chat = game:GetService("Chat")

-- Modify chat format
Chat.MessageSent:Connect(function(player, message, event)
    local rank = player.Op and "&c[OP]" or "&7[Player]"
    event:SetMessage(rank .. " &f" .. player.Name .. " &8» &f" .. message)
end)

-- Cancel certain messages
Chat.MessageSent:Connect(function(player, message, event)
    if message:lower():find("bad_word") then
        event:SetCancelled(true)
        player:SendMessage("&cWatch your language!")
    end
end)
```

---

### DataStore Service

In-memory key-value storage shared across all scripts. Resets on server restart.  
*(For persistent storage, use file I/O with `allow-unsafe-libs: true` in config.)*

```lua
local DataStore = game:GetService("DataStore")
```

| Method | Description |
|---|---|
| `DataStore:Set(key, value)` | Store any Lua value |
| `DataStore:Get(key, default)` | Get a value, with optional default |
| `DataStore:Has(key)` | Check if key exists |
| `DataStore:Delete(key)` | Remove a key |
| `DataStore:GetAll()` | Returns table of all key-value pairs |
| `DataStore:Increment(key, amount)` | Add to a number value, returns new value |
| `DataStore:Clear()` | Remove all data |

```lua
local DataStore = game:GetService("DataStore")
local Players = game:GetService("Players")

-- Track kills
Players.PlayerAdded:Connect(function(player)
    if not DataStore:Has("kills:" .. player.Name) then
        DataStore:Set("kills:" .. player.Name, 0)
    end
end)

-- Increment kill count on PlayerDied
Players.PlayerDied:Connect(function(victim, killer, event)
    if killer and killer._type == "Player" then
        local newKills = DataStore:Increment("kills:" .. killer.Name)
        killer:SendMessage("&aKills: " .. newKills)
    end
end)
```

---

### Workspace Service

```lua
local Workspace = game:GetService("Workspace")
```

| Method/Property | Description |
|---|---|
| `Workspace:GetWorld(name)` | Get a world by name |
| `Workspace:GetWorlds()` | All loaded worlds |
| `Workspace:NewLocation(world, x, y, z)` | Create a Location |
| `Workspace.DefaultWorld` | The main world |

---

## Object Reference

### Player Object

**Properties** — read directly, write with `=`:

```lua
player.Name          -- read-only  (string)
player.UUID          -- read-only  (string)
player.DisplayName   -- read/write (string, supports & colors)
player.Health        -- read/write (number, 0–MaxHealth)
player.MaxHealth     -- read-only  (number)
player.FoodLevel     -- read/write (number, 0–20)
player.Saturation    -- read/write (number, 0–20)
player.Level         -- read/write (number)
player.Exp           -- read/write (number, 0.0–1.0 progress)
player.TotalExp      -- read/write (number)
player.GameMode      -- read/write ("SURVIVAL"|"CREATIVE"|"ADVENTURE"|"SPECTATOR")
player.Flying        -- read/write (bool, also sets AllowFlight)
player.AllowFlight   -- read/write (bool)
player.Op            -- read/write (bool)
player.Location      -- read-only  (Location)
player.World         -- read-only  (World)
player.Velocity      -- read/write (Vector3)
player.Inventory     -- read-only  (Inventory)
player.Sneaking      -- read-only  (bool)
player.Sprinting     -- read-only  (bool)
player.Online        -- read-only  (bool)
player.Ping          -- read-only  (number, ms)
player.Locale        -- read-only  (string, e.g. "en_us")
player.Address       -- read-only  (string, IP address)
```

**Methods:**

```lua
player:SendMessage("&aHello!")
player:SendTitle("&6Title", "&fSubtitle", fadeIn, stay, fadeOut)
player:SendActionBar("&7Action bar text")
player:Teleport(location)
player:Kick("reason")
player:Ban("reason")
player:GiveItem(item)
player:ClearInventory()
player:GetItemInHand()            -- returns Item
player:GetInventory()             -- returns Inventory
player:PlaySound("ENTITY_PLAYER_LEVELUP", volume, pitch)
player:AddEffect("SPEED", durationTicks, amplifier)
player:RemoveEffect("SPEED")
player:GetActiveEffects()         -- returns table of effect tables
player:HasPermission("permission.node")  -- returns bool
player:Heal()                     -- set health to max
player:Feed()                     -- set hunger and saturation to max
player:Kill()                     -- set health to 0
```

**Property assignment examples:**

```lua
player.Health = 10               -- take damage (set to half heart)
player.GameMode = "CREATIVE"     -- change gamemode
player.Flying = true             -- enable flight
player.Level = 30                -- set XP level
player.Velocity = Vector3.new(0, 2, 0)  -- launch up
```

---

### World Object

```lua
world.Name           -- read-only  (string)
world.Environment    -- read-only  ("NORMAL", "NETHER", "THE_END")
world.Time           -- read/write (number, 0–24000)
world.Storm          -- read/write (bool)
world.Thundering     -- read/write (bool)
world.Seed           -- read-only  (number)
world.PlayerCount    -- read-only  (number)

world:GetPlayers()                -- table of Players in this world
world:GetBlockAt(x, y, z)        -- returns Block
world:SpawnEntity(location, type) -- returns Entity (e.g. "ZOMBIE")
world:StrikeLightning(location)   -- real lightning (damage)
world:StrikeLightningEffect(location) -- visual only
world:CreateExplosion(location, power)
world:PlaySound(location, soundName, volume, pitch)
```

---

### Location Object

Created via `Instance.new("Location", {...})` or `Workspace:NewLocation(...)`.

```lua
location.X        -- read/write
location.Y        -- read/write
location.Z        -- read/write
location.Yaw      -- read/write
location.Pitch    -- read/write
location.World    -- read-only

location:GetBlock()              -- Block at this location
location:GetWorld()              -- World
location:DistanceTo(other)       -- distance to another location
location:Add(x, y, z)           -- returns new location offset by x/y/z
location:Clone()                 -- returns a copy
```

---

### Block Object

```lua
block.Type     -- read/write (material name, e.g. "STONE")
block.X        -- read-only
block.Y        -- read-only
block.Z        -- read-only
block.Empty    -- read-only (bool, true if AIR)
block.Liquid   -- read-only (bool)
block.Location -- read-only (Location)
block.World    -- read-only (World)

block:BreakNaturally()
block:GetRelative(x, y, z)     -- returns adjacent Block
block:GetLocation()             -- returns Location
block:GetWorld()                -- returns World
```

---

### Item Object

Created via `Instance.new("Item", {...})` or `noxium.item(material, amount)`.

```lua
item.Type          -- read/write (material name)
item.Amount        -- read/write (stack size)
item.Name          -- read/write (display name, & colors)
item.Lore          -- read/write (table of strings)
item.Enchantments  -- read-only (table of enchant→level)
item.Empty         -- read-only (bool)
item.Glow          -- write-only (bool, adds glow effect)

item:SetName("&bName")
item:SetLore({"&7Line 1", "&7Line 2"})
item:AddEnchantment("SHARPNESS", 5)
item:AddFlag("HIDE_ENCHANTS")
item:SetGlow(true)
item:Clone()
```

**Creating items:**

```lua
-- Simple
local item = Instance.new("Item", {
    Material = "DIAMOND_SWORD",
    Amount = 1,
    Name = "&b&lFrost Blade",
    Lore = {"&7A legendary sword", "&cDeals ice damage"},
    Glow = true,
    Enchantments = {
        SHARPNESS = 5,
        FIRE_ASPECT = 2,
        LOOTING = 3,
    }
})
player:GiveItem(item)
```

---

### Inventory Object

```lua
inv.Size               -- total slots

inv:GetItem(slot)      -- Item at 1-indexed slot
inv:SetItem(slot, item)
inv:AddItem(item)      -- add to first empty slot
inv:RemoveItem(item)
inv:Contains(material)  -- bool
inv:Clear()
inv:GetMainHand()       -- Item in main hand
inv:GetOffHand()        -- Item in off hand
inv:GetHelmet()
inv:GetChestplate()
inv:GetLeggings()
inv:GetBoots()
```

---

### Entity Object

Non-player entities (zombies, creepers, etc.):

```lua
entity.Type       -- "ZOMBIE", "CREEPER", etc.
entity.Name
entity.UUID
entity.Location   -- read/write
entity.World
entity.Velocity   -- read/write
entity.Alive      -- bool
entity.Dead       -- bool
entity.Health     -- read/write (LivingEntity)
entity.MaxHealth  -- read-only  (LivingEntity)

entity:Remove()
entity:Teleport(location)
entity:Damage(amount)          -- LivingEntity only
entity:AddEffect("SLOWNESS", 200, 2)  -- LivingEntity only
entity:SetVelocity(Vector3.new(0, 1, 0))
```

---

### Vector3

```lua
-- Create
local v = Vector3.new(x, y, z)

-- Constants
Vector3.zero     -- (0, 0, 0)
Vector3.one      -- (1, 1, 1)
Vector3.up       -- (0, 1, 0)
Vector3.down     -- (0,-1, 0)
Vector3.forward  -- (0, 0,-1)
Vector3.back     -- (0, 0, 1)

-- Properties
v.X, v.Y, v.Z

-- Methods
v:Length()         -- magnitude
v:LengthSquared()  -- faster squared magnitude
v:Normalize()      -- unit vector
v:Dot(other)       -- dot product (number)
v:Cross(other)     -- cross product (Vector3)

-- Arithmetic operators
local sum  = v + other    -- Vector3 + Vector3
local diff = v - other    -- Vector3 - Vector3
local scaled = v * 2      -- Vector3 * number
local neg = -v            -- negate

-- Usage with player velocity
player.Velocity = Vector3.new(0, 2, 0)   -- launch player up
```

---

## Instance.new()

Factory for creating Noxium objects:

```lua
-- Item
local sword = Instance.new("Item", {
    Material = "NETHERITE_SWORD",
    Amount = 1,
    Name = "&4Dark Blade",
    Lore = {"&7A weapon of darkness"},
    Glow = true,
    Enchantments = { SHARPNESS = 5 }
})

-- Location
local spawn = Instance.new("Location", {
    World = "world",
    X = 0, Y = 65, Z = 0,
    Yaw = 90, Pitch = 0,
})

-- GUI (same as GuiService:New)
local menu = Instance.new("Gui", {
    Title = "&6Shop",
    Rows = 3,
})

-- Vector3 (same as Vector3.new)
local vel = Instance.new("Vector3", { X = 0, Y = 1, Z = 0 })
```

---

## GUI System — Full Guide

### Slot Layout (3-row example)

```
Row 1:   0  1  2  3  4  5  6  7  8
Row 2:   9 10 11 12 13 14 15 16 17
Row 3:  18 19 20 21 22 23 24 25 26
```

Center of 3 rows = slot **13**  
Common pattern (ring border, center items):

```lua
-- 5-row menu with border and centered 3x3 grid
local menu = GuiService:New("Chest", { Title = "&6Menu", Rows = 5 })
menu:Border({ Material = "GRAY_STAINED_GLASS_PANE", Name = " " })
-- Center slots: 10,11,12,13,14,15,16 / 19..25 / 28..34
```

### Confirmation Dialog

```lua
local function confirmMenu(player, message, onConfirm, onCancel)
    local gui = GuiService:New("Chest", { Title = "&cConfirm Action", Rows = 3 })
    gui:FillAll({ Material = "BLACK_STAINED_GLASS_PANE", Name = " " })

    gui:SetSlot(11, {
        Material = "LIME_WOOL",
        Name = "&a&lConfirm",
        Lore = { "&7" .. message },
        OnClick = function(p)
            gui:Close(p)
            if onConfirm then onConfirm(p) end
        end,
    })
    gui:SetSlot(13, {
        Material = "PAPER",
        Name = "&f" .. message,
    })
    gui:SetSlot(15, {
        Material = "RED_WOOL",
        Name = "&c&lCancel",
        OnClick = function(p)
            gui:Close(p)
            if onCancel then onCancel(p) end
        end,
    })

    gui:Open(player)
end
```

---

## Event Reference

Register events via signals on services, or using the `noxium.on()` compat API.

### Player Events

```lua
local Players = game:GetService("Players")

Players.PlayerAdded:Connect(function(player) end)
Players.PlayerRemoving:Connect(function(player) end)
Players.PlayerDied:Connect(function(player, killer, event)
    -- killer may be nil
    event:SetDeathMessage("&c" .. player.Name .. " died!")
    event:KeepInventory(true)
    event:KeepLevel(true)
end)
Players.PlayerRespawned:Connect(function(player, event)
    event:SetRespawnLocation(location)
end)
Players.PlayerChatted:Connect(function(player, message, event)
    event:SetMessage(newMessage)
    event:SetCancelled(true)
end)
```

### Using `noxium.on()` (legacy compat)

If you prefer event-table style (like the old API or Skript-like style), it still works:

```lua
noxium.on("PlayerJoin", function(e)
    e.Player:SendMessage("Welcome!")       -- e.Player is the player
    e:SetMessage("&a" .. e.Player.Name .. " joined!")
end)

noxium.on("BlockBreak", function(e)
    local block = e.Block
    if block.Type == "DIAMOND_ORE" then
        e.Player:SendMessage("&bFound diamond!")
    end
    -- e:SetCancelled(true)
end)
```

### All Available Signal Names

| Signal Name | Event Table Fields |
|---|---|
| `PlayerAdded` | `player` |
| `PlayerRemoving` | `player` |
| `PlayerDied` | `player, killer, event{SetDeathMessage, KeepInventory, KeepLevel}` |
| `PlayerRespawned` | `player, event{RespawnLocation, SetRespawnLocation}` |
| `PlayerChatted` | `player, message, event{SetMessage, SetCancelled}` |
| `PlayerJoin` | `event{Player, Message, SetMessage}` |
| `PlayerQuit` | `event{Player, Message, SetMessage}` |
| `PlayerChat` | `event{Player, Message, SetMessage, Cancel, SetCancelled}` |
| `PlayerCommand` | `player, command, event{SetCancelled}` |
| `PlayerDeath` | `event{Player, Killer, DeathMessage, SetDeathMessage, KeepInventory, KeepLevel}` |
| `PlayerRespawn` | `event{Player, RespawnLocation, SetRespawnLocation}` |
| `PlayerMove` | `event{Player, From, To, SetCancelled}` |
| `PlayerInteract` | `event{Player, Action, Block, Item, SetCancelled}` |
| `PlayerDropItem` | `event{Player, Item, SetCancelled}` |
| `PlayerPickupItem` | `event{Player, Item, SetCancelled}` |
| `PlayerGameModeChange` | `event{Player, NewGameMode}` |
| `PlayerToggleFlight` | `event{Player, Flying, SetCancelled}` |
| `PlayerToggleSneak` | `event{Player, Sneaking}` |
| `BlockBreak` | `event{Player, Block, SetCancelled, SetDropItems}` |
| `BlockPlace` | `event{Player, Block, BlockAgainst, Item, SetCancelled}` |
| `EntityDamage` | `event{Entity, Cause, Damage, SetDamage, SetCancelled}` |
| `EntityDamageByEntity` | `event{Entity, Damager, Damage, SetDamage, SetCancelled}` |
| `EntityDeath` | `event{Entity, Killer, DroppedExp, SetDroppedExp, GetDrops}` |
| `CreatureSpawn` | `event{Entity, SpawnReason, SetCancelled}` |
| `WeatherChange` | `event{World, WeatherState}` |

---

## Color Codes

Use `&` + code in any string passed to Noxium functions:

| Code | Color/Style |
|---|---|
| `&0–9` | Black through Blue |
| `&a` | Green | `&b` | Aqua | `&c` | Red |
| `&d` | Light Purple | `&e` | Yellow | `&f` | White |
| `&l` | **Bold** | `&o` | *Italic* | `&n` | Underline |
| `&m` | ~~Strike~~ | `&k` | Obfuscated | `&r` | Reset |

```lua
player:SendMessage("&c&lError! &r&7Something went wrong.")
player:SendTitle("&6&lWelcome!", "&f" .. player.Name, 10, 60, 20)
```

---

## Full Examples

### Complete Hub Plugin

```lua
-- hub.nox
local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local Commands   = game:GetService("Commands")
local GuiService = game:GetService("GuiService")
local DataStore  = game:GetService("DataStore")

-- Spawn location
local SPAWN = Instance.new("Location", { World = "world", X = 0, Y = 65, Z = 0 })

-- Teleport new players to spawn
Players.PlayerAdded:Connect(function(player)
    RunService:Delay(10, function()
        if player.Online then
            player:Teleport(SPAWN)
            player:SendTitle("&6Welcome!", "&f" .. player.Name)
        end
    end)
    -- Track joins
    DataStore:Increment("total_joins")
end)

-- Custom join/quit messages
Players.PlayerAdded:Connect(function(player)
    RunService:Delay(1, function()
        game:Broadcast("&a▶ &f" .. player.Name .. " &7joined (&a" ..
            Players.NumPlayers .. "&7/" .. Players.MaxPlayers .. "&7)")
    end)
end)

Players.PlayerRemoving:Connect(function(player)
    game:Broadcast("&c◀ &f" .. player.Name .. " &7left")
end)

-- /spawn command
Commands:Register("spawn", "Teleport to spawn", function(player, args)
    player:Teleport(SPAWN)
    player:SendMessage("&aTeleported to spawn!")
    player:PlaySound("ENTITY_ENDERMEN_TELEPORT", 1, 1)
end)

-- /fly command
Commands:Register("fly", "Toggle flight", function(player, args)
    if player._type == "Console" then return end
    player.Flying = not player.Flying
    player:SendMessage(player.Flying and "&a✈ Flying ON" or "&c✈ Flying OFF")
end)

-- /stats command
Commands:Register("stats", "View server stats", function(player, args)
    player:SendMessage("&6&lServer Stats:")
    player:SendMessage("&7  Online: &f" .. Players.NumPlayers .. "/" .. Players.MaxPlayers)
    player:SendMessage("&7  Total joins: &f" .. DataStore:Get("total_joins", 0))
    player:SendMessage("&7  Version: &f" .. game.Version)
end)

-- /menu command
local menu = GuiService:New("Chest", { Title = "&6&l✦ Hub Menu", Rows = 3 })
menu:Border({ Material = "CYAN_STAINED_GLASS_PANE", Name = " " })
menu:SetSlot(11, {
    Material = "COMPASS", Name = "&b&lSpawn", Lore = {"&7Return to spawn"},
    Glow = true,
    OnClick = function(p) p:Teleport(SPAWN); p:SendMessage("&aTo spawn!"); menu:Close(p) end,
})
menu:SetSlot(13, {
    Material = "FEATHER", Name = "&e&lFly", Lore = {"&7Toggle fly mode"},
    OnClick = function(p)
        p.Flying = not p.Flying
        p:SendMessage(p.Flying and "&aFly ON" or "&cFly OFF")
        menu:Close(p)
    end,
})
menu:SetSlot(15, {
    Material = "GOLDEN_APPLE", Name = "&a&lHeal", Lore = {"&7Full heal & feed"},
    Glow = true,
    OnClick = function(p) p:Heal(); p:Feed(); p:SendMessage("&aHealed!"); menu:Close(p) end,
})

Commands:Register("menu", "Open the hub menu", function(player, args)
    if player._type == "Console" then return end
    menu:Open(player)
end)

-- Repeating announcement
local tips = {
    "&6[Tip] &fUse /menu to access the hub!",
    "&6[Tip] &fType /spawn to go back to spawn!",
    "&6[Tip] &fHave fun and be nice!",
}
local tipIdx = 1
RunService:Every(6000, function()
    game:Broadcast(tips[tipIdx])
    tipIdx = (tipIdx % #tips) + 1
end)

print("hub.nox loaded!")
```

### Anti-Cheat Example

```lua
-- anticheat.nox
local Players = game:GetService("Players")
local DataStore = game:GetService("DataStore")

-- Prevent fly hacking in survival
noxium.on("PlayerMove", function(e)
    local player = e.Player
    if player.GameMode == "SURVIVAL" and player.Flying then
        player.Flying = false
        player:SendMessage("&c[AntiCheat] Fly hacking is not allowed!")
    end
end)

-- Prevent fall damage void trick (optional: allow in certain worlds)
noxium.on("EntityDamage", function(e)
    if e.Cause == "VOID" then
        -- Teleport back instead of void death
        local player = e.Entity
        if player._type == "Player" then
            e:SetCancelled(true)
            local world = player.World
            local loc = Instance.new("Location", {
                World = world.Name,
                X = player.Location.X, Y = 65, Z = player.Location.Z
            })
            player:Teleport(loc)
            player:SendMessage("&cYou were saved from the void!")
        end
    end
end)

print("anticheat.nox loaded!")
```

---

## Management Commands

All require `noxium.admin` permission (default: op). Alias: `/nox`

| Command | Description |
|---|---|
| `/noxium list` | List all `.nox` files and load status |
| `/noxium load <name>` | Load a script |
| `/noxium unload <name>` | Unload and stop a script |
| `/noxium reload [name]` | Reload one or all scripts |
| `/noxium info <name>` | Show details (file, load time, commands) |
| `/noxium help` | Show command list |

---

## Configuration (`config.yml`)

```yaml
debug: false                  # verbose Lua error stack traces
allow-unsafe-libs: false      # enable io/os Lua libraries
max-instructions: 100000      # 0 = unlimited (prevent infinite loops)

scripts:
  verbose-loading: true       # log each script name on startup
  fail-fast: false            # stop loading if any script fails
```

---

*Noxium v1.0.0 — Luau-style Lua scripting for Minecraft Paper 1.21+*
