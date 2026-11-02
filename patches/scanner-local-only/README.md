# scanner-local-only: the circuit scanner panel only shows for its own player

**Asset:** `Content/Equipment/CircuitScanner/FGNetworkScanner.uasset` (Blueprint `FGNetworkScanner`, Event Graph)

**Released:** 2026-11-01, rev 2

**Built against:** TrueGrid 1.3.4 (`59f320b`), SML 3.12.0, UE 5.6.1

## Status

| Where | Status |
|---|---|
| Single player | tested, works normally (2026-11-01) |
| Multiplayer, listen server (host + client) | not yet tested |
| Dedicated server | not yet tested |

## Summary
In multiplayer, scanner panels show up on other players' screens and get stuck there. That
happens because the scanner never asks "does this scanner belong to the player at this
machine?". This patch adds that check:
- every place that shows, hides, creates or updates the panel now uses it;
- the panel is removed when the scanner goes away.

The scan itself is unchanged.

## The problem, as seen in game
- When I equip my scanner, its panel appears on the host's screen and stays open for him.
- When the host equips his, his panel appears on mine and stays open for me.
- If a stuck panel is showing and I open my own, mine overlays on top.

## How the original works
A copy of every player's scanner exists on each machine that can see that player: the
holder's own, the host's, and other nearby clients'. Every copy runs the same start-up, equip
and unequip code.

- **Start-up (BeginPlay):**
  - it gets the holder's Player Controller from the scanner's owner;
  - it creates the panel widget (`Debup`) for that controller;
  - it asks the server to run the same setup again (`Set Refs`).
- **Equip / start-up decision:** it decides by machine type:
  - dedicated server: nothing;
  - host: the "Authority" path;
  - client: the "Remote" path.

  Either path adds the panel to the screen, then turns on the floating text of TrueGrid
  machines within 200 m and starts the scan on the server.
- **Unequip:** the same decision, then removes the panel and turns everything off.
- **Every frame (Tick):** updates the panel's on-screen markers. There's no check that the panel
  exists.
- **Nothing when the scanner is destroyed:** there's no End Play, so a panel can outlive its scanner.

So on the host, my scanner's copy takes the host path. On my machine, the host's scanner's
copy takes the client path. There, the host's controller doesn't exist, the panel is created
for whoever is local (me), and it's shown. Nothing removes it later.

## Why it was probably written this way
"Is this the server / do I have authority" is a very common stand-in for "is this mine". It
is correct as long as a scanner exists only on its holder's machine and the server. It also
gives the right result in single player and on a dedicated server, so the bug only shows with
a host plus a client.

The start-up code and the equip code both show the panel on purpose: on a client, the scanner
can already be equipped by the time it starts up.

## The change (five parts)
**"Is this mine?"** means two Branches in a row:
1. the scanner's own `Player Controller` variable is valid;
2. that controller is the local one (`Is Local Controller`).

The original code already sets that variable before these decisions, and already uses it for
the panel's owner. Other players' controllers don't exist on your machine, so the first check
is false there. On the host, they exist but aren't local, so the second check is false.

| Part | Where | Change |
|---|---|---|
| C1 | Equip / start-up decision | "is this mine?" replaces the machine-type checks. Yes → add the panel and start everything, as before. |
| C2 | Unequip decision | The same, in front of removing the panel. |
| C1+C2 | Old branches | The `Is Server` branches, the four `Switch Has Authority` nodes, and `Is Dedicated Server` / `Is Server` are removed. |
| C3 | Panel creation | The panel is only created when "is this mine?" is true; otherwise it skips to the existing next step. |
| C4 | Every frame | The Tick work only runs if the panel exists. |
| C5 | New End Play | If the panel exists: remove it and turn the floating text off. |

**Not changed:**
- the scan (still runs on the host every 0.25 s, started and stopped by the holder's machine);
- `Set Refs`, `Toggle Hide`, and input handling;
- the show/hide DoOnce pair.

**Result per machine:**

| Machine | Before | After |
|---|---|---|
| Single player | panel shows | panel shows (same) |
| The holder's own machine | panel shows | panel shows (same) |
| Another client, for my scanner | panel could appear and stick | nothing |
| Host, for a client's scanner | show/hide sequence ran, floating text toggled, log errors every frame | nothing |
| Dedicated server | nothing | nothing (same) |

**Everyone in a session needs the patched version:** each machine's copy only fixes what
appears on that machine's screen.

## Risks and things to watch
- **Starting and stopping the scan for a client's scanner.** It now depends only on that
  client's request to the host. Before, the host's copy also started it locally. It's the same
  path the original already relies on with dedicated servers. **Test:** as a client, the
  readout updates while you look at buildings.
- **Player Controller not set at start-up.** If the scanner's owner weren't set yet when it
  starts up on your own machine, no panel is created until the scanner is re-equipped. That's
  unlikely: the original's own network setup uses the same value.
- **Quitting to the menu:** End Play also runs there, and briefly searches nearby buildings to
  hide their floating text while the world shuts down. At worst that adds log lines.
- **Mixed versions:** an unpatched player in the session can still show stray panels for others.
- **Known cosmetic issue:** in the editor, one new Branch sits inside the "fuse break map hud
  icon" comment box. It only affects how the graph looks, not behaviour.
- **Left as it was:**
  - other players' copies still send `Set Refs` (the host drops it with a warning);
  - start-up could in theory try to show the panel one frame before it exists.

  Both are unchanged from the original.

## Testing

| # | Test | Expect | Result |
|---|---|---|---|
| 1 | Single player: equip, unequip | panel shows, hides | works normally |
| 2 | Single player: die while equipped | panel gone | works normally |
| 3 | Host equips, unequips | only the host sees his panel | not yet tested |
| 4 | Client equips, unequips | only the client sees theirs; host's floating text doesn't flicker | not yet tested |
| 5 | Both equipped | each sees exactly one panel, their own | not yet tested |
| 6 | Client looks at TrueGrid buildings | readout updates | not yet tested |
| 7 | Die / swap tool / leave and rejoin while equipped | no panel left on any screen | not yet tested |
| 8 | Host's `FactoryGame.log` | no `Accessed None ... Debup` lines | not yet tested |

## How to apply it
Both routes start from the original asset at `59f320b`.

**Headless (bp-patch, from the folder with `sf-modenv.sh`):**
```bash
./sf-modenv.sh export ./TrueGrid
./bp-patch/bp-patch.sh lint  <patch.json> <that export>      # 89 op(s), 0 problem(s)
./bp-patch/bp-patch.sh check <patch.json> --source ./TrueGrid
./bp-patch/bp-patch.sh apply <patch.json> --source ./TrueGrid
./sf-modenv.sh pull ./TrueGrid
./sf-modenv.sh export ./TrueGrid
./bp-patch/bp-patch.sh diff <before export> <after export> /TrueGrid/Equipment/CircuitScanner/FGNetworkScanner
./sf-modenv.sh build ./TrueGrid
```
The diff should match `diff.txt`.

**In the Unreal Editor:** follow `editor-steps.md`, then Compile (no errors) and Save.

## History
- **rev 1 (not released):** "is this mine?" used `Get Instigator Character -> Is Locally
  Controlled`. That broke the panel everywhere, including single player. The scanner has no
  instigator until it's equipped, and the panel is created at start-up, before that. So no panel
  was ever created.
- **rev 2 (2026-11-01):** uses the scanner's own `Player Controller`, the reference the original
  code already relies on. Tested in single player: works normally.
