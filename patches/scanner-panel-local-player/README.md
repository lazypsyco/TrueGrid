# Scanner panel only for its own player

<!-- bp-patch v0.13 write-up. Edit freely; nothing in this file is regenerated. -->

Patch `scanner-panel-local-player`, rev 2. Asset `/TrueGrid/Equipment/CircuitScanner/FGNetworkScanner`, graph `EventGraph`.
Files: [patch.json](patch.json) (the change), [editor.md](editor.md) (the same change in the Unreal Editor),
[diff.txt](diff.txt) (pseudo-code before/after; written by `bp-patch.sh apply` / `rebuild`).

## Problem

The circuit scanner's panel (`Widget_DebugScreen`, variable `Debup`) is shown and hidden on the wrong
machines in multiplayer. Observed as a non-host client on a friend's listen server, before any patch:

- when I equip my scanner, my panel also appears on the host's screen and stays there;
- when the host equips theirs, their panel appears on my screen and stays there;
- a stuck panel and a fresh one end up overlaid.

Single player is not affected, which is why the author would not have seen it.

## Original code and why it was done that way

`FGNetworkScanner`'s `ReceiveBeginPlay` creates the panel unconditionally (`CreateWidget` with the
scanner's `Player Controller` as owner) and `Set Visibility(false)`. The equip decision (reached from
`WasEquipped`, and from `BeginPlay` when already equipped) is

```
IsDedicatedServer ? SwitchHasAuthority[Remote] : (IsServer ? SwitchHasAuthority[Authority] : SwitchHasAuthority[Remote])
  -> Debup.AddToViewport, Set HUD Visibility(true), Toggle Scanning(true) (Server RPC), Toggle Custom Wires
```

and unequip mirrors it with `RemoveFromParent` and the `false` calls. `Tick` reads `Debup` on every
machine; there is no `EndPlay`.

Why it was probably written this way: the decision asks *"am I the server?"*, which is the natural
question when the code was tested in single player (host and owner are the same machine) and on a
dedicated server (where `IsDedicatedServer` routes to the client side). It is the wrong question for a
listen server with several players: equipment is replicated to every machine that can see the holder
(owner relevancy), so the host's copy of *my* scanner and my copy of *the host's* scanner both run the
same `WasEquipped` and both decide "show". The `IsServer`/`SwitchHasAuthority` combination picks a
branch on every machine; none of the branches asks whether the panel belongs to the local player.

## Fix

Replace "am I the server?" with "is this scanner mine?", using data the original code already relies on:
the scanner's own `Player Controller` variable (set from the owning character before the RCO lookup and
the widget creation). A player's `PlayerController` exists only on the server and on that player's own
machine, so `IsValid(Player Controller)` is false on other clients and `Player Controller.IsLocalController`
is false on the host for a remote player's scanner.

- **C1** equip decision: `IsValid(Player Controller)` → `IsLocalController` → `AddToViewport` etc.
- **C2** unequip decision: same test → `RemoveFromParent` etc.
- **C1+C2 clean-up**: the two `IsServer` Branches, the four `Switch Has Authority` nodes,
  `IsDedicatedServer` and `IsServer` are removed.
- **C3** `BeginPlay`: the panel is only created when the test passes; both `else` paths continue at the
  existing `Set Visibility` so the rest of the set-up is unchanged.
- **C4** `Tick`: the per-frame panel work runs only when `Debup` is valid (it now exists only on the
  owning machine).
- **C5** new `Event End Play`: if `Debup` is valid, `RemoveFromParent` and `Set HUD Visibility(false)`,
  so a scanner destroyed while equipped (death, leaving) takes its panel and the building HUD with it.

Everything else (the scan itself, `Toggle Scanning` Server RPC, custom wires) is untouched.

## Risks and side effects

- The scan for a remote player's scanner is now started and stopped only by that player's
  `Toggle Scanning` Server RPC (the host no longer also starts it locally). Dedicated servers already
  rely on this path. If the RPC is dropped, the readout never updates: test "readout updates while you
  hold it" in multiplayer.
- If `GetOwner()` were not yet resolved at `BeginPlay` on the owning client, `Player Controller` would be
  None and no panel would be created (the original would have created an owner-less one). Not seen.
- `EndPlay` runs on world teardown too (quit to menu), doing a 200 m box overlap and `Show HUD` calls on
  buildings being destroyed. Worst case: log lines. Could be restricted to `EndPlayReason == Destroyed`.
- Unpatched machines in a session still show stray panels. Everyone in a session must run the patched
  build; SML's join check (both sides require the remote's version to be >= their own) enforces this as
  soon as the fork's `.uplugin` version differs from upstream's.
- Pre-existing and untouched: other players' scanner copies still call `Set Refs` (dropped by the host
  with a warning) and read `RCO Ref` on None.
- One new Branch (`IfThenElse_13`) sits inside the "fuse break map hud icon" comment box. Cosmetic.

## Status

applied (rev 2), tested in single player: panel shows on equip, hides on unequip, gone after death. Multiplayer not yet tested.

## Testing

Single player (done, rev 2, 2026-11-01): equip → panel shown; unequip → hidden; die while equipped → gone.

Host + client (pending):
- each equips and unequips; both equipped at once;
- the client's readout updates while holding the scanner;
- die, swap tool, leave/rejoin while equipped;
- the host's log has no `Accessed None … Debup`.

## History

- rev 1 (2026-11-01): "is this mine" = `GetInstigatorCharacter → IsLocallyControlled`. Applied and built;
  single player showed no panel at all: `GetInstigatorCharacter` is null at `BeginPlay` (before equip),
  so C3 never created the panel. Reverted.
- rev 2 (2026-11-01): `Player Controller` variable → `IsValid` → `IsLocalController`. Single player OK.
- 2026-11-02: moved to the fork's `patches/` layout (bp-patch v0.13, format 2, deterministic node GUIDs).
  Same 89 operations as the tested rev 2; the asset is rebuilt from the pristine copy so its node GUIDs
  match the patch (the graph is identical, the bytes are not).
