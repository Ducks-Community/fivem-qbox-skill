# FiveM Qbox AI Skill (`fivem-qbox`)

An AI coding skill (`SKILL.md` + progressive reference guides) for modern **FiveM Lua 5.4** development on the **Qbox (`qbx_core`)** framework. Designed to stop AI coding assistants from generating legacy QBCore boilerplate, `Wait(0)` polling loops, manual callback events, and insecure client-trusted logic.

## What It Enforces

- **`ox_lib` First**: Uses `lib.callback`, `lib.points`, `lib.zones`, `lib.cache` / `lib.onCache`, `lib.requestModel`, `lib.requestAnimDict`, `lib.waitFor`, `lib.addKeybind`, `lib.cron`, and `lib.notify` instead of reinventing utilities.
- **Zero-Loop Event-Driven (`0.00ms` Idle)**: Eliminates `while true do Wait(0)` threads, `GetDistanceBetweenCoords`, repeated `PlayerPedId()`, and `[E]` key-check loops in favor of StateBags, `lib.onCache`, `lib.points`, and keybinds.
- **Native Qbox & Ecosystem APIs**: Targets `qbx_core` (`exports.qbx_core`, `QBX.PlayerData`, `qbx.spawnVehicle`), `qbx_vehicles`, `ox_inventory`, `pma-voice`, and includes an explicit decision matrix between **`ox_target`** (multi-option / targeted entities) and **`sleepless_interact`** (fast, single-action fixed points).
- **Strict LuaLS Typing & Modularity**: Requires `---@class`, `---@field`, `---@param`, and `---@return` annotations, `lua54 'yes'`, `use_experimental_fxv2_oal 'yes'`, `local`-only scoping, guard clauses, `require`-based modules, and deterministic `onResourceStop` cleanup.
- **Server-Authoritative Security & `oxmysql`**: Enforces server-side resolution of config IDs, player/job/distance/cooldown validation, `CanCarryItem` + atomic transactions, and parameterized `MySQL.*.await` queries (`?` placeholders).
- **Mandatory `lib.locale()` Localization**: Enforces `ox_libs { 'locale' }` with JSON locale files (`locales/tr.json` + `locales/en.json`).

## Repository Structure

```text
fivem-qbox/
├── SKILL.md                             # Core rules, prohibitions, interaction decision matrix & checklist
├── references/
│   ├── ox_lib-recipes.md                # Callbacks, points, zones, cache, asset loading, keybinds, cron, locale
│   ├── qbox-ecosystem.md                # fxmanifest, qbx_core, qbx_vehicles, ox_inventory, ox_target, sleepless_interact, pma-voice
│   ├── security-and-db.md               # Trust boundaries, server validation template, oxmysql .await patterns
│   └── luals-and-clean-code.md          # LuaLS annotations, modular require structure, StateBags, onResourceStop cleanup
├── README.md
└── LICENSE
```

## Installation

### Antigravity (Global — all workspaces)

```bash
git clone https://github.com/Ducks-Community/fivem-qbox-skill.git ~/.gemini/config/skills/fivem-qbox
```

### Antigravity (Workspace / Project-local)

```bash
git clone https://github.com/Ducks-Community/fivem-qbox-skill.git .agents/skills/fivem-qbox
```

### Claude Code / Other `SKILL.md`-Compatible Agents

Clone into your agent's skills directory (for example, Claude Code global skills):

```bash
git clone https://github.com/Ducks-Community/fivem-qbox-skill.git ~/.claude/skills/fivem-qbox
```
