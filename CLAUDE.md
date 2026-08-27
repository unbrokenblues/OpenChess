# CLAUDE.md — Dicky's Smart Chess Board (forked from joojoooo/OpenChess)

> Read this before touching anything. This fork exists to build a **translucent,
> battery-powered, LED-lit chess board** that plays against Stockfish and teaches
> chess — reusing joojoooo/OpenChess's proven hardware design (Hall-effect sensor
> matrix, LED move hints) but replacing the on-device software (Lichess API, paid
> Bluetooth emulation, web UI) with our own simpler firmware.

**Upstream:** https://github.com/joojoooo/OpenChess (fork, ESP32-based — the one
with the PNP-transistor fix for the original Concept-Bytes PCB's design flaws)
**Build guide (reference only, not our software plan):** https://joojoooo.github.io/OpenChess
**Our fork:** https://github.com/unbrokenblues/OpenChess

---

## Why fork instead of build from scratch

The hardest part of this build — reliable 64-square Hall-effect sensing, LED
move-hint sequencing (including castling/en passant/promotion), and a
field-tested 3D-printable board — is already solved and debugged by joojoooo's
fork and multiple independent community builders (confirmed via real build
photos/comments on the MakerWorld page). We're reusing that hardware design and
wiring approach wholesale. What we're **not** reusing: the on-device Stockfish
engine, Lichess API integration, and the paid Bluetooth board-emulation feature —
our own firmware replaces all of that with a WiFi bridge to a Mac-hosted
Stockfish + python-chess script, plus a companion webpage for teaching features
(opening names, eval, commentary) instead of an on-board web UI.

---

## Key deviations from upstream — what WE are doing differently

### Power: embedded battery (upstream has none)
Upstream is wired-only (USB-C from a wall adapter). We're adding:
- **ESP32 board with an onboard JST battery connector + charge IC** (same board
  type as BusGlance/Bubble — confirmed via photo to have this built in). This
  means **no separate TP4056/boost module needed** — plug the battery straight
  into the board's existing JST connector, same USB-C port used for flashing also
  charges it.
- **Battery:** reusing an existing spare 2000mAh LiPo (connector confirmed to
  match).
- **Low-battery sensing:** 2× 100kΩ voltage divider from battery+ to **GPIO35**
  (confirmed free/unused by upstream's pin map — see GPIO section below), same
  formula as BusGlance (`divider=2`).
- **Low-battery indication:** flash the WS2812B grid in a **distinct pattern**
  (e.g. slow synchronized pulse across all 64 squares) rather than reusing "red,"
  since upstream's own firmware already uses red for in-game captures — a
  whole-board pulse avoids confusion with a single-square capture cue.
- **USB-C CC pulldowns:** add 2× 5.1kΩ resistors on CC1/CC2 for reliable direct
  USB-C-to-C charging (independently confirmed necessary both by our own
  ESP32-HARDWARE-PLAYBOOK.md from BusGlance, and by a real OpenChess builder in
  the MakerWorld comments who hit the exact same issue).
- **Realistic power draw estimate:** ~250-350mA average during active play
  (ESP32+WiFi dominates at ~150-250mA; sensor matrix only powers one column of 8
  at a time via the shift register, ~50-70mA; LEDs are ~0 at rest, brief spikes
  only during hints/animations). A 2000mAh pack gives ~6-7 hours of active play
  per charge — plenty for this use case, not a power-hungry system.

### Level shifting: reusing existing stock, not buying upstream's suggested module
Using **SN74HCT125N** (quad non-inverting unidirectional buffer, 3.3V→5V) — a
chip already on hand (shared stock bought for the Bubble project, which uses the
same chip for its own LED matrix data line). This is exactly upstream's own
recommendation ("MUST be 74HCT125 or 74AHCT125... NOT a generic bidirectional
I2C/SPI/UART logic level converter" — those are too slow/timing-imprecise for
WS2812B's tight one-way signal). One chip's 4 channels cover our exact 4 signals
(LED data + shift register clock/latch/data).

### Board finish: translucent print, not upstream's default
Printing the board in **translucent (or just light-colored basic) PLA/PETG**
rather than an opaque color, since the whole point is backlighting through the
board surface. Real-world note: basic light-colored PLA at a thin enough top
layer is naturally light-permeable — a dedicated "translucent" filament isn't
strictly required, ordinary white/natural PLA diffuses LED light adequately at
the print's intended wall thickness. Confirmed square size: **30mm × 30mm**,
measured directly off our own printed 2×2 test tile (not just the CAD bounding
box divided by 8).

### Control board: hand-wired perfboard, matching community practice
NOT using Concept-Bytes' manufactured PCB (documented design flaws — wrong OE'
wiring, 3.3V/5V logic mismatch, sensor overcurrent risk that can damage the MCU).
Using small blank prototyping perfboards instead (same approach the designer and
every successful community builder used) — currently sourcing 5×7cm boards
(6pcs) after confirming via a real build photo that 4×6cm alone is too tight to
hold the ESP32 + shift register + transistors + level shifter + connector pads
together.

### Software: custom firmware, not upstream's feature set
Planning our own firmware rather than compiling upstream's as-is:
- **Occupancy-only sensing** (this actually already matches upstream's default
  behavior — it never identified piece type/color either, just presence/absence
  per square, using legal-move-tracking via python-chess to disambiguate
  castling/en passant/promotion from the occupancy-change pattern alone)
- **No Lichess API, no on-device Stockfish, no paid Bluetooth emulation** — ESP32
  just handles sensor scanning + LED output, bridged over local WiFi to a script
  running on a Mac (python-chess + Stockfish via UCI)
- **Companion webpage** (served by the Mac-hosted script) for opening
  names/eval/commentary — deliberately NOT an on-board screen, since a small
  embedded display would be more parts and less capable than reusing the Mac/phone
  already on hand
- **LED hint color scheme to reuse from upstream's own logic** (verified from
  upstream's `src/board_driver.h` / `src/chess_game.cpp`):
  - Cyan = pick up this piece (origin)
  - White = place here (empty destination)
  - Red = capture here (remove opponent's piece first) — **do not reuse red for
    battery warnings**
  - Purple = square to clear that isn't the destination (en passant only)
  - Green = move confirmed
  - Castling is sequenced (king's move shown/confirmed first, then rook's — never
    simultaneous), not color-coded

### Chess pieces: still undecided
Either (a) reuse an existing physical chess set — drill a shallow hole in each
piece's base, embed magnet(s), re-felt — or (b) 3D print a separate piece model
(was evaluating MakerWorld's "modern chess set" — solid pieces, no pre-made
magnet cavity, base dimensions not published, would need to verify base size
against the confirmed 30mm square before committing). **Not yet decided as of
this writing.**

### Magnets: 8×2mm stacked pairs, N35 grade
2 magnets stacked per piece (4mm total thickness) rather than a single thicker
magnet — matches upstream's own stated alternative ("8×4mm (32pcs) or 8×2mm
(64pcs, stack 2)"). Using N35-graded neodymium specifically (not generic
unspecified "fridge magnets") for reliable field strength. Cavity in piece base
should be cut a bit oversized vs. the magnet (~8.3-8.4mm diameter, ~4.2-4.3mm
deep) to allow easy insertion without heat-softening the plastic, since epoxy
(not friction fit) provides the actual holding strength.

---

## GPIO pin map (classic ESP32 — verify against actual board once in hand)

Confirmed already used by upstream firmware (`src/board_driver.h`):
| Function | GPIO |
|---|---|
| LED data | 32 |
| Shift register clock | 14 |
| Shift register latch | 26 |
| Shift register data | 33 |
| Row inputs (×8) | 4, 16, 17, 18, 19, 21, 22, 23 |

**Free for our battery ADC divider: GPIO 35** (input-only, not in upstream's
"safe GPIO" pool at all since it can't drive outputs — zero conflict). Also free:
13, 25, 27 (in upstream's safe pool but unused by their default wiring), and 34/36/39
(other input-only ADC pins, same class as 35).

⚠️ This pin map assumes a **classic ESP32** — upstream's firmware also supports
ESP32-S3 with a different pinout/ADC map. Re-verify once the actual board is in
hand (run an ADC scan, same technique as BusGlance's `adc_scan` sketch, before
committing wiring).

---

## Full BOM

**Core electronics** (verified against upstream's actual AliExpress sourcing,
~20-25€ before our additions):
- ESP32 dev board (with onboard JST battery connector + charge IC)
- 64× A3144 Hall-effect sensors (TO-92)
- 64× iron discs, 12×1mm
- 64× neodymium magnets, 8×2mm N35 (2 stacked per piece × 32 pieces)
- WS2812B LED strip, 30 LEDs/m, ~3m (64 used)
- 1× 74HC595 shift register (DIP-16)
- 1× USB-C female port
- 8× 10kΩ resistors (sensor pull-ups) + 8× 1kΩ (transistor bases) + 2× 100kΩ
  (battery divider) + 2× 5.1kΩ (USB-C CC pulldowns) — all covered by one 600pc
  30-value 1/4W assortment kit
- 8× PNP transistors (2N3906 or equivalent — covered by a 100pc assorted
  transistor kit)
- 1× SN74HCT125N (already on hand, shared stock with Bubble)
- Perfboard: 5×7cm ×6 (control board — ESP32 + shift register + transistors +
  level shifter + connectors)
- Solid-core hookup wire, 26 AWG, 5 colors × 10m (black=GND, red=5V,
  yellow=rows, blue=columns, green=LED data) — **must be solid core, not
  stranded** (holds shape in the board's unglued printed wire channels; the
  board's wire channels are also only designed for <0.6mm wire diameter)
- 2-part epoxy (magnet embedding — chosen over superglue for better
  gap-filling and resistance to repeated mechanical stress from handling)

**Battery subsystem:**
- Existing spare 2000mAh LiPo (connector confirmed to match board's JST)
- (No separate TP4056/boost needed — handled by the ESP32 board's onboard
  circuit)

**Tools:** soldering iron, multimeter (verify every joint — the designer's own
strongest advice, several builders' failures traced to unverified shorts),
digital caliper (measuring fit/tolerances), flush cutters, wire strippers.

---

## Known hardware gotchas (researched, not guessed)

1. **The original Concept-Bytes manufactured PCB is broken** — wrong 74HC595 OE'
   wiring (tied to 5V, disabling it), 3.3V/5V logic mismatch, sensor overcurrent
   beyond the shift register's rating, can damage a 3.3V MCU. This is why we're
   hand-wiring on perfboard with the PNP-transistor + level-shifter fix, like
   every successful community builder.
2. **Real build time: 15-20 hours**, ~200+ individual solder joints. Not a
   weekend project — budget it like BusGlance (checkpointed, multi-session,
   verify-as-you-go with a multimeter rather than testing all 200 joints at the
   end).
3. **Cheap ESP32 USB-C boards often lack CC pulldown resistors** — a C-to-C
   cable/charger won't enumerate or power the board without them. Fixed by
   adding 2× 5.1kΩ on CC1/CC2 (see BOM).
4. **A3144 Hall sensors are analog-adjacent but used as digital occupancy
   switches here** — no piece-identity detection, matching how DGT and most
   commercial "smart" chess boards (GoChess Mini, ChessUp gen1, Millennium)
   actually work too, despite marketing suggesting otherwise. True piece-ID
   requires embedded electronics per piece (DGT's resonant LC pieces, RFID, or
   Chessnut Air's embedded sensor chips) — out of scope for this build.
5. **Position the Hall sensor near the center of each square**, slightly
   overlapping the LED footprint, per a community builder's tip — avoids a
   detection dead zone.

---

## Open items / TODO
- [ ] Decide chess piece source (reuse existing set vs. print a separate model)
- [ ] Verify actual board's GPIO pin map once ESP32 is in hand (confirm classic
      ESP32 vs S3, confirm GPIO35 truly reads battery voltage via ADC scan)
- [ ] Decide felt sheet / re-felting approach for piece bases (optional,
      depends on piece source decision)
- [ ] Write the actual custom firmware (sensor scan + LED driver + WiFi bridge
      to Mac-hosted Stockfish/python-chess script)
- [ ] Write the Mac-side Stockfish bridge + companion webpage
- [ ] Confirm final BOM order is clean (no accidental unrelated items, no
      missing checkboxes at checkout — this bit us once already, double check
      before every order)
