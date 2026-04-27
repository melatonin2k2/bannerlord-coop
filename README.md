# BannerlordCoop — LAN Co-op Mod for Mount & Blade II: Bannerlord

Lets multiple players play the singleplayer campaign together over LAN.
Each player controls their own hero/party and can trade, fight, recruit, and
build their kingdom as normal on the **same living world** that updates for
everyone in real time.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  HOST MACHINE                                                │
│  - Runs the campaign simulation normally                     │
│  - CoopServer listens on UDP port 47770                      │
│  - Broadcasts party positions every 0.5 s (unreliable UDP)   │
│  - Broadcasts world-state snapshot every 2 s (reliable)      │
│  - Broadcasts time ticks every in-game hour (reliable)       │
│  - Streams the save file to each joining client              │
│  - Receives and validates player action requests             │
└───────────────────────┬─────────────────────────────────────┘
                        │  LiteNetLib UDP
          ┌─────────────┴────────────┐
          │                          │
┌─────────▼──────────┐  ┌───────────▼────────────┐
│  CLIENT A           │  │  CLIENT B               │
│  - Loads host save  │  │  - Loads host save       │
│  - Applies position │  │  - Applies position      │
│    / settlement     │  │    / settlement          │
│    updates          │  │    updates               │
│  - Sends actions    │  │  - Sends actions         │
│    to host          │  │    to host               │
└─────────────────────┘  └──────────────────────────┘
```

### What the host drives (authoritative)
- Campaign time advancement
- All NPC party movement and AI
- Settlement economy (prosperity, food, loyalty, security)
- Garrison sizes and wages
- Diplomacy (war/peace between kingdoms)
- Battle resolution
- Market prices

### What each client drives locally
- Their own hero / party movement (sent to host, host validates)
- Their own battles (UI runs locally; result is synced back)
- Their own trade / recruit menus (result sent to host, synced)

---

## How It Works Step-by-Step

1. **Host** starts a new campaign or loads a save as normal.
2. Host presses **F10** on the campaign map → "Host Game".
3. **Clients** launch Bannerlord with this mod enabled.
4. Clients press **F10** on the main menu (before loading any campaign) → "Join Game" → enter the host's LAN IP.
5. The host's current save is transferred to the client (~10–50 MB).
6. Client automatically loads the save.
7. All players appear on the world map as independent parties.
8. The world (settlements, armies, time) is kept in sync by the host.

---

## Building from Source

### Prerequisites
- Visual Studio 2022 or Rider
- .NET Framework 4.7.2 targeting pack
- Mount & Blade II: Bannerlord installed

### External DLLs (place in `BannerlordCoop/lib/`)
Download and copy these into the `lib/` folder before building:

| File | Source |
|---|---|
| `0Harmony.dll` | [Bannerlord.Harmony NexusMods](https://www.nexusmods.com/mountandblade2bannerlord/mods/2006) |
| `LiteNetLib.dll` | [LiteNetLib GitHub releases](https://github.com/RevenantX/LiteNetLib/releases) |

### Build steps

```bash
# 1. Clone
git clone <repo-url>
cd bannerlord-coop/BannerlordCoop

# 2. Put DLLs in lib/ (see table above)

# 3. Build (adjust BannerlordPath if needed)
dotnet build -c Release /p:BannerlordPath="C:\Program Files (x86)\Steam\steamapps\common\Mount & Blade II Bannerlord"
```

The post-build target automatically copies the output to:
```
<BannerlordPath>\Modules\BannerlordCoop\
```

### Manual module setup (if post-build fails)
```
<BannerlordPath>\Modules\BannerlordCoop\
├── SubModule.xml
└── bin\Win64_Shipping_Client\
    ├── BannerlordCoop.dll
    ├── 0Harmony.dll
    └── LiteNetLib.dll
```

---

## Playing

### Host
1. Enable the mod in the Bannerlord launcher.
2. Start a singleplayer campaign.
3. Press **F10** on the campaign map.
4. Choose **Host Game**.
5. Share your LAN IP (e.g. `192.168.1.x`) with players on the same network.

### Client
1. Enable the mod in the Bannerlord launcher.
2. Go to the **main menu** (do NOT load a campaign).
3. Press **F10**.
4. Choose **Join Game** → enter the host's IP.
5. Wait for the save to transfer (progress shown in the message log).
6. The campaign loads automatically — you are now in the shared world.

### In-game chat
Type `/` to open a chat input (future UI; currently messages appear in the info log).

### Disconnect
Press **F10** → **Disconnect** (or close the game).

---

## Packet Reference

| Packet | Direction | Delivery | Description |
|--------|-----------|----------|-------------|
| `PlayerJoined` | S→C | Reliable | New player connected |
| `PlayerLeft` | S→C | Reliable | Player disconnected |
| `WorldStateFull` | S→C | Reliable | Full snapshot (all parties + settlements) |
| `PartyPositions` | S→C | Unreliable | Position heartbeat for all parties |
| `SettlementUpdate` | S→C | Reliable | Delta for one or more settlements |
| `TimeTick` | S→C | Reliable | Advance campaign clock to host's value |
| `DiplomacyChange` | S→C | Reliable | War declared or peace made |
| `BattleStarted` | S→C | Reliable | Battle engagement notification |
| `BattleEnded` | S→C | Reliable | Battle result summary |
| `ActionResult` | S→C | Reliable | Approve/reject a client action request |
| `ClientPartyPosition` | C→S | Unreliable | Client's own party position heartbeat |
| `RequestMove` | C→S | Reliable | Move player party to position |
| `RequestEnterSettlement` | C→S | Reliable | Enter town/castle/village |
| `RequestAttack` | C→S | Reliable | Attack another party |
| `RequestTrade` | C→S | Reliable | Trade item with settlement |
| `RequestRecruit` | C→S | Reliable | Recruit troops |
| `RequestDiplomacy` | C→S | Reliable | Diplomacy action |
| `ChatMessage` | Both | Reliable | In-game chat |
| `FileTransferBegin` | S→C | Reliable | Save-file header |
| `FileChunk` | S→C | Reliable | 64 KB save-file chunk |
| `FileTransferEnd` | S→C | Reliable | All chunks sent |

---

## Known Limitations & Roadmap

- **Battles**: Client battles currently rely on Bannerlord's local simulation.
  A future version will synchronise battle results through the host.

- **Inventory sync**: Item prices and trade profits sync via world-state but
  detailed inventory is not yet streamed. Client trade is optimistically applied.

- **Multiple clans**: Each client controls their own clan. Clan-level economies
  (workshops, caravans) are not yet individually synced.

- **Save on client**: Clients can save their own state; the save will include
  the full world state as of the last sync.

- **Chat UI**: Currently uses the info log overlay. A proper in-game chat box
  is planned for the next milestone.

- **Siege UI**: Siege management menus work on the host. Client siege
  participation is proxied through the host.

---

## Contributing

PRs welcome. Key areas needing work:
- Full Gauntlet UI panel replacing the InquiryData dialogs
- Battle result synchronisation
- Caravan / workshop sync per player
- Support for more than 2 players (tested with 2, designed for up to 8)

---

## Credits & Prior Art

This mod builds on lessons from:
- [BannerlordCoop by Bannerlord-Coop-Team](https://github.com/Bannerlord-Coop-Team/BannerlordCoop)
- [mountandblade-coop by Andreas1331](https://github.com/Andreas1331/mountandblade-coop)
- [LiteNetLib](https://github.com/RevenantX/LiteNetLib)
- [Harmony](https://github.com/pardeike/Harmony)
