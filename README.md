# Astonia 3 Server

A classic 2D isometric MMORPG server written in C. This codebase powers the game world, handling everything from player connections to NPC AI, combat, quests, and multi-server coordination.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Building](#building)
- [Running](#running)
- [Core Concepts](#core-concepts)
- [File Reference](#file-reference)
- [Data Structures](#data-structures)
- [Game Systems](#game-systems)
- [Adding Content](#adding-content)

---

## Architecture Overview

### Multi-Process Design

Astonia 3 uses a **multi-process architecture** where each game area runs as a separate server process:

```
┌─────────────────────────────────────────────────────────────┐
│                      MySQL Database                          │
│         (Characters, Clans, Mail, Persistence)              │
└─────────────────────────────────────────────────────────────┘
        │              │              │              │
   ┌────┴────┐    ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │ Area 1  │    │ Area 2  │   │ Area 3  │   │ Area N  │
   │ Server  │    │ Server  │   │ Server  │   │ Server  │
   │ :2300   │    │ :2301   │   │ :2302   │   │ :23XX   │
   └────┬────┘    └────┬────┘   └────┬────┘   └────┬────┘
        │              │              │              │
        └──────────────┴──────────────┴──────────────┘
                            │
                    ┌───────┴───────┐
                    │ Game Clients  │
                    └───────────────┘
```

When players move between areas, they disconnect from one server and reconnect to another. Character data is saved to and loaded from the shared database.

### Tick-Based Game Loop

The server runs at **24 ticks per second** (approximately 41.67ms per tick). Each tick:

1. `tick_date()` - Update in-game time (day/night cycle)
2. `tick_timer()` - Process scheduled timers
3. `tick_char()` - Update all characters (movement, AI, combat)
4. `tick_effect()` - Update spell effects and projectiles
5. `tick_clan()` - Process clan updates
6. `tick_club()` - Process club (guild) updates
7. `tick_player()` - Update connected players
8. `tick_login()` - Handle pending logins
9. `pflush()` - Flush database writes
10. `io_loop()` - Process network I/O
11. `tick_chat()` - Relay chat messages

---

## Building

### Prerequisites

- GCC compiler
- MySQL client libraries
- zlib (for network compression)
- pthreads

### Compilation

```bash
make clean
make
```

This produces the `server` executable.

---

## Running

### Command Line Options

```bash
./server -a <areaID> [-m <mirror>] [-i <serverID>] [-d] [-c]
```

| Option | Description |
|--------|-------------|
| `-a <areaID>` | **Required.** Area ID this server manages (1-37) |
| `-m <mirror>` | Mirror number for load balancing (default: 0) |
| `-i <serverID>` | Server ID for client IP translation |
| `-d` | Run as daemon (background process) |
| `-c` | Disable concurrent database access |

### Example

```bash
# Start area 1 (starting zone)
./server -a 1

# Start area 3 as a daemon
./server -a 3 -d
```

### Required Files

The server expects these directories:
- `./area<N>/` - Zone definition files for each area
- `./item/` - Item template definitions
- `./char/` - Character template definitions

---

## Core Concepts

### Characters (cn)

Everything that moves in the game world is a **character** - players, NPCs, and monsters. Characters are stored in the global `ch[]` array and referenced by index (`cn`).

```c
// Access character 42
ch[42].name;      // Character name
ch[42].hp;        // Current hit points
ch[42].x, ch[42].y;  // Map position
ch[42].flags;     // CF_PLAYER, CF_WARRIOR, etc.
```

### Items (in)

All objects are **items** - weapons, armor, potions, doors, and world objects. Items are stored in `it[]` and referenced by index (`in`).

```c
// Access item 100
it[100].name;     // Item name
it[100].flags;    // IF_TAKE, IF_USE, IF_WEAPON, etc.
it[100].carried;  // Character carrying this (0 = on ground)
```

### Map Tiles (m)

The world is a 256x256 grid of tiles per area. Access via `map[x + y * MAXMAP]`.

```c
// Check if tile is blocked
if (map[x + y * MAXMAP].flags & MF_MOVEBLOCK) {
    // Cannot walk here
}
```

### Effects (fn)

Spells, projectiles, and visual effects use the `ef[]` array.

### Drivers

**Drivers** provide behavior for NPCs and items:
- **Character Drivers (CDR_*)**: Control NPC AI
- **Item Drivers (IDR_*)**: Handle item special behaviors

---

## File Reference

### Core Server

| File | Purpose |
|------|---------|
| `server.h` | Core definitions: structures, flags, constants |
| `server.c` | Main entry point, game loop, initialization |
| `create.c` | Character/item creation, zone loading |
| `create.h` | Creation API |

### Player System

| File | Purpose |
|------|---------|
| `player.h` | Player connection structure |
| `player.c` | Network handling, client communication |
| `player_driver.c` | Player action processing |
| `io.c` | Low-level network I/O |

### Database

| File | Purpose |
|------|---------|
| `database.h` | Database API |
| `database.c` | MySQL queries, character persistence |

### Combat & Skills

| File | Purpose |
|------|---------|
| `skill.h` / `skill.c` | Skill definitions, experience costs |
| `do.c` | Action execution (attack, cast, use) |
| `effect.h` / `effect.c` | Spell effects, projectiles |
| `death.c` | Death handling, respawn |
| `fight.c` | Combat calculations |

### NPC AI

| File | Purpose |
|------|---------|
| `drvlib.h` / `drvlib.c` | Driver helper functions |
| `libload.c` | Driver registration |
| `talk.c` | NPC dialogue system |
| `lostcon.c` | Disconnected player handling |
| `strategy.c` | Strategic AI behaviors |

### Items

| File | Purpose |
|------|---------|
| `item_id.h` | Item template IDs |
| `container.c` | Container/inventory logic |
| `depot.c` | Player bank storage |
| `store.c` | NPC merchant shops |

### World

| File | Purpose |
|------|---------|
| `map.c` | Map loading, pathfinding |
| `area.c` / `area.h` | Area definitions |
| `light.c` | Lighting calculations |
| `sector.c` | Spatial partitioning |
| `date.c` | In-game time, day/night |

### Social Systems

| File | Purpose |
|------|---------|
| `chat.c` / `chat.h` | Chat channels |
| `clan.c` / `clan.h` | Clan/guild system |
| `club.c` / `club.h` | Club system |
| `tell.c` | Private messages |

### Utilities

| File | Purpose |
|------|---------|
| `mem.c` / `mem.h` | Custom memory allocator |
| `log.c` / `log.h` | Logging system |
| `tool.c` | Utility functions |
| `timer.c` | Scheduled events |

### Area-Specific Code

| File | Purpose |
|------|---------|
| `area1.c` - `area18.c` | Area-specific NPCs and quests |
| `palace.c` | Palace dungeon |
| `dungeon.c` | Generic dungeon system |
| `lab*.c` | Labyrinth areas |

---

## Data Structures

### Character Structure (`struct character`)

```c
struct character {
    unsigned long long flags;    // CF_PLAYER, CF_WARRIOR, etc.
    char name[40];               // Display name
    unsigned int ID;             // Database ID

    // Stats
    short value[2][V_MAX];       // [0]=total, [1]=base
    int hp, mana, endurance;     // Current pools

    // Position
    unsigned short x, y;         // Current position
    unsigned short tox, toy;     // Target position

    // Inventory
    unsigned int item[110];      // Equipment + inventory
    unsigned int gold;           // Currency

    // AI
    unsigned short driver;       // Driver number
    struct data *dat;            // Driver data
};
```

### Item Structure (`struct item`)

```c
struct item {
    unsigned long long flags;    // IF_TAKE, IF_WEAPON, etc.
    char name[40];

    unsigned int value;          // Gold value
    signed short mod_index[5];   // Stat modifier indices
    signed short mod_value[5];   // Stat modifier values

    // Location (mutually exclusive)
    unsigned short x, y;         // On ground
    unsigned short carried;      // Carried by character
    unsigned short contained;    // In container

    unsigned short driver;       // Item driver
};
```

### Map Tile (`struct map`)

```c
struct map {
    unsigned int gsprite;        // Ground sprite
    unsigned int fsprite;        // Foreground sprite
    unsigned short ch;           // Character on tile
    unsigned short it;           // Item on tile
    unsigned int flags;          // MF_MOVEBLOCK, etc.
};
```

---

## Game Systems

### Character Classes

| Class | Flags | Description |
|-------|-------|-------------|
| Warrior | `CF_WARRIOR` | Physical combat focus |
| Mage | `CF_MAGE` | Magic focus |
| Seyan'Du | `CF_WARRIOR \| CF_MAGE` | Hybrid class |
| Arch-* | `CF_ARCH` | Enhanced version |

### Skills (V_* constants)

**Attributes:** `V_WIS`, `V_INT`, `V_AGI`, `V_STR`

**Pools:** `V_HP`, `V_MANA`, `V_ENDURANCE`

**Weapon Skills:** `V_SWORD`, `V_DAGGER`, `V_STAFF`, `V_TWOHAND`, `V_HAND`

**Combat Skills:** `V_ATTACK`, `V_PARRY`, `V_TACTICS`, `V_WARCRY`

**Spells:** `V_BLESS`, `V_HEAL`, `V_FREEZE`, `V_FIREBALL`, `V_FLASH`

### Equipment Slots (WN_* constants)

```
WN_HEAD (1)     - Helmet
WN_NECK (0)     - Amulet
WN_BODY (4)     - Armor
WN_ARMS (3)     - Bracers
WN_BELT (5)     - Belt
WN_LEGS (7)     - Leggings
WN_FEET (10)    - Boots
WN_RHAND (6)    - Weapon
WN_LHAND (8)    - Shield
WN_CLOAK (2)    - Cloak
WN_LRING (11)   - Left ring
WN_RRING (9)    - Right ring
```

### Clan Relations

| State | Code | Description |
|-------|------|-------------|
| Alliance | `CS_ALLIANCE` | Cannot attack, 24h to break |
| Peace | `CS_PEACETREATY` | Cannot attack, 24h to break |
| Neutral | `CS_NEUTRAL` | Cannot attack, 1h to war |
| War | `CS_WAR` | Can attack in clan areas |
| Feud | `CS_FEUD` | Can attack anywhere |

---

## Adding Content

### Creating an NPC

1. Define in zone file (`area<N>/*.npc`):
```
name=Guard Captain
sprite=12345
group=1
flags=CF_RESPAWN,CF_MALE
driver=CDR_SIMPLEBADDY
hp=100
str=20
```

2. If custom behavior needed, create driver in `drvlib.h`:
```c
#define CDR_MYDRIVER    200
```

3. Implement driver and register in `libload.c`

### Creating an Item

1. Define template in `item/*.item`:
```
name=Iron Sword
sprite=50000
flags=IF_TAKE,IF_USE,IF_WNRHAND,IF_SWORD
value=100
mod=V_WEAPON,10
```

2. For special behavior, assign driver:
```
driver=IDR_MYITEM
```

### Creating a Quest

1. Create driver data ID in `drdata.h`:
```c
#define DRD_MYQUEST_PPD  MAKE_DRD(DEV_ID_DB, 100 | PERSISTENT_PLAYER_DATA)
```

2. Create quest data structure
3. Implement NPC driver that manages quest state
4. Use `set_data()`/`get_data()` to track progress

---

## Important Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `TICKS` | 24 | Ticks per second |
| `MAXMAP` | 256 | Map dimensions |
| `MAXCHARS` | varies | Max characters per area |
| `MAXITEM` | varies | Max items per area |
| `POWERSCALE` | 1000 | HP/Mana multiplier |
| `INVENTORYSIZE` | 110 | Player inventory slots |

---

## License

Original code (c) 2001-2008 D. Brockhaus

---

## See Also

- Individual file headers contain detailed documentation
- `drvlib.h` - Complete list of NPC and item drivers
- `drdata.h` - Driver data block IDs
- `server.h` - All flags and constants
