# Patches in this fork

## Why this fork exists
I play TrueGrid in multiplayer as a client on a friend's hosted game, and I ran into bugs
that only show up there. This fork fixes them in TrueGrid's own Blueprint assets, so each
fix is an ordinary change to a `.uasset`. I'm not the author of TrueGrid; the goal is to fix
what I can and keep each change small, documented and easy to review or adopt.

Fixes come from reading the mod's code first, then confirming the problem in game.

## Patches

| Patch | Asset changed | Released | Single player | Multiplayer |
|---|---|---|---|---|
| [scanner-local-only](scanner-local-only/README.md): the circuit scanner panel only shows for the player holding the scanner | `Content/Equipment/CircuitScanner/FGNetworkScanner.uasset` | 2026-11-01 (rev 2) | tested, works normally | not yet tested |

Status words used here:
- **not yet tested**: built, nobody has played with it yet.
- **tested, works normally**: played with it, and it behaved as intended.
- **tested, problem found**: see that patch's README.

## What each patch folder contains
- `README.md`: the full write-up.
  - The problem, as seen in game.
  - How the original code works, and why it was probably written that way.
  - The change.
  - Risks, test status, and history.
- `editor-steps.md`: how to make the same change by hand in the Unreal Editor.
- `patch.json`: the same change as data, for the headless patch tool I used.
- `diff.txt`: before/after of the changed Blueprint as readable pseudo-code. `.uasset` files are
  binary, so git can't show what changed in them; this file does.

## How patches are made
I edit headlessly on Linux, so I don't use the editor window. Each change is a `patch.json`
applied by my own tool, bp-patch. It is an editor-only Unreal plugin plus a script, and it isn't part
of this repo.

The tool:
1. checks that the Blueprint still looks exactly as the patch expects (otherwise it refuses);
2. applies the change;
3. compiles the Blueprint;
4. saves only if it compiles with zero errors.

The `diff.txt` is produced by exporting the Blueprint before and after and comparing the two.

Without the tool, the `editor-steps.md` route gives the same result in the Unreal Editor. Use
the same engine and SML versions as the mod (UE 5.6.1-CSS, SML 3.12.0).

## Built against
- TrueGrid 1.3.4, commit `59f320b`;
- SML 3.12.0;
- Unreal Engine 5.6.1.

A patch is written against one exact state of its asset. If upstream changes that asset, the
patch needs redoing against the new version.
