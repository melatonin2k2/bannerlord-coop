# BannerlordCoop — LAN Co-op Mod for Mount & Blade II: Bannerlord

Lets multiple players play the singleplayer campaign together over LAN.
Each player controls their own hero/party and can trade, fight, recruit, and
build their kingdom as normal — on the **same living world** that updates for
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
| `0Harmony.dll` | [Bannerlord.Harmony NexusMods](https://www.nexusmods.com/mountandblade2bannerlord/mods/2006) or NuGet `Lib.Harmony` |
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
Press **T** to open a chat input. Press **~** (tilde / backtick) to toggle the
persistent chat panel. Slash commands are also routed through the input:
`/trade`, `/buyworkshop`, `/foundcaravan`, `/siege`, `/joinbattle`, `/defend`,
`/flee`.

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
| `RequestTrade` | C→S | Reliable | Acquire merchant lock for trade screen |
| `RequestTradeApply` | C→S | Reliable | Submit a content-validated trade transaction |
| `RequestRecruit` | C→S | Reliable | Recruit troops |
| `RequestDiplomacy` | C→S | Reliable | Diplomacy action |
| `RequestPurchaseWorkshop` | C→S | Reliable | Buy a workshop in a settlement |
| `RequestFoundCaravan` | C→S | Reliable | Found a caravan led by a companion |
| `RequestSiegeAction` | C→S | Reliable | Drive siege menu (build engine / tactic / storm / sally) |
| `BattleJoinOffer` | S→C | Reliable | Adjacent peer invited to a joint battle |
| `BattleJoinResponse` | C→S | Reliable | Reply to `BattleJoinOffer` (Ally/Opp/Decline) |
| `BattleUpgradedToJoint` | S→C | Reliable | Battle promoted to a shared joint mission |
| `JointBattleLootAssigned` | S→C | Reliable | Per-participant loot/gold/prisoner allocation |
| `AgentStateBatch` | P↔P | Unreliable | 30 Hz battle-host → guests agent state stream |
| `HeroInputBatch` | P↔P | Reliable | Guest → battle-host hero input frame stream |
| `AgentDeath` | P↔P | Reliable | Authoritative agent death from battle-host |
| `ChatMessage` | Both | Reliable | In-game chat |
| `FileTransferBegin` | S→C | Reliable | Save-file header |
| `FileChunk` | S→C | Reliable | 64 KB save-file chunk |
| `FileTransferEnd` | S→C | Reliable | All chunks sent |

---

## Known Limitations & Roadmap

- **Battles**: Solo battles resolve locally on the participating peer and
  broadcast the result; joint battles (host + adjacent peers within a 64-unit
  coalesce radius) enter a shared mission with battle-host authority over
  agent state, hero input, and damage. Outcomes reconcile at the campaign
  host, with the battle-host's report canonical.

- **Inventory & trade**: Item rosters sync host→client via hourly
  `InventoryDelta`/`RosterUpdate`. Trade transactions are content-validated
  by the host (`RequestTradeApply`): the client sends the proposed
  `(item, count, side, price)` lines via the `/trade` slash command; the host
  validates funds and stock atomically and confirms or toasts a rejection.

- **Multiple clans / clan economies**: Each client controls their own clan.
  Workshops and caravans sync host→client (`WorkshopUpdate` / `CaravanUpdate`).
  Clients can buy workshops and found caravans via the `/buyworkshop` and
  `/foundcaravan` slash commands (host validates clan funds, ownership, and
  faction relations); a proper menu integration is a planned follow-up.

- **Save on client**: Clients can save their own state. The save includes
  the full world state as of the last sync. Host-pushed saves load via
  `MBSaveLoad → SandBoxGameManager → MBGameManager.StartNewGame`.

- **Chat UI**: Persistent in-game chat panel (`CoopChatPanel`, Gauntlet-based)
  with `InformationManager` overlay fallback. The Gauntlet movie XML is not
  yet shipped — the panel is wired and renders silently in `try/catch` until
  the layout drops in; chat lines remain visible via the overlay fallback.
  Toggle visibility with the `~` (tilde / backtick) key; `T` opens the inline
  input.

- **Siege**: Siege battles flow through the joint-battle path (the 64-unit
  coalesce rule covers besieger leader and garrison co-located at the
  settlement). Siege management actions (build engines, assign tactic, wait,
  pull back, storm, sally out) are proxied to the host via the
  `/siege <action> <partyId> <settlementId> [arg]` slash command and
  `RequestSiegeAction` packet. Some apply paths (engine construction, tactic
  assignment, storm, sally) probe Bannerlord 1.3.15 internals via reflection
  and may reject as "unsupported on this build" if the surface differs;
  pull-back and wait are universally supported.

---

## Contributing

PRs welcome. Key areas needing work:
- `CoopChatPanel.xml` Gauntlet movie file (the C# wiring is shipped; the
  layout is the missing piece — until then the chat panel falls back to the
  `InformationManager` overlay).
- Replacing slash commands with proper menu integration for `/trade`,
  `/buyworkshop`, `/foundcaravan`, and `/siege`.
- Full Bannerlord 1.3.15 surface coverage for the reflection-probed siege
  apply helpers (engine construction, tactic assignment, storm, sally).
- Support for more than 2 players (tested with 2, designed for up to 8).

---

## Credits & Prior Art

This mod builds on lessons from:
- [BannerlordCoop by Bannerlord-Coop-Team](https://github.com/Bannerlord-Coop-Team/BannerlordCoop)
- [mountandblade-coop by Andreas1331](https://github.com/Andreas1331/mountandblade-coop)
- [LiteNetLib](https://github.com/RevenantX/LiteNetLib)
- [Harmony](https://github.com/pardeike/Harmony)
