# Noxium Wiki

> Noxium is a Lua-based scripting plugin for Minecraft (Paper 1.21+).  
> Write `.nox` files in `plugins/noxium/noxiums/` and they load automatically at server start.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [File Structure](#file-structure)
3. [Plugin Management Commands](#plugin-management-commands)
4. [The `noxium` API](#the-noxium-api)
   - [Messaging](#messaging)
   - [Players](#players)
   - [Worlds](#worlds)
   - [Locations](#locations)
   - [Items](#items)
   - [Scheduling](#scheduling)
   - [Events](#events)
   - [Commands](#commands)
   - [Server Info](#server-info)
5. [Object Reference](#object-reference)
   - [Player Object](#player-object)
   - [World Object](#world-object)
   - [Location Object](#location-object)
   - [Block Object](#block-object)
   - [Item Object](#item-object)
   - [Inventory Object](#inventory-object)
   - [Entity Object](#entity-object)
   - [Event Objects](#event-objects)
6. [Event Reference](#event-reference)
7. [Color Codes](#color-codes)
8. [Examples](#examples)
9. [Tips & Patterns](#tips--patterns)

---

## Getting Started

1. Build or download `noxium.jar` and drop it into your `plugins/` folder.
2. Start your server. The folder `plugins/noxium/noxiums/` is created automatically.
3. Create a new file in that folder ending in `.nox`, for example `hello.nox`:

```lua
-- hello.nox
noxium.on("PlayerJoin", function(e)
    e.player:sendMessage("&aHello, " .. e.player:getName() .. "!")
end)
```

4. Run `/noxium reload` in-game or restart the server.  
5. The next player to join will see the hello message.

---

## File Structure

```
plugins/
└── noxium/
    ├── config.yml          ← Noxium configuration
    └── noxiums/            ← Your .nox scripts go here
        ├── welcome.nox
        ├── economy.nox
        └── minigames.nox
```

- Files use the `.nox` extension but are standard **Lua 5.2** scripts.
- Each file is its own isolated environment — variables in one script do not affect another.
- File names must not contain spaces. Use underscores or camelCase.

---

## Plugin Management Commands

All commands require the `noxium.admin` permission (default: op).  
Alias: `/nox`

| Command | Description |
|---|---|
| `/noxium list` | List all `.nox` files and their load status |
| `/noxium load <name>` | Load a noxium by name (without `.nox`) |
| `/noxium unload <name>` | Unload a noxium and cancel all its tasks |
| `/noxium reload [name]` | Reload one noxium, or all if no name given |
| `/noxium info <name>` | Show details about a loaded noxium |
| `/noxium help` | Show the help menu |

---

## The `noxium` API

Every script has access to a global `noxium` table containing all API functions.

---

### Messaging

#### `noxium.broadcast(message)`
Send a message to all online players.
```lua
noxium.broadcast("&aServer restart in 5 minutes!")
```

#### `noxium.log(message)`
Print an info message to the server console, prefixed with your script name.
```lua
noxium.log("Script loaded successfully.")
-- Console: [MyScript] Script loaded successfully.
```

#### `noxium.logWarning(message)`
Print a warning to the console.
```lua
noxium.logWarning("Config value missing, using default.")
```

#### `noxium.logError(message)`
Print an error to the console.
```lua
noxium.logError("Failed to connect to database!")
```

#### `noxium.color(text)`
Translate `&` color codes into Minecraft color codes and return the result.
```lua
local colored = noxium.color("&6Gold &aGreen")
```

---

### Players

#### `noxium.getPlayer(name) → Player | nil`
Get an online player by exact name. Returns `nil` if not online.
```lua
local p = noxium.getPlayer("Steve")
if p then
    p:sendMessage("Found you!")
end
```

#### `noxium.getOnlinePlayers() → table`
Returns a Lua table (1-indexed array) of all online Player objects.
```lua
local players = noxium.getOnlinePlayers()
for i, p in ipairs(players) do
    p:sendMessage("Hello #" .. i)
end
```

#### `noxium.getPlayerCount() → number`
Returns the number of currently online players.
```lua
noxium.broadcast("Players online: " .. noxium.getPlayerCount())
```

#### `noxium.getMaxPlayers() → number`
Returns the server's max player slot count.

---

### Worlds

#### `noxium.getWorld(name) → World | nil`
Get a loaded world by name.
```lua
local world = noxium.getWorld("world")
```

#### `noxium.getWorlds() → table`
Returns all loaded worlds as an array of World objects.

---

### Locations

#### `noxium.location(world, x, y, z [, yaw, pitch]) → Location`
Create a Location object. `world` can be a world name string or a World object.
```lua
local loc = noxium.location("world", 0, 64, 0)
local loc2 = noxium.location("world", 100, 80, 200, 90, 0)
```

---

### Items

#### `noxium.item(material, amount) → Item`
Create an ItemStack.  
Material names are Minecraft material IDs (all uppercase, e.g. `"DIAMOND_SWORD"`).
```lua
local sword = noxium.item("DIAMOND_SWORD", 1)
local apples = noxium.item("APPLE", 64)
player:giveItem(sword)
```

---

### Scheduling

#### `noxium.schedule(ticks, fn) → taskId`
Run `fn` once after `ticks` server ticks (20 ticks = 1 second).
```lua
noxium.schedule(100, function()
    noxium.broadcast("5 seconds passed!")
end)
```

#### `noxium.scheduleRepeating(delayTicks, intervalTicks, fn) → taskId`
Run `fn` repeatedly. `delay` is the initial delay before the first run; `interval` is how often to repeat.
```lua
-- Broadcast every 10 minutes (12000 ticks), starting after 1 minute
noxium.scheduleRepeating(1200, 12000, function()
    noxium.broadcast("&6[Reminder] Vote for us!")
end)
```

#### `noxium.scheduleAsync(ticks, fn) → taskId`
Run `fn` asynchronously after `ticks` ticks. Use for non-Bukkit operations like file I/O.  
⚠️ Do not call Bukkit API from async tasks — use `noxium.schedule` inside if you need to.

#### `noxium.cancelTask(taskId)`
Cancel a scheduled task by its ID.
```lua
local id = noxium.scheduleRepeating(0, 20, function() end)
noxium.cancelTask(id)
```

---

### Events

#### `noxium.on(eventName, fn)`
Register a listener for a Minecraft event. `fn` receives one argument — the event table.

```lua
noxium.on("PlayerJoin", function(e)
    e.player:sendMessage("Welcome!")
end)
```

Multiple listeners can be registered for the same event — all will fire.

See [Event Reference](#event-reference) for all available events.

---

### Commands

#### `noxium.command(name, [description,] fn)`
Register a custom `/command`. The handler receives `(sender, args)` where `args` is a 1-indexed Lua table.

```lua
noxium.command("hello", "Say hello", function(sender, args)
    sender:sendMessage("&aHello, " .. sender:getName() .. "!")
end)

-- With arguments
noxium.command("say", function(sender, args)
    if #args == 0 then
        sender:sendMessage("&cUsage: /say <message>")
        return
    end
    local msg = table.concat(args, " ")
    noxium.broadcast("&f[" .. sender:getName() .. "] " .. msg)
end)
```

- `sender` is a Player object when a player runs the command, or a ConsoleSender object when run from console.
- `args[1]`, `args[2]`, etc. are the command arguments as strings.
- `#args` gives the number of arguments.

---

### Server Info

#### `noxium.getServerName() → string`
#### `noxium.getVersion() → string`
#### `noxium.getScriptName() → string`
Returns the name of the currently running script.

#### `noxium.consoleCommand(command)`
Execute a command as the server console.
```lua
noxium.consoleCommand("say Hello!")
noxium.consoleCommand("op Steve")
```

---

## Object Reference

### Player Object

Methods available on any Player object (e.g. `e.player` in events, or from `noxium.getPlayer()`):

| Method | Returns | Description |
|---|---|---|
| `player:getName()` | string | Player's username |
| `player:getUUID()` | string | Player's UUID |
| `player:getDisplayName()` | string | Display name (may include color) |
| `player:setDisplayName(name)` | — | Set display name |
| `player:sendMessage(msg)` | — | Send a chat message (supports `&` colors) |
| `player:sendTitle(title, subtitle, fadeIn, stay, fadeOut)` | — | Show a title on screen |
| `player:sendActionBar(msg)` | — | Show a message in the action bar |
| `player:getHealth()` | number | Current health (0–20 by default) |
| `player:setHealth(n)` | — | Set health |
| `player:getMaxHealth()` | number | Max health |
| `player:getFoodLevel()` | number | Hunger level (0–20) |
| `player:setFoodLevel(n)` | — | Set hunger |
| `player:getSaturation()` | number | Saturation level |
| `player:setSaturation(n)` | — | Set saturation |
| `player:heal()` | — | Restore full health |
| `player:feed()` | — | Restore full hunger and saturation |
| `player:getLevel()` | number | XP level |
| `player:setLevel(n)` | — | Set XP level |
| `player:getExp()` | number | XP progress (0.0–1.0) |
| `player:setExp(n)` | — | Set XP progress |
| `player:getTotalExp()` | number | Total XP points |
| `player:setTotalExp(n)` | — | Set total XP points |
| `player:getGameMode()` | string | `"SURVIVAL"`, `"CREATIVE"`, `"ADVENTURE"`, `"SPECTATOR"` |
| `player:setGameMode(mode)` | — | Set game mode (uppercase string) |
| `player:isOp()` | bool | Whether the player is OP |
| `player:setOp(bool)` | — | Set OP status |
| `player:isFlying()` | bool | Whether the player is flying |
| `player:setFlying(bool)` | — | Set flying state |
| `player:allowFlight(bool)` | — | Allow/deny flight |
| `player:isSneaking()` | bool | Whether sneaking |
| `player:isSprinting()` | bool | Whether sprinting |
| `player:isOnline()` | bool | Whether the player is online |
| `player:hasPermission(perm)` | bool | Permission check |
| `player:kick(reason)` | — | Kick the player |
| `player:ban(reason)` | — | Ban the player |
| `player:getLocation()` | Location | Current location |
| `player:teleport(location)` | — | Teleport player |
| `player:getWorld()` | World | Current world |
| `player:getInventory()` | Inventory | Player's inventory |
| `player:giveItem(item)` | — | Add item to inventory |
| `player:clearInventory()` | — | Clear all items |
| `player:getItemInHand()` | Item | Item in main hand |
| `player:getVelocity()` | table `{x,y,z}` | Current velocity |
| `player:setVelocity(vec)` | — | Set velocity (`{x=0, y=1, z=0}`) |
| `player:playSound(sound, vol, pitch)` | — | Play a sound to the player |
| `player:addEffect(type, ticks, amplifier)` | — | Add potion effect |
| `player:removeEffect(type)` | — | Remove potion effect |
| `player:getPing()` | number | Player's ping in ms |
| `player:getLocale()` | string | Player's locale (e.g. `"en_us"`) |
| `player:getAddress()` | string | Player's IP address |

---

### World Object

| Method | Returns | Description |
|---|---|---|
| `world:getName()` | string | World name |
| `world:getEnvironment()` | string | `"NORMAL"`, `"NETHER"`, `"THE_END"` |
| `world:getTime()` | number | Current tick time (0–24000) |
| `world:setTime(ticks)` | — | Set world time |
| `world:getPlayers()` | table | All players in this world |
| `world:getPlayerCount()` | number | Number of players |
| `world:getBlockAt(x, y, z)` | Block | Get block at coordinates |
| `world:spawnEntity(location, type)` | Entity | Spawn an entity |
| `world:strikeLightning(location)` | — | Strike real lightning |
| `world:strikeLightningEffect(location)` | — | Strike lightning effect (no damage) |
| `world:setStorm(bool)` | — | Start/stop rain |
| `world:setThundering(bool)` | — | Start/stop thunder |
| `world:isStorm()` | bool | Whether it's raining |
| `world:createExplosion(location, power)` | — | Create an explosion |

---

### Location Object

| Property/Method | Description |
|---|---|
| `location.x` | X coordinate (number) |
| `location.y` | Y coordinate (number) |
| `location.z` | Z coordinate (number) |
| `location.yaw` | Yaw angle |
| `location.pitch` | Pitch angle |
| `location:getX()` | Returns X |
| `location:getY()` | Returns Y |
| `location:getZ()` | Returns Z |
| `location:getYaw()` | Returns yaw |
| `location:getPitch()` | Returns pitch |
| `location:getWorld()` | Returns the World |
| `location:getBlock()` | Returns the Block at this location |
| `location:distanceTo(otherLoc)` | Distance to another location |
| `location:add(x, y, z)` | Returns new location offset by x/y/z |

---

### Block Object

| Method | Returns | Description |
|---|---|---|
| `block:getType()` | string | Material name, e.g. `"STONE"` |
| `block:setType(material)` | — | Change block type |
| `block:getLocation()` | Location | Block's location |
| `block:getWorld()` | World | Block's world |
| `block:getX()` | number | X coordinate |
| `block:getY()` | number | Y coordinate |
| `block:getZ()` | number | Z coordinate |
| `block:isEmpty()` | bool | Whether the block is air |
| `block:isLiquid()` | bool | Whether the block is water/lava |
| `block:breakNaturally()` | — | Break block with drops |

---

### Item Object

| Method | Returns | Description |
|---|---|---|
| `item:getType()` | string | Material name |
| `item:getAmount()` | number | Stack size |
| `item:setAmount(n)` | — | Set stack size |
| `item:getName()` | string | Display name or material name |
| `item:setName(name)` | — | Set custom display name |
| `item:getLore()` | table | List of lore lines |
| `item:setLore(lines)` | — | Set lore (table of strings) |
| `item:isEmpty()` | bool | Whether it's air/nil |

**Creating items:**
```lua
local item = noxium.item("DIAMOND_SWORD", 1)
item:setName("&bFrost Blade")
item:setLore({"&7A sword made of pure frost", "&cDeals 10 damage"})
player:giveItem(item)
```

---

### Inventory Object

Obtained via `player:getInventory()`.

| Method | Returns | Description |
|---|---|---|
| `inv:getSize()` | number | Total slots |
| `inv:getItem(slot)` | Item | Item at slot (1-indexed) |
| `inv:setItem(slot, item)` | — | Set item at slot (1-indexed) |
| `inv:addItem(item)` | — | Add item to first empty slot |
| `inv:removeItem(item)` | — | Remove first matching item |
| `inv:contains(material)` | bool | Whether material is in inventory |
| `inv:clear()` | — | Clear all items |

---

### Entity Object

Entities that are not players have these methods:

| Method | Returns | Description |
|---|---|---|
| `entity:getType()` | string | Entity type, e.g. `"ZOMBIE"` |
| `entity:getName()` | string | Entity name |
| `entity:getUUID()` | string | Entity UUID |
| `entity:getLocation()` | Location | Current location |
| `entity:getWorld()` | World | Current world |
| `entity:teleport(location)` | — | Teleport entity |
| `entity:remove()` | — | Remove entity from world |
| `entity:isAlive()` | bool | Whether entity is alive |
| `entity:isDead()` | bool | Whether entity is dead |
| `entity:getHealth()` | number | Health (LivingEntity only) |
| `entity:setHealth(n)` | — | Set health (LivingEntity only) |
| `entity:getMaxHealth()` | number | Max health (LivingEntity only) |
| `entity:damage(amount)` | — | Deal damage (LivingEntity only) |

---

### Event Objects

All event objects share these patterns. Specific fields depend on the event type.

**Common fields:**
- `e.player` — the Player involved (if applicable)
- `e.entity` — the Entity involved (if applicable)
- `e.block` — the Block involved (if applicable)
- `e:setCancelled(bool)` — cancel the event (prevents default behavior)
- `e:isCancelled()` — check if already cancelled

---

## Event Reference

Register any event with `noxium.on("EventName", function(e) ... end)`.

### Player Events

#### `PlayerJoin`
Fires when a player joins the server.
```lua
-- e.player     → Player
-- e:getMessage()      → string (join message)
-- e:setMessage(msg)   → set join message
noxium.on("PlayerJoin", function(e)
    e:setMessage("&a" .. e.player:getName() .. " joined!")
end)
```

#### `PlayerQuit`
Fires when a player leaves.
```lua
-- e.player     → Player
-- e:getMessage()      → string (quit message)
-- e:setMessage(msg)   → set quit message
```

#### `PlayerChat`
Fires when a player sends a chat message. **Async event.**
```lua
-- e.player          → Player
-- e:getMessage()    → string
-- e:setMessage(msg) → change the chat message
-- e:setCancelled(bool)
noxium.on("PlayerChat", function(e)
    -- Prefix every message with player's level
    local level = e.player:getLevel()
    e:setMessage("[Lv." .. level .. "] " .. e:getMessage())
end)
```

#### `PlayerCommand`
Fires when a player runs a command.
```lua
-- e.player           → Player
-- e:getCommand()     → string (full command with /)
-- e:setCancelled(bool)
```

#### `PlayerDeath`
Fires when a player dies.
```lua
-- e.player              → Player (who died)
-- e.killer              → Player | nil (killer if a player)
-- e:getDeathMessage()   → string
-- e:setDeathMessage(msg)
-- e:keepInventory(bool) → prevent item drops
-- e:keepLevel(bool)     → prevent XP loss
noxium.on("PlayerDeath", function(e)
    e:keepInventory(true)
    e:setDeathMessage("&c" .. e.player:getName() .. " &7was defeated.")
end)
```

#### `PlayerRespawn`
Fires when a player respawns.
```lua
-- e.player                    → Player
-- e:getRespawnLocation()      → Location
-- e:setRespawnLocation(loc)
```

#### `PlayerMove`
Fires every tick a player moves. **High frequency — keep code fast!**
```lua
-- e.player    → Player
-- e.from      → Location (previous)
-- e.to        → Location (new)
-- e:setCancelled(bool)
```

#### `PlayerInteract`
Fires when a player right/left-clicks a block or air.
```lua
-- e.player    → Player
-- e.action    → string: "RIGHT_CLICK_BLOCK", "LEFT_CLICK_BLOCK", "RIGHT_CLICK_AIR", "LEFT_CLICK_AIR"
-- e.block     → Block | nil (clicked block)
-- e.item      → Item | nil (item in hand)
-- e:setCancelled(bool)
noxium.on("PlayerInteract", function(e)
    if e.action == "RIGHT_CLICK_BLOCK" and e.block:getType() == "STONE" then
        e.player:sendMessage("You right-clicked stone!")
    end
end)
```

#### `PlayerDropItem`
Fires when a player drops an item.
```lua
-- e.player  → Player
-- e.item    → Item (dropped item)
-- e:setCancelled(bool)
```

#### `PlayerPickupItem`
Fires when a player picks up an item.
```lua
-- e.player  → Player
-- e.item    → Item
-- e:setCancelled(bool)
```

#### `PlayerGameModeChange`
Fires when a player's game mode changes.
```lua
-- e.player      → Player
-- e.newGameMode → string ("SURVIVAL", "CREATIVE", etc.)
```

---

### Block Events

#### `BlockBreak`
Fires when a player breaks a block.
```lua
-- e.player  → Player
-- e.block   → Block (the broken block)
-- e:setCancelled(bool)
-- e:setDropItems(bool) → whether to drop items
noxium.on("BlockBreak", function(e)
    if e.block:getType() == "DIAMOND_ORE" then
        e.player:sendMessage("&bYou found diamond ore!")
    end
end)
```

#### `BlockPlace`
Fires when a player places a block.
```lua
-- e.player       → Player
-- e.block        → Block (placed block)
-- e.blockAgainst → Block (block placed against)
-- e.item         → Item (item used to place)
-- e:setCancelled(bool)
```

---

### Entity Events

#### `EntityDamage`
Fires when any entity takes damage from any source.
```lua
-- e.entity        → Entity (the victim)
-- e.cause         → string ("FALL", "FIRE", "ENTITY_ATTACK", "PROJECTILE", etc.)
-- e:getDamage()   → number
-- e:setDamage(n)
-- e:setCancelled(bool)
noxium.on("EntityDamage", function(e)
    -- Prevent fall damage
    if e.cause == "FALL" then
        e:setCancelled(true)
    end
end)
```

#### `EntityDamageByEntity`
Fires when an entity damages another entity.
```lua
-- e.entity        → Entity (victim)
-- e.damager       → Entity (attacker)
-- e:getDamage()   → number
-- e:setDamage(n)
-- e:setCancelled(bool)
noxium.on("EntityDamageByEntity", function(e)
    -- Double damage from players
    if e.damager._type == "player" then
        e:setDamage(e:getDamage() * 2)
    end
end)
```

#### `EntityDeath`
Fires when any entity dies.
```lua
-- e.entity          → Entity
-- e.killer          → Player | nil
-- e:getDrops()      → table of Items
-- e:setDroppedExp(n)
```

#### `CreatureSpawn`
Fires when a creature spawns.
```lua
-- e.entity        → Entity
-- e.spawnReason   → string ("NATURAL", "SPAWNER", "EGG", "COMMAND", etc.)
-- e:setCancelled(bool)
noxium.on("CreatureSpawn", function(e)
    -- Prevent natural mob spawning
    if e.spawnReason == "NATURAL" then
        e:setCancelled(true)
    end
end)
```

---

### World Events

#### `WeatherChange`
Fires when a world's weather changes.
```lua
-- e.world          → World
-- e.toWeatherState → string ("STORM" or "CLEAR")
noxium.on("WeatherChange", function(e)
    if e.toWeatherState == "STORM" then
        noxium.broadcast("&9A storm is coming to " .. e.world:getName() .. "!")
    end
end)
```

---

## Color Codes

Use `&` followed by a code in any string passed to Noxium's message functions.  
They are automatically translated to Minecraft color codes.

| Code | Color/Format |
|------|-------------|
| `&0` | Black |
| `&1` | Dark Blue |
| `&2` | Dark Green |
| `&3` | Dark Aqua |
| `&4` | Dark Red |
| `&5` | Dark Purple |
| `&6` | Gold |
| `&7` | Gray |
| `&8` | Dark Gray |
| `&9` | Blue |
| `&a` | Green |
| `&b` | Aqua |
| `&c` | Red |
| `&d` | Light Purple |
| `&e` | Yellow |
| `&f` | White |
| `&l` | **Bold** |
| `&o` | *Italic* |
| `&n` | Underline |
| `&m` | Strikethrough |
| `&k` | Obfuscated |
| `&r` | Reset |

```lua
player:sendMessage("&c&lError! &r&7Something went wrong.")
```

---

## Examples

### Welcome Plugin
```lua
-- welcome.nox
noxium.on("PlayerJoin", function(e)
    local name = e.player:getName()
    e:setMessage("&a► &f" .. name .. " &7joined the game")
    noxium.schedule(40, function()
        if e.player:isOnline() then
            e.player:sendMessage("")
            e.player:sendMessage("&6&l   WELCOME TO THE SERVER")
            e.player:sendMessage("&7   There are &f" .. noxium.getPlayerCount() .. " &7players online.")
            e.player:sendMessage("")
            e.player:sendTitle("&6Welcome!", "&f" .. name, 10, 60, 20)
        end
    end)
end)
```

### Anti-Fall Damage
```lua
-- no_fall.nox
noxium.on("EntityDamage", function(e)
    if e.cause == "FALL" then
        e:setCancelled(true)
    end
end)
```

### Chat Format
```lua
-- chat.nox
noxium.on("PlayerChat", function(e)
    local player = e.player
    local prefix = player:isOp() and "&c[Admin] " or "&7[Player] "
    local name = "&f" .. player:getName()
    local msg = "&7» &f" .. e:getMessage()
    e:setMessage(prefix .. name .. " " .. msg)
end)
```

### Random Loot on Death
```lua
-- death_loot.nox
noxium.on("PlayerDeath", function(e)
    e:keepInventory(true)

    -- Give random item on respawn
    noxium.schedule(5, function()
        local items = {"BREAD", "APPLE", "COOKED_BEEF", "GOLDEN_APPLE"}
        local chosen = items[math.random(#items)]
        e.player:giveItem(noxium.item(chosen, math.random(1, 5)))
        e.player:sendMessage("&7You received a &f" .. chosen .. " &7as a consolation prize.")
    end)
end)
```

### Teleport Hub
```lua
-- hub.nox
local spawn = noxium.location("world", 0, 65, 0, 0, 0)

noxium.command("spawn", "Teleport to spawn", function(sender, args)
    if sender:getName() == "CONSOLE" then
        sender:sendMessage("&cPlayers only.")
        return
    end
    sender:teleport(spawn)
    sender:sendMessage("&aTeleported to spawn!")
    sender:playSound("ENTITY_ENDERMEN_TELEPORT", 1.0, 1.0)
end)

noxium.on("PlayerJoin", function(e)
    -- Teleport new players to spawn
    noxium.schedule(10, function()
        if e.player:isOnline() then
            e.player:teleport(spawn)
        end
    end)
end)
```

### Scoreboard-style Broadcast Counter
```lua
-- counter.nox
local count = 0

noxium.scheduleRepeating(0, 20, function()
    count = count + 1
end)

noxium.command("time", "See how long the server has been running", function(sender, args)
    local seconds = count
    local mins = math.floor(seconds / 60)
    local secs = seconds % 60
    sender:sendMessage("&7Server uptime: &f" .. mins .. "m " .. secs .. "s")
end)
```

---

## Tips & Patterns

### Checking if sender is a player
```lua
noxium.command("fly", function(sender, args)
    if sender._type ~= "player" then
        sender:sendMessage("&cPlayers only!")
        return
    end
    -- now safe to use player-specific methods
    sender:setFlying(true)
end)
```

### Storing per-player data
```lua
local data = {}  -- keyed by player name

noxium.on("PlayerJoin", function(e)
    data[e.player:getName()] = { kills = 0, deaths = 0 }
end)

noxium.on("PlayerDeath", function(e)
    local name = e.player:getName()
    if data[name] then
        data[name].deaths = data[name].deaths + 1
    end
end)
```

### Utility helper functions
```lua
-- Define helpers at the top of your script
local function isPlayer(sender)
    return sender._type == "player"
end

local function requirePermission(sender, perm)
    if not sender:hasPermission(perm) then
        sender:sendMessage("&cYou don't have permission to do that.")
        return false
    end
    return true
end

noxium.command("op2", function(sender, args)
    if not requirePermission(sender, "noxium.admin") then return end
    -- ...
end)
```

### Combining multiple events
```lua
-- Track player positions for a teleport-back feature
local lastSafeLocation = {}

noxium.on("PlayerJoin", function(e)
    lastSafeLocation[e.player:getName()] = e.player:getLocation()
end)

noxium.on("PlayerMove", function(e)
    local player = e.player
    local name = player:getName()
    if player:getGameMode() == "SURVIVAL" then
        local block = e.to:getBlock()
        if not block:isLiquid() then
            lastSafeLocation[name] = e.to
        end
    end
end)

noxium.command("back", "Return to last safe location", function(sender, args)
    local loc = lastSafeLocation[sender:getName()]
    if loc then
        sender:teleport(loc)
        sender:sendMessage("&aTeleported back!")
    end
end)
```

---

## Configuration (`config.yml`)

```yaml
# Enable debug logging
debug: false

# Allow scripts to use io/os Lua libraries (file access, process execution)
# Set to true only if you fully trust your scripts!
allow-unsafe-libs: false

# Maximum instructions per Lua call (0 = unlimited)
# Prevents infinite loops from hanging the server
max-instructions: 100000

scripts:
  # Log each script name when loading
  verbose-loading: true
  # Stop loading all scripts if one fails
  fail-fast: false
```

---

*Noxium v1.0.0 — Lua scripting for Minecraft*
