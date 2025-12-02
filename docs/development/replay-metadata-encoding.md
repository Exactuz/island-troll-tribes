# Replay Metadata Encoding System

> **Date**: 2025-12-02  
> **Status**: Working (MMD Strategy)  
> **Goal**: Record custom match metadata in WC3 replays for later parsing

---

## Overview

The system embeds match metadata (player names, results, match ID, duration, etc.) into Warcraft 3 replay files using the **w3mmd (Warcraft 3 Map Meta Data)** protocol.

---

## Architecture

### Components

| Component | Location | Purpose |
|-----------|----------|---------|
| **Main Encoder** | `island-troll-tribes/wurst/systems/core/MatchMetadataEncoder.wurst` | Controller, builds metadata, calls strategies |
| **MMD Encoder** | `island-troll-tribes/wurst/systems/core/MatchMetadataMMDEncoder.wurst` | MMD protocol encoder |
| **Config** | `island-troll-tribes/wurst/systems/core/MatchMetadataConfig.wurst` | Constants & character set |
| **MMD Decoder** | `ittweb/scripts/replay-metadata-parser/src/mmd/mmdReader.ts` | Parses w3mmd data from replays |

### External Dependencies

- [WurstMMD library](https://github.com/jlfarris91/WurstMMD) - Located at `_build/dependencies/WurstMMD/`

### Data Flow

```
[Game Ends] → [Build Metadata] → [Serialize] → [MMD Encode] → [Replay File]
                                                                    ↓
[Parse Replay] ← [Extract Payload] ← [Read w3mmd Data] ← [.w3g file]
```

---

## MMD-Based Encoding

### How It Works

**Encoder (Wurst):**
```wurst
import MMD

// Logs custom data that gets embedded in replay
MMD.logCustom("itt_version", MAP_VERSION_STRING)
MMD.logCustom("itt_schema", "1")
MMD.logCustom("itt_data_0", "first chunk of payload...")
MMD.logCustom("itt_data_1", "second chunk...")
MMD.logCustom("itt_chunks", "2")
```

**Decoder (TypeScript):**
```typescript
import { readMMDData } from "./mmd/mmdReader.js";

const result = await readMMDData("replay.w3g");
// result.ittMetadata.payload contains the reconstructed data
```

### Data Format in Replay

MMD uses gamecache sync to embed data. The custom data appears as:
- `itt_version` - Map version string
- `itt_schema` - Schema version number
- `itt_data_N` - Payload chunks (N = 0, 1, 2, ...)
- `itt_chunks` - Total number of chunks

### Payload Format

**Schema v2** (current):
```
v2
mapName:Island Troll Tribes
mapVersion:v3.28
matchId:12345678901234
startTime:0
endTime:300
duration:300
playerCount:2
player:0|PlayerName|HUM|0|WIN|150|45|67|100|5|2|1|0|0|0|0
player:1|OtherPlayer|ORC|1|LOSE|80|30|20|50|3|1|0|1|0|0|0
checksum:123456789
END
```

**Player Line Format (v2)**:
```
player:slot|name|race|team|result|dmg|selfHeal|allyHeal|gold|meat|elk|hawk|snake|wolf|bear|panther
```

| Field | Description |
|-------|-------------|
| slot | Player slot index (0-11) |
| name | Player name |
| race | Race code (HUM, ORC, NE, UD, RN) |
| team | Team/tribe ID |
| result | WIN, LOSE, or LEAVE |
| dmg | Damage dealt to enemy trolls |
| selfHeal | Healing done to self |
| allyHeal | Healing done to allies |
| gold | Gold acquired from selling |
| meat | Cooked meat eaten |
| elk | Elk kills |
| hawk | Hawk kills |
| snake | Snake kills |
| wolf | Wolf kills |
| bear | Bear kills |
| panther | Panther kills |

**Schema v1** (legacy, no stats):
```
player:0|PlayerName|HUM|0|WIN
```

### Character Encoding

```
ENCODE_CHARS = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789:|._- \n"
```

---

## Parser Architecture (ittweb)

The replay parser is located at `ittweb/scripts/replay-metadata-parser/`.

### Directory Structure
```
src/
├── cli.ts                    # Command-line interface
├── decodeReplay.ts           # Main decode orchestrator
├── errors.ts                 # Error types
├── types.ts                  # TypeScript types
├── mmd/                      # MMD-based decoding
│   └── mmdReader.ts
├── payload/
│   └── payloadParser.ts
└── spec/
    └── defaultSpec.ts
```

### CLI Commands
```bash
# MMD-based (recommended)
node dist/cli.js mmd replay.w3g --pretty

# With raw output for debugging
node dist/cli.js mmd replay.w3g --raw --pretty
```

### Building the Parser
```bash
cd ittweb/scripts/replay-metadata-parser
npx tsc -p tsconfig.json
```

### Dependencies
- `w3gjs` v3.0.0 - WC3 replay parser library with w3mmd support

---

## Trigger Conditions

Encoding starts when:
1. A tribe wins the game (`VictoryDefeat.wurst`)
2. Debug command `-eg` is executed (`Commands.wurst`)

---

## Build System Peculiarities ⚠️

### Grill Build Behavior

**IMPORTANT**: The grill build command shows misleading error messages but still produces a valid map.

```powershell
grill build base.w3x
```

**What you'll see:**
```
🔥 Grill warming up..
🔥 Ready. Version: <1.4.0.0-jenkins-WurstSetup-165>
🔨 Building project..
Warning: The provided wc3 path wasn't suitable. Falling back to discovery.
❌ There was an issue with the wurst build process.
```

**Reality:** Despite the `❌` error, the map is usually built successfully!

### Where the Built Map Goes
- **Source map**: `base.w3x` (don't use this for testing)
- **Built map**: `_build/Island.Troll.Tribes.v3.28.w3x` (use this!)

### How to Verify Build Success
```powershell
Get-Item "_build\Island.Troll.Tribes.v3.28.w3x" | Select-Object LastWriteTime
```

Check if `LastWriteTime` is recent (after your code changes).

### Common Build Issues

1. **Missing dependencies folder**
   ```powershell
   grill install
   ```

2. **WurstMMD not cloning properly**
   ```powershell
   # Manual clone if needed
   git clone https://github.com/jlfarris91/WurstMMD.git "_build/dependencies/WurstMMD"
   ```

3. **WC3 path errors** - These warnings can be ignored if the map builds

---

## Debug Commands

### Running the Parser
```powershell
cd C:\Users\user\source\repos\ittweb\scripts\replay-metadata-parser

# MMD-based
node dist/cli.js mmd "path\to\replay.w3g" --raw --pretty
```

### In-Game Commands
- `-eg` - Force end game and trigger metadata encoding

### Building the Map
```powershell
cd C:\Users\user\source\repos\island-troll-tribes
grill build base.w3x

# Verify build succeeded
Get-Item "_build\Island.Troll.Tribes.v3.28.w3x" | Select-Object LastWriteTime
```

---

## Encoder Extension (Adding New Strategies)

The encoder architecture supports multiple strategies. To add a new encoder:

```wurst
class MyEncoderStrategy implements MetadataEncoderStrategy
    override function encode(string payload)
        // Your encoding logic here

init
    registerEncoderStrategy(new MyEncoderStrategy())
```

---

## Key Files Reference

### Encoder (Wurst)
- `wurst/systems/core/MatchMetadataEncoder.wurst` - Main controller & metadata building
- `wurst/systems/core/MatchMetadataMMDEncoder.wurst` - MMD-based encoder
- `wurst/systems/core/MatchMetadataConfig.wurst` - Character set & constants

### Decoder (TypeScript)
- `src/mmd/mmdReader.ts` - MMD-based decoder
- `src/payload/payloadParser.ts` - Parses text payload to structured data
- `src/cli.ts` - Command-line interface

---

## Development History

### Strategies Attempted

1. **Order-Based Encoding** ❌ (Removed)
   - Encoded data as unit order IDs
   - Failed because WC3 replays don't record neutral passive player orders
   - Even with player 0's unit, proved unreliable

2. **Chat-Based Encoding** ❌ (Removed)
   - Encoded data as printed chat messages
   - `print()` in Wurst uses `DisplayTextToForce` which doesn't create actual chat events
   - Replays only record actual player-typed chat

3. **MMD-Based Encoding** ✅ (Current)
   - Uses w3mmd standard protocol via WurstMMD library
   - Reliable, well-established standard
   - Built-in support in w3gjs parser library

### Revision History

| Date | Changes |
|------|---------|
| 2025-12-02 | Initial investigation of order-based approach |
| 2025-12-02 | Confirmed neutral passive orders not recorded |
| 2025-12-02 | Implemented modular encoder architecture |
| 2025-12-02 | Tested order encoder with player 0 (unreliable) |
| 2025-12-02 | Tested chat encoder (display text not recorded) |
| 2025-12-02 | Implemented MMD-based encoder (working!) |
| 2025-12-02 | Removed failed strategies, cleaned up codebase |
| 2025-12-02 | **Schema v2**: Added player stats (damage, healing, kills, gold, meat) |
