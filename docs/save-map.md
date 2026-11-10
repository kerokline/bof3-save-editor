# The save, byte by byte, and a review of this editor's fields

A companion to [`story-flags.md`](story-flags.md). It covers the rest of the
save: what each range is, which of this editor's fields the game's code agrees
with, and which it does not.

## Where this comes from

- **Read from code:** the BreathOfFire3PCPort project's source for the 2001 PC
  release, a function-by-function reimplementation of Capcom's code. Each range
  of the save was read once for every function that writes or reads it.
- **Measured on saves:** 80 distinct PC saves of one playthrough, and 20 distinct
  PlayStation saves (US and JP, from memory cards).
- **Limit:** the code read is the PC release's. The PlayStation layout is the
  same apart from the character name's width, and the PlayStation saves agree
  with every claim below that a save can show, but the PlayStation code itself
  was not read for this page.

**Offsets are this editor's:** from the start of the save's first frame, so the
game's data block starts at `+0x200`. Character record offsets are the
PlayStation layout this editor uses.

## 1. Review of the editor's fields

### Wrong or risky

| Field | Where | What the code shows | Suggested change |
|---|---|---|---|
| `fatigue` | record `+0x19` (`+25`) | Not fatigue. `+0x19` is a set of bits that each raise one resistance; it is 0 in all 800 records measured. **Fatigue is `+0x1A`**: the amount of max HP taken off, 0..9, up by one when a member ends a fight down, cleared at an inn. The saves show 1, 2, 3, 5 and 9 there | Read and write `+26` |
| `masters` | `+0xE74` | Two different 3-byte sets (`+0xE74` met, `+0xE77` open), written as one. See [`story-flags.md`](story-flags.md) section 9 | Two fields |
| `constructionPowers` | `+0xEED`, `+0xEEE` | Derived. The game rebuilds both from the faerie records each time it runs the village; an edit is overwritten | Show read-only, or drop |
| `battleCounts` | `+0xED0`, `+0xEDC`, `+0xEE0` | Not three counters of one kind. `+0xED0` counts battles toward the next stage flag and returns to 0. `+0xEDC` and `+0xEE0` are **stamps**: the save's battle count (`+0x954`) at the last mood update and at the last culture level. There are three more stamps the editor does not show: `+0xED4`, `+0xED8`, `+0xEE4` | Treat the five as stamps (below) |
| `stage` | `+0x86E`, 2 bytes | These are event flags 240..255, the end of the event-flag bank. The game sets flag 245 when the culture level reaches 4, and flags 246..255 one at a time from the battle tally | Bits, not a number |
| `position.y.integer` | `+0x22E` | Read as one byte; it is two (`+0x22E..+0x22F`) | Read 2 bytes |
| `position.area` | `+0x224` | Read as one byte; the game stores a u16. The high byte is 0 in every save measured, so this is harmless today | u16 |
| current max HP, AP | record `+0x1C`, `+0x1E` | Derived, like everything in record `+0x1C..+0x3B`: rebuilt on every load from the base block at `+0x3C..+0x5B`, equipment and the master. An edit does nothing | Read-only |

**Stamps.** The game measures faerie progress as "battle count now minus the
stamp", unsigned. A stamp above the battle count reads as a huge age, and
everything it gates happens at once. No save measured has a stamp above its
count. An editor that changes the battle count at `+0x954` must move every stamp
by the same amount: the five above, the dword at `+4` of each faerie record, and
the dword at `+4` of each building.

### Right

Confirmed in code, and in the saves where a save can show it:

- **Characters:** name, level, EXP, equipment (six ids), current HP and AP, base
  max HP and AP, the four base stats, willpower, the nine resistances, surprise,
  reprisal, critical and dodge, the master byte, the apprenticing level, the four
  ability lists, and the six stat-growth bytes (signed, in the order HP, AP,
  power, defence, agility, intelligence).
- **Accuracy** (`+0x58`): the code writes it, but no code was found that reads it.
- **Inventory:** zenny, item ids and counts (four categories of 128, counts to
  99), vital items, the skill list, dragon genes.
- **Party:** active formation, unlocked formations, the two orders.
- **Fishing:** one byte per kind of fish, the largest catch.
- **Counters:** the play clock; the two countdowns at `+0xE7C` and `+0xE80`.
  While a countdown is not all zero, the game refuses one ability in battle. The
  code's own name for the first is "Celerity"; it has no name for the second.
- **Faerie village:** food (`+0xEEA`, 0..99), max jobs (`+0xEEB`), culture
  (`+0xEEC`, 0..7), the 60 records, the 8 buildings, the 60 names.
- **Slot summary:** name, portraits, level, clock, EXP.

### Rules the game relies on

An edit that breaks one of these gives a save the game misreads.

- **Party.** The two orders hold the same ids, with `0xFF` only at the end. The
  byte at `+0x94C` is the row of the party-combination table for the field order
  (bit 7 is set by a load): the load reads the party's files by it, so it must
  match the party. The editor does not model it, so changing who is in the party
  needs more than the two orders.
- **Joined.** Record `+0x07` bit 0 says the member has joined. Heals and the
  equipment count pass over a record without it.
- **Formation.** The active formation is one that is unlocked.
- **Play clock.** All zero reads as a new game.
- **Level and EXP.** If EXP is past the stored level, the next level-up grants
  every level between, with stats and abilities. The EXP curve is chosen by the
  record's position, not by the character id.
- **Master.** The master byte, the apprenticing level and the six growth bytes
  change together. With no master (`0xFF`) the other seven are 0. The
  apprenticing level must not be above the level: the difference is a byte, and a
  wrapped one opens every reward.
- **AP** is not cut to its maximum on load; HP is.
- **Equipment.** The game counts worn items apart from the inventory. Equipping
  in the editor does not take the item out of the inventory.
- **Genes.** `+0xE73` is the top byte of the gene dword and must stay 0.
- **Options and summary.** `+0xEA9` and `+0xEAA` are copies of the window colour
  `+0x27A` and background `+0x27B`.
- **Record 7** (the last one) is reused for guests, and one minigame keeps its
  rank in the first byte of that record's fourth ability list (`+0x7A`). A 5
  there is not an ability.

## 2. The whole save

Tags: **code** the game's code for these bytes was read; **zero** no code touches
them and they are 0 in all 100 saves.

| Offset | Size | What | Tag |
|---|---:|---|---|
| `+0x200` | 16 | Scenario state: chapter, status bits, state, a chapter's own byte, run, step, two u16 counters, 6 unused | code |
| `+0x210` | 10 | The field's event state: `+0x212` the seed marker ([`story-flags.md`](story-flags.md)), `+0x213` kind, `+0x214` state, `+0x215` argument, `+0x216` a u16 timer. A save made at a save point holds `0A 07 FF` at `+0x213` | code |
| `+0x21A` | 6 | | zero |
| `+0x220` | 4 | Four script counters | code |
| `+0x224` | 12 | Area u16, one unused byte, facing (bit 7 a flag), x and z as 16.16 fixed | code |
| `+0x230` | 8 | Scratch that a few chapters reuse | code |
| `+0x238` | 56 | | zero |
| `+0x270` | 2 | Checksum | code |
| `+0x272` | 6 | | zero |
| `+0x278` | 7 | Options: message speed, mono sound, window colour, background, two unused, dash | code |
| `+0x27F` | 17 | | zero |
| `+0x290` | 1,312 | Eight character records, section 3 | code |
| `+0x7B0` | 160 | Chapter flags, 20 rows of 8 | code |
| `+0x850` | 32 | Event flags 0..255 | code |
| `+0x870` | 8 | | zero |
| `+0x878` | 4 | Zenny | code |
| `+0x87C` | 4 | One area's walk state: a u16, a count, a reserve | code |
| `+0x880` | 2 | Active formation, unlocked formations | code |
| `+0x882` | 6 | Field order, battle order | code |
| `+0x888` | 32 | Enemy kinds beaten, 256 bits | code |
| `+0x8A8` | 32 | Abilities learnt, 256 bits | code |
| `+0x8C8` | 32 | Enemy kinds identified, 256 bits | code |
| `+0x8E8` | 4 | Play clock | code |
| `+0x8EC` | 32 | Chest and field-object flags | code |
| `+0x90C` | 32 | Largest catch per kind of fish | code |
| `+0x92C` | 32 | Passage find flags | code |
| `+0x94C` | 1 | Party-combination row | code |
| `+0x94D` | 1 | A copy of the field's script flags | code |
| `+0x94E` | 3 | Fishing: lure id, lure count, rod id, held outside the inventory | code |
| `+0x951` | 1 | Music track to restore | code |
| `+0x952` | 2 | | zero |
| `+0x954` | 4 | Battle count | code |
| `+0x958` | 16 | Four u32 tallies: zenny gained, passages taken, chest contents taken, fishing sessions | code |
| `+0x968` | 12 | A return trip: x, z, area, an "away" byte, a facing | code |
| `+0x974` | 1,024 | Item ids, then counts | code |
| `+0xD74` | 32 | Vital items | code |
| `+0xD94` | 128 | The shared skill list | code |
| `+0xE14` | 1 | A rank 0..12 one area reads for a reward | code |
| `+0xE15` | 5 | | zero |
| `+0xE1A` | 1 | World-map display mode, 0..2 | code |
| `+0xE1B` | 1 | Battles since master 4 was met, to 30 | code |
| `+0xE1C` | 12 | Remembered menu pages and cursors | code |
| `+0xE28` | 24 | Six recent gene combinations, newest first | code |
| `+0xE40` | 48 | Twelve stored gene combinations | code |
| `+0xE70` | 4 | Gene bits (18 used) | code |
| `+0xE74` | 6 | Masters met, masters open | code |
| `+0xE7A` | 2 | | zero |
| `+0xE7C` | 8 | Two countdowns | code |
| `+0xE84` | 28 | | zero |
| `+0xEA0` | 28 | Slot summary. `+0xEA9`, `+0xEAA` option copies; `+0xEB4` the cleared-game byte; `+0xEB6` 4 bytes that hold the rest of the name on the PC release | code |
| `+0xEBC` | 18 | Nine button words | code |
| `+0xECE` | 2 | | zero |
| `+0xED0` | 32 | Faerie village values (section 1) | code |
| `+0xEF0` | 480 | Faerie records, 60 of 8 | code |
| `+0x10D0` | 64 | Faerie buildings, 8 of 8 | code |
| `+0x1110` | 300 | Faerie names, 60 of 5 | code |
| `+0x123C` | 116 | | zero |

**No byte is live without a known use.** The bytes marked zero add up to 463 on
the PC, and every one is 0 in all 80 PC saves and all 20 PlayStation saves.

**A gene combination row** is 4 bytes: up to three gene numbers, `0xFF` after the
last, then a form byte. A form byte of `0xFF` marks an empty row, and an empty row
is four `0xFF` in every save. A new game fills all 18 rows with `0xFF`; zeros
would read as a real combination.

**A faerie record** (8 bytes): `+0` in use, `+1` place (1..8 a building, 9, 10,
11 the three loose places, 0 not placed), `+2` and `+3` state that depends on the
building's kind, `+4` a u32 battle-count stamp.

**A building** (8 bytes): `+0` kind (0 empty, else 4..13), `+1` and `+2` two
choices made on the board, `+3` level, `+4` a u32 battle-count stamp.

## 3. A character record

| Offset | Size | What |
|---|---:|---|
| `+0x00` | 5 | Name (9 on the PC release, and every later offset 4 higher there) |
| `+0x05` | 1 | Character id |
| `+0x06` | 1 | Level |
| `+0x07` | 1 | Flags: bit 0 joined, bit 1 cannot be moved in the party screen |
| `+0x08` | 4 | EXP |
| `+0x0C` | 2 | Status bits |
| `+0x0E` | 6 | Equipment |
| `+0x14` | 4 | Current HP, AP |
| `+0x18` | 1 | Current willpower |
| `+0x19` | 1 | Resistance-raising bits (0 in every save) |
| `+0x1A` | 1 | Fatigue, 0..9 |
| `+0x1B` | 1 | Master, `0xFF` none |
| `+0x1C` | 32 | **Rebuilt on every load** from `+0x3C..+0x5B`: max HP and AP, the four stats, a word, willpower, nine resistances, five rates, three unused |
| `+0x3C` | 4 | Base max HP, AP |
| `+0x40` | 8 | Base power, defence, agility, intelligence |
| `+0x48` | 2 | A base word with no reader found |
| `+0x4A` | 1 | Base willpower |
| `+0x4B` | 9 | Base resistances |
| `+0x54` | 5 | Surprise, reprisal, critical, dodge, accuracy |
| `+0x59` | 3 | Unused |
| `+0x5C` | 40 | Four ability lists of 10 |
| `+0x84` | 1 | Level when the master was joined |
| `+0x85` | 6 | Per-level bonuses the master gives, signed |
| `+0x8B` | 25 | Unused |

## 4. Not checked

- Each range was read once. The measured claims are the ones a save can show;
  the rest is one reading of the code.
- Names the game shows (masters, abilities, fish, faerie jobs) were not checked
  here: the code works in numbers.
- Whether the PlayStation release uses any byte the PC release leaves untouched.
