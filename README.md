# DIY Laser Tag System — Master Design Document

A modular, open-source laser tag system built around the ESP32-S3 microcontroller. Designed to replace the internal circuitry of widely-available commercial laser tag blasters, using a custom PCB that matches the original form factor. Because many manufacturers license the same blaster hardware, a single PCB design provides wide compatibility across brands.

The system scales from two players in a backyard to a full arena with spectator displays — with no required infrastructure beyond the blasters themselves.

---

## Table of Contents

1. [Design Philosophy](#design-philosophy)
2. [Hardware Overview](#hardware-overview)
3. [Microcontroller — ESP32-S3](#microcontroller--esp32-s3)
4. [Communication Architecture](#communication-architecture)
5. [Hit Detection Flow](#hit-detection-flow)
6. [Messaging Protocol](#messaging-protocol)
7. [Reliability Layer](#reliability-layer)
8. [Game Logic & Modifiers](#game-logic--modifiers)
9. [Advanced Modifier System](#advanced-modifier-system)
10. [Game Modes & Configuration](#game-modes--configuration)
11. [Config Sharing](#config-sharing)
12. [System Modes](#system-modes)
13. [Anchor Node Design](#anchor-node-design)
14. [Phone Integration & Game Master Console](#phone-integration--game-master-console)
15. [TV & Spectator Display](#tv--spectator-display)
16. [Power Management](#power-management)
17. [Multi-Channel LoRa (Frequency Hopping)](#multi-channel-lora-frequency-hopping)
18. [IR Encoding — Manchester Encoding](#ir-encoding--manchester-encoding)
19. [OTA Firmware Updates](#ota-firmware-updates)
20. [Match History & Persistent Leaderboards](#match-history--persistent-leaderboards)
21. [ESP32-S3 Pin Reference](#esp32-s3-pin-reference)
22. [Planned Enhancements](#planned-enhancements)

---

## Design Philosophy

These principles govern every decision in the system:

- **Blasters are autonomous** — a full game runs with no external infrastructure. Anchors, phones, and displays are enhancements, not requirements.
- **Event-driven communication** — messages are only sent when something happens. No constant state broadcasting.
- **Layered architecture** — features scale up without breaking core gameplay.
- **Config-as-data** — all game modes are parameter objects, not hardcoded logic. Any game mode can be described, shared, and loaded at runtime.
- **Trust-based** — each blaster owns and reports its own state. Designed for friendly play, not competitive anti-cheat.
- **Open and accessible** — ESP32-S3 on PlatformIO with the Arduino framework. The most widely known embedded platform, with the largest community and the most documented examples. Anyone getting into electronics today is likely to already own one.

---

## Hardware Overview

### Per Blaster

| Component | Part / Type | Notes |
|---|---|---|
| Microcontroller | ESP32-S3 | See MCU section below |
| LoRa radio | SX1276 module | Connected via SPI |
| IR LED | 940nm IR LED | Shot transmission, line-of-sight |
| IR receiver | TSOP38238 or equivalent | Hit detection, demodulating receiver |
| OLED display | I2C OLED (SSD1306 or SH1107) | Health, ammo, status, modifiers |
| Speaker | I2S amplifier (e.g. MAX98357A) + small speaker | Sound effects |
| NeoPixel LEDs | WS2812B chain | All lighting — see LED section |
| Trigger | Tactile or microswitch | Fire input |
| Reload button | Tactile switch | Reload input |
| Button A | Tactile switch | Aux / special ability |
| Button B | Tactile switch | Aux / special ability |
| Vibration motor | ERM motor via MOSFET | Haptic hit feedback |
| Battery | 2× 18650 cells in parallel | ~5,000–6,000mAh combined. User-replaceable. See Power section. |
| Charger IC | MCP73831 or equivalent | Handles USB-to-battery charging with proper charge termination |
| Protection IC | DW01A or equivalent | Over-discharge, over-current, and short circuit protection |

### LED Architecture

All LEDs are driven from a single NeoPixel data line. The chain includes:

- The onboard RGB status LED (replaces the LED wired directly to the original blaster PCB)
- Side accent lights
- Health bar LEDs

Because all pixels share one data line, the number of LEDs can be expanded freely in future revisions with no PCB changes. Team colour, health gradients, hit flash, low ammo warnings, reload animations, and any other effects are all handled purely in firmware. This was chosen over individual LED pins specifically for flexibility — the original blaster approach of separate LED pins locks the design into the physical layout of the existing hardware.

### Anchor Node Hardware — Heltec WiFi LoRa 32

Rather than designing a custom PCB for anchor nodes, off-the-shelf **Heltec WiFi LoRa 32** development boards are used. This is significantly more cost-effective than a custom board for a low-volume, non-player-facing device, and the hardware is an excellent fit for the role.

**Recommended: Heltec WiFi LoRa 32 V4** (V3 is also fully compatible)

| Feature | Detail |
|---|---|
| MCU | ESP32-S3R2 (dual-core LX7, 240MHz, 2MB PSRAM, 16MB flash) |
| LoRa | SX1262, up to 28dBm TX power |
| Display | 0.96" 128×64 OLED (built in) |
| Connectivity | WiFi b/g/n, BLE 5.0, LoRa |
| Battery | Onboard LiPo management, SH1.25-2 connector |
| Power | USB-C, or LiPo battery |
| Sleep current | ~20µA |
| Dimensions | 51.7 × 25.4 × 10.7mm |

**Why Heltec over a custom anchor PCB:**
- ESP32-S3 + SX1262 + OLED + BMS all integrated on one small board — everything the anchor role needs
- V3 and V4 share the same pinout and are interchangeable in firmware
- PlatformIO and Arduino framework support is well-established with extensive community documentation
- Significantly cheaper than fabricating a custom PCB at low volume
- The onboard OLED can display anchor status, relay count, and signal strength without additional hardware
- Battery connector means anchors can be deployed away from power outlets if needed

**One important note:** the V3 cannot use WiFi and Bluetooth simultaneously due to lack of external PSRAM. The V4 adds 2MB PSRAM which makes concurrent WiFi + BLE more capable. For anchor nodes serving a web UI while maintaining LoRa and BLE simultaneously, the V4 is the better choice. V3 is fine for LoRa relay + WiFi-only use.

The anchor nodes do not replace custom blaster PCBs — they are purely the bridge, relay, and dashboard devices placed at the sideline.

---

## Microcontroller — ESP32-S3

### Why ESP32-S3

The ESP32-S3 was chosen over alternatives (including the nRF52840, ESP32-C6, and original ESP32) for the following reasons:

- **Ecosystem** — PlatformIO + Arduino framework support is mature, well-documented, and widely known. The largest pool of community libraries, examples, and answered questions of any embedded platform.
- **Familiarity** — the most common first MCU for people getting into electronics today. Lowers the barrier to community contributions and DIY builds.
- **GPIO count** — 45 GPIOs, more than enough for all blaster peripherals with room to spare. The ESP32-C6 (22 GPIO) was considered but ruled out as too tight.
- **Dual-core** — one core for radio handling (LoRa, IR), one for game logic and display.
- **BLE 5.0** — built-in, no extra hardware needed for future phone app integration.
- **Web server** — the ESP32-S3's WiFi and web server libraries are mature and capable. Anchor nodes can serve a full config UI and live scoreboard from a simple web server with no external hosting.
- **Power management** — significantly better than the original ESP32. Combined with a dedicated BMS, suitable for battery-powered blasters.

### Why Not the Alternatives

| Chip | Reason Not Chosen |
|---|---|
| Original ESP32 | Poor power consumption (~110mA idle on BLE), older silicon |
| ESP32-C6 | Only 22 GPIO — too tight for full blaster hardware, especially with expansion room |
| nRF52840 | Better specs on paper but steeper learning curve, smaller community, less Arduino library support |

---

## Communication Architecture

The system uses two communication technologies, each chosen for what it does best:

| Layer | Technology | Role |
|---|---|---|
| Hit detection | IR (infrared) | Instant, directional, line-of-sight shot registration |
| Game logic | LoRa (SX1276) | Reliable, long-range event communication |
| Configuration | BLE | Low-power phone-to-blaster setup |
| UI / dashboard | WiFi | Optional web UI, scoreboard, spectator display |

### Why IR for Hit Detection

IR is inherently directional and line-of-sight, which is exactly what physical hit detection requires. It cannot be spoofed by a signal coming from behind a wall. It is instantaneous. It requires no pairing or addressing — the receiver simply detects a pulse.

### Why LoRa for Game Logic

LoRa was chosen after evaluating ESP-NOW, Zigbee, Thread, and nRF24L01. The key factors:

- **Range** — LoRa easily covers large outdoor play areas and multi-room indoor arenas. ESP-NOW range is adequate but not generous.
- **Reliability** — LoRa has better penetration through walls and obstacles than 2.4GHz protocols.
- **Latency tolerance** — game logic messages (hit confirmation, damage payloads, status sync) do not need to be instantaneous. A 100–200ms round trip is completely imperceptible in a laser tag context. This makes LoRa's slightly higher latency a non-issue.
- **Duty cycle** — LoRa's duty cycle limitations are not a practical concern for event-driven messaging. Messages are only sent when something happens.

Note: LoRa operates under duty cycle regulations in some regions (particularly Europe, where airtime is legally limited). For event-driven use this is not a practical concern, but worth keeping in mind for international deployment.

---

## Hit Detection Flow

The hit detection system uses a two-phase approach: IR for the physical event, LoRa for the game logic resolution.

```
1.  Shooter pulls trigger
2.  IR LED transmits a short game ID (assigned at lobby — not a MAC address)
3.  Target blaster IR receiver detects the pulse
4.  Target blaster broadcasts via LoRa: "game ID X hit me — send attack payload"
5.  Shooter responds via LoRa with their current attack payload:
      - Base damage value
      - Any active offensive modifiers (bonus damage, piercing, status effects, etc.)
6.  Target blaster resolves the hit locally:
      - Applies incoming payload against own defensive modifiers
      - Updates health, ammo, and status
      - Updates OLED display and NeoPixel LEDs
7.  If target has reactive modifiers (e.g. Thorns):
      - Target sends a reactive payload back to the shooter
      - Shooter resolves the reflected damage locally
```

### Key Design Decisions

**Modifiers are lazily evaluated.** They are not broadcast continuously. They are only transmitted when a hit event occurs and they become relevant. There is no point tracking someone's active modifiers until they are actually hit.

**Each blaster is the authority on its own state.** Health, ammo, and modifiers are managed locally and periodically synced. This is a trust-based system appropriate for friendly play.

**The IR payload is minimal.** The IR transmission carries only the shooter's game ID — a short numeric value assigned at lobby. All richer game data travels over LoRa. This keeps the IR signal short and reliable, and separates physical detection from game logic cleanly.

---

## Messaging Protocol

Inspired by MQTT, all LoRa messages follow a lightweight publish/subscribe topic structure. Every blaster receives every broadcast and ignores topics it does not need. Anchor nodes subscribe to everything and log it all.

### Message Structure

```
[ topic | sender_id | sequence_number | payload ]
```

### Topics

| Topic | Direction | Description |
|---|---|---|
| `game/hit` | Shooter → All | IR hit confirmed. Includes shooter ID, target ID, damage, active offensive modifiers. |
| `game/reactive` | Defender → Shooter | Reactive modifier result (e.g. Thorns damage returned to attacker). |
| `game/status` | Each blaster → All | Periodic full-state snapshot: health, ammo, score, active modifiers. |
| `game/event` | Any → All | Notable events: kill, game over, modifier activated, respawn, low ammo. |
| `game/config` | Master/Anchor → All | Pre-game configuration broadcast. Sent once before game start. |
| `game/sync` | Anchor → All | Game-wide state: scores, team standings, time remaining. |
| `game/anchor` | Anchor → All | Anchor heartbeat and relay status. |

---

## Reliability Layer

LoRa broadcast is fire-and-forget with no built-in acknowledgment. The following patterns compensate for this at the application layer.

### Sequence Numbers

Every blaster increments a counter on each outgoing message. Recipients detect gaps (received message 47 and 49, but not 48) and can request a resend of critical messages or flag that something was missed.

### Idempotent Messages

All message payloads are designed so that receiving the same message twice produces the same result. This makes retries safe — the system can be aggressive about resending without risking double-applying damage or double-counting kills.

### Priority Tiers

Not all messages need the same reliability treatment.

| Priority | Examples | Handling |
|---|---|---|
| Critical | Hit events, kill events, game over | Retry with sequence tracking |
| Normal | Periodic status sync | Send once — self-corrects on next sync cycle |
| Low | Animation triggers, sound cues | Fire and forget — loss is acceptable |

### Periodic State Sync as Safety Net

The `game/status` broadcast is a full snapshot of a blaster's current state. Even if individual event messages are lost, the next status sync corrects inconsistencies across all devices. The game self-heals naturally without requiring perfect delivery on every packet. This is more robust than hardware-level ACKs for this use case — it corrects *everything*, not just the last packet.

---

## Game Logic & Modifiers

### Modifier System

Modifiers are parameter-based and config-driven. Any modifier can be added to a game config without firmware changes. Examples:

| Modifier | Type | Effect |
|---|---|---|
| Thorns | Defensive / reactive | Returns a percentage of incoming damage to the attacker |
| Shield | Defensive | Absorbs a fixed amount of damage before HP is affected |
| Damage boost | Offensive | Multiplies outgoing damage for a duration |
| Armour pierce | Offensive | Bypasses a portion of target's defensive modifiers |
| Slow | Status effect | Applied by attacker, reduces target's fire rate |

### Modifier Evaluation Flow

Offensive modifiers are included in the attack payload when a hit occurs. Defensive modifiers are applied locally by the hit blaster. Reactive modifiers (like Thorns) trigger a return message to the attacker after resolution.

The flow is always: IR hit → attack payload → local resolution → optional reactive return. Modifiers are never transmitted until they are relevant.

---

## Game Modes & Configuration

### Unified Config Model

All game modes are defined as configuration objects, not hardcoded logic. A game mode is simply a set of parameters. This means any new game mode can be created, shared, and loaded without a firmware update.

```json
{
  "version": 1,
  "name": "Custom Game",
  "core": {
    "teams": 2,
    "time_limit": 600
  },
  "player": {
    "hp": 100,
    "ammo": 30,
    "respawn": true
  },
  "combat": {
    "damage": 10,
    "friendly_fire": false
  },
  "modifiers": {
    "enabled": ["shield", "thorns"]
  }
}
```

### Built-in Presets

Presets are predefined configurations. Users can modify any parameter after selecting a preset.

- Free-for-All
- Team Deathmatch
- Quick Play

### Custom Games

- Created via BLE app or WiFi web UI
- Not stored permanently on blasters — loaded at game start
- Fully shareable between users via share codes

---

## Config Sharing

Game configurations can be shared between players as human-readable JSON or as compressed share codes.

```
LTG:Ab3K9xP2Q...
```

Share codes are compact enough to be sent over text message, copied and pasted, or encoded as a QR code. A versioning field in the config ensures older firmware can detect incompatible configs gracefully.

---

## System Modes

The system supports three operating modes. All are compatible with each other — a game can have some blasters in standalone mode, one acting as master, and one or more anchors present simultaneously.

### 1. Standalone Mode

The simplest mode. No external devices required.

- Config set via onboard OLED UI and buttons
- Game starts when all players are ready
- No phone, no anchor, no WiFi required
- Fully functional laser tag out of the box

### 2. Master Blaster Mode

One blaster acts as a portable game controller. This is the primary mode for casual group play with a phone.

- Master blaster connects to phones via BLE
- Optionally hosts a WiFi AP for browser-based setup
- Broadcasts `game/config` to all players over LoRa before game start
- Only one master is allowed — other blasters detect and defer automatically

### 3. Anchor / Arena Mode

Anchor nodes are powered ESP32-S3 devices placed around the play area. They extend range, aggregate game data, and serve the web UI to phones and TV displays.

**Anchor roles:**
- LoRa relay for extended coverage
- WiFi access point or client for phone/TV connection
- Full game state aggregation and logging
- Live scoreboard and stats server
- Spectator display host

---

## Anchor Node Design

### Behavior

Anchors are passive listeners by default. They do not participate in game logic.

- Listen to all LoRa traffic
- Relay only critical messages (not low-priority traffic)
- Maintain a full copy of game state built from received messages
- Log all events for post-game stats

### Relay Rules

To prevent message storms when multiple anchors are present:

- Sequence number tracking — do not relay a message already seen
- TTL (time-to-live) field — decrement on each relay, discard at zero
- Random rebroadcast delay — prevents simultaneous relay collisions
- Optional RSSI filtering — only relay messages received below a signal strength threshold (i.e. only relay weak signals that may not have reached all blasters)

### Losing the Anchor

Because blasters are autonomous, losing an anchor mid-game has no effect on gameplay. The game continues uninterrupted. The anchor is an enhancement layer — its absence degrades the experience (no live scoreboard, no phone dashboard) but does not break the game.

---

## Phone Integration & Game Master Console

The phone acts as a **game master console**, not a game participant. It is used for setup, monitoring, and post-game stats. It connects to a blaster (via BLE) or an anchor node (via WiFi).

### Pre-Game

- Lobby management: assign teams, player names, colours
- Select or build game mode config
- Set health, damage, respawn rules, time limits, modifier rules
- Broadcast config to all blasters

### During Game

- Live dashboard: each player's health, ammo, kills, deaths, active modifiers
- Recent events feed (kills, respawns, modifier activations)
- Optional mid-game rule changes (e.g. enable a power-up, extend time limit)

### Post-Game

- Full match statistics: shots fired, shots hit, accuracy percentage, damage dealt, damage received, kill/death ratio
- Match history across multiple games
- Leaderboards

### Implementation

The phone app is a **mobile web app** served directly from the anchor node's web server — no app store required. Any phone with a browser can use it. This also means the "app" is updatable via firmware without the user doing anything.

---

## TV & Spectator Display

Any browser-capable device on the anchor's WiFi network can load the spectator view. This includes:

- Smart TVs with a built-in browser
- Chromecast / Apple TV / Fire Stick
- A Raspberry Pi or laptop connected to a TV
- Any phone or tablet in spectator mode

The anchor node hosts a lightweight web page that auto-updates via WebSocket. No casting app required. The spectator view shows:

- Live player health bars
- Team scores
- Kill feed / recent events
- Time remaining
- Post-game summary and stats

---

## Power Management

### Battery — 2× 18650 Cells in Parallel

18650 cells were chosen over LiPo pouches for three reasons specific to this use case:

- **Impact and drop resistance** — the rigid steel casing protects the cell from the knocks and drops inherent to laser tag. A LiPo pouch under repeated physical stress can swell or puncture, both of which are safety concerns. 18650s contain any failure inside a rigid shell.
- **Temperature resistance** — thermal runaway threshold for 18650 is significantly higher (~130–150°C) than LiPo pouches (~60–80°C). Blasters left in the sun or running hard in warm weather are a much safer scenario with 18650s.
- **User replaceability** — 18650s are completely standardised, widely available, and cost $2–5 per cell. Players can carry spare cells and swap them in the field. No soldering required.

Two cells wired in parallel (not series) gives a combined capacity of approximately 5,000–6,000mAh at 3.7V nominal, which should comfortably cover a full day of play. The cells must be matched (same model, same charge state) before paralleling to avoid one cell charging the other.

The physical footprint of two 18650s side by side is close to a 4×AA battery pack, making it a natural fit for the blaster's existing battery compartment. Cells should be mechanically secured in a proper holder or with spot-welded nickel strips — loose cells in a holder can cause intermittent power loss from vibration and impacts during play.

### BMS — Two-Chip Solution

The ESP32-S3 has no built-in battery management. A dedicated two-chip solution handles all battery management separately from the MCU:

**Charger IC — MCP73831 (or equivalent)**
- Handles USB-to-battery charging using constant-current / constant-voltage profile
- Proper charge termination — no risk of overcharging
- Supports simultaneous charge-and-use (power path management) natively, avoiding the TP4056's load-sharing problem where a connected load can prevent proper charge termination
- Voltage accuracy ±0.5%, which reduces battery degradation over time compared to cheaper alternatives

**Protection IC — DW01A (or equivalent)**
- Over-discharge protection — disconnects load before cell voltage drops to damaging levels
- Over-current protection — disconnects on shorts or excessive draw
- Short circuit protection — fast response to prevent cell damage from wiring faults

Together these two chips cover the full battery lifecycle safely. The MCU can sleep deeply between events while the BMS handles all battery monitoring independently.

### Firmware Power Strategies

- **BLE preferred over WiFi** for all configuration tasks — significantly lower power draw
- **WiFi disabled during gameplay** on blasters when not needed (anchor nodes keep WiFi on)
- **Setup mode AP timeout** — if a blaster is hosting a WiFi AP for config and no client connects within a set time, the AP shuts down automatically
- **NeoPixel brightness scaling** — LEDs are one of the highest current draws on the board. Firmware should cap brightness dynamically based on battery level to extend play time

---

## ESP32-S3 Pin Reference

### Pin Status Key

| Status | Meaning |
|---|---|
| ✅ Assigned | Allocated to blaster hardware |
| 🟢 Free | Safe for general use or rerouting |
| 🟡 Caution | Strapping pin or variant-dependent |
| 🔴 Avoid | Reserved for flash, PSRAM, USB, or boot |

### Hardware Assignments (19 pins total)

| GPIO | Assignment | Notes |
|---|---|---|
| GPIO 4 | IR transmit | Digital out. Use a current-limiting resistor on the IR LED. |
| GPIO 5 | IR receive | Digital in. Interrupt-capable — ideal for IR receiver module. |
| GPIO 6 | NeoPixel data | Single pin drives entire LED chain. All lighting on one wire. |
| GPIO 7 | Vibration motor | Digital out / PWM. Drive via MOSFET or motor driver, not directly. |
| GPIO 8 | I2S BCLK (speaker) | I2S bit clock. I2S is remappable to any safe GPIO. |
| GPIO 9 | I2S LRCLK (speaker) | I2S word select / LR clock. |
| GPIO 10 | I2S DOUT (speaker) | I2S data out to amplifier (e.g. MAX98357A). |
| GPIO 11 | Trigger | Digital in, active low. Use internal or external pull-up. |
| GPIO 12 | Reload button | Digital in, active low. |
| GPIO 13 | Button A | Digital in, active low. |
| GPIO 14 | Button B | Digital in, active low. |
| GPIO 16 | LoRa SPI MISO | SPI2 default. SPI is remappable — these are recommended defaults. |
| GPIO 17 | LoRa SPI MOSI | SPI2 default. |
| GPIO 18 | LoRa SPI SCK | SPI2 default clock. |
| GPIO 21 | LoRa SPI CS | Chip select for SX1276. Active low. |
| GPIO 22 | LoRa RST | Hardware reset for SX1276. Digital out. |
| GPIO 23 | LoRa IRQ (DIO0) | Interrupt from SX1276 on TX/RX done. Digital in, interrupt-capable. |
| GPIO 38 | I2C SDA (OLED) | I2C is remappable to any safe GPIO. |
| GPIO 39 | I2C SCL (OLED) | Goes HIGH after reset — fine for I2C SCL. |

> I2C, I2S, and SPI are all software-configurable on the S3 and can be remapped to any safe GPIO. Assignments above are sensible defaults — adjust freely to suit PCB routing.

### Peripheral Pin Count Summary

| Peripheral | Interface | Pins |
|---|---|---|
| OLED display | I2C | 2 |
| LoRa SX1276 | SPI + RST + IRQ | 6 |
| IR transmit | Digital out | 1 |
| IR receive | Digital in | 1 |
| I2S speaker | I2S | 3 |
| NeoPixel LEDs (all) | Digital out | 1 |
| Trigger | Digital in | 1 |
| Reload button | Digital in | 1 |
| Button A | Digital in | 1 |
| Button B | Digital in | 1 |
| Vibration motor | Digital out / PWM | 1 |
| **Total** | | **19** |

### Free Pins (Safe for Rerouting or Future Use)

| GPIO | Notes |
|---|---|
| GPIO 1 | ADC1_CH0, touch capable. |
| GPIO 2 | ADC1_CH1, touch capable. |
| GPIO 15 | Good reserve for future hardware. |
| GPIO 24 | Safe general use. |
| GPIO 25 | Safe general use. |
| GPIO 33–37 | Safe on standard S3. **Avoid on S3R8/S3R8V (octal PSRAM) variants.** |
| GPIO 40–42 | Safe. JTAG only matters if using hardware JTAG debug. |
| GPIO 43 | Default UART0 TX. Safe to repurpose if using USB serial. |
| GPIO 44 | Default UART0 RX. Same note as GPIO 43. |
| GPIO 47–48 | Safe general use. Good reserves. |

### Pins to Avoid

**Strapping pins** — sampled at boot to determine operating mode. Connecting hardware that drives these at boot can prevent startup. Use a 1kΩ–10kΩ series resistor if use is unavoidable.

| GPIO | Reason |
|---|---|
| GPIO 0 | Boot mode strapping. Keep floating or weak pull-up. |
| GPIO 3 | JTAG enable strapping. |
| GPIO 45 | SPI flash voltage strapping. Weak pull-down. |
| GPIO 46 | Boot mode strapping. Weak pull-down. |

**USB JTAG pins** — leave alone unless USB debug is explicitly disabled.

| GPIO | Reason |
|---|---|
| GPIO 19 | USB D− |
| GPIO 20 | USB D+ |

**Internal flash / PSRAM** — do not use under any circumstances. Configuring these as I/O will crash the firmware.

| GPIO | Reason |
|---|---|
| GPIO 26–32 | Connected to internal SPI flash/PSRAM. Using any of these WILL crash firmware. |

### Octal PSRAM Warning

If using the **S3R8 or S3R8V** variant (8MB octal PSRAM), GPIO 33–37 are additionally occupied by the PSRAM interface and cannot be used. When selecting a module, confirm whether it uses standard or octal PSRAM. The safest PCB approach is to treat GPIO 33–37 as reserved and route around them.

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                    PLAY AREA                        │
│                                                     │
│  [Blaster 1] ──┐                                    │
│  [Blaster 2] ──┤── LoRa mesh (broadcast)            │
│  [Blaster 3] ──┤                                    │
│  [Blaster N] ──┘                                    │
│                │                                    │
│         [Heltec V3/V4 Anchor] ◄── USB or LiPo       │
│                │                                    │
└────────────────┼────────────────────────────────────┘
                 │
          WiFi Access Point
               ├──── [Phone — Game Master App]
               └──── [TV / Spectator Display]
```

---

## Advanced Modifier System

Modifiers go beyond simple stat changes. The modifier system supports three categories:

### Timed Buffs

Modifiers with a duration, applied at hit or activation and expiring after a set time. Managed locally on each blaster — no continuous network traffic required.

Examples:
- **Damage boost** — multiplies outgoing damage for N seconds
- **Shield** — absorbs incoming damage for N seconds before HP is affected
- **Speed reload** — reduces reload cooldown for N seconds
- **Invulnerability** — brief window of immunity after respawn (prevents spawn camping)

The blaster tracks expiry timestamps locally and removes the modifier when the timer elapses. Active timed buffs are included in the `game/status` periodic sync so other devices stay aware.

### Area Effects

Triggered by a game event (a kill, a capture point, a special button press) and broadcast to all blasters in range via LoRa. Any blaster within the defined "area" — approximated by LoRa signal strength (RSSI) — applies the effect.

Examples:
- **EMP blast** — disables shooting for all players within range for N seconds
- **Healing field** — restores HP to all teammates within range
- **Damage amplifier** — all damage dealt in range is multiplied for a duration

Area effects are implemented as a special `game/event` message with an RSSI threshold field. Receiving blasters compare the received signal strength against the threshold to decide whether they are "in range."

### Respawn Conditions

Configurable per game mode. Options include:

- **Instant respawn** — blaster reactivates immediately after a short invulnerability window
- **Timed respawn** — blaster counts down on the OLED before reactivating
- **Team respawn** — eliminated players wait until a teammate triggers a respawn event (e.g. stands in a zone, presses a button)
- **No respawn** — elimination is permanent for the round (last team standing)

Respawn configuration is part of the game config object and requires no special firmware logic — it is purely parameter-driven.

---

## Multi-Channel LoRa (Frequency Hopping)

By default the system operates on a single LoRa channel. In environments with interference or when multiple independent games are running nearby, frequency hopping provides isolation and resilience.

### How It Works

The SX1262 (used on the Heltec V3/V4) and SX1276 (blaster module) both support programmable frequency. A frequency hopping sequence is defined in the game config and shared with all devices at lobby time. All blasters and anchors cycle through the same sequence in lockstep, synchronized by sequence numbers.

### Channel Plan

Channels are selected from the legal ISM band for the region (915MHz in the US, 868MHz in Europe). A typical hopping sequence might use 8–16 channels spaced 200kHz apart. The hop occurs on a defined interval (e.g. every 500ms or after every N messages).

### Benefits

- Interference resistance — a jammed or congested channel is only occupied briefly
- Game isolation — two games in the same area can use different hopping sequences without cross-talk
- Regulatory compliance — hopping spreads energy across the band, reducing sustained occupancy of any single frequency

### Implementation Note

Frequency hopping requires all devices to stay synchronized. The sequence number in every message doubles as the synchronization clock — if a device misses several messages and loses sync, it can re-acquire by scanning for active traffic. This is handled transparently by the reliability layer.

---

## IR Encoding — Manchester Encoding

The default IR transmission uses simple pulse-width modulation, which is adequate but susceptible to noise from ambient IR sources (sunlight, fluorescent lights, other IR devices).

### Manchester Encoding

Manchester encoding represents each bit as a transition rather than a level — a 0 is a low-to-high transition, a 1 is a high-to-low transition (or vice versa). This has several advantages for IR laser tag:

- **Noise immunity** — sustained IR interference (e.g. sunlight) cannot fake a valid Manchester-encoded signal, since valid signals require transitions at every bit boundary
- **Self-clocking** — the receiver can recover the clock from the data stream itself, making timing less critical
- **DC balanced** — equal numbers of highs and lows on average, which helps the receiver's AGC (automatic gain control) stay locked

### Practical Impact

For outdoor play in sunlight — one of the harshest IR environments — Manchester encoding meaningfully reduces false hits. The tradeoff is slightly more complex firmware on both the transmitter and receiver side, and a modest increase in transmission time per packet. For the short game ID payload being transmitted, this is a negligible cost.

The 38kHz carrier modulation standard to IR remote control receivers (like the TSOP38238) is retained — Manchester encoding operates at the data layer on top of the carrier, not at the carrier level.

---

## OTA Firmware Updates

Each blaster and anchor node supports over-the-air firmware updates. This avoids the need to physically connect every blaster to a computer for updates, which becomes impractical as the number of devices grows.

### Blaster OTA — Via WiFi

Each blaster can connect to a WiFi network (either a home network or an anchor node AP) to pull a firmware update. The update process:

1. Player holds a button combination on boot to enter OTA mode
2. Blaster connects to a known WiFi SSID (stored in config)
3. Blaster checks a firmware server (anchor node or local network host) for a newer version
4. If available, downloads and flashes the update using the ESP32 OTA partition
5. Reboots into new firmware

The ESP32-S3 Arduino framework includes robust OTA support via the `ArduinoOTA` and `HTTPUpdate` libraries, making this straightforward to implement.

### Anchor Node OTA — Via Heltec Web Interface

Heltec boards can be updated via the same WiFi OTA mechanism. The anchor's web UI can include an "update firmware" page that accepts a `.bin` file upload directly from the browser — no PlatformIO or USB connection required.

### Version Management

Every firmware build includes a semantic version string. The `game/config` broadcast includes the expected firmware version for the current session. Blasters running mismatched firmware versions display a warning on the OLED and prompt the player to update before joining.

---

## Match History & Persistent Leaderboards

Match history and leaderboards are stored on anchor nodes, not on blasters. Blasters are stateless between games by design — they receive config at game start and do not retain history.

### Storage on Anchor Nodes

Heltec V4 boards include 16MB of flash, of which a portion can be allocated as a simple filesystem (LittleFS or SPIFFS) for storing match records. Each completed match is written as a JSON record:

```json
{
  "match_id": "2024-08-15-001",
  "mode": "Team Deathmatch",
  "duration": 480,
  "players": [
    {
      "id": 1,
      "name": "Player 1",
      "team": 0,
      "kills": 5,
      "deaths": 2,
      "shots_fired": 84,
      "shots_hit": 31,
      "damage_dealt": 310,
      "damage_received": 180
    }
  ],
  "winner": "Team A"
}
```

### Leaderboard

Aggregated from stored match records. Available stats per player across all recorded matches:

- Total kills / deaths / K:D ratio
- Accuracy (shots hit / shots fired)
- Total damage dealt and received
- Games played / games won
- Favourite game mode

### Access

Match history and leaderboards are served by the anchor node's web server and accessible from any browser on the WiFi network — the same interface used for the live game dashboard. No separate app or database required.

---

*Document reflects design decisions as of initial planning phase. All pin assignments, hardware choices, and protocol details are subject to revision during PCB design and firmware development.*

*This document was written with the assistance of Claude AI*
