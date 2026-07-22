# Soccer Merge — Version 0.2 Implementation Plan

**Product direction:** A football **club-building** game driven by merge mechanics. The merge board grows the club's physical world — not individual player development. Customization systems (club name, location, colors, stadium style, facilities, team identity) are **out of scope for v0.2**.

**Scope:** Version 0.2 only. Do not implement club customization, multiple merge chains, a Club Scene, or dual-currency economy.

**Constraint:** This document is a plan only. Do not apply changes until each item is reviewed and executed deliberately.

---

## Current baseline (do not change without reason)

| Element | Current state |
|---|---|
| Scene | `Game Scene` (only scene) |
| Grid | 80 × 80 px virtual grid (`SnapToGrid` extension) |
| Global objects | `MergeItem`, `Producer` |
| Scene objects | `EnergyText`, `EnergyBar`, `LevelProgress`, `SoccerFieldBackground` |
| Merge chain | 100 levels in global `MergeChain` JSON string; levels 0–10 named, 11–99 placeholder |
| MergeItem visuals | Single sprite (`soccer ball 1.png`) for all levels |
| Producer bug | Energy deducted **before** spawn slot check; energy lost if all 4 adjacent cells are full |
| No-energy feedback | Event `/7` (Else on producer click) exists but has **empty actions** |
| Persistence | None |
| Extensions in use | `SnapToGrid`, `ButtonStates`, `PanelSpriteContinuousBar` |

### Event block map (reference for this plan)

Events are numbered as they appear top-to-bottom in `Game Scene` today:

| ID | Trigger | Purpose |
|---|---|---|
| `/1` | Scene begins | Init `MaxEnergy`, `Energy`, `LastEnergyTimestamp` |
| `/2` | Every frame | Energy regeneration math |
| `/3` | Every frame | Update `EnergyText` and `EnergyBar` |
| `/4` | Scene begins | Init `MergeItem` position vars; set starter levels at fixed coords |
| `/5` | `MergeItem` dropped | Snap, merge, reject-on-occupied logic |
| `/6` | `Producer` clicked + `Energy > 0` | Deduct energy, find free adjacent cell, spawn Level 0 item |
| `/7` | `Producer` clicked + `Energy <= 0` | Else branch — currently does nothing |
| `/8` | Every frame | ForEach `MergeItem` → update `LevelProgress` text |

---

## Version 0.2 merge chain (levels 0–9)

Replace the **names** (and later **sprites**) for the first ten tiers. Levels 10–99 stay as placeholders until a future version.

| Level | Display name | Theme |
|---|---|---|
| 0 | Football | Starting object |
| 1 | Goal | First structure |
| 2 | Practice Pitch | Training ground |
| 3 | Small Bleachers | Early seating |
| 4 | Local Ground | Community venue |
| 5 | Covered Stand | Weather protection |
| 6 | Small Stadium | Enclosed venue |
| 7 | Modern Stadium | Upgraded architecture |
| 8 | Football Complex | Multi-facility site |
| 9 | World-Class Stadium | Peak tier (v0.2 cap) |

Level 9's `nextLevel` should remain `10` (chain continues internally) but v0.2 UI shows **"Levels: X / 10"** until the full chain is redesigned.

---

## Item 1 — Fix producer: deduct energy only on successful spawn

### How to implement in the GDevelop visual editor

1. Open **Game Scene → Events**.
2. Locate event **`/6`** (`Producer` clicked AND `Energy > 0`).
3. **Remove** the action `Energy = max(0, Energy - 1)` from the **top-level actions** of `/6` (it currently runs immediately on click).
4. Scroll to the **last sub-event** of `/6` — the one with condition `Found = True` that runs `Create MergeItem`.
5. **Add** the action `Energy = max(0, Energy - 1)` as the **first action** in that sub-event, **before** `Create MergeItem`.
6. Confirm event **`/7`** (Else: clicked + `Energy <= 0`) still has no energy deduction — it should remain feedback-only (see Item 2).

**Optional hardening:** At the start of `/6` top-level actions, add `Found = False` to reset the boolean each click (it is already declared on the event with default `false`, but an explicit reset prevents stale state if event order changes).

### Affected event block / object

| Target | Change |
|---|---|
| Event `/6` | Move energy deduction from parent actions → `Found = True` sub-event |
| Object `Producer` | No object edit required |
| Global variable `Energy` | Write timing changes only |

### Variables / resources required

| Name | Scope | Role |
|---|---|---|
| `Energy` | Global | Decremented only after confirmed free cell |
| `Found` | Event `/6` local | Must be `True` before deduction (spawn confirmed) |
| `SpawnX`, `SpawnY` | Event `/6` local | Unchanged |

No new resources.

### Exact expected behavior

| Action | Expected result |
|---|---|
| Click producer, at least one adjacent cell empty | Level 0 `MergeItem` spawns; `Energy` decreases by 1 |
| Click producer, all 4 adjacent cells occupied | Nothing spawns; `Energy` **unchanged** |
| Click producer, `Energy = 0` | Nothing spawns; handled by `/7` (Item 2) |
| Rapid double-click with 1 energy | At most one spawn and one deduction (GDevelop processes one click per frame) |

### Rollback procedure

1. In event `/6`, move `Energy = max(0, Energy - 1)` back to the **parent actions** (first action after vibration).
2. Remove it from the `Found = True` sub-event.
3. Preview and confirm energy is again deducted even when no item spawns (original buggy behavior restored).

---

## Item 2 — Add clear "not enough energy" feedback

### How to implement in the GDevelop visual editor

1. **Create a scene object:**
   - Right-click `Game Scene` → **Add a new object** → **Text**.
   - Name: `EnergyWarningText`.
   - Initial text: `Not enough energy!`
   - Font size: 18–24, color: red or amber.
   - Position: near `EnergyText` (e.g. X=10, Y=58) on layer **`UI`**.
   - In object properties, set **Hidden at start** ✓ (or opacity 0 via an opacity behavior).

2. **Add a scene variable** on event `/7` (or as a layout variable):
   - Name: `EnergyWarningTimer`
   - Type: number, default `0`

3. **Edit event `/7`** (Else: `Producer` clicked AND `Energy <= 0`):
   - Add action: `EnergyWarningText` → Text → Set text to `"Not enough energy!"` (if not static).
   - Add action: show `EnergyWarningText` (Visibility → visible, or Opacity → 255).
   - Add action: `EnergyWarningTimer = 2` (seconds to display).

4. **Add a new top-level event** after `/7`:
   - Condition: `EnergyWarningTimer > 0`
   - Action: `EnergyWarningTimer = EnergyWarningTimer - TimeDelta()`
   - Sub-event condition: `EnergyWarningTimer <= 0`
   - Sub-event action: hide `EnergyWarningText`

5. **Optional polish:** In `/7`, add `DeviceVibration::StartVibration(100)` (longer buzz than the success vibration) or play a "denied" sound (see sound hook pattern in Item 4).

### Affected event block / object

| Target | Change |
|---|---|
| Event `/7` | Add show-warning actions (currently empty) |
| New event (e.g. `/7b`) | Timer countdown + hide warning |
| New object `EnergyWarningText` | Scene-only UI text on `UI` layer |

### Variables / resources required

| Name | Scope | Role |
|---|---|---|
| `EnergyWarningTimer` | Scene or event `/7` local | Seconds remaining for warning display |
| `EnergyWarningText` | Scene object | Visible feedback label |

Optional resource: `sfx_denied.wav` (if using audio instead of/in addition to text).

### Exact expected behavior

| Action | Expected result |
|---|---|
| Click producer with `Energy > 0` and free cell | Normal spawn; **no** warning shown |
| Click producer with `Energy = 0` | No spawn; `EnergyWarningText` appears for ~2 seconds |
| Warning visible, energy regenerates above 0 | Warning still fades on timer (does not persist indefinitely) |
| Click producer repeatedly at 0 energy | Timer resets to 2 s each click (warning stays visible / re-triggers) |

### Rollback procedure

1. Delete the warning timer event block.
2. Clear actions from event `/7` (restore empty Else).
3. Delete object `EnergyWarningText` from `Game Scene`.
4. Delete `EnergyWarningTimer` variable if created.

---

## Item 3 — Distinct visuals and visible names for Levels 0–9

### How to implement in the GDevelop visual editor

This item has three parts: **chain data**, **sprites**, and **runtime display**.

#### Part A — Update merge chain names

1. **Project → Global variables → `MergeChain`.**
2. Edit the JSON string so keys `"0"` through `"9"` use the new names from the table above.
3. Leave `"10"` onward unchanged for now.

#### Part B — Add 10 sprites to `MergeItem`

1. Open object **`MergeItem`** (global object editor).
2. Open the **Animations / sprites** editor.
3. Create **10 animations** (one per level), named exactly: `Lv0`, `Lv1`, … `Lv9`:

   | Animation | Suggested asset direction |
   |---|---|
   | `Lv0` | Football (existing `soccer ball 1.png` is fine) |
   | `Lv1` | Goal frame / net |
   | `Lv2` | Small grass training pitch |
   | `Lv3` | Wooden bleachers |
   | `Lv4` | Local park ground with boundary |
   | `Lv5` | Covered stand / roof section |
   | `Lv6` | Small enclosed stadium |
   | `Lv7` | Modern stadium exterior |
   | `Lv8` | Complex aerial / multi-building |
   | `Lv9` | Large world-class stadium |

4. Import PNG assets via **Resources** panel before assigning to animations.
5. Set default animation to `Lv0`.

#### Part C — Switch animation when level changes

Add logic in **three places** (same sub-event pattern in each):

**C1 — Event `/4` (scene start, starter items):**  
After each `Set MergeItem.Level` action, add:  
`MergeItem` → Change animation → `"Lv" + ToString(MergeItem.Level())`  
(GDevelop expression: use `ToString(MergeItem.Level())` concatenated, or a sub-event per known level during testing.)

**C2 — Event `/5` merge success sub-event (`NextLevel != -1`):**  
After `Set MergeItem.Level = NextLevel`, add:  
`MergeItem` → Change animation → expression based on `NextLevel`.

**C3 — Event `/6` spawn sub-event (`Found = True`):**  
After `Set MergeItem.Level = 0`, add:  
`MergeItem` → Change animation → `"Lv0"`.

**Recommended approach:** Create an **external event** `ApplyMergeItemVisuals` with one sub-event per level (0–9):

```
Conditions: MergeItem.Level = N
Actions:
  - MergeItem: Change animation → LvN
  - ItemNameLabel: Update text from MergeChain[N].name  (see label below)
```

Call it via **Link** after every level assignment and after loading save data (Item 5).

#### Part D — Visible name label

1. Add a **Text** object **`ItemNameLabel`** to `MergeItem` as a **child object** (or use a separate object positioned each frame — child is simpler).
   - Font size: 10–12, centered below sprite.
   - Default text: blank.
2. In `ApplyMergeItemVisuals`, set text to:  
   `MergeChain[ToString(MergeItem.Level())].name`  
   (Use JSON parsing expression or a lookup sub-event chain if GDevelop expression access is awkward — see note below.)

**GDevelop JSON note:** Reading nested JSON from a string variable may require a **JavaScript event** or pre-splitting names into a second structure. **Simplest v0.2 approach:** ten sub-events (`Level = 0` → set text `"Football"`, etc.) inside `ApplyMergeItemVisuals`. Refactor to data-driven lookup in v0.3.

#### Part E — Update progress counter

In event **`/8`**, change the text expression from `/ 100` to `/ 10` for v0.2 display cap.

### Affected event block / object

| Target | Change |
|---|---|
| Global variable `MergeChain` | Names for keys 0–9 |
| Object `MergeItem` | 10 animations `Lv0`–`Lv9` |
| New child/link `ItemNameLabel` (optional object) | Name display |
| Events `/4`, `/5`, `/6`, `/8` | Animation + text updates |
| New external event `ApplyMergeItemVisuals` | Centralized visual apply (recommended) |
| Resources | 9 new PNG sprites (Lv1–Lv9; Lv0 may reuse existing) |

### Variables / resources required

| Name | Type | Role |
|---|---|---|
| `MergeChain` | Global string (JSON) | Name lookup per level |
| `MergeItem.Level` | Object variable | Drives animation selection |
| `Lv0`…`Lv9` animations | Object resource | Distinct visuals |
| 9–10 PNG files | Resources | Stadium growth art |

### Exact expected behavior

| State | Expected result |
|---|---|
| Newly spawned item | Shows `Lv0` animation + label **"Football"** |
| Merge two Level 2 items | Result shows `Lv3` animation + **"Small Bleachers"** |
| Starter item at Level 2 on scene load | Correct animation + name before any interaction |
| Level 10+ items (if reached via old chain) | Fall back to `Lv9` animation or a generic `LvSuper` animation (define one fallback sub-event) |
| `LevelProgress` UI | Reads e.g. `Levels: 4 / 10` when highest on board is Level 3 |

### Rollback procedure

1. Revert `MergeChain` JSON names to previous values (backup before edit).
2. Remove animation-switch actions from `/4`, `/5`, `/6`; delete external event `ApplyMergeItemVisuals` if created.
3. Remove extra animations from `MergeItem`; restore single default sprite.
4. Delete `ItemNameLabel` child object.
5. Restore `/8` text to `/ 100`.

---

## Item 4 — Pop effect and sound hook on successful merge

### How to implement in the GDevelop visual editor

#### Part A — Pop effect (extend existing tween)

Event **`/5` → sub-event `NextLevel != -1`** already runs:

```
MergeItem → Tween → Add object scale tween ("mergePop", easeOutBack, 0.3)
```

**Enhance:**

1. Open `MergeItem` → verify **Tween** behavior exists (it does).
2. In the merge success sub-event, **before** the scale tween, add:  
   `MergeItem` → Scale → set to `0.6` (quick pre-shrink).
3. Keep existing scale tween to `1` with `easeOutBack`.
4. **Optional:** Add opacity tween or a temporary **Particle emitter** object `MergeVFX` created at `NewXPosition`, `NewYPosition` and destroyed after 0.5 s.

#### Part B — Sound hook

1. Import a placeholder SFX (e.g. `sfx_merge.wav`) into **Resources**.
2. Add action in the **same merge success sub-event**, after `Create MergeItem`:  
   **Audio → Play sound** → `sfx_merge.wav`, volume 80, loop no.
3. If no audio file is ready, add the action with a **missing resource comment** in the event (comment event: "Hook: assign sfx_merge.wav here") — structure the event now so sound is one action away.

**Do not add sound to failed merges or rejected drops** — only the successful `NextLevel != -1` path.

### Affected event block / object

| Target | Change |
|---|---|
| Event `/5` → `/5/1/1` (`NextLevel != -1`) | Enhanced tween + audio action |
| Object `MergeItem` | Tween behavior (existing) |
| Optional `MergeVFX` | Scene or global particle object |

### Variables / resources required

| Name | Type | Role |
|---|---|---|
| Tween behavior | `MergeItem` behavior | Scale pop (existing) |
| `sfx_merge.wav` | Resource | Merge success sound (placeholder OK) |
| `NewXPosition`, `NewYPosition` | Event `/5` local | VFX placement |

No new variables required.

### Exact expected behavior

| Action | Expected result |
|---|---|
| Merge two equal items (valid merge) | New item scales up with bounce; merge SFX plays once |
| Drop item on empty cell | No SFX; no merge pop |
| Drop item on occupied different-level cell | Item snaps back; no SFX |
| Merge at max chain level (`NextLevel = -1`) | No merge; no SFX |

### Rollback procedure

1. Remove audio action from `/5/1/1`.
2. Remove pre-shrink scale action and optional `MergeVFX` create/destroy events.
3. Restore original single tween action only.
4. Remove `sfx_merge.wav` from resources if added.

---

## Item 5 — Save and restore game state

### How to implement in the GDevelop visual editor

#### Part A — Enable Storage extension

1. **Project Manager → Extensions → Search "Storage".**
2. Add **Storage** (built-in) extension.

#### Part B — Define save schema

Use a single storage key: `"SoccerMerge_v02_save"`

| Field | Type | Source |
|---|---|---|
| `energy` | number | Global `Energy` |
| `lastEnergyTimestamp` | number | Global `LastEnergyTimestamp` |
| `maxLevelFound` | number | Event `/8` local `MaxLevelFound` |
| `items[]` | array | Each `MergeItem`: `level`, `x`, `y` |

**Promote `MaxLevelFound` to a global variable** (recommended) so both `/8` and save/load can access it without scope issues.

#### Part C — Save event

Add a new event group **`SaveSystem`** at the bottom of `Game Scene`:

**Trigger:** `Scene` → At scene end **OR** use **Timer** every 30 s + **Window is going to close** (platform-dependent; for web/mobile use periodic timer).

**Actions (pseudocode sequence in visual editor):**

1. `Storage → Write number in storage` → key `"energy"`, value `Energy`
2. `Storage → Write number in storage` → key `"lastEnergyTimestamp"`, value `LastEnergyTimestamp`
3. `Storage → Write number in storage` → key `"maxLevelFound"`, value `MaxLevelFound`
4. `Storage → Write number in storage` → key `"itemCount"`, value `Count(MergeItem)`
5. **For each** `MergeItem` (ForEach):
   - Write `item_N_level`, `item_N_x`, `item_N_y` using a scene variable index `SaveIndex` incremented each iteration

**Simpler v0.2 alternative:** Serialize items as one JSON string in a scene variable, then write one key `"boardJson"`. Use a **JavaScript code event** if ForEach indexing is cumbersome — document the JS in a comment either way.

#### Part D — Load event

Add to event **`/1`** (Scene begins) **after** energy init — or in a dedicated **`LoadSystem`** group run once at start:

1. `Storage → Exists in storage` → if false, skip load (first run uses scene-placed items).
2. Read `energy`, `lastEnergyTimestamp`, `maxLevelFound`.
3. **Delete all** existing `MergeItem` instances.
4. Loop `itemCount`: create `MergeItem` at saved x/y, set level, call `ApplyMergeItemVisuals`.
5. Run energy regen check immediately (event `/2` logic) so offline time is applied on load.

**Important:** Load must run **before** or **replace** event `/4` starter setup. Recommended order at scene start:

```
1. Load save (if exists) → populate board + globals
2. Else → run existing /4 starter placement
3. Apply visuals to all MergeItems
```

#### Part E — Conflict with event `/4`

Event `/4` currently hard-sets levels at fixed coordinates on every scene start. **Disable or wrap `/4`** in condition `SaveExists = false` so loaded boards are not overwritten.

### Affected event block / object

| Target | Change |
|---|---|
| New group `SaveSystem` | Periodic + on-close save |
| New group `LoadSystem` or extend `/1` | Restore on scene start |
| Event `/4` | Guard with "no save exists" condition |
| Event `/8` | `MaxLevelFound` promoted to global |
| All `MergeItem` instances | Deleted and recreated on load |
| Extension **Storage** | New dependency |

### Variables / resources required

| Name | Scope | Role |
|---|---|---|
| `Energy` | Global | Saved / restored |
| `LastEnergyTimestamp` | Global | Saved / restored |
| `MaxLevelFound` | Global (promoted) | Saved / restored |
| `SaveIndex` | Scene temp | ForEach write/read helper |
| `SaveExists` | Scene temp | Branch first-run vs returning player |
| Storage keys | Browser/local storage | Persisted data |

No art resources.

### Exact expected behavior

| Scenario | Expected result |
|---|---|
| First launch | Default board from scene editor; energy 100 |
| Merge, spawn, quit, reopen | Board items, levels, positions restored exactly |
| Energy was 47 with timestamp T | Restored; offline regen applied on load |
| Highest level on board was 5 | `LevelProgress` shows `Levels: 6 / 10`; `MaxLevelFound = 5` |
| Save corrupted / missing keys | Fall back to first-run defaults; no crash |

### Rollback procedure

1. Remove `SaveSystem` and `LoadSystem` event groups.
2. Remove Storage extension if unused elsewhere.
3. Remove `SaveExists` guard from `/4`; restore original scene-start behavior.
4. Move `MaxLevelFound` back to event-local on `/8` if desired.
5. Clear browser storage key manually for testing: `SoccerMerge_v02_save` (or delete via a debug action).

---

## Item 6 — Reset-save option for testing

### How to implement in the GDevelop visual editor

1. Add a **Text button** or **Sprite button** object: `ResetSaveButton`.
   - Place on `UI` layer, corner position (e.g. bottom-right).
   - Label: `"Reset Save"` (dev-only; hide in production builds later).
   - Add **ButtonFSM** behavior (same as `Producer`) for click detection.

2. Add event:

   **Conditions:** `ResetSaveButton` clicked  
   **Actions:**
   - `Storage → Delete from storage` → all keys used in Item 5 (or delete parent key)
   - `Storage → Delete from storage` → `"SoccerMerge_v02_save"` (if using monolithic key)
   - **Reload scene** (`Scene → Reload scene`) OR delete all `MergeItem` + reset globals to defaults matching `/1` and `/4`

3. **Reload scene** is preferred — guarantees clean state identical to first launch.

### Affected event block / object

| Target | Change |
|---|---|
| New object `ResetSaveButton` | UI layer, dev testing |
| New event `/ResetSave` | Clear storage + reload |
| Item 5 save keys | Deleted on reset |

### Variables / resources required

| Name | Role |
|---|---|
| Storage keys from Item 5 | Targets for deletion |
| `ButtonFSM` behavior | Click handling (reuse `ButtonStates` extension) |

Optional: small button sprite or plain text object.

### Exact expected behavior

| Action | Expected result |
|---|---|
| Tap Reset Save | Storage cleared; scene reloads |
| After reset | Default 6 starter items, energy 100, no saved board |
| After reset → merge → close → reopen | New save works independently |

### Rollback procedure

1. Delete `ResetSaveButton` object.
2. Delete reset event block.
3. No impact on production save logic if button is removed.

---

## Item 7 — Manual test checklist

Use this checklist after implementing Items 1–6. Record pass/fail and notes.

### Energy & producer

- [ ] **P1** — At 5 energy, click producer with free cell → energy becomes 4, one Level 0 item appears.
- [ ] **P2** — Surround producer on all 4 sides; click at 5 energy → energy stays 5, no item created.
- [ ] **P3** — At 0 energy, click producer → no spawn; warning appears (Item 2).
- [ ] **P4** — Warning fades within ~2 seconds without interaction.
- [ ] **P5** — At 0 energy, spam-click producer → warning re-triggers; no negative energy.
- [ ] **P6** — Wait for regen (or temporarily set `EnergyRegenMs` to 5000 for test) → energy increases, bar and text update.

### Merge mechanics

- [ ] **M1** — Drag two Level 0 items onto same cell → one Level 1 item; pop tween plays; SFX plays.
- [ ] **M2** — Drag Level 0 onto Level 1 same cell → items reject (snap back); no merge; no SFX.
- [ ] **M3** — Drop item on empty cell → stays put; position vars updated.
- [ ] **M4** — Merge to Level 9 → correct `Lv9` art and name **"World-Class Stadium"**.
- [ ] **M5** — `LevelProgress` updates after each new highest level discovered.

### Visuals & names (Levels 0–9)

- [ ] **V0** — Spawned item shows football art + "Football".
- [ ] **V1–V9** — Merge sequentially (use debug or pre-placed pairs) → each level distinct art and correct name.
- [ ] **V10** — Starter items on scene load show correct art for their preset levels (1, 1, 2).

### Save & load

- [ ] **S1** — Fresh install / after reset → default board, energy 100.
- [ ] **S2** — Move items, spend energy, merge once → close preview → reopen → exact board restored.
- [ ] **S3** — Note energy and timestamp → close for 5+ min (or lower regen ms) → reopen → energy increased correctly.
- [ ] **S4** — `LevelProgress` and `MaxLevelFound` match pre-close state.
- [ ] **S5** — Reset Save → returns to S1 state; new session saves independently.

### Regression

- [ ] **R1** — Drag still snaps to 80×80 grid.
- [ ] **R2** — UI layer elements visible above gameplay.
- [ ] **R3** — Two producers both spawn correctly.
- [ ] **R4** — No console errors on load/save/reset.

### Test setup tips

| Goal | How |
|---|---|
| Fast energy regen | Temporarily set global `EnergyRegenMs = 5000` during test |
| Fill producer adjacency | Spawn items manually into all 4 adjacent cells |
| Skip to Level N | Temporarily add debug sub-event: key press → create item at level N |
| Inspect storage | Browser DevTools → Application → Local Storage (HTML5 export) |

---

## Recommended implementation order

Execute in this sequence to minimize rework and keep rollback isolated:

```
1. Item 1  — Producer energy fix          (small, high-impact bug fix)
2. Item 2  — Energy warning               (pairs with Item 1)
3. Item 3  — Visuals + names              (largest content task)
4. Item 4  — Merge pop + sound            (depends on Item 3 merge path)
5. Item 5  — Save/load                    (must follow visuals apply logic)
6. Item 6  — Reset save                   (depends on Item 5)
7. Item 7  — Run manual test checklist    (verification gate)
```

---

## Pre-implementation backup (mandatory)

Before any v0.2 work in GDevelop:

1. Duplicate `game.json` → `game.json.v01-backup`
2. Export project as `game.zip.v01-backup`
3. Note current GDevelop version: **5.6.274**

**Full v0.2 rollback:** Restore backup file and re-open in GDevelop. Delete storage keys manually if save schema was tested.

---

## Out of scope for v0.2 (explicit)

Do **not** implement in this version:

- Club name, location, colors, stadium style chooser
- Team identity / player development systems
- Club Scene or facility building UI
- Second merge chain or producer types
- Orders / quests / monetization
- Levels 10–99 art or renaming
- Converting to folder project (recommended for v0.3, not required here)

---

## Version gate

**v0.2 is complete when:** all Item 7 checklist tests pass, Items 1–6 are implemented, and no changes were made to club customization systems (none exist yet).

**Next version preview (v0.3):** Club Scene stub, promote `MaxLevelFound` to unlock facility slots, folder project migration, data-driven name lookup without ten sub-events.
