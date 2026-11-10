# Story flags and progress state

Notes for setting story progress in a save. They come from the Breath of Fire III
PC port project (BreathOfFire3PCPort), which has the game's code as readable
source, and from 38 real saves (PC, US and JP) checked against that reading.

**Offsets here are save-file offsets, as this editor uses them**: the start of the
save's 8 KiB block on the card, where the game's data begins at `+0x200`. So
zenny is `+0x878`, as in `saveInventory.ts`.

What is measured, read or untested is said in each section. The PC executable is
the 2001 Chinese PC port, which derives from the Japanese release.

## 1. Where the state is

| What | Offset | Size | Notes |
|---|---|---:|---|
| Chapter number | `+0x200` | 1 | 0..15 in play. 17 is the ending, 16 the title demo |
| Chapter working bytes | `+0x201..+0x209` | 9 | State, run, step, timer. Zero at a chapter's start |
| "Event flags seeded" marker | `+0x212` | 1 | 1 in every save seen. Section 3 |
| Chapter flag rows | `+0x7B0 + 8 * chapter` | 8 each | 64 flags per chapter, chapters 0..19 |
| Event flags | `+0x850` | 32 | Flags 0..255 |
| Chest and pick-up flags | `+0x8EC` | 32 | One flag per field object that gives something |
| A second find bank | `+0x92C` | 32 | With a counter at `+0x95C` |
| Dragon genes | `+0xE70` | 3 | 18 bits. Already edited here |
| "Game cleared" in the slot summary | `+0xEB4` | 1 | Section 4 |

**Bit order:** flag `n` is bit `n & 7` of byte `n >> 3` of its bank, low bit first.

The two bytes this editor calls the faerie village stage (`+0x86E`) are bytes 30
and 31 of the event flags, so they are event flags 240..255.

## 2. How a chapter starts and ends

Read from the code and checked against saves.

- **A chapter ends** when its last scene sets an "advance" bit. Nothing changes
  until the party next enters an area.
- **On that area entry** the game adds one to the chapter number and starts the
  new chapter. Chapter 15 goes to 17.
- **Starting a chapter** zeroes the working bytes and clears **flags 0..31** of
  the new chapter's row. Flags 32..63 are not cleared, and are still zero in a
  normal game.

So:

- **The start of chapter N** is: chapter number N, working bytes zero, row N all
  zero, and every earlier row as that chapter left it.
- **A finished chapter is not "all 64 flags set".** Each chapter leaves its own
  pattern. Flags are of several kinds:

| Kind | Meaning |
|---|---|
| Progress | Set on every route |
| Once-only | Marks a first-entry scene as done |
| Optional | Set only if the player does a thing, such as an optional fight |
| Choice | One of two or more alternatives |
| Party pick | One flag of a group, chosen by who is in the party |
| Transient | Set, then cleared again before the chapter ends |
| State | A lever or switch position. It can change back |
| Unused | No code touches it |

A finished row usually stays as it was left. Known exceptions: chapter 9 changes
chapter 6's flag 62, chapter 15 writes chapter 14's flags 1..3, and the ending
clears chapter 15's flags 9..22.

### Example: the rows of one complete playthrough

From a PC save made after the ending. Other routes differ in the optional, choice
and party-pick flags, so treat this as one valid finished state, not the only one.

| Chapter | Flags set |
|---:|---|
| 0 | 0..7 10 |
| 1 | 0..7 9..13 15 18 20..22 24..29 57..60 62 |
| 2 | 0..1 3..8 12 15..32 35..39 41 46..51 54..56 62..63 |
| 3 | 1..30 |
| 4 | 1..8 10 12 |
| 5 | 0..2 5..6 8..12 14..20 23..26 29..35 40..54 59..62 |
| 6 | 0..1 3..22 26..31 33..40 42 44 59..63 |
| 7 | 0..5 7..12 |
| 8 | 0..17 19..22 24..26 |
| 9 | 0..8 11..19 21..30 32..36 38..39 44 46..50 52 54..58 61..62 |
| 10 | 0..2 4..11 13..26 32..44 46..47 52..61 |
| 11 | 0..2 4..12 14..19 |
| 12 | 0..2 20..25 30 36..45 47..48 56 63 |
| 13 | 0..31 |
| 14 | 0 4 6..31 48..49 53 |
| 15 | 0..6 32..47 (after the ending cleared 9..22) |

Event flags in that save: 3..7 13..15 24..27 29 32 35 41..42 45..52 54..56 58..64
67..69 71..72 77 79..80 82..84 89..93 96..101 103..104 106..115 121..123 125..128
132 134..135 143..149 152 245..255.

Chapter 0 has two optional flags, 8 and 9: two optional fights. A save that
fought all five of the opening's fights has 0..10.

## 3. A new game

- The game does not clear the event flags at New Game. On the first frame in the
  field it copies a 16-byte table over event flags 0..127, then sets the marker
  byte `+0x212` to 1. (Read in the PC code. The PSX has the same routine.)
- **That table sets flags 3, 25 and 27 and nothing else.** A save made one frame
  into a new PC game shows exactly those three. They are set in every one of the
  38 saves.
- The marker is 1 in every save seen: 19 PC, 7 US, 5 JP.

To build a new-game state: event flags 3, 25 and 27 set, all other flags zero,
marker `+0x212` = 1. **If the marker is 0, the game copies the table again on the
first field frame and wipes event flags 0..127.** That last point is read from the
code, not tried.

The chapter 0 start also fills `+0xE28..+0xE6F` (72 bytes) with `0xFF`.

## 4. A cleared game

After the credits, one answer to the last prompt:

- sets event flag 146,
- clears chapter 15's flags 9..22,
- puts the chapter number back to 15 with the working bytes zero.

The save code copies event flag 146 into the slot summary at `+0xEB4`. Measured on
one PC cleared save: flag 146 set, chapter 15, and `+0xEB4` = 1, where every other
save has 0.

## 5. Dragon genes

**Setting a gene's bit is enough to grant it.** Played on PC: a save edited to own
gene 16 before the scene that gives it. The gene worked in battle at once, the
scene still played and gave its other gene, and the story went on.

Four genes are given by story scenes, not found in the field: 0 and 6 in chapter
3, and 3 and 16 together in chapter 8. The scenes do not check which genes are
owned. There is no need to set a chapter flag with a gene.

## 6. Chests and pick-ups

Opening a chest or picking up an item sets a flag in the bank at `+0x8EC`, not a
chapter flag. The flag number comes from the object's data in the area file. A
few one-off items use event flags instead.

## 7. Flag numbers held in tables of the executable

Some code takes its flag number from a small table. These are the values, read
from one copy of the PC `BOF3.exe`. The addresses are PC addresses, given for
reference.

- **Bank** says which flags the numbers index: `event` is the event flags
  (`+0x850`), `chapter N` is that chapter's row, `current` is the row of whichever
  chapter is running.
- `-` is an empty entry.
- **Confidence** is how sure the table's layout is: `read` from the code, or
  `likely`.

(sha256 `15984158e0badb4b2f9471c1e9003bb18b2ab11aff47af2376f9ebcd31dfb8cb`). Each row is a table the game reads a story-flag number from.

| Table | Address | Bank | Values | What the entries are | Confidence |
|---|---|---|---|---|---|
| Area41_TriggerFlags | `0x5F5F68` | event | 131 132 | area 41 object triggers 59 and 60; the code reads 0x5F5F2D + object +0x86, unbounded | likely |
| Area44_Switches | `0x5F7082` | event | 22 22 23 23 | area 44 floor switches (x, z, facing, flag, tail kind), toggled | read |
| Area49_Zones | `0x5FBB70` | event | 2 2 2 3 3 2 3 | area 49 zones (x0, z0, x1, z1, direction, flag); set turns the direction by 4 | read |
| Area49_CellSwitches | `0x5FBC44` | event | 2 2 2 2 2 3 3 3 3 | area 49 cell switches (x, z, pose, flag), toggled | read |
| Area52_CellSwitches | `0x5FE0F0` | event | 36 36 38 37 | area 52 cell switches (x, z, pose nibble, flag, tail state), toggled | read |
| Area77_CellSwitches | `0x60BBCC` | event | 5 6 | area 77 cell switches (x, z, pose, flag), toggled | read |
| Area86_Switches | `0x611C7C` | event | 8 9 10 | area 86 floor switches (x, z, facing, flag), toggled | read |
| Area108_HookCells | `0x61E63C` | event | 24 25 26 27 | area 108 cell hook entries (x, z, direction, flag), toggled | read |
| Area112_HookCells | `0x61FF80` | event | 56 57 58 59 | area 112 cell hook entries (x, z, flag-bits byte, flag); each set once | read |
| Area112_HookFlags | `0x61FFA0` | event | 60 61 62 63 | area 112: flags an entry's byte 2 clears (bits 0..3) or sets (bits 4..7), by bit & 3 | read |
| Area117_Rects | `0x6215A8` | event | 133 | area 117 rectangle (x0, z0, x1, z1, facing, flag); set reverses the facing | read |
| Area117_Switches | `0x62160C` | event | 133 | area 117 floor switch (x, z, facing, flag), toggled | read |
| Area118_Rects | `0x621DF0` | event | 133 | area 118 rectangle (x0, z0, x1, z1, facing, flag); set reverses the facing | read |
| Area118_Switches | `0x621E3C` | event | 133 | area 118 floor switch (x, z, facing, flag), toggled | read |
| Area139_CellEntries | `0x62EDBC` | event | 92 93 | area 139 cell entries (x, z, direction nibble, flag), toggled | read |
| Area140_CellEntries | `0x62EFF8` | event | 94 95 96 97 98 99 100 101 | area 140 cell entries (x, z, direction, flag, effect, rectangle); set, cleared by effect 0x71 | read |
| Area109_SwitchFlags | `0x65DE60` | event | 14 15 16 | area 109 switch: the three flags of the pattern, bits 0..2 | read |
| Area167_Switch | `0x63C594` | event | 71 | area 167 cell switch (x, z, facing nibble, flag), toggled | read |
| EffectKind18_06_Flags | `0x65DB84` | event | 8 9 10 | effect kind 0x18 sub-kind 6: the flag of set +0x36; a one-shot turn request | read |
| EffectKind18_07_FlagRows | `0x65DDB4` | - | 3 20 20 | sub-kind 7 doors: the Cond_Flags row of door +0xB; 0..19 a chapter row, 20..24 the event bank | read |
| EffectKind18_07_FlagBits | `0x65DDB8` | by-row | 4 (+64*(row-20) when row>=20) 131 (+64*(row-20) when row>=20) 132 (+64*(row-20) when row>=20) | sub-kind 7 doors: the flag in the row EffectKind18_07_FlagRows gives, same index | read |
| EffectKind18Sub0C_Flags | `0x65DFC4` | event | 18 19 20 21 11 0 0 0 | sub-kind 0x0C: the flag by column +0x36; set opens | read |
| EffectKind18Sub13_Flags | `0x65E010` | chapter 9 | 0 12 13 15 | sub-kind 0x13 by +0xB: the flag that shows the panel | read |
| EffectKind18Sub13_Faces | `0x65E014` | chapter 9 | 12 13 15 17 | sub-kind 0x13 by +0xB: the flag that picks the face | read |
| EffectKind18_GateFlags | `0x65E034` | chapter 9 | 12 13 15 17 | sub-kind 0x0F gate flag by +0x3A; also sub-kind 0x50's wall texture flag by +0xB (effect_5c.cpp:302) | read |
| EffectKind18Sub50_Flags | `0x65E094` | chapter 9 | 0 12 13 15 | sub-kind 0x50 wall by +0xB (1..2 as the start leaves it) | read |
| EffectKind18Sub14_Flags | `0x65E31C` | chapter 9 | 38 39 | sub-kind 0x14 by place +0x36: the flag that starts the spawn | read |
| EffectKind18Sub22_Flags | `0x65E348` | event | 60 62 61 63 | sub-kind 0x22 by place +0x36: set lowers, clear raises | read |
| EffectKind18Sub42_Places | `0x65E8E0` | event | 95 94 98 | sub-kind 0x42 places (x, z, variant, flag) by +0x36 | read |
| EffectKind18Sub5C_Flags | `0x65F38C` | event | 134 135 | sub-kind 0x5C two-panel door by variant +0x36: set while shut | read |
| Scena10_Pickups | `0x661528` | current | 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 | chapter 10 pick-up records (flag less 0x20, item less 0x4E, x word, z word) | read |
| Scena15_Battles | `0x6618C0` | current | 32 34 37 39 40 41 42 44 46 47 | chapter 15 battle records, +8 the flag set after the battle; index unchecked, count from the table extent | likely |
| Scenario_SubInitTable | `0x662CD8` | event | flags set: 3 25 27 | bit image copied over event flags 0..127 at a New Game | read |
| Area_MusicSets | `0x669A48` | by-row | (not read: layout not known (unknown)) | music sets: +0 a pointer to (+4) 3-byte entries (track, row, flag); the flag is behind a pointer | unknown |

## 8. What is not checked

- The table values are from the PC executable. They have not been compared with
  the PlayStation discs. What has been compared: the chapter number, the chapter
  rows and the event flags agree between PC, US and JP saves made at the same
  point, and the rows of later US and JP saves fit the same code reading.
- Most of the flag meanings are one reading of the code. The saves confirm which
  flags get set, not always why.
- The full per-chapter maps (every flag, what sets it and what it gates) are in
  the BreathOfFire3PCPort repository under `docs/story-flags/`, once that branch
  is merged.

## 9. Two fields of this editor, checked against the code

The rest of the editor's fields, and a map of the whole save, are in
[`save-map.md`](save-map.md).

Both names are right. One of the two is written wrongly.

### `+0x881`, unlocked formations: right

- The tactics screen opens cell `n` of the formation grid when bit `n` is set.
- Bits 0..2 are set in every save from the first one (`0x07`).
- Bits 3, 4 and 5 are each set once by a master (ids 11, 13 and 14) when an
  apprentice has gained 3 levels under them. The same bits make those masters
  show as "available" in the masters window. Master 12's reward at 3 levels is
  an item instead, kept by event flag 108.
- The saves of one playthrough show `0x07`, then `0x1F`, then `0x3F`.

### `+0xE74`, masters: two fields, not one written twice

`loadInventory` reads 3 bytes at `+0xE74`; `saveInventory` writes them to
`+0xE74` and again to `+0xE77`. The game keeps two different 17-bit sets there:

| Offset | Set when | Used for |
|---|---|---|
| `+0xE74` | a master's introduction opens | "met" |
| `+0xE77` | the introduction closes | the masters window lists these |

They are equal most of the time, which is why the copy looks right. They differ
in two cases:

- **Master 9.** The meeting sets the first bit; a later story step sets the
  second. Three of the saves examined have bit 9 met and not open. Saving one of
  them through the editor opens that master early.
- **Master 4.** Met and not open is the state in which the game counts battles
  in the byte at `+0xE1B`, up to 30, and a rest sets the count back to 0.
  Saving in that state through the editor ends the count.

Suggested fix: load and save `+0xE74` and `+0xE77` as two fields of 3 bytes.
