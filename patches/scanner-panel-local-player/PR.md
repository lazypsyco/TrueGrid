# Scanner panel only for its own player


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

## Change (pseudo-code diff of the Blueprint graphs)

```diff
diff -ru before/graphs/EventGraph__EventGraph.txt after/graphs/EventGraph__EventGraph.txt
--- before/graphs/EventGraph__EventGraph.txt	2026-11-02 19:58:55.291943605 -0700
+++ after/graphs/EventGraph__EventGraph.txt	2026-11-02 19:58:55.417818497 -0700
@@ -13,7 +13,13 @@
 // COMMENT "set text" covers: ForEachLoop, GetArrayItem_6, MakeArray_7, Text
 // COMMENT "run scan" covers: Array_Add, Building Ref, Check Building, Check Network, Circuit Id, CustomEvent_6, ExecutionSequence_4, MakeArray_1, MakeStruct_0, Map_Find, Scan Building, Toggle Hide, TrueGrid Network Server
 // COMMENT "cosmetic wires" covers: CustomEvent_5, DynamicCast_1, DynamicCast_3, ExecutionSequence_5, ForEachLoop, GetLinearColorOptionValue, GetMaterial, GetSubsystem_1, GetWireInstanceMeshes, K2_GetComponentScale, Multiply_LinearColorLinearColor, SelectInt, Select_2, Select_3, Select_4, Select_5, SetVectorParameterValue, SetWorldSca
-// COMMENT "fuse break map hud icon" covers: Array_LastIndex, Array_Length, BP_CreateEntry, CompareInt, CustomEvent_9, Delay, Event_4, ExecutionSequence_6, ExecutionSequence_7, ExecutionSequence_8, ForEachLoop, ForLoop, Gate, GetAllEntries, GetArrayItem_1, GetArrayItem_2, Greater_IntInt, IfThenElse_8, K2_GetActorLocation, Macro, RemoveEntry, S
+// COMMENT "fuse break map hud icon" covers: Array_LastIndex, Array_Length, BP_CreateEntry, CompareInt, CustomEvent_9, Delay, Event_4, ExecutionSequence_6, ExecutionSequence_7, ExecutionSequence_8, ForEachLoop, ForLoop, Gate, GetAllEntries, GetArrayItem_1, GetArrayItem_2, Greater_IntInt, IfThenElse_13, IfThenElse_8, IsValid, K2_GetActorLocatio
+
+EVENT ReceiveEndPlay(EndPlayReason:EEndPlayReason)
+  if IsValid(Object=Debup)
+    [then]
+      Debup.RemoveFromParent()
+      Set HUD Visibility(Visible=false)
 
 EVENT Set Cosmetic Wire Polish(Wire Ref:FGBuildable, Polish Voltage Tier:Enum_VoltageTier, Polish Power Type:Enum_PowerType)
   $DynamicCast_1#1 = Cast<Build_PowerLine_C>(Wire Ref)
@@ -30,32 +36,34 @@
               $ForEachLoop#0.Array Element.SetWorldScale3D(NewScale_X=$ForEachLoop#0.Array Element.K2_GetComponentScale().ReturnValue_X, NewScale_Y=Select(NewEnumerator0=1.000000, NewEnumerator1=1.000000, NewEnumerator2=2.000000, NewEnumerator3=4.000000, NewEnumerator5=4.000000, NewEnumerator6=4.000000, Index=Polish Voltage Tier), NewScale_Z=Select(NewEnumerator0=1.000000, NewEnumerator1=1.000000, NewEnumerator2=2.000000, NewEnumerator3=4.000000, NewEnumerator5=4.000000, NewEnumerator6=4.000000, Index=Polish Voltage Tier))
 
 EVENT ReceiveTick(DeltaSeconds:real)
-  Debup.ScreenLocator.SetVisibility(InVisibility=Select(Option 0=ESlateVisibility::Hidden, Option 1=ESlateVisibility::HitTestInvisible, Index=(Array_Length(TargetArray=TrueGrid Subsys.Problem Helper Locations) > 0)))
-  sequence
-    [then_0]
-      if (Array_Length(TargetArray=TrueGrid Subsys.Problem Helper Locations) > 0)
-        [then]
-          sequence
-            [then_0]
-              CompareInt#7(Input=Array_Length(TargetArray=TrueGrid Subsys.Problem Helper Locations), CompareWith=Array_Length(TargetArray=Debup.ScreenLocator.GetAllEntries()))
-                [>]
-                  ForLoop#9(FirstIndex=1, LastIndex=(Array_Length(TargetArray=TrueGrid Subsys.Problem Helper Locations) - Array_Length(TargetArray=Debup.ScreenLocator.GetAllEntries())))
-                    [LoopBody]
-                      $BP_CreateEntry#85 = Debup.ScreenLocator.BP_CreateEntry()
-                      Set_Add(TargetSet=Screen Locator Popups, NewItem=$BP_CreateEntry#85.ReturnValue)
-                [<]
-                  ForLoop#8(FirstIndex=1, LastIndex=(Array_Length(TargetArray=Debup.ScreenLocator.GetAllEntries()) - Array_Length(TargetArray=TrueGrid Subsys.Problem Helper Locations)))
+  if IsValid(Object=Debup)
+    [then]
+      Debup.ScreenLocator.SetVisibility(InVisibility=Select(Option 0=ESlateVisibility::Hidden, Option 1=ESlateVisibility::HitTestInvisible, Index=(Array_Length(TargetArray=TrueGrid Subsys.Problem Helper Locations) > 0)))
+      sequence
+        [then_0]
+          if (Array_Length(TargetArray=TrueGrid Subsys.Problem Helper Locations) > 0)
+            [then]
+              sequence
+                [then_0]
+                  CompareInt#7(Input=Array_Length(TargetArray=TrueGrid Subsys.Problem Helper Locations), CompareWith=Array_Length(TargetArray=Debup.ScreenLocator.GetAllEntries()))
+                    [>]
+                      ForLoop#9(FirstIndex=1, LastIndex=(Array_Length(TargetArray=TrueGrid Subsys.Problem Helper Locations) - Array_Length(TargetArray=Debup.ScreenLocator.GetAllEntries())))
+                        [LoopBody]
+                          $BP_CreateEntry#85 = Debup.ScreenLocator.BP_CreateEntry()
+                          Set_Add(TargetSet=Screen Locator Popups, NewItem=$BP_CreateEntry#85.ReturnValue)
+                    [<]
+                      ForLoop#8(FirstIndex=1, LastIndex=(Array_Length(TargetArray=Debup.ScreenLocator.GetAllEntries()) - Array_Length(TargetArray=TrueGrid Subsys.Problem Helper Locations)))
+                        [LoopBody]
+                          Debup.ScreenLocator.RemoveEntry(EntryWidget=Debup.ScreenLocator.GetAllEntries()[Array_LastIndex(TargetArray=Debup.ScreenLocator.GetAllEntries())])
+                          Set_Remove(TargetSet=Screen Locator Popups, Item=Debup.ScreenLocator.GetAllEntries()[Array_LastIndex(TargetArray=Debup.ScreenLocator.GetAllEntries())])
+                [then_1]
+                  $Set_ToArray#91 = Set_ToArray(A=Screen Locator Popups)
+                  ForEachLoop#10(Array=$Set_ToArray#91.Result)
                     [LoopBody]
-                      Debup.ScreenLocator.RemoveEntry(EntryWidget=Debup.ScreenLocator.GetAllEntries()[Array_LastIndex(TargetArray=Debup.ScreenLocator.GetAllEntries())])
-                      Set_Remove(TargetSet=Screen Locator Popups, Item=Debup.ScreenLocator.GetAllEntries()[Array_LastIndex(TargetArray=Debup.ScreenLocator.GetAllEntries())])
-            [then_1]
-              $Set_ToArray#91 = Set_ToArray(A=Screen Locator Popups)
-              ForEachLoop#10(Array=$Set_ToArray#91.Result)
-                [LoopBody]
-                  Macro#16(Entry Ref=$ForEachLoop#10.Array Element, World Location=TrueGrid Subsys.Problem Helper Locations[$ForEachLoop#10.Array Index])
-    [then_1]
-      Gate#15(bStartClosed=true) [Enter]
-      Macro#14(Entry Ref=Debup.CircuitID0, World Location=Set_GetItemByIndex(TargetSet=TrueGrid Subsys.Infected Machines, Index=0).K2_GetActorLocation())
+                      Macro#16(Entry Ref=$ForEachLoop#10.Array Element, World Location=TrueGrid Subsys.Problem Helper Locations[$ForEachLoop#10.Array Index])
+        [then_1]
+          Gate#15(bStartClosed=true) [Enter]
+          Macro#14(Entry Ref=Debup.CircuitID0, World Location=Set_GetItemByIndex(TargetSet=TrueGrid Subsys.Infected Machines, Index=0).K2_GetActorLocation())
 
 EVENT Enable Circuit 0 Polish()
   sequence
@@ -109,17 +117,25 @@
           DelayUntilNextTick()
           RCO Ref = Player Controller.GetRemoteCallObjectOfClass(inClass=RCO_TrueGrid_C)
           RCO Ref.Circuit Scanner = self
-          $CreateWidget_1 = CreateWidget(Class=Widget_DebugScreen_C, OwningPlayer=Player Controller)
-          Debup = $CreateWidget_1#1.ReturnValue
-          Target.SetVisibility(bNewVisibility=false, bPropagateToChildren=false)
+          if IsValid(Object=Player Controller)
+            [then]
+              if Player Controller.IsLocalController()
+                [then]
+                  $CreateWidget_1 = CreateWidget(Class=Widget_DebugScreen_C, OwningPlayer=Player Controller)
+                  Debup = $CreateWidget_1#1.ReturnValue
+                  Target.SetVisibility(bNewVisibility=false, bPropagateToChildren=false)
+                [else]
+                  -> goto SetVisibility
+            [else]
+              -> goto SetVisibility
     [then_1]
       if IsEquipped()
         [then]
           WaitForPlayerState(PlayerController=Player Controller)
-          if IsDedicatedServer()
+          if IsValid(Object=Player Controller)
             [then]
-              Switch Has Authority#29() [Execute]
-                [Remote]
+              if Player Controller.IsLocalController()
+                [then]
                   Debup.AddToViewport(ZOrder=-1)
                   DoOnce#13(Start Closed=false)
                   sequence
@@ -132,14 +148,6 @@
                           LogDisplay(Str="TURN OFF SCANNER DEBUG")
                     [then_1]
                       DoOnce#25(Start Closed=false) [Reset]   (other input of a macro shown elsewhere)
-            [else]
-              if IsServer()
-                [then]
-                  Switch Has Authority#30() [Execute]
-                    [Authority]
-                      -> goto AddToViewport
-                [else]
-                  -> goto Switch Has Authority
 
 EVENT Toggle Scanning(Enable:bool)  [Server Reliable]
   if Enable
@@ -197,10 +205,10 @@
 
 EVENT WasUnEquipped()
   Parent::WasUnEquipped()
-  if IsDedicatedServer()
+  if IsValid(Object=Player Controller)
     [then]
-      Switch Has Authority#28() [Execute]
-        [Remote]
+      if Player Controller.IsLocalController()
+        [then]
           Debup.RemoveFromParent()
           DoOnce#25(Start Closed=false)
           sequence
@@ -211,14 +219,6 @@
               -> continues at IfThenElse_7 (shown under ReceiveBeginPlay)
             [then_1]
               DoOnce#13(Start Closed=false) [Reset]   (other input of a macro shown elsewhere)
-    [else]
-      if IsServer()
-        [then]
-          Switch Has Authority#31() [Execute]
-            [Authority]
-              -> goto RemoveFromParent
-        [else]
-          -> goto Switch Has Authority
 
 EVENT Update Cosemetic Wires()
   ForEachLoop#6(Array=TrueGrid Subsys.Powerline Actors)
diff -ru before/graphs.txt after/graphs.txt
--- before/graphs.txt	2026-11-02 19:58:55.454939948 -0700
+++ after/graphs.txt	2026-11-02 19:58:55.460827648 -0700
@@ -1,4 +1,4 @@
-  EventGraph   EventGraph  (252 nodes)
+  EventGraph   EventGraph  (270 nodes)
   Function     UserConstructionScript  (1 nodes)
   Function     Scan Building  (73 nodes)
   Function     Make Text  (1 nodes)
```

The same change as Unreal Editor steps: `patches/scanner-panel-local-player/editor.md` in the fork. Made with bp-patch (headless Blueprint graph patches).
