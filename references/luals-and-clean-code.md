# LuaLS Typing, Modularity & Event-Driven Cleanup

Write clean, strongly typed Lua 5.4 code with zero global namespace pollution, shallow nesting, StateBag synchronization, and deterministic resource cleanup.

## 1. Strict LuaLS Type Annotations

Annotate every function (`---@param`, `---@return`) and custom structure (`---@class`, `---@field`, `---@type`, `---@enum`) so the Lua Language Server catches mismatches immediately.

```lua
---@enum WorkshopState
local WorkshopState = {
    IDLE = 1,
    WORKING = 2,
    COOLDOWN = 3,
}

---@class CraftRecipe
---@field item string Target item name in ox_inventory
---@field amount integer Output quantity
---@field duration integer Crafting duration in milliseconds
---@field ingredients table<string, integer> Map of itemName -> requiredCount

---@class CraftStation
---@field id string Unique station identifier
---@field coords vector3 World position of the bench
---@field job? string Optional required job name
---@field recipes CraftRecipe[]

---@type table<string, CraftStation>
local stations = {}

---@param stationId string
---@param recipeIndex integer
---@return CraftRecipe?
local function findRecipe(stationId, recipeIndex)
    local station = stations[stationId]
    if not station then return nil end

    return station.recipes[recipeIndex]
end
```

## 2. Anti-Spaghetti Modularization

### Rules
1. **Everything `local` by default**: Never create implicit global variables or global helper functions across files. Return a module table and import it with `require`.
2. **Guard Clauses (Early Returns)**: Invert negative checks and return early. Never exceed 3 levels of indentation.
3. **Single-Responsibility Helpers**: Keep each function focused on one task (<40 lines when practical). Split large client/server scripts into domain modules (e.g., `client/targets.lua`, `client/peds.lua`, `server/callbacks.lua`, `server/db.lua`).
4. **No Magic Numbers**: Move coordinates, distances, durations, and item tables into `config/client.lua`, `config/server.lua`, or `config/shared.lua`.

### Refactoring Example (Spaghetti vs. Clean)

```lua
-- BAD: Deep nesting, repeated natives, magic numbers
function DoWork(station)
    if station ~= nil then
        if IsPedInAnyVehicle(PlayerPedId(), false) == false then
            if QBX.PlayerData.job.name == 'mechanic' then
                -- 50 lines of crafting logic...
            end
        end
    end
end

-- GOOD: Early returns, cached ped, LuaLS types, modular helper
---@param station CraftStation
---@return boolean canUse
local function canUseStation(station)
    if cache.vehicle then return false end
    if not QBX.PlayerData or not QBX.PlayerData.job then return false end
    if station.job and QBX.PlayerData.job.name ~= station.job then return false end
    if not QBX.PlayerData.job.onduty then return false end

    return true
end
```

### Splitting into Modules with `require`

```lua
-- client/peds.lua
local M = {}

---@type integer[]
local spawnedPeds = {}

---@param model string | integer
---@param coords vector4
---@return integer?
function M.spawn(model, coords)
    local hash = lib.requestModel(model, 5000)
    if not hash then return nil end

    local ped = CreatePed(0, hash, coords.x, coords.y, coords.z - 1.0, coords.w, false, true)
    SetModelAsNoLongerNeeded(hash)
    FreezeEntityPosition(ped, true)
    SetEntityInvincible(ped, true)
    SetBlockingOfNonTemporaryEvents(ped, true)

    spawnedPeds[#spawnedPeds + 1] = ped
    return ped
end

function M.cleanup()
    for i = 1, #spawnedPeds do
        local ped = spawnedPeds[i]
        if DoesEntityExist(ped) then
            DeleteEntity(ped)
        end
    end
    table.wipe(spawnedPeds)
end

return M
```

```lua
-- client/main.lua
local pedManager = require 'client.peds'
```

## 3. Event-Driven Synchronization via StateBags

Prefer **StateBags** over broadcasting `TriggerClientEvent(-1, ...)` whenever syncing entity or player state across nearby clients.

```lua
-- Server: set replicated state on an entity (automatically streams to clients entering scope)
Entity(vehicle).state:set('fuelLevel', 85.0, true)

-- Client: react without polling
AddStateBagChangeHandler('fuelLevel', nil, function(bagName, _, value)
    if type(value) ~= 'number' then return end

    local entity = GetEntityFromStateBagName(bagName)
    if entity == 0 or not DoesEntityExist(entity) then return end

    -- Apply updated fuel state once when changed
end)
```

For local-only flags (e.g., busy lock), pass `false` as the third argument to `state:set` so it never triggers network traffic:

```lua
LocalPlayer.state:set('isCrafting', true, false)
```

## 4. Deterministic Resource Cleanup (`onResourceStop`)

Every resource that spawns peds/props/blips or registers `lib.points`, `lib.zones`, `ox_target`, or `sleepless_interact` entries **must** clean them up when stopped so live restarts (`ensure my_resource`) leave zero ghost entities or duplicate interactions.

```lua
---@type integer[]
local spawnedEntities = {}
---@type integer[]
local createdBlips = {}
---@type CPoint[]
local activePoints = {}
---@type string[]
local interactIds = {}

AddEventHandler('onResourceStop', function(resourceName)
    if resourceName ~= cache.resource then return end

    for i = 1, #interactIds do
        exports.sleepless_interact:removeCoords(interactIds[i])
    end

    for i = 1, #activePoints do
        activePoints[i]:remove()
    end

    for i = 1, #createdBlips do
        if DoesBlipExist(createdBlips[i]) then
            RemoveBlip(createdBlips[i])
        end
    end

    for i = 1, #spawnedEntities do
        local entity = spawnedEntities[i]
        if DoesEntityExist(entity) then
            exports.ox_target:removeLocalEntity(entity)
            DeleteEntity(entity)
        end
    end
end)
```
