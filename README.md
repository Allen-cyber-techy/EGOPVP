<p align="center">
  <img src="EGOPVP LOGO/egopvp_logo.png" alt="EGOPVP Logo" width="200"/>
</p>

<h1 align="center">EGOPVP — FiveM Competitive PvP Server</h1>

<p align="center">
  <em>A custom-engineered multiplayer PvP practice server built on the FiveM platform, featuring ranked matchmaking, persistent clan hierarchies, and a fully server-authoritative economy — designed and developed as a solo full-stack project.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Framework-QB--Core-blue?style=flat-square" alt="QB-Core"/>
  <img src="https://img.shields.io/badge/Language-Lua-purple?style=flat-square" alt="Lua"/>
  <img src="https://img.shields.io/badge/Frontend-NUI%20(HTML%2FCSS%2FJS)-orange?style=flat-square" alt="NUI"/>
  <img src="https://img.shields.io/badge/Database-MySQL%20(XAMPP)-green?style=flat-square" alt="MySQL"/>
  <img src="https://img.shields.io/badge/Status-Private%20Repository-red?style=flat-square" alt="Private"/>
</p>

---

> **Note:** This repository is **private** to protect proprietary game logic and anti-exploit architecture. The documentation below provides a high-level technical overview of the systems I designed and implemented. Database schemas and architectural decisions are described in detail to demonstrate competency in data modelling, server-side programming, and systems design.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Technology Stack](#technology-stack)
- [System Architecture](#system-architecture)
- [Core Systems](#core-systems)
  - [Ranking & Elo System](#1-ranking--elo-system)
  - [Game Mode Engine](#2-game-mode-engine)
  - [Persistent Clan System](#3-persistent-clan-system)
  - [Battlepass & In-Game Store](#4-battlepass--in-game-store)
  - [Statistics & Leaderboard Pipeline](#5-statistics--leaderboard-pipeline)
- [Custom NUI Layer](#custom-nui-layer)
- [Technical Challenges & Solutions](#technical-challenges--solutions)
- [Database Architecture](#database-architecture)
- [Project Structure](#project-structure)

---

## Project Overview

EGOPVP is a competitive FiveM PvP server that I architected and developed from scratch as a solo developer. The server supports concurrent players across three distinct game modes with real-time stat tracking, ranked progression, and persistent clan-based social structures.

Unlike typical FiveM roleplay servers that rely on pre-built community scripts, every core gameplay system in EGOPVP is **custom-coded** — from the matchmaking lobby pipeline to the Elo-based ranking algorithm to the server-authoritative economy that prevents client-side exploits.

**Key Metrics:**
- **15+ custom resource modules** designed and implemented from scratch
- **10+ normalized database tables** across a relational MariaDB schema
- **3 concurrent game modes** with independent state machines and lifecycle management
- **Full client-server NUI integration** for all player-facing interfaces

---

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Game Framework** | QB-Core (Lua) | Server-side resource framework, player identity management, and event bus |
| **Scripting Language** | Lua 5.4 | All server-side and client-side game logic |
| **Database** | MySQL (InnoDB) via XAMPP | Persistent storage with ACID-compliant transactions and indexed queries |
| **Frontend (NUI)** | HTML5 / CSS3 / Vanilla JS | In-game HUD, menus, leaderboards, and clan management UI |
| **Networking** | FiveM Native Event System | Client↔Server RPC, entity synchronization, and state replication |
| **Version Control** | Git | Full history of iterative development |

---

## System Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        FiveM Game Client                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  Client Lua   │  │  NUI Layer   │  │   Game Renderer (GTA V)  │ │
│  │  (Input, Sync)│  │  (HTML/CSS/JS)│  │   (Entity Management)   │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────────────────────┘ │
└─────────┼─────────────────┼───────────────────────────────────────┘
          │ TriggerServerEvent│ SendNUIMessage / NUI Callbacks
          ▼                  ▼
┌────────────────────────────────────────────────────────────────────┐
│                       FiveM Server Runtime                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  Game Mode    │  │  Economy &   │  │  Stats, Rankings &       │ │
│  │  Controllers  │  │  Store Logic │  │  Leaderboard Engine      │ │
│  │  (DM/TB/OW)   │  │  (Server-Auth)│  │  (Elo, Snapshots)       │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────────┘ │
│         │                  │                     │                 │
│  ┌──────┴──────────────────┴─────────────────────┴───────────────┐ │
│  │                   QB-Core Framework Layer                      │ │
│  │         (Player Management, Shared Objects, Events)            │ │
│  └──────────────────────────┬────────────────────────────────────┘ │
└─────────────────────────────┼─────────────────────────────────────┘
                              │ oxmysql / Prepared Statements
                              ▼
                 ┌────────────────────────┐
                 │    MySQL (InnoDB)      │
                 │   10+ Indexed Tables   │
                 │   ACID Transactions    │
                 └────────────────────────┘
```

---

## Core Systems

### 1. Ranking & Elo System

The competitive ranking system implements a **modified Elo algorithm** that awards or deducts rank points (`rank_points`) based on per-match performance rather than a simple win/loss binary.

**XP Calculation Factors:**
- Kill count, death count, and K/D ratio within the match
- Headshot accuracy (`headshot_rate`)
- Damage dealt vs. damage taken differential
- Opponent rank tier delta (higher-ranked opponents yield more points)
- Win/loss streaks (`current_win_streak`, `best_win_streak`) as score multipliers

**Rank Tier Progression:**
Players ascend through discrete rank tiers (`rank_tier`) with associated point thresholds. The system tracks both current and historical peak ranks (`peak_rank_tier`, `peak_rank_points`) per player and per game mode, enabling season-end reward distribution based on highest attained rank.

**Seasonal Architecture:**
Rank data is seasonally partitioned — at the end of each competitive season, final standings are archived into `ego_season_stats` and player ranks are soft-reset, preserving historical performance data for longitudinal analysis.

---

### 2. Game Mode Engine

Each game mode operates as an independent **finite state machine** with its own lifecycle, spawn logic, scoring rules, and win conditions, all managed through custom server-side event handlers.

| Mode | Description | Key Technical Characteristics |
|---|---|---|
| **Deathmatch** | Free-for-all arena combat | Per-player score tracking, dynamic spawn-point rotation to prevent spawn-camping, configurable kill limits and time bounds |
| **Team Battle** | Team-based objective combat | Team assignment and balancing algorithms, shared team score aggregation, synchronized round transitions |
| **Open War** | Large-scale multi-team warfare | Zone-based territorial control, team selection lobby with real-time roster display, optimized entity syncing for high player counts (see [Technical Challenges](#technical-challenges--solutions)) |

**Match Lifecycle:**
```
IDLE → LOBBY (player queue) → LOADING (map/entity setup) → ACTIVE (gameplay) → SCORING (stat commit) → CLEANUP → IDLE
```

All game mode transitions are managed server-side to prevent client-side state desynchronization. Player spawn protection is implemented as a timed invulnerability window with a visual ghost effect visible to all clients.

---

### 3. Persistent Clan System

The clan system provides a full social layer with **role-based access control (RBAC)**, persistent shared statistics, and a hierarchical management structure — all backed by a normalized relational schema.

**Data Model:**
- **`clans`** — Core clan entity with metadata (name, tag, description, logo), leader reference, and aggregate stats (`total_kills`, `total_wins`, `level`, `xp`)
- **`clan_members`** — Junction table linking players to clans with role-based permissions (`leader`, `officer`, `member`)
- **`clan_invites`** — Bidirectional invitation/request system with uniqueness constraints

**Access Control Matrix:**

| Permission | Leader | Officer | Member |
|---|:---:|:---:|:---:|
| Invite Players | ✓ | ✓ | ✗ |
| Kick Members | ✓ | ✓ | ✗ |
| Promote / Demote | ✓ | ✗ | ✗ |
| Edit Settings | ✓ | ✗ | ✗ |
| Accept Join Requests | ✓ | ✓ | ✗ |
| Disband Clan | ✓ | ✗ | ✗ |

**Clan Progression:** Clans earn XP through member activity (10 XP per kill, 50 XP per win) aggregated at the clan level, advancing through a level system gated by cumulative XP thresholds.

---

### 4. Battlepass & In-Game Store

Both the battlepass and the in-game store operate under a **server-authoritative model** — all currency transactions, item grants, and progression unlocks are validated and executed exclusively on the server. The client never directly modifies economy state.

**Anti-Exploit Design:**
- Currency balances are stored server-side in MariaDB and never exposed to the client as mutable state
- All purchase requests undergo server-side validation (sufficient balance, item availability, duplicate purchase checks)
- Battlepass tier progression is computed from server-tracked match completions and challenge progress, not client-reported values
- Transaction atomicity is ensured via database-level operations to prevent race conditions during concurrent purchases

**Economy Flow:**
```
Client Request → Server Validation → DB Transaction (atomic) → Inventory Update → Client Confirmation
```

---

### 5. Statistics & Leaderboard Pipeline

The statistics engine spans **7 interconnected database tables** to support real-time tracking, historical analysis, and multi-dimensional leaderboard queries.

**Table Architecture:**

| Table | Purpose | Key Indexes |
|---|---|---|
| `ego_player_stats` | Aggregate lifetime stats per player | `rank_points DESC`, `kd_ratio DESC`, `total_kills DESC` |
| `ego_player_mode_stats` | Per-mode stat partitioning (DM, TB, OW) | `(mode, rank_points DESC)` |
| `ego_stats_snapshots` | Daily stat deltas for time-series leaderboards | `(citizenid, snapshot_date)` |
| `ego_match_history` | Per-match granular records with MVP flags | `match_id`, `created_at DESC` |
| `ego_player_achievements` | Unlockable milestones with claim tracking | `(citizenid, achievement_id)` |
| `ego_seasons` | Season metadata and lifecycle management | `is_active` |
| `ego_season_stats` | Archived per-season final standings | `(season_id, citizenid)` |

**Computed Columns:** Ratios such as `kd_ratio` and `win_rate` are recomputed on every stat mutation rather than calculated at query time, enabling O(1) leaderboard reads via pre-sorted indexes.

---

## Custom NUI Layer

All player-facing interfaces are implemented as **custom NUI (Native UI) panels** — web pages rendered inside the game client via Chromium Embedded Framework and communicating with Lua scripts through FiveM's NUI messaging API.

**Implemented Interfaces:**
- **Mega Menu** (`ego-megamenu`) — Central hub for all player actions: matchmaking, stats, clan management, store, and settings
- **PvP HUD** (`ego-pvphud`) — Real-time heads-up display showing match state, kill feed, scores, and timers
- **Clan Management Panel** (`ego-clans/html`) — Full CRUD interface for clan operations with role-aware UI rendering
- **Leaderboard UI** — Sortable, filterable leaderboards across multiple stat dimensions and time windows
- **Loading Screen** (`qb-loading`) — Custom branded loading experience
- **Character Selection** (`qb-multicharacter`) — Multi-character lobby interface

**Communication Pattern:**
```
NUI (JS) ──SendNUIMessage──▶ Client Lua ──TriggerServerEvent──▶ Server Lua ──SQL──▶ MariaDB
                                                                      │
NUI (JS) ◀──SetNuiFocus/JS── Client Lua ◀──TriggerClientEvent──── Server Lua
```

---

## Technical Challenges & Solutions

### 1. Entity Synchronization in Large-Scale Open War

**Problem:** FiveM's default entity synchronization (OneSync) creates significant bandwidth overhead during high-player-count Open War matches, leading to rubber-banding and desynchronization when 20+ players operate in close proximity.

**Solution:**
- Implemented **distance-based entity culling** — players beyond a configurable radius are de-prioritized from the sync scope, reducing per-tick network payload
- Applied **server-side position batching** — instead of relying on per-frame native sync, critical position data for combat calculations is batched and broadcast at a reduced tick rate while maintaining client-side interpolation for visual smoothness
- Optimized **weapon damage processing** server-side to avoid duplicate hit registration caused by sync latency, using authoritative damage validation with a server-side hit queue

### 2. Race Condition Prevention in Economy Transactions

**Problem:** Concurrent purchase requests (e.g., during store sales or battlepass tier claims) risked double-spending if two requests were processed before either committed to the database.

**Solution:** All currency mutations are wrapped in **atomic database operations** using MySQL's transactional guarantees (InnoDB engine). Balance checks and deductions occur within a single prepared statement execution, eliminating the read-modify-write race window.

### 3. Spawn Protection Synchronization

**Problem:** Ghost-effect spawn protection needed to be visually synchronized across all clients while remaining authoritatively timed on the server.

**Solution:** Server broadcasts the protection state and expiry timestamp; each client independently renders the ghost visual effect and removes it based on the server-provided timestamp rather than a local timer, preventing desync between visual state and actual invulnerability.

---

## Database Architecture

The complete database schema comprises **10+ tables** across a single MySQL (InnoDB) database managed via XAMPP, designed with the following principles:

- **Normalization:** Data is normalized to 3NF to eliminate redundancy (e.g., mode-specific stats are partitioned into `ego_player_mode_stats` rather than denormalized columns)
- **Indexing Strategy:** Composite and descending indexes on frequently queried columns (`rank_points DESC`, `(mode, rank_points DESC)`) to support O(log n) leaderboard queries
- **Referential Integrity:** Foreign key constraints with `ON DELETE CASCADE` ensure data consistency across clan-related tables
- **Temporal Data:** `created_at` / `updated_at` timestamps on all mutable tables with `ON UPDATE CURRENT_TIMESTAMP` triggers for automatic audit trails
- **Uniqueness Constraints:** Compound unique keys (e.g., `(citizenid, snapshot_date)`, `(citizenid, achievement_id)`) prevent duplicate records at the database level

```
┌─────────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│  ego_player_stats│────▶│ ego_player_mode_stats │     │   ego_seasons    │
│  (lifetime agg.) │     │  (per-mode breakout)  │     │ (season config)  │
└────────┬────────┘     └──────────────────────┘     └────────┬─────────┘
         │                                                     │
         ▼                                                     ▼
┌─────────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│ego_stats_snapshot│     │  ego_match_history    │     │ ego_season_stats │
│ (daily deltas)  │     │  (per-match records)  │     │ (archived ranks) │
└─────────────────┘     └──────────────────────┘     └──────────────────┘

┌─────────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│     clans       │────▶│    clan_members       │     │   clan_invites   │
│ (clan entities) │     │  (player ↔ clan)      │     │ (invite/request) │
└─────────────────┘     └──────────────────────┘     └──────────────────┘
```

---

## Project Structure

```
resources/
├── [qbxx]/                          # Core custom modules
│   ├── qb-core/                     # QB-Core framework (modified)
│   ├── ego-pvphud/                  # Custom PvP HUD (NUI)
│   ├── ego-megamenu/                # Central navigation menu (NUI)
│   ├── ego-openwar/                 # Open War game mode controller
│   ├── ego-teambattle/              # Team Battle game mode controller
│   ├── ego-stats/                   # Statistics engine & leaderboards
│   ├── ego-clans/                   # Clan system with RBAC & NUI
│   ├── ego-battlepass/              # Seasonal battlepass progression
│   ├── ego-shop/                    # In-game store (server-authoritative)
│   ├── ego-economy/                 # Currency & transaction management
│   ├── ego-customization/           # Player cosmetic customization
│   ├── ego-nametags/                # Overhead player name rendering
│   ├── ego-announcements/           # Server-wide broadcast system
│   ├── ego-admin/                   # Admin tooling & moderation
│   ├── ego-discord/                 # Discord integration bridge
│   └── qb-pvparena/                 # Deathmatch arena controller
├── [lobby]/
│   └── rex_egolobby/                # Lobby & matchmaking pipeline
├── [killfeed]/
│   └── an_killfeed/                 # Real-time kill feed overlay
├── [voice]/                         # Proximity voice chat integration
├── [standalone]/                    # Utility & dependency resources
└── [OX]/
    └── ox_inventory/                # Inventory management layer
```

---

<p align="center">
  <strong>This repository is private to protect proprietary game logic and anti-exploit systems.</strong><br/>
  The systems documented above were designed, architected, and implemented independently as a demonstration of competency in full-stack systems programming, relational database design, client-server networking, and real-time multiplayer game development.
</p>

<p align="center">
  <em>Developed by Allen — 2025–2026</em>
</p>
