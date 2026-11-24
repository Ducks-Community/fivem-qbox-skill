# Security & Database Standards

FiveM clients are completely untrusted. Treat every client-to-server call (`lib.callback`, `RegisterNetEvent`) as potentially spoofed by an attacker.

## 1. Server-Authoritative Trust Boundaries

**Rules:**
- **Never accept gameplay values from the client**: Do not accept prices, item names, reward amounts, coordinates, player money deltas, or permission flags from client arguments.
- **Accept only intent and identifiers**: Clients send an action ID, config key, shop index, or entity `netId`; the server looks up authoritative items, prices, coordinates, and job rules from `config/server.lua` or `config/shared.lua`.
- **Deduct before granting & check outcomes**: Remove currency/items first, verify the removal succeeded, check `CanCarryItem` before giving rewards, and refund if a downstream step fails.
- **Rate-limit every endpoint**: Guard callbacks and net events with a per-source timestamp table (`GetGameTimer()`) to prevent spam/dupe exploits.

## 2. Server Validation Template

Follow this exact validation order inside every server `lib.callback.register` or `RegisterNetEvent`:

```lua
-- server/shop.lua
local serverConfig = require 'config.server'
local sharedConfig = require 'config.shared'

---@type table<integer, integer>
local cooldowns = {}

---@param source integer
---@param durationMs integer
---@return boolean
local function checkCooldown(source, durationMs)
    local now = GetGameTimer()
    local expiresAt = cooldowns[source] or 0
    if now < expiresAt then return false end

    cooldowns[source] = now + durationMs
    return true
end

AddEventHandler('playerDropped', function()
    cooldowns[source] = nil
end)

---@param source integer
---@param shopId string
---@param itemIndex integer
---@return boolean success, string? errorKey
lib.callback.register('my_resource:server:buyItem', function(source, shopId, itemIndex)
    -- 1. Rate-limit check
    if not checkCooldown(source, 1000) then
        return false, 'error.rate_limited'
    end

    -- 2. Input type & config resolution (ignore any client-sent prices or names)
    if type(shopId) ~= 'string' or type(itemIndex) ~= 'number' then
        return false, 'error.invalid_request'
    end

    local shop = sharedConfig.shops[shopId]
    local entry = shop and shop.items[itemIndex]
    if not shop or not entry then
        return false, 'error.invalid_request'
    end

    -- 3. Player lookup
    local player = exports.qbx_core:GetPlayer(source)
    if not player then
        return false, 'error.no_player'
    end

    -- 4. Permission / job / duty check
    if shop.requiredJob then
        local job = player.PlayerData.job
        if job.name ~= shop.requiredJob or not job.onduty then
            return false, 'error.unauthorized'
        end
    end

    -- 5. Server-side distance check
    local ped = GetPlayerPed(source)
    local playerCoords = GetEntityCoords(ped)
    if #(playerCoords - shop.coords) > (serverConfig.maxInteractDistance or 5.0) then
        return false, 'error.too_far'
    end

    -- 6. Inventory capacity & atomic payment
    if not exports.ox_inventory:CanCarryItem(source, entry.item, 1) then
        return false, 'error.inventory_full'
    end

    if not player.Functions.RemoveMoney('cash', entry.price, 'shop-purchase') then
        return false, 'error.insufficient_funds'
    end

    local added = exports.ox_inventory:AddItem(source, entry.item, 1)
    if not added then
        player.Functions.AddMoney('cash', entry.price, 'shop-purchase-refund')
        return false, 'error.failed'
    end

    return true
end)
```

## 3. `oxmysql` Best Practices

Always load `@oxmysql/lib/MySQL.lua` in `fxmanifest.lua`, use the coroutine `.await` API (never legacy callback pyramids), and **always** bind parameters with `?` placeholders. String concatenation (`..`) or `string.format` in SQL queries is strictly prohibited.

```lua
-- server/database.lua

---@class PlayerProgressRow
---@field citizenid string
---@field level integer
---@field experience integer

---@param citizenId string
---@return PlayerProgressRow?
local function getPlayerProgress(citizenId)
    return MySQL.single.await(
        'SELECT citizenid, level, experience FROM player_progress WHERE citizenid = ? LIMIT 1',
        { citizenId }
    )
end

---@param citizenId string
---@return integer level
local function getPlayerLevel(citizenId)
    local level = MySQL.scalar.await(
        'SELECT level FROM player_progress WHERE citizenid = ? LIMIT 1',
        { citizenId }
    )
    return level or 1
end

---@param citizenId string
---@param level integer
---@param experience integer
---@return integer affectedRows
local function savePlayerProgress(citizenId, level, experience)
    return MySQL.update.await(
        'INSERT INTO player_progress (citizenid, level, experience) VALUES (?, ?, ?) ON DUPLICATE KEY UPDATE level = VALUES(level), experience = VALUES(experience)',
        { citizenId, level, experience }
    )
end
```

### Multi-Step Atomic Writes (`MySQL.transaction.await`)

Wrap multi-table updates (e.g., transferring ownership or logging economy deltas) in `MySQL.transaction.await`:

```lua
---@param fromCitizenId string
---@param toCitizenId string
---@param vehicleId integer
---@return boolean committed
local function transferVehicleRecord(fromCitizenId, toCitizenId, vehicleId)
    return MySQL.transaction.await({
        {
            query = 'UPDATE player_vehicles SET citizenid = ? WHERE id = ? AND citizenid = ?',
            values = { toCitizenId, vehicleId, fromCitizenId },
        },
        {
            query = 'INSERT INTO vehicle_transfer_logs (vehicle_id, from_citizenid, to_citizenid) VALUES (?, ?, ?)',
            values = { vehicleId, fromCitizenId, toCitizenId },
        },
    })
end
```

### High-Frequency Queries (`MySQL.prepare.await`)

For repeated identical queries (e.g., batch syncing), use `MySQL.prepare.await` for prepared statement performance:

```lua
local rows = MySQL.prepare.await(
    'SELECT plate, garage FROM player_vehicles WHERE citizenid = ?',
    { citizenId }
)
```
