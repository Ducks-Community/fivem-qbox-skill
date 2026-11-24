# ox_lib Patterns & Recipes

Use `ox_lib` primitives instead of writing custom threads, distance loops, asset-loading loops, or event-pair callbacks.

## 1. Callbacks (`lib.callback`)

Never pair `TriggerServerEvent` with `TriggerClientEvent` just to request data or receive an action result.

```lua
-- server/callbacks.lua
---@param source integer
---@param itemId string
---@return boolean success, string? errorKey
lib.callback.register('my_resource:server:craftItem', function(source, itemId)
    local player = exports.qbx_core:GetPlayer(source)
    if not player then return false, 'error.no_player' end

    if type(itemId) ~= 'string' then
        return false, 'error.invalid_request'
    end

    -- Perform server-side validation, then mutate state
    return true
end)
```

```lua
-- client/main.lua
---@param itemId string
local function requestCraft(itemId)
    local success, errorKey = lib.callback.await('my_resource:server:craftItem', false, itemId)
    if not success then
        lib.notify({
            title = locale('notify.title'),
            description = locale(errorKey or 'error.failed'),
            type = 'error',
        })
        return
    end

    lib.notify({
        title = locale('notify.title'),
        description = locale('notify.craft_success'),
        type = 'success',
    })
end
```

## 2. Points & Zones (`lib.points`, `lib.zones`)

Replace manual coordinate-checking `Wait(500)` / `Wait(0)` loops with `lib.points.new` or `lib.zones`. Only run frame-level drawing inside `nearby` when strictly required (e.g., 3D markers).

```lua
---@type CPoint[]
local activePoints = {}

---@param coords vector3
---@param distance number
---@param data table
local function registerSpot(coords, distance, data)
    local point = lib.points.new({
        coords = coords,
        distance = distance,
        spotData = data,
        onEnter = function(self)
            -- Spawn local prop/ped or attach target here
        end,
        onExit = function(self)
            -- Clean up local prop/ped or target here
        end,
        -- Only define `nearby` if frame rendering (e.g. DrawMarker) is genuinely needed
        nearby = function(self)
            if self.currentDistance < 10.0 then
                DrawMarker(2, self.coords.x, self.coords.y, self.coords.z, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.25, 0.25, 0.25, 200, 160, 80, 180, false, true, 2, false, nil, nil, false)
            end
        end,
    })

    activePoints[#activePoints + 1] = point
end
```

Use `lib.zones.box` or `lib.zones.sphere` when entering/exiting polygonal or rectangular boundaries:

```lua
local shopZone = lib.zones.box({
    coords = vec3(25.7, -1345.0, 29.5),
    size = vec3(6.0, 8.0, 3.0),
    rotation = 0.0,
    onEnter = function()
        LocalPlayer.state:set('inShopZone', true, false)
    end,
    onExit = function()
        LocalPlayer.state:set('inShopZone', false, false)
    end,
})
```

## 3. Cache & Reactive Listeners (`lib.cache`, `lib.onCache`)

Never poll `PlayerPedId()`, `GetVehiclePedIsIn()`, or `GetCurrentPedWeapon()` in threads. Read `cache.ped`, `cache.vehicle`, `cache.seat`, `cache.weapon`, and `cache.coords`, or react to changes via `lib.onCache`.

```lua
---@param vehicle integer | false
lib.onCache('vehicle', function(vehicle)
    if not vehicle then
        -- Player exited vehicle
        return
    end

    local class = GetVehicleClass(vehicle)
    -- React once when entering a vehicle
end)

---@param seat integer | false
lib.onCache('seat', function(seat)
    if seat ~= -1 then return end
    -- React only when player moves into driver seat
end)
```

## 4. Asset Loading & Async Waiting (`lib.requestModel`, `lib.requestAnimDict`, `lib.waitFor`)

Never write `while not HasModelLoaded(model) do Wait(10) end` loops. Always release models with `SetModelAsNoLongerNeeded` after spawning.

```lua
---@param model string | integer
---@param coords vector4
---@return integer? ped
local function spawnLocalPed(model, coords)
    local hash = lib.requestModel(model, 5000)
    if not hash then return nil end

    local ped = CreatePed(0, hash, coords.x, coords.y, coords.z - 1.0, coords.w, false, true)
    SetModelAsNoLongerNeeded(hash)

    FreezeEntityPosition(ped, true)
    SetEntityInvincible(ped, true)
    SetBlockingOfNonTemporaryEvents(ped, true)

    return ped
end

---@param ped integer
---@param dict string
---@param clip string
---@param duration integer
local function playAnim(ped, dict, clip, duration)
    if not lib.requestAnimDict(dict, 5000) then return end
    TaskPlayAnim(ped, dict, clip, 8.0, -8.0, duration, 1, 0.0, false, false, false)
    RemoveAnimDict(dict)
end
```

Use `lib.waitFor` when waiting for an asynchronous condition (e.g., network entity replication) with an explicit timeout:

```lua
---@param netId integer
---@return integer? entity
local function waitForNetEntity(netId)
    return lib.waitFor(function()
        if NetworkDoesEntityExistWithNetworkId(netId) then
            local entity = NetToEnt(netId)
            if DoesEntityExist(entity) then return entity end
        end
    end, locale('error.entity_timeout'), 5000)
end
```

## 5. Keybinds (`lib.addKeybind`) & Scheduled Jobs (`lib.cron`)

Replace `while true do Wait(0) if IsControlJustPressed(...)` with `lib.addKeybind`:

```lua
lib.addKeybind({
    name = 'my_resource_toggle',
    description = locale('keybind.toggle_desc'),
    defaultKey = 'F6',
    onPressed = function()
        if not cache.vehicle then return end
        -- Trigger action cleanly without any loop
    end,
})
```

Replace server `while true do Wait(60000)` threads with `lib.cron.new`:

```lua
-- Runs every 15 minutes
lib.cron.new('*/15 * * * *', function()
    -- Periodic server cleanup or payout
end)
```

## 6. Mandatory Localization (`lib.locale()`)

Configure `fxmanifest.lua`:

```lua
ox_libs {
    'locale',
}

files {
    'locales/*.json',
}
```

Initialize `lib.locale()` in shared/client/server entrypoints and maintain `locales/tr.json` alongside `locales/en.json` (which `ox_lib` uses as the default fallback):

```json
// locales/tr.json
{
    "notify": {
        "title": "Atölye",
        "craft_success": "%s başarıyla üretildi."
    },
    "error": {
        "no_player": "Oyuncu verisi bulunamadı.",
        "invalid_request": "Geçersiz istek.",
        "failed": "İşlem tamamlanamadı.",
        "entity_timeout": "Varlık zamanında yüklenemedi."
    }
}
```

```json
// locales/en.json
{
    "notify": {
        "title": "Workshop",
        "craft_success": "Successfully crafted %s."
    },
    "error": {
        "no_player": "Player data not found.",
        "invalid_request": "Invalid request.",
        "failed": "Action could not be completed.",
        "entity_timeout": "Entity failed to load in time."
    }
}
```

## 7. Menus, Progress Bars & Notifications

Use `lib.registerContext`, `lib.progressBar`, and `lib.notify` rather than custom draw loops or legacy `QBCore.Functions.Progressbar`:

```lua
---@param duration integer
---@param labelKey string
---@return boolean completed
local function runActionProgress(duration, labelKey)
    return lib.progressBar({
        duration = duration,
        label = locale(labelKey),
        useWhileDead = false,
        canCancel = true,
        disable = { move = true, car = true, combat = true },
    })
end
```
