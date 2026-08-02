# True Grid Information

## Transformers:

Transformers connect different voltage tiers together. Higher voltages allow power to travel longer distances and support larger loads before requiring additional transformers.

Typical grid layouts might look like:
+ LV ⇄ MV ⇄ HV  (main factory grid)
+ HV ⇄ SV       (for long-distance transmission)
+ HV → QV       (for extremely high power systems)

___
## Power Converters:

Different machines now run on different Power Types. Converters change the circuit to the correct type to operate.

For example:
+ Smelters require Direct Current (DC)
+ Constructors require Alternating Current (AC)
+ Long Distance Power Transportation uses (SC)

[See full list here!](https://github.com/lazypsyco/TrueGrid/blob/main/Resources/Building%20Guide.md)

---
## Space Elevator Integration:

Each tier demands a sustained high voltage load while consuming items: higher tiers draw more power than the last.
Tickets are returned out of the back of the Space Eleveator. Simply sink the holder item to recieve the refund.

| Phase | Name | Power Requirment | Bonus AWESOME SINK points |
| --- | :--- | --- | --- |
| 1 | Distribution Platform | 300 MW | +10%
| 2 | Construction Dock | 1,500 MW | +20%
| 3 | Main Body | 5,000 MW | +30%
| 4 | Propulsion | 12,500 MW | +50%
| 5 | Assembly | 50,000 MW | +75%
| 6 | "Endless Mode"| 125,000 MW | +100%

Supplying above the minimum threshold pays out bonus AWESOME Sink points. You can set the threshold inside the Space Elevator GUI. 

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

+ Ineffecient Machines
  + There is an inherent inefficiency in all power conversions; consuming more power than is converted. 
  + Starts at 90% (increasing power consumption by 10%)
  + This effect can be reduced as a M.A.M. research.
+ Minimum Voltage Threshold
  + TrueGrid buildings require a minimum voltage to stay operational, regardless of what power is used.
  + Starts at 10% of the Maximum Conversion Rate of the building.
  + This effect can be reduced as a M.A.M. research.
+ Distance Loss 
  + Circuits have a max size and production capacity is reduced linearly the bigger it gets. (Circuit has 50% of max capacity at half the max range)
    + LV - 500 meter max
    + MV - 2,000 meter max
    + HV - 5,000 meter max
    + SV - 100,000 meter max
    + QV - 250 meter max