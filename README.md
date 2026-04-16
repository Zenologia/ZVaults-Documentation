# 🔐 ZVaults
*Developed by Zenologia*

A per-player, per-vault cooldown system for Minecraft trial vaults — built for Paper 26.1.x servers on Java 25.
Drop it in and every `TRIAL_KEY` / `OMINOUS_TRIAL_KEY` interaction is gated by a configurable delay, tracked by block position + player UUID, with three pluggable storage backends and a polished admin command suite.

ZVaults wraps the vanilla trial vault mechanic without replacing it: vanilla still dispenses loot, and ZVaults decides whether the player is allowed to re-roll the vault yet. Cooldowns are only recorded after vanilla *confirms* loot was dispensed, so missed interactions never eat a player's cooldown.

## What it does (and doesn't) do

- ✅ Per-player, per-vault cooldown tracking (keyed on `world:x:y:z` + UUID).
- ✅ Separate durations for normal and ominous vaults.
- ✅ Timestamp is only written after vanilla confirms the vault actually dispensed loot.
- ✅ Auto-clears the vault's rewarded-players entry when a player's cooldown expires, so they can open again with no admin action needed.
- ✅ Three storage backends out of the box — YAML (default), SQLite, and MySQL (all SQL paths via HikariCP).
- ✅ Admin force-unlock (sneak + right-click) that clears the vault's rewarded-players list so the vault can dispense again.
- ✅ `zvaults.bypass` that skips the cooldown **check** but still records the timestamp — so revoking bypass puts the player on cooldown from their last open.
- ✅ Live cooldown tuning via `/zvaults setdelay` — no restart needed.
- ✅ Per-player and global cooldown wipes.
- ❌ Not a loot-table editor; ZVaults does not change what the vault drops.
- ❌ Not a throttle per time window — it enforces one cooldown per (player, vault) pair at a time, not N opens per hour.
- ❌ Storage backend (`storage.type`) is **not** hot-reloadable — changing it requires a full server restart.
- ❌ Trial keys only; the listener ignores any other held item.

---

## ⚙️ FEATURES

- 🔐 Per-player / per-vault cooldown tracking
- 🗝️ Independent cooldowns for **normal** and **ominous** vaults
- ✨ Deferred cooldown write — only commits after vanilla dispenses loot
- 🔁 Automatic rewarded-player reset when a cooldown expires (no manual step to re-open)
- 💾 Three storage backends (YAML, SQLite, MySQL) selectable at enable-time
- 🛡️ Admin force-unlock (sneak + right-click with a trial key)
- 🎯 Bypass permission that still records timestamps (respected on revoke)
- 🔧 Runtime tuning: `/zvaults getdelay`, `/zvaults setdelay <n> [o]`
- 🧹 Admin reset tools: `/zvaults resetglobal`, `/zvaults resetplayer <name>`
- 🔄 `/zvaults reload` re-reads `config.yml` + `messages.yml` without a restart
- 🎨 Fully configurable chat prefix, deny sound, deny-sound pitch
- 📝 9 branch-specific cooldown-remaining messages (minutes × seconds matrix)
- 🧰 Async prune task that trims expired entries on an interval
- 🐛 Optional debug logging of every confirmed cooldown write
- ⚡ Paper 26.1 native — registered via the new Paper command API, not legacy plugin.yml command declarations

---

## Requirements

- Paper **26.1.2+** (api-version `26.1`)
- Java **25+**
- A writable `plugins/ZVaults/` directory
- *(MySQL backend only)* A reachable MySQL/MariaDB server with a pre-created database

---

## Optional dependencies

- Any permissions plugin (LuckPerms, etc.) — for fine-grained control over `zvaults.*` nodes beyond the op default
- Bundled: HikariCP 6.2.1, sqlite-jdbc 3.49.1.0, mysql-connector-j 9.2.0 (shadowed into the plugin jar — no external downloads required)

---

## Installation

1. Drop `ZVaults-<version>.jar` into your server's `plugins/` folder.
2. Start the server once — `config.yml` and `messages.yml` auto-generate under `plugins/ZVaults/`.
3. *(Optional)* edit `config.yml` — adjust cooldown durations, storage backend, etc.
4. Reload or restart:
   ```
   /zvaults reload
   ```
   > **Note:** If you change `storage.type`, run a **full server restart**. Switching backends at runtime is not supported.

---

## Configuration (`config.yml`)

Defaults shown:

```yaml
# ZVaults Configuration
# Cooldown in seconds before a player can re-open the same vault
cooldowns:
  normal: 30
  ominous: 120

# Chat message prefix—prepended automatically to all messages.
# See messages.yml for all message templates.
prefix: "&b[&6ZVaults&b]&7"

# Sound played when a player is denied (on cooldown)
deny-sound: BLOCK_ANVIL_LAND
deny-sound-pitch: 2.0

# Auto-save/prune interval in seconds
autosave-interval-seconds: 300

# Debug logging controls (defaults are off to avoid log spam)
debug:
  cooldown-start-log: false

# Storage backend
storage:
  type: yaml # yaml | sqlite | mysql
  mysql:
    host: localhost
    port: 3306
    database: zvaults
    username: root
    password: ""
```

| Key | Default | Effect |
|---|---|---|
| `cooldowns.normal` | `30` | Cooldown in seconds after opening a **normal** vault (trial key). |
| `cooldowns.ominous` | `120` | Cooldown in seconds after opening an **ominous** vault (ominous trial key). |
| `prefix` | `"&b[&6ZVaults&b]&7"` | Chat prefix applied to every ZVaults message. |
| `deny-sound` | `BLOCK_ANVIL_LAND` | Any valid Bukkit sound name. Falls back to `BLOCK_ANVIL_LAND` if the name is invalid. |
| `deny-sound-pitch` | `2.0` | Pitch of the deny sound (typical range `0.5`–`2.0`). |
| `autosave-interval-seconds` | `300` | How often (seconds) the async prune task sweeps storage for expired entries. **Floor: 10.** Not a live-save interval — writes already happen on every open. |
| `debug.cooldown-start-log` | `false` | When `true`, logs one INFO line per confirmed cooldown write (see [Debug logging](#debug-logging)). |
| `storage.type` | `yaml` | `yaml`, `sqlite`, or `mysql`. Requires a full restart to change. |
| `storage.mysql.host` | `localhost` | MySQL host. |
| `storage.mysql.port` | `3306` | MySQL port. |
| `storage.mysql.database` | `zvaults` | MySQL database name (must already exist). |
| `storage.mysql.username` | `root` | MySQL username. |
| `storage.mysql.password` | `""` | MySQL password. |

---

## Messages (`messages.yml`)

All cooldown-denied messages are picked from one of **nine** keys based on the remaining time (minutes × seconds, with zero / one / many branches). The prefix from `config.yml` is prepended automatically — **do not** re-embed it in message strings.

```yaml
cooldown-denied:
  many-minutes-many-seconds: "&7You may re-open this vault in &e{minutes} minutes&7 and &e{seconds} seconds&7."
  many-minutes-one-second:   "&7You may re-open this vault in &e{minutes} minutes&7 and &e1 second&7."
  many-minutes-zero-seconds: "&7You may re-open this vault in &e{minutes} minutes&7."
  one-minute-many-seconds:   "&7You may re-open this vault in &e1 minute&7 and &e{seconds} seconds&7."
  one-minute-one-second:     "&7You may re-open this vault in &e1 minute&7 and &e1 second&7."
  one-minute-zero-seconds:   "&7You may re-open this vault in &e1 minute&7."
  zero-minutes-many-seconds: "&7You may re-open this vault in &e{seconds} seconds&7."
  zero-minutes-one-second:   "&7You may re-open this vault in &e1 second&7."
  zero-minutes-zero-seconds: "&7You may re-open this vault in &eless than a second&7."

force-unlock:
  success: "&aVault force-unlocked. All players may now open this vault."

admin:
  getdelay:      "&7Normal cooldown: &e{normal}s&7 | Ominous cooldown: &e{ominous}s&7."
  setdelay:      "&aCooldowns updated. &7Normal: &e{normal}s&7 | Ominous: &e{ominous}s&7."
  reset-global:  "&aAll cooldown data wiped."
  reset-player:  "&aCooldown data wiped for &e{player}&a."
  reload:        "&aConfiguration reloaded."
```

### Tokens

| Token | Where it appears | Meaning |
|---|---|---|
| `{minutes}` | `cooldown-denied.*-minutes-*` | Minutes remaining. |
| `{seconds}` | `cooldown-denied.*-seconds` | Seconds-of-last-minute remaining. |
| `{normal}` | `admin.getdelay`, `admin.setdelay` | Current normal cooldown in seconds. |
| `{ominous}` | `admin.getdelay`, `admin.setdelay` | Current ominous cooldown in seconds. |
| `{player}` | `admin.reset-player` | The target player's name, as typed. |

Color codes use `&` (e.g. `&a` green, `&c` red, `&7` gray).

---

## Commands

All commands are subcommands of `/zvaults`. Tab-completion is provided for the subcommand name and, on `/zvaults resetplayer`, for online player names.

| Command | Description | Permission |
|---|---|---|
| `/zvaults getdelay` | Print current normal and ominous cooldown durations. | `zvaults.admin.getdelay` |
| `/zvaults setdelay <normal> [ominous]` | Update cooldowns live. `<normal>` is required; `[ominous]` is optional. Values must be `≥ 0`. Saves `config.yml` and reloads settings in place. | `zvaults.admin.setdelay` |
| `/zvaults resetglobal` | Wipe **all** cooldown data from cache + storage. | `zvaults.admin.reset.global` |
| `/zvaults resetplayer <player>` | Wipe cooldown data for one player. Works on offline players — the UUID is resolved from the username you provide. | `zvaults.admin.reset.player` |
| `/zvaults reload` | Re-read `config.yml` and `messages.yml`. Emits a warning if `storage.type` changed (requires a restart to take effect). | `zvaults.admin.reload` |

> **Note:** Only the `/zvaults` root command is registered — there are no built-in aliases.

---

## Permissions

| Node | Default | Effect |
|---|---|---|
| `zvaults.use` | `true` (all players) | Required to interact with vaults under the ZVaults cooldown system. Negate it and the player's vault clicks are silently blocked by the plugin. |
| `zvaults.bypass` | `op` | Skips the cooldown **check** — the player can always open the vault. The timestamp is still recorded, so revoking this node subjects the player to cooldown from their last open. |
| `zvaults.forceunlock` | `op` | Allows sneak + right-click with a trial key to clear the vault's rewarded-players list (see [Force unlock](#force-unlock)). |
| `zvaults.admin.getdelay` | `op` | `/zvaults getdelay`. |
| `zvaults.admin.setdelay` | `op` | `/zvaults setdelay`. |
| `zvaults.admin.reset.global` | `op` | `/zvaults resetglobal`. |
| `zvaults.admin.reset.player` | `op` | `/zvaults resetplayer`. |
| `zvaults.admin.reload` | `op` | `/zvaults reload`. |
| `zvaults.admin.*` | `op` | Wildcard that grants all five `zvaults.admin.*` subcommand nodes. Does **not** grant `zvaults.use` or `zvaults.bypass`. |

---

## Storage backends

ZVaults ships three storage backends. The backend is selected once at enable-time from `storage.type` in `config.yml` — **changing it later requires a full server restart**. All three back an in-memory cache that is hydrated from the active backend on startup; every cooldown read hits the cache, and writes are mirrored to both cache and backend.

### YAML (default)

- File: `plugins/ZVaults/data.yml`
- Layout: one top-level key per player UUID, each with a child map of `world:x:y:z → epochMs`.
- Writes happen **synchronously** on every confirmed open (not only at autosave time).
- Empty-file guard: the plugin will **not** create `data.yml` until the first real cooldown is committed. Stopping the server with no recorded cooldowns leaves the data folder clean.
- No external dependencies. Ideal for single-server setups or small networks.

Example snippet from a live `data.yml`:

```yaml
b7a3e0f2-4c1a-4f44-9c35-9b3c5e4f1e77:
  world:128:65:-241: 1714765432100
  world_nether:12:70:-8: 1714765489200
```

### SQLite

- File: `plugins/ZVaults/ZVaults.db`
- Driver: bundled sqlite-jdbc 3.49.1.0 (no external download required).
- Pool: HikariCP with a max pool size of 1 connection (pool name `ZVaults-SQLite`).
- Schema auto-created on first connect (see below).
- Good fit if you want ACID semantics on a single server without running a separate database process.

### MySQL / MariaDB

- Driver: bundled MySQL Connector/J 9.2.0 (no external download required).
- JDBC URL template: `jdbc:mysql://<host>:<port>/<database>?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC`
- Pool: HikariCP with a max pool size of 10 connections (pool name `ZVaults-MySQL`).
- The database must already exist — ZVaults does not issue `CREATE DATABASE`.
- Recommended for networks where multiple servers need to share cooldown state against a common DB. Note: ZVaults does not push live cache-invalidation events between servers — each server's cache is hydrated once at enable-time.

### SQL schema (SQLite + MySQL)

Both SQL backends bootstrap the same table on first connection:

```sql
CREATE TABLE IF NOT EXISTS cooldowns (
  player_uuid   VARCHAR(36)  NOT NULL,
  vault_key     VARCHAR(255) NOT NULL,
  last_open_ms  BIGINT       NOT NULL,
  PRIMARY KEY (player_uuid, vault_key)
);
```

Upserts use `ON CONFLICT(player_uuid, vault_key) DO UPDATE` on SQLite and `ON DUPLICATE KEY UPDATE` on MySQL. Prune is a single `DELETE FROM cooldowns WHERE (? - last_open_ms) > ?`.

### Choosing a backend

| Backend | Start cost | Best for | Notes |
|---|---|---|---|
| `yaml` | None | Small servers, quick install | Human-readable; every write flushes to disk. |
| `sqlite` | Minimal | Single server wanting DB semantics | Max 1 pool connection; zero ops overhead. |
| `mysql` | External DB | Multi-server setups sharing cooldowns | Requires pre-created database and network access. |

> **Reminder:** changing `storage.type` at runtime is **not** supported. Run a full server restart after swapping backends. `/zvaults reload` will log a warning if it detects `storage.type` changed and will not switch the live backend.

---

## Bypass behavior

`zvaults.bypass` (default `op`) controls **only** the cooldown *check* — it does **not** stop ZVaults from recording the timestamp.

### What bypass does

- Skips the "is this player still on cooldown?" gate — the player is never denied.
- Still records a timestamp **after** vanilla confirms the vault dispensed loot. The write path is identical to a non-bypass open.
- The timestamp lands in cache + active storage backend (and, for YAML, the `data.yml` file is flushed immediately).

### Why that matters

Because the timestamp is always recorded, revoking `zvaults.bypass` puts the player on cooldown **from their most recent open**. There is no grace period and no "catch-up" window.

Concretely:

1. Op opens vault A at `t = 0s` with bypass → timestamp `0s` committed.
2. Op opens vault A again at `t = 15s` with bypass → timestamp refreshed to `15s`.
3. Admin revokes `zvaults.bypass` at `t = 20s`.
4. Op right-clicks vault A at `t = 21s` — denied, cooldown remaining = `normal-cooldown − 6s`.

### What bypass does **not** do

- Does **not** skip the force-unlock path — a sneaking player with a trial key always goes through the force-unlock branch (which requires `zvaults.forceunlock`, not `zvaults.bypass`).
- Does **not** suppress the debug trace — `debug.cooldown-start-log: true` will still log bypass-triggered cooldown writes.
- Does **not** imply `zvaults.use`. If `zvaults.use` is negated, the player's vault click is blocked before the bypass branch is ever reached.

---

## Force unlock

Vanilla trial vaults keep a per-vault list of players who've already been rewarded. Until an entry clears, that player cannot receive loot from the vault regardless of cooldowns. ZVaults provides an admin-only escape hatch for this:

### How to trigger it

**Sneak** (Shift) + **right-click** a `VAULT` block while holding a **trial key** (`TRIAL_KEY` or `OMINOUS_TRIAL_KEY`). The player must have the `zvaults.forceunlock` permission (default `op`).

### What it does

1. Clears the vault's list of already-rewarded players.
2. Persists the cleared list to the vault block.
3. Cancels the interaction so the click does not count as an open (the trial key is not consumed).
4. Sends the `force-unlock.success` message to the invoking player.

### What it does **not** do

- **Does not** wipe ZVaults cooldown timestamps — players who are on cooldown remain on cooldown until their individual timers expire (or an admin runs `/zvaults resetplayer` / `/zvaults resetglobal`).
- **Does not** dispense loot. It only allows the vault to dispense again on a subsequent normal right-click.
- **Does not** care whether the vault is normal or ominous — both variants share the same rewarded-player mechanic.
- **Does not** require `zvaults.use` or `zvaults.bypass`. The only permission checked in this branch is `zvaults.forceunlock`.

### When to use it

- A player lost their loot due to a crash / rollback and vanilla still considers them "rewarded".
- You want to reset a vault for a scheduled event without restarting the world.
- You need to verify vault behavior during testing.

---

## Autosave / prune

Despite the name, `autosave-interval-seconds` is **not** a write-back / flush interval. Every confirmed cooldown is already committed synchronously one tick after the interaction — there is no accumulated state waiting for this timer. The interval instead drives a **prune** sweep.

### What the task actually does

An async task runs every `autosave-interval-seconds` seconds. On each run it:

1. Computes a retention window of `max(cooldowns.normal, cooldowns.ominous) × 1000` milliseconds.
2. Drops every cooldown entry older than that window from the active storage backend.
3. Drops the same entries from the in-memory cache.
4. For the YAML backend, flushes `data.yml` — but only if the file already exists or there is non-empty state to persist.

### Interval floor

If you configure a value below `10`, the loader silently floors it to `10` seconds to avoid a busy-loop. There is no hard ceiling, but values above the combined cooldown durations offer no additional benefit.

### Shutdown behavior

On server shutdown the plugin runs one final prune and then closes the storage handle. Every confirmed cooldown was already persisted at open-time, so there is no backlog of in-memory writes waiting for shutdown to flush — the shutdown prune only trims expired entries.

### Tuning guidance

- Leave at `300` (5 min) unless you have a specific reason to change it. Expired entries are harmless in memory; the sweep exists so `data.yml` / the `cooldowns` table does not grow unbounded over long uptime.
- If you want aggressive reclamation (e.g. very short cooldowns on a lobby server) lower it to `30`–`60`.
- If your cooldowns are very long (multi-hour), you can safely raise it to `900`+ — there's no functional impact.

---

## Debug logging

ZVaults keeps runtime chatter low by default. There is a single opt-in debug switch for operators who want visibility into cooldown writes.

### `debug.cooldown-start-log`

```yaml
debug:
  cooldown-start-log: false
```

When set to `true`, every confirmed cooldown write emits one **INFO** line to the server log:

```
[ZVaults] Cooldown start confirmed: player=<name>, vaultKey=<world:x:y:z>, epochMs=<epochMs>
```

This fires:

- After vanilla dispenses loot (only confirmed opens produce a line).
- For both normal and ominous vaults.
- For bypass users as well — the write happens on the same path.

Use it to verify:

- That bypass users still record a timestamp.
- That the vault block you're testing against is actually being reached by the plugin.
- That the open → dispense → confirmation sequence is firing in the right order.

Leave it `false` on production servers. Every vault open produces one INFO line, which will quickly dominate the log on a busy server.

### Fine-grained traces

Additional `FINE`-level traces are emitted at several points in the interaction flow — every time ZVaults sees a vault right-click, when bypass is in play, on each step of the success-confirmation path, and when a cooldown ends and the reward is reset. These are filtered out by Paper's default log level. To see them, raise the plugin's log level via `logging.properties`:

```properties
com.zenologia.zvaults.level = FINE
```

This level of detail is intended for debugging, not day-to-day production use.

---

## Data files created in `plugins/ZVaults/`

The plugin's data folder is `plugins/ZVaults/`. Files listed with an **✓** are created by the plugin automatically; others only appear once specific conditions are met.

| File | Created by | When it appears |
|---|---|---|
| ✓ `config.yml` | Plugin | On first enable (copied from bundled resource if absent). |
| ✓ `messages.yml` | Plugin | On first enable (copied from bundled resource if absent). |
| `data.yml` | YAML backend | Only after the first cooldown commit. An empty-file guard prevents it from being materialized until there is real state to persist. |
| `ZVaults.db` | SQLite backend | On first enable when `storage.type: sqlite`. |

### About `data.yml`

- Human-readable; safe to inspect with any text editor while the server is offline.
- Editing while the server is running will be overwritten by the in-memory cache on the next write.
- Deleting it while the server is offline is equivalent to running `/zvaults resetglobal` on next startup.

### About `ZVaults.db`

- SQLite 3 database, single `cooldowns` table (see [Storage backends](#storage-backends) for the schema).
- Open it with `sqlite3 plugins/ZVaults/ZVaults.db` for ad-hoc inspection.
- Safe to delete while offline for a full reset.

### About MySQL

- No files are created inside `plugins/ZVaults/` — all state lives in the configured external database.
- Reset by truncating the `cooldowns` table or running `/zvaults resetglobal`.

### What ZVaults does **not** create

- No `update.yml`, no backup folder, no migration files.
- No log files beyond the normal server log.
- No subdirectories under `plugins/ZVaults/`.

---

## Edge cases & gotchas

Behavior that isn't obvious from the config or command list, grouped by where admins usually bump into it.

### Permissions

- `zvaults.use` defaults to `true` — **all players** are under cooldown logic by default. Negating it causes ZVaults to cancel the click and silently deny both the block and item interaction. The player sees no cooldown message and no deny sound; the vault simply doesn't open.
- The `zvaults.use` check is only evaluated on a **non-sneaking** click. A sneaking right-click with a trial key takes the force-unlock branch first, so a player with only `zvaults.forceunlock` (and no `zvaults.use`) can still force-unlock a vault.
- `zvaults.bypass` and `zvaults.forceunlock` are independent — granting one does **not** grant the other.
- `zvaults.admin.*` does **not** include `zvaults.use` or `zvaults.bypass`. Admins who also need to open vaults need those nodes explicitly.

### Cooldown keys

- Cooldowns are tracked per **block position**, not per vault "type" or loot table. Two vaults side-by-side have independent cooldowns.
- The vault key format is `world:x:y:z` using integer block coordinates. If you move a vault (e.g. via WorldEdit copy), the old cooldown entries at the original coordinates remain until pruned.
- Dimension is part of the key — a vault at the same x/y/z in the Nether and Overworld are distinct.

### Normal vs ominous vaults

- Both variants share the same cooldown logic; only the duration differs.
- `cooldowns.ominous` is read independently of `cooldowns.normal`. You can set either to `0` to disable that variant's cooldown (though `0` will still record timestamps — it just never denies).
- The prune sweep's retention window is `max(normal, ominous) × 1000` milliseconds.

### Trial keys and force unlock

- Only sneaking with a `TRIAL_KEY` or `OMINOUS_TRIAL_KEY` triggers the force-unlock branch. Any other held item (or an empty hand) is ignored entirely by the plugin.
- Force unlock does not consume the trial key and does not dispense loot — the player must right-click again (without sneaking) to trigger the vanilla dispense.
- If the player is sneaking but lacks `zvaults.forceunlock`, ZVaults does **not** apply cooldown enforcement to that click — the plugin bails out of its own logic and lets vanilla handle the interaction normally. There is no permission combination that makes a sneaking + trial-key click hit the cooldown gate; sneaking is always routed to the force-unlock branch, which either unlocks the vault or defers to vanilla.

### Storage & reload

- `/zvaults reload` re-reads `config.yml` and `messages.yml`, and restarts the autosave timer with the new interval. It does **not** swap storage backends — changing `storage.type` requires a full server restart.
- Shared MySQL: each server keeps its own in-memory cache hydrated once at enable-time. If you reset cooldowns via SQL on one server while others are running, their caches will still serve the stale timestamps until they restart or the entries are individually overwritten.

### Server timing

- Cooldowns are measured in wall-clock milliseconds, not tick counts. Changing the server clock (NTP jumps, VM host migration) can cause cooldowns to appear shorter/longer than configured for already-running timers.
- The cooldown write is deferred **one tick** after the interaction so vanilla can confirm the reward. If the server shuts down in that tick window the timestamp is lost — the player is not penalized.

### Vault state

- ZVaults never modifies loot tables, trial-spawner configs, or ominous-trial-key recipes.
- ZVaults **does** modify the vault's rewarded-players list in two places during a normal open: (1) just before the click is allowed through, to remove the player so vanilla will dispense to them again, and (2) when the cooldown elapses, to remove the player again so they can re-open without admin intervention. Between those two points the plugin only *reads* the list — one tick after the click it checks whether vanilla actually re-added the player (proof the dispense happened) before writing the cooldown timestamp. The force-unlock branch performs a broader version of the same clear for every player on the vault.
- Removing the vault block while a player has an active cooldown leaves an orphan entry in storage. The prune sweep cleans it up once the retention window elapses.

---

## Troubleshooting

Quick-reference for the most common "why isn't this working?" questions.

### Cooldowns aren't being enforced

1. Confirm the player does **not** have `zvaults.bypass`. Ops receive this node by default, so any op-level account will bypass the check.
2. Confirm the player has `zvaults.use` (default `true`). If they don't, the click is cancelled outright — you'll see no cooldown message, no deny sound, and the vault won't open.
3. Enable `debug.cooldown-start-log: true` and open a vault — you should see `Cooldown start confirmed: ...` in the log. If you don't, the right-click isn't reaching ZVaults at all (likely another plugin is cancelling the event before ZVaults runs).
4. Check `cooldowns.normal` / `cooldowns.ominous` are non-zero. Zero is a valid value that records timestamps but never denies.

### Cooldowns apply to op players who shouldn't be affected

This happens when the player has a stored timestamp but no longer has `zvaults.bypass`. Common causes: a permissions plugin is explicitly negating `zvaults.bypass` for their group, or they recorded a timestamp earlier while un-opped. Bypass only skips the *check* — it never deletes existing timestamps. Fix by running `/zvaults resetplayer <name>` or by restoring `zvaults.bypass` for that player.

### The "cooldown" message shows the wrong time

`messages.yml` selects one of nine branches based on the remaining time (minutes × seconds, with zero/one/many variants) and fills in the `{minutes}` and `{seconds}` tokens. If a placeholder isn't rendering, confirm the token is spelled `{minutes}` / `{seconds}` exactly — they are case-sensitive and use curly braces, not `%...%`.

### `data.yml` isn't appearing in the data folder

This is intentional. The YAML backend does not create `data.yml` until the first cooldown is committed. Have any player (bypass or not) successfully open a vault and the file will appear on the next write.

### Changing `storage.type` in `config.yml` has no effect

`storage.type` is only read once at plugin enable. `/zvaults reload` will log a warning and keep the currently-active backend. **Restart the server** to switch backends.

### `/zvaults` commands return "unknown command"

This means the plugin failed to enable. Check the startup log for a `Failed to enable ZVaults: ...` line. Common enable-time failures:

- Running on a non-Paper server (Spigot, Bukkit) — see [Requirements](#requirements).
- Java 25 is not installed or not the runtime.
- MySQL backend selected but the database cannot be reached.

### MySQL errors on startup

- **`Communications link failure`** — check host/port/firewall.
- **`Access denied for user ...`** — check username/password and `GRANT` for the target database.
- **`Unknown database 'X'`** — create the database first; ZVaults does not auto-create it.
- The MySQL driver is bundled. A missing-driver error indicates a damaged jar — re-download the release.

### Force unlock doesn't do anything

- Player must have `zvaults.forceunlock` (default `op`).
- Player must be **sneaking** when right-clicking.
- Player must be holding a `TRIAL_KEY` or `OMINOUS_TRIAL_KEY` in the **main hand**. Off-hand clicks are ignored.
- Target block must be a `VAULT` block. Trial spawners are not vaults.

### Too much log spam after enabling debug

`debug.cooldown-start-log: true` emits one INFO line per vault open. On a busy server this will dominate the log. Switch it back to `false` and use the `FINE`-level traces described in the [Debug logging](#debug-logging) section if you need deeper traces without flooding the main log.

---

## Uninstall

Removing ZVaults is a three-step operation. None of it affects vanilla vault blocks, loot tables, or players' inventories.

1. **Stop the server.** Don't hot-remove — the plugin needs a clean shutdown to close its storage handle.
2. **Delete the jar.** Remove the ZVaults plugin jar from your `plugins/` folder.
3. **Delete the data folder (optional).** Remove `plugins/ZVaults/` if you also want to discard cooldown state. If you plan to reinstall later, leaving the folder in place preserves all configuration and cooldown history.

### If you used MySQL

`plugins/ZVaults/` contains only `config.yml` and `messages.yml` — cooldown state lives in the external database. To fully remove state:

```sql
DROP TABLE cooldowns;
```

(Or drop the whole database if ZVaults owned it.)

### Leftover cooldown state

After uninstall there is no residual effect on the Minecraft world:

- Vault blocks keep whatever rewarded-players list vanilla already tracks.
- Trial spawners are untouched.
- No scheduled tasks remain once the server is stopped — the autosave timer is cancelled on shutdown.

### Reinstall checklist

If you later reinstall ZVaults onto a server that still has the data folder:

- `config.yml` + `messages.yml` are preserved as-is.
- `data.yml` / `ZVaults.db` are re-loaded into the cache on enable.
- MySQL: the existing `cooldowns` table is reused (the `CREATE TABLE IF NOT EXISTS` is idempotent).

---

## 🧑‍💻 Author

**ZVaults** is developed and maintained by **Zenologia**.

- GitHub: [@Zenologia](https://github.com/Zenologia)

If ZVaults saves you time on your server, a ⭐ on the repository is the best way to say thanks.
