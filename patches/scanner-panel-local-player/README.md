# Scanner panel only for its own player

<!-- bp-patch write-up. Edit freely; nothing in this file is regenerated. -->

Patch `scanner-panel-local-player`, **rev 3**. Asset `/TrueGrid/Equipment/CircuitScanner/FGNetworkScanner`,
graph `EventGraph`. Written against TrueGrid 1.3.4, upstream commit `59f320b`.

Files:
- [patch.json](patch.json): the change.
- [editor.md](editor.md): the same change made by hand in the Unreal Editor.
- [diff.txt](diff.txt): pseudo-code before/after.
- [PR.md](PR.md): this text, for the pull request (it links to `diff.txt` and `editor.md`).

## Problem

In multiplayer, the circuit scanner's panel (`Widget_DebugScreen`, held in the variable `Debup`) is
shown and hidden on the wrong machines. Observed as a non-host client on a friend's listen server,
before any patch:

- when I equip my scanner, my panel also appears on the host's screen and stays there;
- when the host equips theirs, their panel appears on my screen and stays there;
- a stuck panel and a fresh one end up overlaid.

Single player is not affected, which is why it went unnoticed.

## Original code (upstream 59f320b)

Read from an export of the unmodified asset.

**Set-up.** `ReceiveBeginPlay` waits for valid subsystems, calls `Set Refs`, then runs a two-branch
Sequence. The `Set Refs` custom event (a Reliable Server RPC) runs **the same Sequence**.
- **Then 0** stores the subsystem, casts `GetOwner()` to `FGCharacterPlayer` and sets
  `Player Controller` from it. It then waits `Delay Until Next Tick`, sets `RCO Ref` and
  `RCO Ref.Circuit Scanner = self`, and **creates the panel unconditionally**:
  `Create Widget(Widget_DebugScreen, Owning Player = Player Controller)`, `Debup = …`, then hides the
  `Target` crosshair.
- **Then 1**: if the scanner is already equipped, `Wait For Player State`, then the equip decision
  below.

The Sequence runs on every machine that has a copy of the scanner. On the server it can run several
times per scanner: once from `ReceiveBeginPlay`, once more from the `Set Refs` call there (a Server RPC
called on the server runs immediately), and again when the owning client's own `Set Refs` arrives.

**Equip / unequip decision.** This is reached from `WasEquipped`, from `WasUnEquipped`, and from the
set-up's Then 1:

```
Is Dedicated Server ? Switch Has Authority[Remote]
                    : (Is Server ? Switch Has Authority[Authority] : Switch Has Authority[Remote])
  equip:   Debup.AddToViewport → DoOnce → Set HUD Visibility(true), Toggle Scanning(true) [Server RPC],
           Toggle Custom Wires(setting), debug log; then re-arm the unequip DoOnce
  unequip: Debup.RemoveFromParent → DoOnce → Set HUD Visibility(false), Toggle Scanning(false),
           Toggle Custom Wires(false), debug log; then re-arm the equip DoOnce
```

Each `Switch Has Authority` has exactly one output wired, as shown. Nothing else hangs off these six
nodes or off `Is Dedicated Server` / `Is Server`.

**Tick** works on `Debup` on every machine: screen-locator entries for problem locations and the
Circuit-0 locator. There is no `EndPlay`.

**Why it was probably written this way.** The decision asks *"am I the server?"*. That's the natural
question when testing in single player (host and owner are the same machine) and on a dedicated server
(where `Is Dedicated Server` routes to the client side). It is the wrong question on a listen server
with several players. Equipment is replicated to every machine that can see its holder, so the host's
copy of my scanner and my copy of the host's scanner both run `WasEquipped`, and every branch decides
"show". None of them asks whether the scanner belongs to the local player.

## Fix

Decide **once**, at panel creation, whether this scanner is the local player's (**C3**). Everywhere
else, act only where the panel exists (`IsValid(Debup)`). Node references are bp-patch GUID prefixes.

- **C3 · create the panel only for the local player.** In the set-up Sequence, after
  `Set RCO Ref.Circuit Scanner` (`1FAB4FD0`), two Branches:
  1. `IsValid(Player Controller)`, then
  2. `Player Controller.IsLocalController`.

  Only if both pass does it reach `Create Widget` (`412FCDFD`) and `Set Debup`. Both False outputs
  continue at the existing `Target.SetVisibility(false)` (`F3CCAB74`), so the rest of the set-up is
  unchanged.
  - A player's `PlayerController` exists only on the server and on that player's own machine. So the
    first test fails on other clients, and the second fails on the host for a remote player's scanner.
  - This also stops the server from creating widgets during its extra passes through the Sequence.
- **C1 · equip.** The Branch that decided on `Is Dedicated Server` (`FB573060`) now decides on
  `IsValid(Debup)`. True goes straight to `Add to Viewport` (`B79EA23E`); False goes nowhere. This also
  covers the set-up's "already equipped" path: if that path runs before the panel exists, it now skips
  instead of calling `Add to Viewport` on None.
- **C2 · unequip.** The same at the unequip Branch (`FC2E064E`), leading to `Remove from Parent`
  (`9E399337`).
- **C1+C2 clean-up.** Removed: the two `Is Server` Branches, the four `Switch Has Authority` macros, then
  the unconnected `Is Dedicated Server` and `Is Server`.
- **C4 · Tick.** Everything after `Event Tick` runs only if `IsValid(Debup)`. Every node in Tick works
  on the panel, so nothing else is skipped.
- **C5 · new `Event End Play`.** If `IsValid(Debup)` **and** `EndPlayReason == Destroyed`:
  `Debup.RemoveFromParent`, then `Set HUD Visibility(false)`. A scanner destroyed while equipped (death,
  dropping it) takes its panel and the buildings' sky text with it. On world teardown (quit, leaving,
  level change) the viewport goes anyway, so the overlap and `Show HUD` calls are skipped.

Not touched: the scan itself, `Toggle Scanning` (a Reliable Server RPC), custom wires, `Set Refs`, and
`Circuit Scanner Info`.

## What changes per machine

| Machine / scanner | Before | After |
|---|---|---|
| Single player, or the host's own scanner | panel; equip runs locally | same |
| Host, a remote player's scanner | **equip runs on the host:** panel attempt, sky text and wire colours change on the host's screen, and the host starts the scan timer locally | nothing on the host's screen; the scan is started by the owner's Reliable `Toggle Scanning` RPC |
| Client, own scanner | panel; equip runs | same |
| Client, another player's scanner | **panel shows on the wrong screen**; its `Toggle Scanning` RPC is dropped (not the owner) | nothing |
| Dedicated server | equip: nothing (Remote only); but `Create Widget` was called for every scanner | nothing, and no widget is created |

## Risks and side effects

- **Scan start for remote scanners.** The server-side scan for a remote player's scanner is now started
  and stopped only by that player's `Toggle Scanning` RPC, which is Reliable. Dedicated servers already
  depended on this path. A destroyed scanner takes its server timer with it.
- **Start-up race (pre-existing).** The set-up's Then 0 creates the panel after
  `Delay Until Next Tick`, while Then 1 waits `Wait For Player State` and then shows the panel if the
  scanner is already equipped. Both are latent. If Then 1 resumes first, the panel is not shown until
  the next equip; rev 2 gave an `Accessed None` in that case. It is expected to be fine, because the
  delay is registered first.
  - Covered by the tests "load a save while holding it" and "respawn with it equipped".
  - Structural fix if a test fails: run the "already equipped?" check after `Set Visibility` in Then 0,
    instead of in a parallel branch.
- **Owner not resolved at `BeginPlay`.** If `GetOwner()` isn't resolved yet, the `Cast To FGCharacterPlayer`
  fails and no panel is created. This is the same as the original, whose `Create Widget` sat inside the
  same cast's success path.
- **Unpatched machines** in a session still show stray panels. SML's join check enforces matching
  versions as soon as the fork's `.uplugin` version differs from upstream's.
- **Not verifiable offline.** The pins of `EqualEqual_ByteByte`, including the `EndPlayReason` (enum) →
  byte link, can only be checked in Unreal, by `tg check` or `tg apply`. `IsLocalController` was already
  applied in Unreal by rev 2.
- **Pre-existing and untouched:**
  - other players' scanner copies call `Set Refs` (dropped with a warning) and read `RCO Ref` on None;
  - `Circuit Scanner Info` reads `Debup` without a check (safe: it is only reached through the owner's RCO).
- **Cosmetic:** some added nodes sit inside the "DEBUG" and "fuse break map hud icon" comment boxes.

## Status

rev 3:
- **Lint:** 0 problems against the pristine export of `59f320b` (2 warnings: functions whose pins
  can't be checked offline).
- **`tg check` and `tg apply` passed in Unreal (2026-11-04).** In-game testing: see Testing.
- **EventGraph size:** 252 → 266 nodes predicted (rev 2: 270, which matches what Unreal produced).
- **`diff.txt` is Unreal's**, written by `tg apply`. Before applying, it was predicted offline (bp-patch's
  model + renderer; the method reproduced the Unreal-applied rev 2 exactly).

## Testing

**Single player:** equip → panel shown; unequip → hidden; die while equipped → gone. Rev 2 passed these
on 2026-11-01; repeat them for rev 3.

**Host + client** (pending):
- each player equips and unequips; both equipped at once;
- the client's readout updates while holding the scanner;
- die, swap tool, leave and rejoin while equipped;
- load a save while holding the scanner; respawn with it equipped (start-up race);
- quit to menu while holding it: no `Show HUD` / `Set HUD Visibility` log lines;
- neither log has `Accessed None … Debup`.

**Expected, and not a failure of this patch:**
- On a client, the screen locator stays empty, and the problem boxes and cosmetic wire colours don't
  show. They read server-only subsystem state (`Problem Helper Locations`, `Infected Machines`,
  `Circuits`) and the client's empty fuse-helper mesh.
- That is a separate issue.

## History

- **rev 1 (2026-11-01):** "is this mine" = `GetInstigatorCharacter → IsLocallyControlled`. Single player
  showed no panel: `GetInstigatorCharacter` is null at `BeginPlay` (before equip), so the panel was
  never created. Reverted.
- **rev 2 (2026-11-01):** the `Player Controller` variable → `IsValid` → `IsLocalController`, repeated
  at create, equip and unequip. 89 ops. Single player OK.
- **2026-11-02:** moved to the fork's `patches/` layout (bp-patch v0.13, deterministic node GUIDs).
- **rev 3 (2026-11-04):**
  - equip and unequip decide on `IsValid(Debup)` instead of repeating the controller test, so C3 is the
    single decision and the start-up race's `Accessed None` path is removed;
  - EndPlay clean-up only for `EndPlayReason == Destroyed`;
  - preconditions, clean-up, C3 and C4 unchanged from rev 2;
  - 83 ops;
  - all documents rewritten against the pristine export.
