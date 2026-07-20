*Your power grid should feel like infrastructure, not just a line of poles.*

TrueGrid adds voltage tiers, transformers, and real electrical mechanics to Satisfactory's power network. Power must now be stepped up for long-distance transmission, then stepped back down before it reaches the factory. Flesh out the Power Network first then the factory. And a single wire can no longer carry *200 gajillion MW across the entire map.*
___

## Core Features:

+ Voltage tiers limit how much power each circuit can carry.
+ Transformers and converters step power up for transmission and back down for distribution.
+ Plan your grid before you build, or pay for it later.
+ Space elevator tiers demand sustained high-voltage loads.
+ Optional: power losses, conversion inefficiency, and AC phase management.

___
## Voltage Tiers:

Different voltage levels determine how much power a circuit can carry and how efficiently it travels long distances.

| Tier |	Name |Capacity | Distance Loss
|---|---|---:|:---:|
|_______|___________________|______________|____________________|
LV  | Low Voltage |	500 MW	| High
MV  | Medium Voltage	| 1,000 MW	| Medium
HV  | High Voltage	 | 5,000 MW | Low
SV  | Transmission | 30,000 MW	| Minimal
QV  | Quantum Voltage	| 60,000 MW	| Very High

Higher voltages are designed for long-distance transmission and high capacity loads, while lower voltages are intended for local factory distribution. QV carries more power than anything else in the grid. It just doesn't like to go very far.

Transformers allow power to be stepped between these tiers as needed.

How a typical grid flows: 

    Generators → SV Transmission → HV Backbone → Transformers & Converters → Machines
___


## Recommended...

For Players who enjoy:

+ Large-scale infrastructure projects
+ Complex factory logistics
+ Engineering-style problem solving
+ Making the power grid part of gameplay

#### Not Recommended...

For Players who prefer:

+ Simple vanilla power systems
+ Fast playthroughs
+ Minimal infrastructure management

Screenshots:

---
A Typical Power Setup 

![alt text](https://github.com/lazypsyco/TrueGrid/blob/5ecb12ca533f3b059143d5cfd1081074b483688e/Resources/Backbone.png?raw=true "A Setup Demonstrating Transmission from Power Production to Factory")

---
A Simple Transformer Network
![alt text](https://github.com/lazypsyco/TrueGrid/blob/main/Resources/Transformers.png?raw=true  "A Setup Demonstrating Transforming between LV MV and HV voltage tiers")

---
Phase shift management setups

WIP

Inductors

Capacitors







## Transformers:

Transformers connect different voltage tiers together. Higher voltages allow power to travel longer distances and support larger loads before requiring additional transformers.

Typical grid layouts might look like:

LV ⇄ MV ⇄ HV  (main factory grid)
HV ⇄ SV       (for long-distance transmission)
HV → QV       (for extremely high power systems)

Large factories will often use high voltage backbones with local step-down transformers.
___
## Power Converters:

Converters change electricity between circuit types. Different machines run on different current — match the converter to the machine.

For example:

Smelters require Direct Current (DC)
Constructors require Alternating Current (AC)

---
## Space Elevator Integration:

Each tier demands a sustained high-voltage load while consuming items. Higher tiers draw harder — plan your transmission capacity around it.

Supplying above the minimum threshold pays out bonus AWESOME Sink points. Overbuild your grid and it earns its keep.

Configurable Options:

Item Multiplier    - Game Phase item counts cost more per tick.
Game Phase Recipes - completely change the costs of each Tier.
and options for adding more items to the list.

___
## Complexity Options:

Optional mechanics for players who want deeper simulation.

#### Phase Shift (AC Systems)
Machines on an AC grid introduce reactive power, reducing the usable capacity of that circuit.
Additional buildings help manage this effect.

Capacitor:
Offsets Phase Shift on the circuit.
Improves usable capacity.

Inductor:
Stabilizes circuits with rapidly changing loads.
Increases total Phase Shift on the grid.

#### Power Loss:

Electrical systems are not perfectly efficient and this mod introduces several sources of power loss:

Distance Loss – longer cables lose more power.
Conversion Loss – transformers and converters consume energy.
Minimum Voltage Inefficiency – lightly loaded grids waste power.
