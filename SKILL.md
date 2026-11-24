---
name: fivem-qbox
description: >-
  Use this skill whenever creating, editing, refactoring, reviewing, or debugging
  FiveM Lua resources for Qbox (qbx_core). Enforces ox_lib-first patterns, zero-loop
  event-driven architecture (0.00ms idle resmon), ox_target vs sleepless_interact
  selection, strict LuaLS type annotations, server-authoritative security, oxmysql
  await queries, and lib.locale() localization (no NUI).
---

# FiveM Qbox & ox_lib Development Standard

Write idiomatic, modular Lua 5.4 resources strictly targeting **Qbox (`qbx_core`)** and the **Overextended (`ox_*`)** ecosystem. Never reinvent utilities that `ox_lib` or Qbox already provide, and never introduce client polling loops or client-trusted state.

## Core Principles

1. **`ox_lib` First**: Use `lib.callback`, `lib.points`, `lib.zones`, `lib.cache` / `lib.onCache`, `lib.requestModel`, `lib.requestAnimDict`, `lib.waitFor`, `lib.addKeybind`, `lib.cron`, `lib.notify`, `lib.registerContext`, `lib.progressBar`, and `lib.locale()` instead of custom wrappers or raw natives.
2. **Event-Driven & Zero-Loop (`0.00ms` Idle)**: Achieve `0.00ms` idle `resmon`. Replace continuous `Wait(0)` threads with `lib.points`, `lib.zones`, `ox_target`, `sleepless_interact`, `lib.addKeybind`, `lib.onCache`, and **StateBags** (`AddStateBagChangeHandler`).
3. **Server-Authoritative Security**: Treat the client as untrusted. Clients send intent and config keys/indices only; the server validates player state, job/group/ACE permissions, distance, cooldowns, and inventory space before mutating state.
4. **Strict LuaLS Typing**: Annotate every function (`---@param`, `---@return`) and non-trivial table/config (`---@class`, `---@field`, `---@type`, `---@enum`). Keep `lua54 'yes'` and `use_experimental_fxv2_oal 'yes'` in `fxmanifest.lua`.
5. **Modular Anti-Spaghetti Structure**: Keep every variable/function `local`, enforce guard clauses (early returns, max 3 nesting levels), split large features into `require`-d modules, and clean up spawned entities/zones/targets on `onResourceStop`.
6. **Mandatory Localization (`lib.locale()`)**: Never hardcode user-facing strings in Lua. Load `ox_libs { 'locale' }` in `fxmanifest.lua`, maintain `locales/tr.json` (primary) and `locales/en.json` (fallback), and format via `locale('key', ...)`.

## Hard Prohibitions

| Legacy / Anti-Pattern (NEVER USE) | Required Replacement |
| :--- | :--- |
| `exports['qb-core']:GetCoreObject()` / `QBCore.*` | `exports.qbx_core:*`, `@qbx_core/modules/playerdata.lua` (`QBX.PlayerData`), `@qbx_core/modules/lib.lua` (`qbx.*`) |
| `TriggerServerEvent` + `TriggerClientEvent` request/response pairs | `lib.callback.register` & `lib.callback.await` |
| `while true do Wait(0)` / `Citizen.CreateThread` / `Citizen.Wait` | `lib.points`, `lib.zones`, `ox_target`, `sleepless_interact`, `lib.addKeybind`, `lib.onCache`, `AddStateBagChangeHandler`, modern `CreateThread`/`Wait` |
| Repeated `PlayerPedId()`, `GetVehiclePedIsIn()`, `GetEntityCoords(PlayerPedId())` | `cache.ped`, `cache.vehicle`, `cache.seat`, `cache.coords` (`ox_lib` cache) |
| `GetDistanceBetweenCoords(x1, y1, z1, x2, y2, z2, ...)` | `#(coordsA - coordsB)` vector arithmetic |
| `DrawText3D` / `lib.showTextUI` + `Wait(0)` + `IsControlJustPressed` | `sleepless_interact` (single-action fixed point) or `ox_target` (multi-option / targeted entity) |
| Manual `while not HasModelLoaded(...)` / `HasAnimDictLoaded(...)` loops | `lib.requestModel`, `lib.requestAnimDict`, `lib.waitFor` |
| Placing `config/server.lua` inside `shared_scripts` | List shared/client configs in `files` (`require 'config.client'`), keep `config/server.lua` server-only via `require 'config.server'` |
| Hardcoded Turkish/English notification, label, or menu strings | `lib.locale()` with `locales/tr.json` and `locales/en.json` |
| Trusting client-sent prices, item names, quantities, coords, or job checks | Server resolves config tables by ID and enforces all checks server-side |

## Interaction Selection: `ox_target` vs `sleepless_interact`

Choose the interaction library by interaction ergonomics — **never** fall back to a `Wait(0)` `[E]` key-check loop:

- **Use `ox_target` (Default)** when the player must deliberately aim/select among multiple actions or target dynamic entities:
  - Vehicle actions (trunk, hood, repair, lockpick, impound)
  - Player-to-player actions (cuff, search, escort, revive)
  - Multi-option NPCs (shops with multiple menus, mission hubs)
  - Dynamic world objects/models (ATMs, dumpsters, props, vehicle parts)
- **Use `sleepless_interact`** for fast, single-action fixed world points where opening the target eye is unnecessary friction:
  - Fixed doors, elevators, garages, stashes, wardrobes/lockers
  - Crafting benches, duty toggles, seating/rest spots
  - Single-action static NPCs
- **Ambiguous Cases**: If a resource's interaction style is genuinely unclear between multi-option targeting and quick single-point interaction, ask the user before implementing.

## Reference Guides (Read as Needed)

Read the relevant reference file before writing code in that area:

- [ox_lib Recipes](./references/ox_lib-recipes.md) — Read for `lib.callback`, `lib.points`, `lib.zones`, `lib.cache`/`lib.onCache`, asset requests (`lib.requestModel`, `lib.requestAnimDict`, `lib.waitFor`), `lib.addKeybind`, `lib.cron`, `lib.locale()`, menus, and progress bars.
- [Qbox Ecosystem](./references/qbox-ecosystem.md) — Read for Qbox resource structure, `fxmanifest.lua`, `qbx_core`, `qbx_vehicles`, `ox_inventory`, `ox_target`, `sleepless_interact`, and `pma-voice` APIs.
- [Security & Database](./references/security-and-db.md) — Read before writing any server event, `lib.callback.register`, economy/item logic, or `oxmysql` database query.
- [LuaLS & Clean Code](./references/luals-and-clean-code.md) — Read for `---@class`/`---@param`/`---@return` standards, modular `require` architecture, guard clauses, StateBags, and `onResourceStop` cleanup.

## Pre-Delivery Checklist

Before finishing any FiveM Qbox task, verify:
- [ ] `fxmanifest.lua` sets `lua54 'yes'`, `use_experimental_fxv2_oal 'yes'`, `ox_libs { 'locale' }`, and never exposes `config/server.lua` to the client.
- [ ] Zero idle loops (`0.00ms` idle `resmon`); no `Citizen.*`, `GetCoreObject`, `GetDistanceBetweenCoords`, or `[E]` `Wait(0)` polling.
- [ ] Interactions follow the `ox_target` vs `sleepless_interact` matrix and clean up in `onResourceStop`.
- [ ] Every function and custom data structure has `---@param`, `---@return`, and `---@class` LuaLS annotations.
- [ ] All server callbacks/events validate player, permission, distance, cooldown/rate-limit, and inventory capacity server-side; all SQL queries use `MySQL.*.await` with `?` placeholders.
- [ ] All user-facing strings use `locale(...)` backed by `locales/tr.json` and `locales/en.json`.
