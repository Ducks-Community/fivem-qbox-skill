# Qbox & Ecosystem Standards

Target **Qbox (`qbx_core`)** natively alongside `ox_lib`, `ox_inventory`, `ox_target`, `sleepless_interact`, `qbx_vehicles`, and `pma-voice`.

## 1. Resource Structure & `fxmanifest.lua`

Keep server configs isolated from client downloads by loading modules with `require` and exposing only client/shared configs via `files`.

```text
my_resource/
├── fxmanifest.lua
├── config/
│   ├── client.lua
│   ├── server.lua        # NEVER listed in shared_scripts or files
│   └── shared.lua
├── locales/
│   ├── en.json
│   └── tr.json
├── client/
│   └── main.lua
├── server/
│   └── main.lua
└── shared/
    └── types.lua         # Optional LuaLS @class / @enum definitions
```

```lua
fx_version 'cerulean'
game 'gta5'

name 'my_resource'
author 'Server Team'
version '1.0.0'

lua54 'yes'
use_experimental_fxv2_oal 'yes'

ox_libs {
    'locale',
}

shared_scripts {
    '@ox_lib/init.lua',
    '@qbx_core/modules/lib.lua',
}

client_scripts {
    '@qbx_core/modules/playerdata.lua',
    'client/*.lua',
}

server_scripts {
    '@oxmysql/lib/MySQL.lua',
    'server/*.lua',
}

files {
    'config/client.lua',
    'config/shared.lua',
    'locales/*.json',
}

dependencies {
    'ox_lib',
    'qbx_core',
    'ox_inventory',
}
```

Load configs via `require`:

```lua
-- client/main.lua
local config = require 'config.client'
local shared = require 'config.shared'

-- server/main.lua
local serverConfig = require 'config.server'
local shared = require 'config.shared'
```

## 2. `qbx_core` Native APIs

Never call `exports['qb-core']:GetCoreObject()`.

### Server Player Access & Notifications

```lua
---@param source integer
---@return table? player
local function getValidPlayer(source)
    local player = exports.qbx_core:GetPlayer(source)
    if not player then return nil end
    return player
end

-- Lookup by CitizenID
local target = exports.qbx_core:GetPlayerByCitizenId(citizenId)

-- Server-to-client notification via qbx_core or ox_lib
exports.qbx_core:Notify(source, locale('notify.saved'), 'success')
```

### Client Player Data (`QBX.PlayerData`)

`@qbx_core/modules/playerdata.lua` automatically keeps `QBX.PlayerData` synchronized on the client:

```lua
---@param jobName string
---@return boolean
local function hasJob(jobName)
    local job = QBX.PlayerData and QBX.PlayerData.job
    return job ~= nil and job.name == jobName and job.onduty == true
end
```

### Vehicle Spawning & Deletion (`qbx.spawnVehicle`, `qbx_core:DeleteVehicle`)

Spawn vehicles server-side using `qbx.spawnVehicle` (from `@qbx_core/modules/lib.lua`), and delete via Qbox rather than raw `DeleteEntity` on persisted vehicles:

```lua
-- Server-side spawn
local netId, veh = qbx.spawnVehicle({
    model = joaat(modelName),
    spawnSource = spawnCoords,
    warp = GetPlayerPed(source),
    props = vehicleProps,
})

-- Server-side delete (handles persistence cleanly)
exports.qbx_core:DeleteVehicle(veh)

-- Client-side helper (non-persisted local/client utility)
qbx.deleteVehicle(vehicleEntity)
```

## 3. `qbx_vehicles` APIs

Use `qbx_vehicles` exports for owned vehicle records instead of raw SQL mutations or legacy `qb-vehiclekeys` wrappers.

```lua
-- Register a newly owned vehicle
local vehicleId, err = exports.qbx_vehicles:CreatePlayerVehicle({
    model = 'sultan',
    citizenid = player.PlayerData.citizenid,
    garage = 'pillboxgarage',
    props = { model = 'sultan', plate = plate },
})

-- Resolve vehicleId and read full vehicle data
local vehicleId = exports.qbx_vehicles:GetVehicleIdByPlate(plate)
if vehicleId then
    local vehicleData = exports.qbx_vehicles:GetPlayerVehicle(vehicleId)
end

-- Check plate uniqueness before generating
local exists = exports.qbx_vehicles:DoesPlayerVehiclePlateExist(plate)

-- Remove owned vehicle record ('citizenid' | 'license' | 'plate' | 'vehicleId')
local success, delErr = exports.qbx_vehicles:DeletePlayerVehicles('vehicleId', vehicleId)
```

## 4. `ox_inventory` APIs

Perform all inventory mutations on the server. Always check `CanCarryItem` before adding items and check return booleans.

```lua
-- Server-side item operations
local canCarry = exports.ox_inventory:CanCarryItem(source, 'bandage', 2)
if not canCarry then
    return false, 'error.inventory_full'
end

local removed = exports.ox_inventory:RemoveItem(source, 'cloth', 3)
if not removed then
    return false, 'error.missing_materials'
end

local added = exports.ox_inventory:AddItem(source, 'bandage', 2)

-- Count / Search
local count = exports.ox_inventory:GetItemCount(source, 'lockpick')
local slots = exports.ox_inventory:Search(source, 'slots', 'water')

-- Stash registration (call on server resource start)
exports.ox_inventory:RegisterStash('workshop_stash', locale('stash.label'), 40, 100000, false)
```

### Inventory Hooks (`registerHook`)

Use `registerHook` strictly for validation (`swapItems` returning `false`) or metadata enrichment (`createItem` returning a metadata table) — never perform heavy DB writes inside a hook callback:

```lua
local hookId = exports.ox_inventory:registerHook('swapItems', function(payload)
    if payload.toType == 'trunk' or payload.toType == 'glovebox' then
        return false
    end
end, {
    itemFilter = { police_evidence = true },
})
```

## 5. `ox_target` vs `sleepless_interact`

### `ox_target` (Multi-Option & Deliberate Targeting)

Store zone/target IDs so they can be removed cleanly on exit or `onResourceStop`:

```lua
local zoneId = exports.ox_target:addBoxZone({
    coords = vec3(452.0, -980.0, 30.6),
    size = vec3(1.5, 1.5, 2.0),
    rotation = 0.0,
    options = {
        {
            name = 'armory_weapons',
            icon = 'fa-solid fa-gun',
            label = locale('target.weapons'),
            groups = { police = 0 },
            onSelect = function()
                exports.ox_inventory:openInventory('stash', 'police_armory')
            end,
        },
        {
            name = 'armory_gear',
            icon = 'fa-solid fa-vest',
            label = locale('target.gear'),
            groups = { police = 0 },
            onSelect = openGearMenu,
        },
    },
})

-- Cleanup: exports.ox_target:removeZone(zoneId)
-- For entities: exports.ox_target:addLocalEntity(entity, options) / removeLocalEntity(entity, optionNames)
-- For models/vehicles: exports.ox_target:addModel(models, options) / addGlobalVehicle(options)
```

### `sleepless_interact` (Fast, Single-Action Fixed Points)

Use for quick world points, stashes, lockers, doors, garages, crafting benches, or single-action NPCs:

```lua
local interactId = 'workshop_bench_1'

exports.sleepless_interact:addCoords(vec3(120.5, -1040.2, 29.2), {
    id = interactId,
    label = locale('interact.open_bench'),
    icon = 'fa-solid fa-wrench',
    distance = 1.8,
    groups = { mechanic = 0 },
    canInteract = function()
        return not cache.vehicle
    end,
    onSelect = function()
        openCraftingMenu()
    end,
})

-- Cleanup:
-- exports.sleepless_interact:removeCoords(interactId)
-- exports.sleepless_interact:addLocalEntity(ped, options) / removeLocalEntity(ped, optionName)
```

## 6. `pma-voice` APIs

Control radio/call channels and read voice proximity reactively via StateBags (never poll voice mode in a loop):

```lua
-- Client-side channel control (0 disconnects)
exports['pma-voice']:setRadioChannel(42)
exports['pma-voice']:setCallChannel(0)
exports['pma-voice']:setVoiceProperty('radioEnabled', true)

-- Read proximity directly from StateBag
local proximity = LocalPlayer.state.proximity -- { index, distance, mode }

-- React to proximity changes via StateBag handler
AddStateBagChangeHandler('proximity', ('player:%s'):format(cache.serverId), function(_, _, value)
    if not value then return end
    -- value.index, value.distance, value.mode
end)
```
