*Your power grid should feel like infrastructure, not just a line of poles.*

TrueGrid adds voltage tiers, transformers, and real electrical mechanics to Satisfactory's power network. Power must now be stepped up for long-distance transmission, then stepped back down before it reaches the factory. Flesh out the Power Network first then the factory. And a single wire can no longer carry *200 gajillion MW across the entire map.*
___

## Core Features:

+ Voltage tiers limit how much power each circuit can carry.
+ Transformers and converters step power up for transmission and back down for distribution.
+ Plan your grid before you build, or pay for it later.
+ Space elevator tiers demand sustained high-voltage loads.
+ Follows Base Game Progression Tiers and unlocked via the Milestone Terminal.
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

---
Report Issues here: https://github.com/lazypsyco/TrueGrid/issues

---
Screenshots:

---
A Typical Power Setup 

![alt text](https://github.com/lazypsyco/TrueGrid/blob/5ecb12ca533f3b059143d5cfd1081074b483688e/Resources/Backbone.png?raw=true "A Setup Demonstrating Transmission from Power Production to Factory")

---
A Simple Transformer Network
![alt text](https://github.com/lazypsyco/TrueGrid/blob/main/Resources/Transformers.png?raw=true  "A Setup Demonstrating Transforming between LV MV and HV voltage tiers")

---
Phase Shift Management
![alt text](https://github.com/lazypsyco/TrueGrid/blob/main/Resources/phase%20Shift.png?raw=true "the ratio between Inductors to Capacitors for net 0MW Phase Shift")

---
Inductor Power Smoothing
![alt text](https://github.com/lazypsyco/TrueGrid/blob/main/Resources/Inductors.png?raw=true "Side by Side of the same grid with different inductor counts")
Distance Loss – longer cables lose more power.
Conversion Loss – transformers and converters consume energy.
Minimum Voltage Inefficiency – lightly loaded grids waste power.
