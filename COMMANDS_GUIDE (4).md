# PvP Core — The Complete Commands & Configuration Guide

**Mod Version:** 2.2.0 · **Minecraft:** 1.21.11 · **Loader:** Fabric 0.19.3+ · **Java:** 21+
**Type:** 100% server-side — vanilla clients can join without installing anything.

---

## Table of Contents

1. [Introduction & Design Philosophy](#1-introduction--design-philosophy)
2. [Requirements & Installation](#2-requirements--installation)
3. [The Permission System (1.21.11)](#3-the-permission-system-minecraft-12111)
4. [The Lobby Hotbar — GUI-First Design](#4-the-lobby-hotbar--gui-first-design)
5. [Command Quick Reference (Master Table)](#5-command-quick-reference-master-table)
6. [Player Commands — Full Documentation](#6-player-commands--full-documentation)
   - [`/duel`](#duel) · [`/queue`](#queue) · [`/ranked`](#ranked) · [`/leavequeue`](#leavequeue) · [`/leave`](#leave) · [`/duelaccept` / `/dueldeny`](#duelaccept--dueldeny-new-in-210)
   - [`/profile`](#profile) · [`/stats`](#stats) · [`/leaderboard`](#leaderboard)
   - [`/spectate`](#spectate) · [`/rematch`](#rematch) · [`/lobby`](#lobby-new-in-220) · [`/party`](#party)
7. [Admin Commands — Full Documentation](#7-admin-commands--full-documentation)
   - [`/arena`](#arena) · [`/pvpadmin`](#pvpadmin) · [`/pvpdebug`](#pvpdebug)
8. [The GUI System — Complete Walkthrough](#8-the-gui-system--complete-walkthrough)
9. [The Kit Editor — Deep Dive](#9-the-kit-editor--deep-dive)
10. [The Party System](#10-the-party-system)
11. [The Chat System (Duel Isolation)](#11-the-chat-system-duel-isolation)
12. [The Per-Duel Scoreboard](#12-the-per-duel-scoreboard)
13. [Configuration Reference — settings.json](#13-configuration-reference--settingsjson)
14. [Configuration Reference — guis.json](#14-configuration-reference--guisjson)
15. [Configuration Reference — messages.json](#15-configuration-reference--messagesjson)
16. [ELO & Matchmaking Reference](#16-elo--matchmaking-reference)
17. [Data Storage & File Layout](#17-data-storage--file-layout)
18. [Audit Log](#18-audit-log)
19. [Color Codes & Formatting](#19-color-codes--formatting)
20. [Integration & Compatibility Notes](#20-integration--compatibility-notes)
21. [Troubleshooting & FAQ](#21-troubleshooting--faq)
22. [What Changed in 2.2.0 (10 Fixes)](#22-what-changed-in-220)
23. [History: What Changed in 2.1.0](#23-history-what-changed-in-210)
24. [History: What Changed in 2.0.0](#24-history-what-changed-in-200)

---

## 1. Introduction & Design Philosophy

PvP Core is a complete practice-PvP plugin (mod) for Fabric servers: ranked and unranked matchmaking with ELO, direct duels, kits with per-player editable layouts, parties, spectating, leaderboards, statistics, and a full admin toolkit — all of it **100% server-side**.

The single most important design rule of this mod:

> **Everything is reachable through GUIs. Commands are an optional convenience for power users and admins — never a requirement.**

That means:

- A brand-new player can join with a **pure vanilla client**, click the hotbar items, browse kits, edit a kit layout in a double-chest editor, queue for ranked matches, create and manage a party, duel friends, spectate matches, and check the leaderboards **without ever typing a single command**.
- Kit editing happens inside a **54-slot double-chest GUI** with a full item palette, an enchanting menu, and a save button (plus automatic saving when the menu closes). No commands, no dropping items on the ground, no inventory gymnastics.
- Text input (arena names, renames) is done through an **anvil text-field GUI**, not chat prompts.
- Admins get a full **Admin GUI** (reload, save, arenas, kits, set-lobby) and an **Arena Editor GUI** — although the command equivalents remain available because server operators often script them.

The second most important rule:

> **The mod never fights other mods.** It does not touch game modes (your lobby plugin keeps full control of that), it waits for third-party login/auth GUIs to finish before applying lobby state, and it never locks inventories while another screen is open.

---

## 2. Requirements & Installation

| Component | Requirement |
|---|---|
| Minecraft Server | 1.21.11 (vanilla-compatible Fabric server) |
| Fabric Loader | 0.19.3 or newer |
| Fabric API | 0.141.x+ for 1.21.11 (any recent build) |
| Java | 21 or newer |
| Clients | **Vanilla works.** No client mod needed. |

**Installation steps:**

1. Drop `pvpcore-2.2.0.jar` into your server's `mods/` folder alongside Fabric API.
2. Start the server once. The mod creates `config/pvp/` with three config files:
   - `settings.json` — all gameplay & behavior settings
   - `guis.json` — every GUI, every item, every slot
   - `messages.json` — every message the mod sends
3. Stop the server (or use `/pvpadmin reload` later) after editing configs.
4. Create your first arenas: `/arena create <id>` then stand where spawn 1 should be and run `/arena setspawn1 <id>` (or do it all in the Arena Editor GUI — see §8.9).
5. Players are ready to play. Default kits (`nodebuff`, `classic`, `gapple`) are generated automatically.

**Server sizing note:** the mod was designed for small servers (~2 GB RAM, 1 vCPU). All data is stored in flat JSON files with atomic writes — no external database is needed or used.

---

## 3. The Permission System (Minecraft 1.21.11)

Minecraft 1.21.11 replaced the old numeric `hasPermissionLevel(n)` checks with the named `PermissionLevel` system. PvP Core uses these levels:

| Level | Name | Who has it by default | PvP Core usage |
|---|---|---|---|
| 0 | `ALL` | Every player | All player commands & GUIs |
| 1 | `MODERATORS` | Moderators | Not used by PvP Core |
| 2 | `GAMEMASTERS` | Operators (`/op`) | All admin commands & GUIs (`/arena`, `/pvpadmin`, `/pvpdebug`) |
| 3 | `ADMINS` | — | Same as level 2 for PvP Core (level 2+ passes) |
| 4 | `OWNERS` | Server owner | Same as level 2 for PvP Core |

Practical summary:

- **Players** need nothing. Every gameplay feature is available to everyone.
- **Admins** need `/op` (or a permission mod granting `GAMEMASTERS`), which unlocks the three admin command trees and the admin GUIs.
- Admins also **bypass the lobby inventory lock** so they can manage their lobby freely (see §13.2 `lobby.lockInventory`).

---

## 4. The Lobby Hotbar — GUI-First Design

When a player finishes logging in (see `lobby.loginSyncMode` in §13.2 — the mod waits for third-party login GUIs), they receive the lobby hotbar. **Every slot, material, name, and lore is configurable in `guis.json`** (§14.1). These are the defaults:

| Slot | Item | Name | Action |
|---|---|---|---|
| 0 | Iron Sword | **Unranked Queue** | Opens the kit picker for the unranked queue |
| 1 | Netherite Sword | **Ranked Queue** | Opens the kit picker for the ranked (ELO) queue |
| 2 | Blaze Rod | **Duel a Player** | Opens the opponent picker, then the kit picker |
| 3 | Book | **Kits** | Opens the kit list (view/edit layouts) |
| 4 | Name Tag | **Party** | **Auto-creates a party** (if none) and opens the Party GUI |
| 5 | *(empty — leaderboard item removed in 2.1.0; re-enable it in guis.json if you want it back)* | — | — |
| 6 | Player Head (your skin) | **Profile** | Opens your profile (stats & history) |
| 7 | Ender Eye | **Spectate** | Opens the ongoing-duels browser |
| 8 | Gray Glass Pane | *(filler)* | — |

**State-aware switching** (also fully configurable):

| Situation | Change |
|---|---|
| Player joined the **unranked queue** | The hotbar shows **only the Redstone “Leave Queue” item** — every other item is hidden until they leave the queue |
| Player joined the **ranked queue** | Same — only the **Leave Queue** item is shown |
| Player is **in a party** | Slot 4 becomes the **“Party (Open Menu)”** variant item |
| Player is **spectating** | Slot 7 becomes an **Ender Pearl “Return to Lobby”** item |

The profile head in slot 6 is a **real player head carrying the viewer's own GameProfile** — it renders the player's *current actual skin* and works correctly with SkinsRestorer-style mods (§20).

The hotbar is **locked** while the player is in the lobby (items can't be moved, dropped, or swapped) — configurable via `lobby.lockInventory`. See §22 (fix #1) for details.

---

## 5. Command Quick Reference (Master Table)

Legend: `<>` required · `[]` optional · **(A)** = admin (GAMEMASTERS level) · **(G)** = has a GUI equivalent

| Command | Arguments | What it does |
|---|---|---|
| `/duel` **(G)** | `<player> [kit]` | Challenge a player (kit picker opens if no kit given) |
| `/queue` **(G)** | `[kit] [ranked]` | Join a queue (kit picker opens if no kit given) |
| `/ranked` **(G)** | `[kit]` | Join the ranked queue (kit picker opens if no kit given) |
| `/leavequeue` **(G)** | — | Leave any queue you are in |
| `/leave` **(G)** | — | Leave your duel (forfeit), queue, or spectate session — one command for all |
| `/duelaccept` | — | Accept a pending duel request (target of the chat button) |
| `/dueldeny` | — | Deny a pending duel request (target of the chat button) |
| `/profile` **(G)** | — | Open your own profile GUI |
| `/stats` **(G)** | `[player]` | View statistics (player picker opens if no target given) |
| `/leaderboard` **(G)** | `[type] [kit]` | Print or open leaderboards (`type` = `wins` \| `elo`) |
| `/spectate` **(G)** | `[player]` | Spectate a player's ongoing duel (browser opens if no target) |
| `/rematch` **(G)** | — | Accept a pending rematch offer |
| `/lobby` **(G)** | — | Teleport yourself to the lobby point *(new in 2.2.0)* |
| `/party` **(G)** | — | Open the party menu in chat (auto-creates a party if configured) |
| `/party create` | — | Create a party |
| `/party invite` | `<player>` | Invite a player |
| `/party accept` | — | Accept a pending invite |
| `/party deny` | — | Deny a pending invite |
| `/party leave` | — | Leave your party |
| `/party kick` | `<player>` | (Leader) Kick a member |
| `/party promote` | `<player>` | (Leader) Transfer leadership |
| `/party disband` | — | (Leader) Disband the party |
| `/party list` | — | Print the member list |
| `/party chat` | — | Toggle party-only chat mode |
| `/party menu` | — | Reopen the clickable party chat menu *(new in 2.2.0)* |
| `/party menu invite` | — | List clickable players you can invite *(new in 2.2.0)* |
| `/party menu duel` | — | List clickable party leaders to duel *(new in 2.2.0)* |
| `/party menu manage` | — | List members with [PROMOTE]/[KICK] buttons (leader) *(new in 2.2.0)* |
| `/party duel` | `<player>` | (Leader) Challenge another party — kit picker → rounds picker *(new in 2.2.0)* |
| `/arena` **(A)(G)** | — | Open the Arena management GUI |
| `/arena list` **(A)** | — | Print all arenas |
| `/arena create` **(A)** | `<id>` | Create an arena (in your current world) |
| `/arena setspawn1` **(A)** | `<id>` | Set spawn 1 to your position |
| `/arena setspawn2` **(A)** | `<id>` | Set spawn 2 to your position |
| `/arena setspectator` **(A)** | `<id>` | Set spectator spawn to your position |
| `/arena <id> enable` **(A)** | — | Enable the arena |
| `/arena <id> disable` **(A)** | — | Disable the arena |
| `/arena <id> validate` **(A)** | — | Check the arena is ready for duels |
| `/arena <id> teleport` **(A)** | — | Teleport to the arena |
| `/arena <id> delete` **(A)** | — | Delete the arena (must be unused) |
| `/pvpadmin` **(A)(G)** | — | Open the Admin GUI |
| `/pvpadmin reload` **(A)** | — | Reload all three config files |
| `/pvpadmin save` **(A)** | — | Force-save all data to disk |
| `/pvpadmin setlobby` **(A)** | — | Set the lobby point to your position |
| `/pvpadmin audit` **(A)** | — | Show the last 15 audit-log entries |
| `/pvpadmin kit create` **(A)** | `<id> [name]` | Create a new kit **from your current inventory** (armor, hotbar and inventory are copied exactly) |
| `/pvpadmin kit delete` **(A)** | `<id>` | Delete a kit |
| `/pvpadmin kit list` **(A)** | — | Print all kit ids |
| `/pvpdebug` **(A)** | — | Print a live summary (duels/queues/parties/arenas) |
| `/pvpdebug state` **(A)** | `<player>` | Full session state dump for a player |
| `/pvpdebug duels` **(A)** | — | List all active duels |
| `/pvpdebug arenas` **(A)** | — | List arenas with status |
| `/pvpdebug queues` **(A)** | — | Total players queued |

---
## 6. Player Commands — Full Documentation

Every command below follows the same 12-field documentation format: Syntax · Permission · Description · Arguments · Examples · GUI Equivalent · Aliases · Success Output · Error Cases · Related Config · Related Commands · Notes & Tips.

---

### `/duel`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/duel <player> [kit]` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | Sends a direct duel challenge to another online player. If you omit the kit, the kit picker GUI opens pre-bound to that opponent — pick a kit and the challenge is sent. If you supply a kit id, the challenge is sent immediately. The target receives a title notification plus an accept/deny GUI. |
| 4 | **Arguments** | `<player>` — an online player's name (tab-completes). `[kit]` — a kit id (tab-completes), e.g. `nodebuff`, `classic`, `gapple`. |
| 5 | **Examples** | `/duel Steve` → opens the kit picker bound to Steve. `/duel Steve nodebuff` → instantly sends a NoDebuff challenge to Steve. |
| 6 | **GUI Equivalent** | Lobby slot 2 (**Blaze Rod — Duel a Player**) → opponent picker → kit picker. Identical flow. |
| 7 | **Aliases** | None |
| 8 | **Success Output** | You: “Challenge sent to Steve. Kit: nodebuff”. Target: big title **DUEL REQUEST**, subtitle “Steve challenged you!” — wait, the *challenger's* name — plus a 3-row accept/deny GUI (§8.6). |
| 9 | **Error Cases** | Target offline/not found (Brigadier error); target busy (already dueling/queued → `duel.challenge-busy`); challenge expires after `duel.rematchExpirySeconds`-style timeout (both sides are notified); target denies (sender is notified). |
| 10 | **Related Config** | `duel.*` (countdown, timeout, heals — §13.3); messages `duel.challenge-sent`, `duel.challenge-received-title/subtitle`, `duel.challenge-expired`, `duel.challenge-denied`, `duel.challenge-busy`. |
| 11 | **Related Commands** | `/rematch` (post-duel), `/queue` (matchmaking instead), `/spectate`. |
| 12 | **Notes & Tips** | Challenges are one-shot: they expire automatically, and a player can only have one incoming challenge screen at a time. Party leaders should use the Party GUI's **Party Duel** button for team fights instead. |

---

### `/queue`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/queue [kit] [ranked]` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | Join the matchmaking queue. Without arguments, the kit picker GUI opens in unranked mode. With a kit id you join that kit's queue directly. The second optional argument is a boolean: `true` = ranked (ELO at stake), `false`/omitted = unranked. While queued, your hotbar's queue item becomes **Leave Queue**. The matchmaker pairs players by ELO proximity that widens with waiting time (§16). |
| 4 | **Arguments** | `[kit]` — kit id (tab-completes). `[ranked]` — `true`/`false` (tab-completes). |
| 5 | **Examples** | `/queue` → unranked kit picker GUI. `/queue nodebuff` → join unranked NoDebuff queue. `/queue gapple true` → join **ranked** Gapple queue. |
| 6 | **GUI Equivalent** | Lobby slot 0 (**Unranked Queue**) or slot 1 (**Ranked Queue**) → kit picker → click a kit to queue. Click the **Leave Queue** redstone item to leave. |
| 7 | **Aliases** | `/ranked` is a dedicated shortcut for ranked mode. |
| 8 | **Success Output** | “You joined the unranked queue for kit nodebuff.” — plus periodic status messages with your position and current ELO search range. On match: “Match found! Kit: … \| Arena: …”. |
| 9 | **Error Cases** | Already in a queue → `queue.already` (leave first). Unknown kit → `kit.not-found`. Offline/invalid state → command fails silently (0). |
| 10 | **Related Config** | `queue.matchmakingIntervalTicks`, `queue.statusIntervalTicks` (§13.6); `elo.*` (§13.5). |
| 11 | **Related Commands** | `/ranked`, `/leavequeue`, `/duel`. |
| 12 | **Notes & Tips** | Ranked and unranked queues are separate — you can only be in one queue at a time. Disconnecting automatically removes you from any queue. |

---

### `/ranked`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/ranked [kit]` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | Identical to `/queue <kit> true` — joins the **ranked** queue where ELO is on the line. Without a kit, the kit picker opens in ranked mode (every kit item shows your current ELO with that kit). |
| 4 | **Arguments** | `[kit]` — kit id (tab-completes). |
| 5 | **Examples** | `/ranked` → ranked kit picker GUI. `/ranked classic` → join ranked Classic queue. |
| 6 | **GUI Equivalent** | Lobby slot 1 (**Netherite Sword — Ranked Queue**). |
| 7 | **Aliases** | `/queue <kit> true` |
| 8 | **Success Output** | “You joined the ranked queue for kit classic.” — with periodic position/range status. |
| 9 | **Error Cases** | Same as `/queue`. |
| 10 | **Related Config** | Same as `/queue`; ELO change messages `duel.elo-change`. |
| 11 | **Related Commands** | `/queue`, `/leavequeue`. |
| 12 | **Notes & Tips** | Winning a ranked duel raises your ELO for that specific kit; losing lowers it. See §16 for the K-factor and search-range expansion tables. |

---

### `/leavequeue`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/leavequeue` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | Leaves whatever queue (ranked or unranked) you are currently in. |
| 4 | **Arguments** | None |
| 5 | **Examples** | `/leavequeue` |
| 6 | **GUI Equivalent** | Click the **Redstone — Leave Queue** hotbar item (the only hotbar item while queued). |
| 7 | **Aliases** | None |
| 8 | **Success Output** | “You left the queue.” |
| 9 | **Error Cases** | Not in a queue → “You are not in a queue.” |
| 10 | **Related Config** | Message `queue.left`, `queue.not-queued`. |
| 11 | **Related Commands** | `/queue`, `/ranked`, `/leave`. |
| 12 | **Notes & Tips** | Leaving is instant and penalty-free. The hotbar reverts to the normal lobby items automatically. |

---

---

### `/leave` *(new in 2.1.0)*

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/leave` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | The universal “get me out of here” command. In priority order: forfeits your current duel (the opponent wins), leaves your queue, or returns you to the lobby from spectate mode. |
| 4 | **Arguments** | None |
| 5 | **Examples** | `/leave` mid-duel → you forfeit and return to the lobby. `/leave` while queued → you leave the queue. |
| 6 | **GUI Equivalent** | The Leave Queue hotbar item covers queues; there is no GUI forfeit button — this command is the duel exit. |
| 7 | **Aliases** | None |
| 8 | **Success Output** | Duel: “You left the duel. Your opponent wins.” · Queue: “You left the queue.” · Spectate: returns you to the lobby. |
| 9 | **Error Cases** | Not in a duel, queue or spectate → “You are not in a duel or a queue.” |
| 10 | **Related Config** | Messages `leave.duel-left`, `leave.queue-left`, `leave.nothing`. |
| 11 | **Related Commands** | `/leavequeue`, `/spectate`, `/duelaccept`. |
| 12 | **Notes & Tips** | Forfeiting works during the countdown too. Stats and ELO are applied exactly like a normal loss — a ranked forfeit costs ELO. |

---

### `/duelaccept` & `/dueldeny` *(new in 2.1.0)*

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/duelaccept` · `/dueldeny` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | Accept or deny the duel request you have received. These are the commands behind the **clickable [ACCEPT] / [DENY] chat buttons** — you normally click the button in chat instead of typing. |
| 4 | **Arguments** | None |
| 5 | **Examples** | Click **[ACCEPT]** in chat (runs `/duelaccept`) · Type `/dueldeny` manually. |
| 6 | **GUI Equivalent** | The chat buttons themselves — the old accept/deny chest GUI is gone by default (see `duel.challengeViaChat`). |
| 7 | **Aliases** | None |
| 8 | **Success Output** | Accept: the duel starts immediately. Deny: the challenger is notified that you denied. |
| 9 | **Error Cases** | No pending request → “You have no pending duel request.” |
| 10 | **Related Config** | Messages `duel.challenge-chat-header`, `duel.challenge-accept-button`, `duel.challenge-deny-button`, `duel.challenge-chat-expire`; `settings.json → duel.challengeViaChat`. |
| 11 | **Related Commands** | `/duel`, `/leave`. |
| 12 | **Notes & Tips** | Requests expire after 30s (45s for party duels). Both buttons show hover tooltips describing what they do. |

---

### `/profile`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/profile` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | Opens your own profile GUI: your real-skin head, global stats (wins/losses/ELO per kit) and kit statistics. |
| 4 | **Arguments** | None |
| 5 | **Examples** | `/profile` |
| 6 | **GUI Equivalent** | Lobby slot 6 (**Player Head — Profile**). |
| 7 | **Aliases** | None |
| 8 | **Success Output** | The Profile GUI opens (5 rows): head at slot 13, Global Stats at 11, Kit Stats at 15, Close at 31. |
| 9 | **Error Cases** | Used from console → “This can only be used by players.” |
| 10 | **Related Config** | `guis.json → guis.profile.*` (§14.2); the head renders your current skin, SkinsRestorer-compatible. |
| 11 | **Related Commands** | `/stats` (view others), `/leaderboard`. |
| 12 | **Notes & Tips** | The head item is built from your live server-side GameProfile, so it always matches the skin you are actually wearing. |

---

### `/stats`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/stats [player]` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | View the duel statistics of any online player. Without an argument, a player-picker GUI opens; click a head to view that player's stats GUI. |
| 4 | **Arguments** | `[player]` — an online player's name (tab-completes). |
| 5 | **Examples** | `/stats` → player picker GUI. `/stats Alex` → opens Alex's stats immediately. |
| 6 | **GUI Equivalent** | Lobby slot 6 (Profile) → **Kit Stats**; or Player Picker via `/stats` with no argument. |
| 7 | **Aliases** | None |
| 8 | **Success Output** | A 6-row Statistics GUI with per-kit W/L and ELO lines (`stats.header` / `stats.line`). |
| 9 | **Error Cases** | Player not found → Brigadier error (`player-not-found`). Console use without argument → player-only error. |
| 10 | **Related Config** | Messages `stats.header`, `stats.line`; `guis.json → guis.stats.*`. |
| 11 | **Related Commands** | `/profile`, `/leaderboard`. |
| 12 | **Notes & Tips** | Stats GUI shows one line per kit — wins, losses and ELO — plus the global aggregate. |

---

### `/leaderboard`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/leaderboard [type] [kit]` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | Two modes: with no arguments it opens the interactive leaderboard GUI (paged, switchable between Wins and ELO); with arguments it prints the top 10 directly into chat. |
| 4 | **Arguments** | `[type]` — `wins` or `elo` (tab-completes). `[kit]` — a kit id or `global` for all kits combined. |
| 5 | **Examples** | `/leaderboard` → GUI. `/leaderboard elo nodebuff` → chat: top 10 by ELO in NoDebuff. `/leaderboard wins global` → top 10 by wins overall. |
| 6 | **GUI Equivalent** | Lobby slot 5 (**Emerald — Leaderboards**). In the GUI: Nether Star = Global, click the Emerald (slot 49) to switch Wins ↔ ELO, arrows to page. |
| 7 | **Aliases** | None |
| 8 | **Success Output** | Chat mode prints a header `-------- Top ELO [nodebuff] --------` followed by `#1 Player - 1420` style lines. Empty board → “No entries yet. Be the first!” |
| 9 | **Error Cases** | Invalid type → treated as unknown kit/type yields empty list; unknown kit → empty entry list. |
| 10 | **Related Config** | Messages `leaderboard.header`, `leaderboard.entry`, `leaderboard.empty`; `guis.json → guis.leaderboard.*`. Leaderboards refresh every autosave cycle (`storage.saveIntervalSeconds`). |
| 11 | **Related Commands** | `/stats`, `/profile`. |
| 12 | **Notes & Tips** | The GUI shows up to 10 entries per page with page arrows; the chat version always prints exactly the top 10. |

---

### `/spectate`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/spectate [player]` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | Spectate an ongoing duel. Without an argument, a browser GUI lists all live duels (both fighters + kit); click one to join. With a player name, you jump straight into that player's current duel. While spectating you are invisible to fighters (configurable) and your hotbar becomes a single **Return to Lobby** item. |
| 4 | **Arguments** | `[player]` — an online player who is currently dueling. |
| 5 | **Examples** | `/spectate` → duel browser GUI. `/spectate Alex` → spectate Alex's duel directly. |
| 6 | **GUI Equivalent** | Lobby slot 7 (**Ender Eye — Spectate**). |
| 7 | **Aliases** | None |
| 8 | **Success Output** | “You are now spectating Alex vs Sam.” + teleport to the arena's spectator spawn. |
| 9 | **Error Cases** | Target not dueling / no duels at all → “There are no ongoing duels right now.” |
| 10 | **Related Config** | `spectate.enabled`, `spectate.hideFromFighters` (§13.9); `chat.isolateDuels` + `spectate.*` chat settings (§11). |
| 11 | **Related Commands** | — |
| 12 | **Notes & Tips** | Spectators' chat goes to the duel audience only (if `chat.isolateDuels` is on). **No game mode is ever changed** — the mod never touches game modes (§20); it uses its own spectator handling, so it will not conflict with your lobby plugin. |

---

### `/rematch`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/rematch` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | Accepts a pending rematch offer. After every duel ends, both players get a rematch GUI; if the other player already offered you a rematch, this command (or the GUI's accept button) starts the next duel instantly with the same kit and arena. |
| 4 | **Arguments** | None |
| 5 | **Examples** | `/rematch` |
| 6 | **GUI Equivalent** | The **Rematch?** GUI (3 rows) that appears after a duel: LIME_DYE “Rematch” at 11, RED_DYE “Decline” at 15. |
| 7 | **Aliases** | None |
| 8 | **Success Output** | “Rematch accepted!” → new countdown starts. |
| 9 | **Error Cases** | No pending offer → “You are not in a queue.” (generic no-action message); offer expired (`duel.rematchExpirySeconds`, default 20s) → “Rematch declined.” |
| 10 | **Related Config** | `duel.rematchExpirySeconds` (§13.3); messages `duel.rematch-sent/accepted/declined`. |
| 11 | **Related Commands** | `/duel`. |
| 12 | **Notes & Tips** | Rematch offers are mutual — whoever accepts first triggers the duel. The rematch reuses the same kit; a fresh arena is picked. |

---
### `/lobby` *(new in 2.2.0)*

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/lobby` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | Teleports you to the configured **lobby point** — the same location admins set with `/pvpadmin setlobby` and where players return after duels (`lobby.teleportAfterDuel`). |
| 4 | **Arguments** | None |
| 5 | **Examples** | `/lobby` → “Teleported to the lobby.” |
| 6 | **GUI Equivalent** | None (the lobby point is also where duel finishers go when `lobby.teleportAfterDuel` is true) |
| 7 | **Aliases** | None |
| 8 | **Success Output** | `lobby.teleported` — “Teleported to the lobby.” |
| 9 | **Error Cases** | `not-in-lobby` when used while inside a duel (finish or leave it first) |
| 10 | **Related Config** | `lobby.lobbyWorld` / `lobbyX/Y/Z` / `lobbyYaw/Pitch` (§13.2), `lobby.teleportOnJoin`, `lobby.teleportAfterDuel` |
| 11 | **Related Commands** | `/pvpadmin setlobby` (admin, sets the point), `/leave` |
| 12 | **Notes & Tips** | Set the point once with `/pvpadmin setlobby` — everyone can then warp back with `/lobby`. The lobby point defaults to overworld 0, 64, 0 until an admin changes it. |

---
### `/party`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/party [subcommand] [arguments]` |
| 2 | **Permission** | Everyone (`ALL`) |
| 3 | **Description** | The party system hub. Bare `/party` opens the **party menu in chat** — and **automatically creates a party for you if you're not in one** (configurable via `party.autoCreateOnItemClick`, default true). All management happens through clickable chat buttons; the subcommands are the text alternative. *(Changed in 2.2.0: parties no longer use a chest GUI.)* |
| 4 | **Arguments** | See the subcommand table below. |
| 5 | **Examples** | `/party` → chat menu (auto-creates). `/party invite Alex`. `/party menu duel` → clickable list of party leaders. `/party duel Steve` → kit picker → rounds picker → party challenge. |
| 6 | **GUI Equivalent** | Lobby slot 4 (**Name Tag — Party**). The chat menu contains: [INVITE] (clickable player names), [CHAT: ON/OFF], [DUEL] (clickable party leaders), [MANAGE] ([PROMOTE]/[KICK] per member, leader only), [LEAVE], [DISBAND] (leader), [CLOSE]. Invites arrive as [ACCEPT] / [DENY] chat buttons. |
| 7 | **Aliases** | None |
| 8 | **Success Output** | Per subcommand — see table below. |
| 9 | **Error Cases** | `not-in-party`, `already-in-party`, `party.full` (max `party.maxSize`), `not-leader` for leader-only actions, `party.target-in-duel` / `party.you-in-duel` / `party.cant-while-dueling` (no party actions while dueling — 2.2.0), invites expire after `party.inviteExpirySeconds` (default 60s). |
| 10 | **Related Config** | `party.maxSize`, `party.inviteExpirySeconds`, `party.membersCanInvite`, `party.autoCreateOnItemClick` (§13.7); chat format `chat.partyFormat` (§13.8). |
| 11 | **Related Commands** | `/duel` (1v1), `/party duel <leader>` (party vs party). |
| 12 | **Notes & Tips** | Creating a party instantly swaps your lobby slot 4 item to the **“Party (Open Menu)”** variant. If `party.membersCanInvite` is false, only the leader may invite. Players inside a duel can neither invite, be invited, nor accept invites (2.2.0). |

**`/party` subcommands:**

| Subcommand | Syntax | Leader only? | Effect & feedback |
|---|---|---|---|
| *(none)* | `/party` | — | Auto-creates a party (if configured) and prints the **chat menu** |
| `create` | `/party create` | — | “You created a party! Invite players with the party menu.” |
| `invite` | `/party invite <player>` | No (if `membersCanInvite`) | Sends a **chat invite with [ACCEPT]/[DENY] buttons** (title **PARTY INVITE**) |
| `accept` | `/party accept` | — | Join the party whose invite you accepted — “You joined Alex's party!”; members see “Alex joined the party!” |
| `deny` | `/party deny` | — | Decline — inviter sees “Alex declined the party invite.” |
| `leave` | `/party leave` | — | Leave your party — members are notified |
| `kick` | `/party kick <player>` | ✔ | Remove a member — “Steve was kicked from the party.” |
| `promote` | `/party promote <player>` | ✔ | Transfer leadership — “Steve is now the party leader.” |
| `disband` | `/party disband` | ✔ | Dissolve the party — everyone is notified |
| `list` | `/party list` | — | Prints `Party (3): Leader, Member2, Member3` (leader in gold) |
| `chat` | `/party chat` | — | Toggles party-only chat mode — “Party chat: ON/OFF” |
| `menu` | `/party menu` | — | Reprints the clickable party chat menu *(2.2.0)* |
| `menu invite` | `/party menu invite` | — | Lists clickable online players that can be invited *(2.2.0)* |
| `menu duel` | `/party menu duel` | — | Lists clickable party leaders you can challenge *(2.2.0)* |
| `menu manage` | `/party menu manage` | ✔ | Lists members with **[PROMOTE] / [KICK]** buttons *(2.2.0)* |
| `menu close` | `/party menu close` | — | Closes the menu (reopens any time via `/party`) *(2.2.0)* |
| `duel` | `/party duel <leader>` | ✔ | Challenge another party: kit picker → **rounds picker** → party-vs-party duel *(2.2.0)* |

---

## 7. Admin Commands — Full Documentation

All admin commands require **GAMEMASTERS (op level 2+)**. Every admin feature also has a GUI equivalent — the command versions exist for scripting and quick terminal work.

---

### `/arena`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/arena [subcommand]` or `/arena <id> <action>` |
| 2 | **Permission** | GAMEMASTERS (op level 2+) |
| 3 | **Description** | Full arena lifecycle management: create, position spawns, enable/disable, validate, teleport, delete. Bare `/arena` opens the Arena List GUI. |
| 4 | **Arguments** | `<id>` — arena id (word, tab-completes). Subcommands: `list`, `create <id>`, `setspawn1 <id>`, `setspawn2 <id>`, `setspectator <id>`, then per-arena: `enable`, `disable`, `validate`, `teleport`, `delete`. |
| 5 | **Examples** | `/arena create arena1` · `/arena setspawn1 arena1` (stand where spawn 1 belongs) · `/arena arena1 validate` · `/arena arena1 disable` |
| 6 | **GUI Equivalent** | `/arena` or the Admin GUI → **Arenas** (Grass Block). The Arena List GUI (6 rows) lists arenas as items; left-click opens the **Arena Editor GUI** (5 rows): Set Spawn 1 (RED_BED, 19), Set Spawn 2 (BLUE_BED, 20), Set Spectator Spawn (ENDER_EYE, 21), Teleport (ENDER_PEARL, 23), Toggle Enabled (LIME_DYE, 24), Validate (EMERALD, 25), Rename via anvil (ANVIL, 31), Delete (TNT, 40), Back (36). |
| 7 | **Aliases** | None |
| 8 | **Success Output** | `arena.created` “Arena arena1 created! Set its spawns in the arena editor.” · `arena.spawn-set` “arena1: Spawn 1 set to your current location.” · `arena.valid` “Arena arena1 is valid and ready for duels!” |
| 9 | **Error Cases** | Duplicate id → “Arena already exists”. Unknown id → “Unknown arena”. Delete while in use → “That arena is currently in use.” Validate failure → “Arena arena1 is not ready: {reason}” (e.g. missing spawns). |
| 10 | **Related Config** | Arenas persist to `data/pvp/arenas.json` (§17). All arena GUI items configurable in `guis.json → guis.arenas / guis.arena_editor`. |
| 11 | **Related Commands** | `/pvpadmin`, `/pvpdebug arenas`. |
| 12 | **Notes & Tips** | Spawn positions capture **position + look direction**, so face where the fighter should face before setting. An arena is auto-reserved while in use (status FREE/IN_USE) — the matchmaker only picks FREE, enabled, valid arenas. `create` binds the arena to **your current world**; all three spawns must be in the same world. |

---

### `/pvpadmin`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/pvpadmin [subcommand]` |
| 2 | **Permission** | GAMEMASTERS (op level 2+) |
| 3 | **Description** | The admin control panel. Bare `/pvpadmin` opens the **Admin GUI** (6 rows): Reload Configs (REPEATER, 20), Save Data (HOPPER, 21), Arenas (GRASS_BLOCK, 22), Kits (BOOK, 23), Set Lobby (BEACON, 24), Close (49). |
| 4 | **Arguments** | Subcommands: `reload`, `save`, `setlobby`, `audit`, `kit create <id>`, `kit delete <id>`, `kit list`. |
| 5 | **Examples** | `/pvpadmin reload` · `/pvpadmin setlobby` (stand at the lobby spot) · `/pvpadmin kit create sumo` |
| 6 | **GUI Equivalent** | The Admin GUI above; **Set Lobby** uses your current position exactly like the command. |
| 7 | **Aliases** | None |
| 8 | **Success Output** | `reload` → “Reloaded all configs in 4ms.” · `save` → “All data saved.” · `setlobby` → “Lobby spawn set to your current location.” · `kit create` → “Kit sumo created.” |
| 9 | **Error Cases** | Non-op → command not visible (permission gate). `kit create` with existing id → “Kit already exists”. |
| 10 | **Related Config** | `reload` re-reads `settings.json`, `guis.json`, `messages.json` and re-applies lobby items to everyone. `setlobby` writes `lobby.lobbyWorld/lobbyX/Y/Z/lobbyYaw/lobbyPitch`. |
| 11 | **Related Commands** | `/arena`, `/pvpdebug`. |
| 12 | **Notes & Tips** | `reload` is hot — configs apply instantly without restart, but active duels are not interrupted. `save` flushes player data, kits, arenas and the audit log to disk immediately (bypassing the autosave interval). New kits start empty — edit their default layout in `data/pvp/kits.json` or let players build layouts in the kit editor. All admin actions are written to the audit log (§18). |

**`/pvpadmin audit`** prints the last 15 audit entries in the format `[timestamp] Actor: action`.

**`/pvpadmin kit` subcommands:**

| Subcommand | Effect |
|---|---|
| `kit create <id>` | Create an empty kit with id (lowercased), display name `&f<id>`, STONE icon |
| `kit delete <id>` | Delete a kit permanently (player layouts referencing it become stale) |
| `kit list` | Print every kit id + display name |

---

### `/pvpdebug`

| # | Field | Details |
|---|---|---|
| 1 | **Syntax** | `/pvpdebug [state <player> \| duels \| arenas \| queues]` |
| 2 | **Permission** | GAMEMASTERS (op level 2+) |
| 3 | **Description** | Live diagnostics for troubleshooting. Bare `/pvpdebug` prints a one-line summary; subcommands dump details. |
| 4 | **Arguments** | `state <player>` — full session dump; `duels` — all active duels; `arenas` — arena statuses; `queues` — total queued players. |
| 5 | **Examples** | `/pvpdebug` · `/pvpdebug state Alex` · `/pvpdebug duels` |
| 6 | **GUI Equivalent** | None (terminal-only diagnostics). |
| 7 | **Aliases** | None |
| 8 | **Success Output** | Summary: `PvP Core: duels=2 queues=5 parties=3 arenas=8 (6 usable)`. State: `Alex: state=DUELING loginSynced=true duel=nodebuff (COUNTDOWN) party=3 members queued=false`. |
| 9 | **Error Cases** | Player not found → Brigadier error. |
| 10 | **Related Config** | `general.debug` in `settings.json` enables verbose logging. |
| 11 | **Related Commands** | `/pvpadmin`, `/arena`. |
| 12 | **Notes & Tips** | `state` is the first tool to reach for when a player reports being “stuck” — it shows their PlayerState (`LOBBY` / `QUEUEING` / `DUELING` / `SPECTATING`), whether login-sync fired, their duel, party and queue status. |

---
## 8. The GUI System — Complete Walkthrough

Every GUI is a server-driven chest screen — vanilla clients render it with zero installation. All titles, row counts, item materials, slots, names, lores, amounts and glow flags are configurable in `guis.json` (§14).

### 8.1 Menu flow map

```
Lobby hotbar
 ├── Unranked Queue (slot 0) ──► Kit Picker ──► (queued: hotbar shows Leave Queue)
 ├── Ranked Queue  (slot 1) ──► Kit Picker (ranked)
 ├── Duel          (slot 2) ──► Player Picker ──► Kit Picker ──► Challenge GUI (target)
 ├── Kits          (slot 3) ──► Kit List ──► Kit Stats ──► Kit Editor ──► Item Palette
 │                                                        │            └► Enchant GUI
 │                                                        └─ (Back)
 ├── Party         (slot 4) ──► Party GUI ──► Player Picker (invite) / Party Duel
 ├── Leaderboards  (slot 5) ──► Leaderboard GUI
 ├── Profile       (slot 6) ──► Profile GUI ──► Stats / Kit Stats
 └── Spectate      (slot 7) ──► Spectate GUI (live duels)
```

### 8.2 Kit Picker (queue)
5 rows, one item per kit (the kit's `displayItem`), prev/next arrows (36/44), Back (49). In ranked mode each kit item's lore shows your ELO. Click = join queue for that kit.

### 8.3 Kit List & Kit Stats
**Kit List** (5 rows): info book at 4, **Create Kit** at 42 *(admins only, 2.1.0)*, page arrows at 39/41, Close at 40. Left-click a kit → **Kit Stats** (per-kit W/L/ELO); middle/drop-click a kit (admins) → delete it. **Kit Stats** (5 rows): *Edit Layout* anvil (38) → Kit Editor, *Edit Template* command block (42, **admins only, 2.1.0**) → Kit Editor in template mode, Back (40), Close (44).

### 8.4 Duel Challenges — Chat Buttons + Rounds Picker *(changed in 2.1.0/2.2.0)*

Duel requests are **clickable [ACCEPT] / [DENY] chat buttons** with hover tooltips (set `duel.challengeViaChat: false` for the old chest GUI). Since 2.2.0, challenging a player (or another party) first asks **how many rounds** the duel should have: after picking a kit, a **Rounds picker** (3 rows: Best of 1 / 3 / 5 / 7, capped by `duel.maxBestOf`) opens and the challenge is sent with that format. Best-of duels play an intermission between rounds (`duel.roundIntermissionSeconds`), the scoreboard shows the round score, and the round loser spectates until the next round starts (`duel.loserSpectates`).

### 8.5 Player Picker
6 rows of online-player heads (real skins). Page arrows (45/53), page indicator (49), Back (48). Used for duels, stats, and party invites — the action depends on which menu opened it.

### 8.6 Party Menu — in Chat *(changed in 2.2.0)*

The party system lives entirely in **chat**: clicking the Party hotbar item (or `/party`) prints a clickable menu — [INVITE] (lists clickable players), [CHAT: ON/OFF], [DUEL] (lists other party leaders), [MANAGE] ([PROMOTE]/[KICK] per member, leader only), [LEAVE], [DISBAND] (leader) and [CLOSE]. Invites arrive as **[ACCEPT] / [DENY] chat buttons**. No chest GUI is used for parties anymore.

### 8.7 Profile / Stats / Leaderboard GUIs
**Profile** (5 rows): real-skin head (13), Global Stats (11), Kit Stats (15), Close (31). **Stats** (6 rows): one line per kit + global aggregate; Back (45), Close (53). **Leaderboard** (6 rows): NETHER_STAR global (4), type-switch EMERALD (49, `Top by: {type}`), page arrows (45/53), Back (48).

### 8.8 Spectate GUI — 6 rows
Lists every live duel as an item showing both fighters and the kit; ENDER_EYE info at 4; page arrows; Close (49). Click a duel → teleported to its spectator spawn, hotbar becomes the Ender Pearl **Return to Lobby** item (click it again to return).

### 8.9 Admin GUI & Arena Editor
**Admin GUI** (6 rows): Reload (20) · Save (21) · Arenas (22) · Kits (23) · Set Lobby (24) · Close (49). **Arena List** (6 rows): LIME_DYE *Create Arena* (45), info (4), arrows (48/52), Back (49); arena items: left-click edit, right-click teleport. **Arena Editor** (5 rows): Set Spawn 1 (19) / Set Spawn 2 (20) / Set Spectator (21) each capture your live position **and look direction**, Teleport (23), Toggle Enabled (24), Validate (25), **Rename** opens the anvil input (31), Delete (40), Back (36).

### 8.10 Anvil text input
All text input (arena rename, etc.) happens in an anvil GUI: type in the text field, then click the LIME_DYE **Confirm** (slot 1). Nothing can be taken out of the anvil — it's purely a text field. This keeps the mod 100% chat-prompt-free.

---

## 9. The Kit Editor — Deep Dive

The kit editor is the heart of the practice experience: a **true double chest (54 slots)** where you physically arrange your kit. No commands, no inventory dropping.

### 9.1 Layout

| Chest slots | Purpose |
|---|---|
| 0 – 3 | **Armor**: helmet, chestplate, leggings, boots |
| 4 – 8 | Filler glass (armor info stand at 5) |
| 9 – 17 | **Hotbar** — your 9 in-fight hotbar slots, in order |
| 18 – 44 | **Main inventory** — your 27 backpack slots, in order |
| 45 | **Back** arrow (saves first) |
| 46 | **Enchant slot** (yellow glass) — place an item here to enchant it |
| 49 | **Save Kit** (lime dye) |
| 50 | **Clear Kit** (red dye) |
| 51 | **Item Palette** (chest) — browse every item in the game |
| 52 | **Enchant Item** (enchanted book) — opens the enchant menu for slot 46 |
| 53 | **Close** (barrier) — auto-saves |

### 9.2 How editing works
- Items can be **freely moved** between the editable slots (armor 0–3, hotbar 9–17, inventory 18–44) with normal pick-up/place clicks — real item movement, server-validated.
- **Item Palette** (51): a paged 6-row browser of every obtainable item; clicking one adds a copy into your first free kit slot.
- **Enchanting**: place an item in slot 46 → click **Enchant** (52) → the Enchant GUI lists every enchantment as a book with its current level in the lore; **left-click cycles the level up**; at max level, clicking again **removes** the enchantment.
- **Saving**: click **Save** (49) for an explicit save with confirmation message — or just **close the GUI**; anything changed is saved automatically. Nothing is ever lost: even the item sitting on your cursor is absorbed back into the layout when the menu closes.
- **Clear** (50) empties the whole layout (asks nothing — but auto-save makes it recoverable only by re-editing).

### 9.3 What a kit is
A kit defines the **default layout** (40 positions: 4 armor + 9 hotbar + 27 inventory). Your personal layout starts as a copy of the default and diverges as you edit. Kit definitions live in `data/pvp/kits.json`; personal layouts are stored inside your player data file. Default kits: `nodebuff`, `classic`, `gapple`.

---

## 10. The Party System

Parties are ad-hoc groups for casual play and team duels. Since 2.2.0 the party system lives entirely in **chat** — no chest GUI.

- **Creation**: click the Party hotbar item (auto-creates, see `party.autoCreateOnItemClick`) or `/party create`.
- **Membership**: max `party.maxSize` (default 8). Invites arrive as **clickable [ACCEPT] / [DENY] chat buttons** and expire after `party.inviteExpirySeconds` (default 60s). If `party.membersCanInvite` is true (default), any member can invite; otherwise leader-only.
- **Leadership**: the creator leads; `promote` transfers. Leader-only actions: kick, promote, disband, party duels.
- **Party chat**: toggle with the [CHAT] button or `/party chat` — while ON your chat goes only to party members with the `[Party]` format.
- **Party duels**: the [DUEL] button (or `/party duel <leader>`) lists other parties — pick a kit, pick the rounds, and both parties fight a team duel.
- **Duel lock (2.2.0)**: players inside a duel can't invite, be invited, or accept invites — both sides get a clear chat error instead.
- **Hotbar integration**: the moment you join/create a party, your slot-4 item becomes the “Party (Open Menu)” variant; leaving/disbanding restores the default item.

---

## 11. The Chat System (Duel Isolation)

The mod routes **all chat server-side** through its chat router:

| Rule | Default | Config key |
|---|---|---|
| Players **inside a duel** chat only with the duel's participants | on | `chat.isolateDuels` |
| **Global chat is hidden from** players inside a duel (they see it again the moment the duel ends) | on | `chat.hideGlobalFromDuelists` |
| Duel chat is also shown to that duel's spectators | on | `chat.duelChatIncludesSpectators` |
| Spectator chat goes to the duel audience | on | spectate chat handling |
| Party chat mode (per-player toggle) | optional | `/party chat` |

**Formats** (all configurable, §13.8):

```
globalFormat: &7{player}&8: &f{message}
duelFormat:   &8[&6Duel&8] &7{player}&8: &f{message}
partyFormat:  &8[&5Party&8] &d{player}&8: &f{message}
specFormat:   &8[&5Spec&8] &7{player}&8: &f{message}
```

Messages longer than 256 characters are trimmed. Commands are never intercepted — only plain chat is routed, via the Fabric `ALLOW_CHAT_MESSAGE` event (signed-chat bookkeeping stays intact). Set `chat.manageChat: false` to disable all chat handling and leave chat 100% vanilla.

---

## 12. The Per-Duel Scoreboard & The Lobby Scoreboard

### 12.1 Per-duel scoreboard

Every duel gets its **own sidebar scoreboard**, sent via packets only to that duel's participants and spectators, and removed the instant the duel ends. Vanilla sidebar conflicts are avoided by using per-duel objective names (`pvpd_<duelid>`).

Default layout (every line configurable in `settings.json → scoreboard.lines`):

```
&6&lDUEL
&7&m----------------
&6{kit}
&e{player1} &f{score1} &8- &f{score2} {player2}
&7Round: &f{round}&7/&f{rounds} &8- &7{duration}
&7Hits: &f{hits1} &8/ &f{hits2}
&7Ping: &a{ping1}ms &8/ &a{ping2}ms
```

Available placeholders per line: `{player1}` `{player2}` `{kit}` `{duration}` `{hits1}` `{hits2}` `{ping1}` `{ping2}` `{hp1}` `{hp2}` `{arena}` — and since 2.2.0 also **`{round}` `{rounds}` `{score1}` `{score2}`** (current round, total rounds of the best-of, and each team's round wins). In **team duels**, the scoreboard instead lists **every fighter with live ping and hearts** (`Name - 43ms 8❤`). Updates every `scoreboard.updateIntervalTicks` (default 20 = 1s). Disable entirely with `scoreboard.enabled: false`.

### 12.2 Lobby scoreboard *(new in 2.2.0)*

Every player **in the lobby or queue** now sees their own sidebar scoreboard with their stats and your server IP — fully customizable in `settings.json → lobbyScoreboard`:

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `true` | Show the lobby scoreboard at all |
| `title` | `&6&lPvP CORE` | Sidebar header (supports & colors) |
| `serverIp` | `&eplay.example.com` | **Set this to your server address** — shown by the `{ip}` placeholder |
| `updateIntervalTicks` | `20` | Refresh rate (20 = once per second) |
| `lines` | *(see below)* | Every line of the sidebar, placeholders applied per player |

Default lines:

```
&8&m------------------
&7Player: &f{player}
&7Rank: &f#{rank} &8(&f{elo}&8)
&7Wins: &a{wins} &7Losses: &c{losses}
&7Streak: &f{streak}
&7Online: &f{online}&7/&f{max}
&7Ping: &f{ping}ms
{ip}
&8&m------------------
```

Placeholders: `{player}` `{online}` `{max}` `{ip}` `{ping}` `{wins}` `{losses}` `{duels}` `{winrate}` `{elo}` `{rank}` `{streak}` `{beststreak}` `{party}` `{queue}` `{world}` `{date}` `{time}`.

The lobby scoreboard **hides automatically** the moment a duel starts (the duel sidebar takes over) or while spectating, and comes back when the player returns to the lobby. It's per-player — everyone sees their own stats, rank and ping.

---
## 13. Configuration Reference — `settings.json`

Location: `config/pvp/settings.json`. The file is written with **every option included** on first boot, is hot-reloadable (`/pvpadmin reload`), and merges your edits over defaults so new options appear automatically after mod updates.

### 13.1 `general`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `prefix` | string | `&8[&6PvP&8] &r` | Prefix used by messages that opt into it |
| `debug` | bool | `false` | Verbose logging |

### 13.2 `lobby`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `giveItems` | bool | `true` | Give the lobby hotbar at all |
| `lockInventory` | bool | `true` | **Lock lobby inventories** — blocks moving/dropping/swapping items while in lobby state. Never applies while any screen is open or before login-sync. |
| `adminBypassLock` | bool | `false` | *(2.2.0)* Admins are locked **by default**; set `true` to let ops move items freely |
| `blockItemUse` | bool | `true` | *(2.2.0)* Block **using** items (food, pearls, potions) while in lobby/queue/spectate |
| `blockWorldInteraction` | bool | `true` | *(2.2.0)* Block **placing / breaking / interacting** with blocks in the lobby |
| `lockSpectators` | bool | `true` | *(2.2.0)* Also lock the inventory of players spectating a duel |
| `clearInventoryAfterDuel` | bool | `true` | *(2.2.0)* Wipe the inventory when a duel ends — players return with **only the lobby hotbar** (set `false` to restore their pre-duel items instead) |
| `teleportAfterDuel` | bool | `true` | *(2.2.0)* Teleport players to the **lobby point** when their duel ends (set `false` to return them to where they stood) |
| `loginSyncMode` | string | `SMART` | **Third-party login GUI compatibility**: `SMART` waits until the player *moves* or a timeout elapses; `TIMEOUT` always waits the full timeout; `INSTANT` applies immediately (servers without login GUIs). |
| `loginTimeoutTicks` | int | `200` | Fallback wait (200 ticks = 10s) before lobby state applies no matter what |
| `invulnerable` | bool | `true` | Keep lobby/queue players at full health & food |
| `teleportOnJoin` | bool | `false` | Teleport players to the lobby point when they join |
| `lobbyWorld` / `lobbyX/Y/Z` / `lobbyYaw/Pitch` | — | overworld / 0, 64, 0 / 0, 0 | The lobby point (set in-game via `/pvpadmin setlobby`; players warp back with `/lobby`) |

### 13.3 `duel`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `countdownSeconds` | int | `3` | 3-2-1 countdown before FIGHT |
| `endDurationSeconds` | int | `3` | Seconds the end screen stays before returning to lobby |
| `timeoutMinutes` | int | `15` | Hard duel time limit (0 = unlimited) |
| `disconnectGraceSeconds` | int | `10` | Reconnect window before a disconnected fighter forfeits |
| `disableHunger` | bool | `true` | Keep food full during duels |
| `healOnStart` | bool | `true` | Full heal at duel start |
| `healAfterDuel` | bool | `true` | Full heal when returning to lobby |
| `rematchExpirySeconds` | int | `20` | Rematch offer window |
| `showEndTitles` | bool | `true` | VICTORY/DEFEAT title screens |
| `challengeViaChat` | bool | `true` | Duel requests use clickable chat buttons instead of a chest GUI (§8.4) |
| `defaultBestOf` | int | `1` | *(2.2.0)* Round count for queue matches, `/duel` and rematch-less challenges |
| `maxBestOf` | int | `9` | *(2.2.0)* Highest best-of players may pick in the Rounds GUI (BO1–BO9, odd values) |
| `roundIntermissionSeconds` | int | `5` | *(2.2.0)* Pause between rounds of a best-of duel |
| `loserSpectates` | bool | `true` | *(2.2.0)* The round loser spectates until the next round; the match loser spectates the arena until they return |

### 13.4 `elo`

| Key | Type | Default |
|---|---|---|
| `startRating` | int | `1000` |
| `kFactor` | int | `32` |
| `range0to10s` | int | `50` |
| `range10to20s` | int | `100` |
| `range20to30s` | int | `150` |
| `range30plus` | int | `250` |

### 13.5 `queue`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `matchmakingIntervalTicks` | int | `10` | Matchmaker pass frequency (0.5s) |
| `statusIntervalTicks` | int | `100` | Queue-status message frequency (5s) |

### 13.6 `party`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `maxSize` | int | `8` | Max members per party |
| `inviteExpirySeconds` | int | `60` | Invite expiry |
| `membersCanInvite` | bool | `true` | Non-leaders may invite |
| `autoCreateOnItemClick` | bool | `true` | Clicking the party item creates a party automatically |

### 13.7 `chat`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `manageChat` | bool | `true` | Master switch — `false` leaves chat 100% vanilla |
| `isolateDuels` | bool | `true` | Duel chat isolation (§11) |
| `hideGlobalFromDuelists` | bool | `true` | Hide global chat from duelists |
| `duelChatIncludesSpectators` | bool | `true` | Spectators see duel chat |
| `globalFormat` / `duelFormat` / `partyFormat` / `specFormat` | string | see §11 | Chat formats |

### 13.8 `scoreboard`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `enabled` | bool | `true` | Per-duel scoreboard (§12) |
| `title` | string | `&6&lDUEL` | Sidebar title |
| `lines` | list | see §12 | Sidebar lines with placeholders |
| `updateIntervalTicks` | int | `20` | Refresh rate |

### 13.9 `spectate`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `enabled` | bool | `true` | Spectating feature |
| `hideFromFighters` | bool | `true` | Spectators invisible to fighters |

### 13.10 `storage`

| Key | Type | Default | Meaning |
|---|---|---|---|
| `saveIntervalSeconds` | int | `60` | Autosave + leaderboard refresh interval |

### 13.11 `lobbyScoreboard` *(new in 2.2.0)*

| Key | Type | Default | Meaning |
|---|---|---|---|
| `enabled` | bool | `true` | Show the lobby sidebar scoreboard |
| `title` | string | `&6&lPvP CORE` | Sidebar header (& colors supported) |
| `serverIp` | string | `&eplay.example.com` | Your server address, shown by `{ip}` |
| `updateIntervalTicks` | int | `20` | Refresh rate (20 = 1/s) |
| `lines` | list | *(§12.2)* | Every sidebar line; placeholders: `{player}` `{online}` `{max}` `{ip}` `{ping}` `{wins}` `{losses}` `{duels}` `{winrate}` `{elo}` `{rank}` `{streak}` `{beststreak}` `{party}` `{queue}` `{world}` `{date}` `{time}` |

### 13.12 `kit` *(new in 2.2.0)*

| Key | Type | Default | Meaning |
|---|---|---|---|
| `editorBaseOnOriginal` | bool | `true` | The player kit editor always starts from the **original kit template** (as set the first time); saving stores the player's personal layout. Set `false` to keep editing the last saved layout instead. |

---

## 14. Configuration Reference — `guis.json`

Location: `config/pvp/guis.json`. Controls **every item in every GUI**: slot, material, amount, name, lore, glow, and whether it's enabled at all. Change a hotbar item to a diamond? Move the leaderboard to slot 3? Rename “Party” to “Team”? All here. Hot-reloads with `/pvpadmin reload`, then re-applies to everyone's hotbar instantly.

### 14.1 Lobby items — `lobby.items.<key>`

Keys: `unranked`, `ranked`, `duel`, `kits`, `party`, `party_active`, `leaderboard`, `profile`, `spectate`, `queue_leave`, `spectate_return`, `filler`.

```json
"unranked": {
  "enabled": true,
  "slot": 0,
  "material": "IRON_SWORD",
  "amount": 1,
  "name": "&b&lUnranked Queue",
  "lore": ["&7Click to browse kits", "&7and join the unranked queue."],
  "glow": false
}
```

- `slot` must be 0–8 (hotbar). Overlapping keys are fine — state-aware variants replace each other.
- The `profile` item renders the **viewer's own skin head** automatically (material `PLAYER_HEAD`).
- Set `enabled: false` to remove an item entirely (e.g. disable spectating by hiding the item).

### 14.2 Chest GUIs — `guis.<guiKey>`

Each GUI has `title`, `rows` (1–6), `items.<itemKey>` entries with the same ItemSpec schema as above (slots are chest slots 0–53), and — for `kit_list` / `queue` — a **`kitSlots` list** controlling exactly which chest slots the kit icons occupy (2.1.0: customizable kit icon slots):

```json
"kit_list": {
  "title": "&8Kits",
  "rows": 5,
  "kitSlots": [10,11,12,13,14,15,16, 19,20,21,22,23,24,25, 28,29,30,31,32,33,34],
  "items": { ... }
}
```

Every GUI also has a `filler` item (the pane used to pad empty slots) — configurable like any other item.

| GUI key | Purpose | Default rows |
|---|---|---|
| `kit_list` | Browse kits | 5 |
| `kit_stats` | Per-kit stats | 5 |
| `kit_editor` | The 54-slot editor | 6 |
| `palette` | Item palette browser | 6 |
| `enchant` | Enchantment picker | 6 |
| `queue` | Kit picker (queue/duel) | 5 |
| `player_picker` | Online-player browser | 6 |
| `challenge` | Duel request dialog | 3 |
| `rematch` | Rematch dialog | 3 |
| `party_invite` | Party invite dialog | 3 |
| `party` | Party management | 6 |
| `stats` | Statistics view | 6 |
| `profile` | Profile view | 5 |
| `leaderboard` | Leaderboards | 6 |
| `spectate` | Live duel browser | 6 |
| `confirm` | Generic yes/no dialog | 3 |
| `admin` | Admin panel | 6 |
| `arenas` | Arena list | 6 |
| `arena_editor` | Arena editing | 5 |

Placeholders available in names/lores: `{player}`, `{kit}`, `{arena}`, `{status}`, `{count}`, `{online}`, `{max}`, `{page}`, `{pages}`, `{type}`.

⚠️ **Geometry warning:** for the `kit_editor`, slots 0–3/9–17/18–44 are the editable kit area and 45–53 hold the control buttons — if you remap buttons, keep them out of the editable ranges.

---

## 15. Configuration Reference — `messages.json`

Location: `config/pvp/messages.json`. **Every message the mod sends** is here with `&` color codes and `{placeholders}`. Unknown keys fall back to defaults, and your file gains new keys automatically on reload. The complete key list (defaults shown in the generated file):

**General:** `prefix`, `no-permission`, `player-not-found` `{player}`, `player-only`, `not-in-lobby`, `reload-done` `{ms}`, `data-saved`.

**Lobby/login:** `lobby.applied`, `lobby.items-locked`, `lobby.teleported`.

**Queue:** `queue.joined` `{type}` `{kit}`, `queue.left`, `queue.not-queued`, `queue.status` `{kit}` `{type}` `{position}` `{range}`, `queue.matched` `{kit}` `{arena}`, `queue.already`.

**Duel:** `duel.challenge-sent` `{player}` `{kit}`, `duel.challenge-received-title`, `duel.challenge-received-subtitle` `{player}`, `duel.challenge-expired` `{player}`, `duel.challenge-expired-sender` `{player}`, `duel.challenge-denied` `{player}`, `duel.challenge-busy` `{player}`, `duel.countdown-title` `{seconds}`, `duel.fight-title`, `duel.win-title`, `duel.win-subtitle` `{loser}`, `duel.lose-title`, `duel.lose-subtitle` `{winner}`, `duel.draw-title`, `duel.started` `{kit}` `{player1}` `{player2}` `{arena}`, `duel.ended` `{winner}` `{loser}` `{kit}` `{duration}`, `duel.draw` `{player1}` `{player2}` `{duration}`, `duel.timeout`, `duel.elo-change` `{kit}` `{player1}` `{delta1}` `{rating1}` `{player2}` `{delta2}` `{rating2}`, `duel.opponent-disconnected` `{player}` `{seconds}`, `duel.opponent-returned` `{player}`, `duel.won-by-disconnect`, `duel.rematch-sent` `{player}`, `duel.rematch-accepted`, `duel.rematch-declined`, `duel.opponent-left`.

**Kit:** `kit.not-found` `{kit}`, `kit.layout-saved` `{kit}`, `kit.layout-reset` `{kit}`, `kit.cleared`, `kit.item-added` `{item}`, `kit.enchanted` `{enchant}` `{level}`, `kit.enchant-removed` `{enchant}`, `kit.renamed` `{name}`, `kit.created` `{kit}`, `kit.deleted` `{kit}`, `kit.no-item-to-enchant`, `kit.save-hint`.

**Party:** `party.created`, `party.disbanded`, `party.left`, `party.joined` `{player}`, `party.player-joined` `{player}`, `party.player-left` `{player}`, `party.kicked` `{player}`, `party.you-kicked`, `party.invite-sent` `{player}`, `party.invite-received-title`, `party.invite-received-subtitle` `{player}`, `party.invite-expired`, `party.invite-denied` `{player}`, `party.promoted` `{player}`, `party.you-promoted`, `party.full` `{max}`, `party.already-in-party`, `party.not-in-party`, `party.not-leader`, `party.chat-on`, `party.chat-off`, `party.no-parties`, `party.challenge-sent` `{leader}`, `party.challenge-received-title`, `party.challenge-received-subtitle` `{leader}` `{kit}`.

**Spectate:** `spectate.started` `{player1}` `{player2}`, `spectate.ended`, `spectate.no-duels`, `spectate.return`.

**Stats/leaderboard:** `stats.header` `{player}`, `stats.line` `{kit}` `{wins}` `{losses}` `{elo}`, `leaderboard.header` `{type}` `{kit}`, `leaderboard.entry` `{position}` `{player}` `{value}`, `leaderboard.empty`.

**Arena:** `arena.created` `{arena}`, `arena.deleted` `{arena}`, `arena.spawn-set` `{arena}` `{spawn}`, `arena.enabled` `{arena}`, `arena.disabled` `{arena}`, `arena.valid` `{arena}`, `arena.invalid` `{arena}` `{reason}`, `arena.teleported` `{arena}`, `arena.renamed` `{name}`, `arena.none`, `arena.in-use`.

**Admin/input:** `admin.gui-opened`, `admin.lobby-set`, `admin.audit-recorded` `{entry}`, `input.enter-name`, `input.invalid-name`.

---

## 16. ELO & Matchmaking Reference

**Ratings** start at `elo.startRating` (1000) per kit, tracked per-kit per-player. After a ranked duel, both players' ratings move by the standard Elo formula with K = `elo.kFactor` (32):

```
expected = 1 / (1 + 10^((opponent - you) / 400))
delta    = round(K * (actual - expected))     // actual = 1 win, 0 loss, 0.5 draw
```

**Search-range expansion** (ranked queue wait time → allowed ELO gap):

| Waiting | Allowed rating difference |
|---|---|
| 0–10 s | ±`range0to10s` (50) |
| 10–20 s | ±`range10to20s` (100) |
| 20–30 s | ±`range20to30s` (150) |
| 30 s + | ±`range30plus` (250) |

The matchmaker runs every `queue.matchmakingIntervalTicks` (10 ticks). Unranked queues match in join order without ELO. Arena selection picks any FREE, enabled, valid arena.

---

## 17. Data Storage & File Layout

No database — flat JSON with **atomic writes** (temp file + move) so a crash can never corrupt data.

```
config/pvp/
 ├── settings.json      all behavior settings (§13)
 ├── guis.json          every GUI item (§14)
 └── messages.json      every message (§15)

data/pvp/
 ├── kits.json          kit definitions (default layouts)
 ├── arenas.json        arena spawn points & status
 ├── audit.json         admin audit log (§18)
 ├── players/<uuid>.json   per-player stats, ELO, layouts, party-chat flag
 └── pending/<uuid>.json   crash-safety inventory snapshots (auto-removed after restore)
```

Autosave runs every `storage.saveIntervalSeconds` (60s) and on every player quit; `/pvpadmin save` forces an immediate flush.

---

## 18. Audit Log

Every admin action is recorded to `data/pvp/audit.json` and shown with `/pvpadmin audit` (last 15): arena create/delete/spawn-set/enable/disable, kit create/delete, config reloads, lobby-set, force-saves. Each entry stores the actor name, epoch timestamp, and action text. The log is capped and rotated by size.

---

## 19. Color Codes & Formatting

All `guis.json` names/lores, `messages.json` texts, chat formats and scoreboard lines support:

| Code | Color | Code | Color | Code | Color |
|---|---|---|---|---|---|
| `&0` | black | `&4` | dark red | `&8` | dark gray |
| `&1` | dark blue | `&5` | dark purple | `&9` | blue |
| `&2` | dark green | `&6` | gold | `&a` | green |
| `&3` | dark aqua | `&7` | gray | `&b` | aqua |
| `&c` red | `&d` light purple | `&e` yellow | `&f` white | | |

Formatting: `&l` bold · `&o` italic · `&n` underline · `&m` strikethrough · `&r` reset. Use `&&` to escape a literal `&`. Hex colors are not supported in config strings (vanilla-safe `&` codes only).

---

## 20. Integration & Compatibility Notes

1. **Login / auth mods** (AuthMe-style Fabric ports, etc.): the mod never touches players before *login-sync* fires. With `loginSyncMode: SMART` (default), lobby items and the inventory lock are applied only after the player **moves** or the 10-second timeout passes — so third-party login GUIs work untouched, and their buttons remain clickable. The inventory lock also only ever applies while the player's own inventory screen is the active screen — clicks inside any other GUI (ours or another mod's) are never intercepted.
2. **No game-mode management, period.** The mod never calls `setGameMode`/`changeGameMode` — not for spectating, not for duels, not for the lobby. Your lobby plugin stays in full control of game modes. Spectator handling is done with the mod's own position/visibility management.
3. **SkinsRestorer & skin mods**: profile heads are built from each player's **live server-side GameProfile**, so restored skins show correctly on heads, in the player picker, and in party lists.
4. **Vanilla clients**: zero client-side requirements. All GUIs are server-driven vanilla screens; the scoreboard uses packets.
5. **Other chat mods**: PvP Core routes plain chat packets itself (that's how duel isolation works). If you run a heavy chat-formatting mod, set `chat.isolateDuels: false` to hand chat back to the other mod (losing duel isolation).
6. **World management**: arenas record their world id; the mod teleports across worlds as needed. Multiverse-style setups work as long as world ids stay stable.

---

## 21. Troubleshooting & FAQ

**“My lobby items are gone / players have no hotbar items.”**
Check `lobby.giveItems` is true. If you run a login GUI, ensure `loginSyncMode` is `SMART` (default) and the timeout (`loginTimeoutTicks`, 200 = 10s) is longer than your login flow. `/pvpdebug state <player>` shows `loginSynced=true/false`.

**“A player is stuck / can't queue.”**
`/pvpdebug state <player>` — look at `state` (should be `LOBBY` when idle). If `DUELING` with no duel alive, the disconnect-grace cleanup will resolve it within `duel.disconnectGraceSeconds`; a server restart also fully restores state (snapshots are crash-safe on disk).

**“Duels never start / 'no arenas'.”**
An arena needs: spawns 1 & 2 set, enabled, and in a loaded world. Verify with `/arena <id> validate`. `/pvpdebug arenas` shows `spawns=true/false`.

**“Clicks in another mod's GUI got blocked.”**
That's not this mod — the lock only applies when the player's *own* inventory screen is open. If you still suspect it, set `lobby.lockInventory: false` and re-test.

**“Global chat doesn't reach dueling players.”**
That's by design (`chat.hideGlobalFromDuelists: true`) — it restores the moment the duel ends. Toggle the key to disable.

**“ELO feels too swingy / too slow.”**
Tune `elo.kFactor` (lower = slower, default 32) and the search ranges. Changes apply immediately (no restart).

**“Can I move the hotbar items?”**
Yes — `guis.json → lobby.items.*` → change `slot` (0–8), `/pvpadmin reload` after.

**“Where do I edit what the scoreboard shows?”**
`settings.json → scoreboard.lines` — full placeholder list in §12.

**“How do I reset a player's data?”**
Delete `data/pvp/players/<uuid>.json` while they're offline.

---

## 22. What Changed in 2.2.0

All 10 issues from the third live-testing round are fixed:

| # | Issue | Fix |
|---|---|---|
| 1 | Party invites should be chat, not GUI | The whole party system moved to **clickable chat**: invites arrive as [ACCEPT]/[DENY] buttons, the Party item opens a chat menu ([INVITE] · [CHAT] · [DUEL] · [MANAGE] · [LEAVE] · [DISBAND]), new `/party menu` + `/party duel` subcommands |
| 2 | Kit editor should base on the original kit | `kit.editorBaseOnOriginal` (default true): the editor always starts from the **original kit template**; saving stores the personal layout — no more compounding edits |
| 3 | Settable lobby location + teleport | `/pvpadmin setlobby` now pairs with a player-facing **`/lobby`** command; `lobby.teleportAfterDuel` (default true) sends duel finishers to the lobby point |
| 4 | Clear inventory after duels | `lobby.clearInventoryAfterDuel` (default true): duel leftovers are wiped — players return with only the lobby hotbar |
| 5 | Loser enters spectator mode | `duel.loserSpectates` (default true): the **round loser** spectates during the intermission of best-of matches; the **match loser** spectates the arena until they click Return / use `/leave` / accept a rematch |
| 6 | Custom round counts | **Best-of-N duels**: challenging a player or party opens a **Rounds picker** (BO1/3/5/7, capped by `duel.maxBestOf`); scoreboard shows round score; rematch keeps the format |
| 7 | Chat isolation still broken | This was the same root cause already fixed in 2.1.0 (formats read from the wrong file + packet-cancel breaking chat chains); 2.2.0 keeps the corrected `ALLOW_CHAT_MESSAGE` router and re-verified it |
| 8 | Lobby items still movable / usable | Full lock-down: item **use** blocked (`lobby.blockItemUse`), block **place/break/interact** blocked (`lobby.blockWorldInteraction`), spectators locked (`lobby.lockSpectators`), and **admins are locked too** unless `lobby.adminBypassLock` is true |
| 9 | Dueling players shouldn't accept invites | No party invites, invite-accepts or duel-accepts while either side is dueling — both sides get clear chat errors (`party.you-in-duel`, `party.target-in-duel`, `party.cant-while-dueling`) |
| 10 | Lobby scoreboard | New per-player **lobby sidebar** with stats, ELO rank, streak, ping, party, online count and your **server IP** — title, IP and every line configurable (`lobbyScoreboard` section, §13.11); auto-hides during duels/spectating |

Also in 2.2.0: the duel scoreboard gains `{round}`/`{rounds}`/`{score1}`/`{score2}` placeholders, `/leave` also works during the round intermission, and a disconnect no longer breaks an ongoing intermission.

---

## 23. History: What Changed in 2.1.0

All 10 issues from the second live-testing round are fixed:

| # | Issue | Fix |
|---|---|---|
| 1 | “Missing item: filler” in every GUI | Every GUI now has its own configurable `filler` item; the fallback for `filler` is a clean gray pane — the “Missing item” text can never appear for it again |
| 2 | Items duplicated in the inventory | Root cause: rejected clicks left the client's optimistic click applied (diff-based sync sent nothing). Every rejected click/cancel now triggers a **full slot resync** (`updateToClient`); quitting also restores stashed items |
| 3 | Leave a duel with a command | New **`/leave`** — forfeits the duel (opponent wins), or leaves the queue / spectate mode |
| 4 | Queue hotbar clutter | While queued, the hotbar shows **only** the Leave Queue item |
| 5 | Admin kit creation | **`/pvpadmin kit create <id> [name]`** copies the admin's **current inventory** (armor + hotbar + inventory) into a new kit; also a **Create Kit** button in the Kits GUI (anvil name input) |
| 6 | Customizable kit icon slots | New **`kitSlots`** list in guis.json for the kit list & kit picker GUIs |
| 7 | Remove the leaderboard item | The leaderboard hotbar item is gone (disabled in defaults; re-enable in guis.json if wanted — `/leaderboard` still works) |
| 8 | Admin view & edit kits | Kit Stats gains an **Edit Template** button (admins): the kit editor saves directly into the kit template everyone receives; admins can also delete kits from the kit list |
| 9 | Accept/deny in chat, not GUI | Duel requests now show **clickable [ACCEPT] / [DENY] chat buttons** with hover tooltips (old GUI available via `duel.challengeViaChat: false`) |
| 10 | Global & duel chat not working | Two root causes fixed: the router read formats from messages.json where the keys never existed (chat literally printed “globalFormat”), and cancelling the signed chat packet broke chat-chain bookkeeping. Chat now routes through the Fabric `ALLOW_CHAT_MESSAGE` event with formats read from settings.json; new `chat.manageChat` master switch |

Also in 2.1.0: GUI slot self-healing (slots outside their GUI are reset to defaults on load — fixes old configs where the kit list Close button sat at slot 49 of a 45-slot GUI), and nav slots corrected in defaults.

---

## 24. History: What Changed in 2.0.0

Version 2.0.0 was the first live-testing release — all 15 issues from the initial round were fixed:

| # | Issue | Fix |
|---|---|---|
| 1 | Players could move default lobby items | Server-side inventory lock: clicks, drops and off-hand swaps in the lobby are cancelled and re-synced (mixin on the play network handler) |
| 2 | Kit editing shouldn't need commands/inventory | Full GUI editing — no commands anywhere in the flow |
| 3 | Conflict with server login GUIs (buttons unclickable) | Login-sync modes; the mod stays fully hands-off until the player moves or a timeout passes; lock never applies inside any open GUI |
| 4 | No game-mode management wanted | All game-mode code removed — spectating uses the mod's own handling; lobby plugins keep control |
| 5 | Hide global chat during duels | Chat router isolates duel chat and hides global chat from duelists; restored automatically at duel end |
| 6 | Party item should auto-create parties | Clicking the Party item creates the party instantly (configurable) |
| 7 | Everything configurable | Every item, name, lore, slot, title and message lives in `guis.json` / `messages.json` / `settings.json` |
| 8 | Profile head must match current skin | Heads built from the live server GameProfile — SkinsRestorer-compatible |
| 9 | Kit editor in a double chest with save/auto-save | 54-slot editor with Save button **and** auto-save on close (cursor items are absorbed too) |
| 10 | Item slot positions wrong | All slots are config-driven with sane defaults; state-aware hotbar switching |
| 11 | Everything should be GUI, no commands | Complete GUI coverage of every feature (including anvil text input); commands remain as optional extras |
| 12 | Admins must be able to configure every detail | Full config surface + Admin GUI + hot `/pvpadmin reload` |
| 13 | Per-duel scoreboard with players, ping, stats | Packet-based per-duel sidebars with `{ping1}`/`{ping2}`/hits/duration; team mode lists everyone with live ping & hearts |
| 14 | Remove the settings item | The settings hotbar item is gone (no key is even generated anymore) |
| 15 | Party config items after creating a party | Party creation swaps in the “Party (Open Menu)” item; full Party GUI with invite/chat/duel/disband/leave |

Plus this build hardens: 1.21.11 permission API migration (GAMEMASTERS checks), Gson serialization crash on first boot fixed, and a full clean server-boot verification pass.

---

*PvP Core 2.2.0 — built for Minecraft 1.21.11 / Fabric. Server-side only. MIT licensed.*
