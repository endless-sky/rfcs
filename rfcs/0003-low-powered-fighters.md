| RFC                  | 3                     |
| -------------------- | :-------------------- |
| Feature Name         | Low Powered Flight    |
| Affected audience    | Game Devs and Players |
| RFC PR               | [PR-3][PR-3]          |
| Relevant issues/RFCs | [samrocketman AI overhaul #6726][ES-6726],<br />[low powered flight #6724][ES-6724],<br />[Carriers pick up fighters #4241][ES-4241] |

---



# Summary

### Benefits of battery powered flight

Low powered flight is intended to add more versatility to fighter builds.  This
enables players to have a more diverse outfit configuration without changing the
fighter limitations on weight and outfit space.

By removing the requirement of power generation the outfit space can be used for
alternative outfits, instead.  For example, a dagger could fit mining lasers and
a cargo expansion while still keeping shield.

For min-maxing outfit space a player can equip a low powere generation outfit
which means overall worst case battery drain is negative and recharge is
extremely slow.  The benefit is the fighter would get slightly extended battery
flight time and the fighter should still return to the carrier.

### Limitation of battery powered flight

In the ember wastes, an ion storm can completely wipe out a battery powered
squadron instantly.  For this reason, the entire escort fleet should participate
in recharging fighters and escort carriers should actively pick up their own
fighter complement.

Fighter flight time is restricted by amount of reserve power available.

### Non-suicide fighters

One of the main reasons to avoid using boxwings is they're vulnerable in combat.
For the use case of mining, it is beneficial for a battery powered boxwing to
avoid deployment if there are hostile enemies nearby.  More generically, if a
fighter has no weapons equipped it should avoid deploying or even return to the
parent carrier if hostiles enter space.

Fighters should avoid deploying if there is not adequate power for the fighter
to reasonably engage.  In the original [samrocketman AI overhaul][ES-6726], `10
seconds` of worst case power usage is the metric used for reasonable flight and
was settled on after multiple tweaks through play testing.

### Other comments

Visually, the AI for fighter recovery and return for recharge is appealing and
interesting to watch the more number of fighters and carriers are involved.

# Motivation

This RFC is to help me avoid wasting time.

This adds a huge gameplay element which enables more diverse configuration and
visually appealing AI behavior.

Low powered fighters have been play tested cumulatively over 1000 hours.  The
following quality of life features should be packaged into the same pull request
for low powered flight.  Otherwise, the gameplay experience for low powered
fighters will be poor without any one element.

# Detailed Design

Due to calculation limitations the Ship class needs to pre-calculate and cache
calculated values in order to limit performance drawbacks on the engine.  The
way I handled this in [samrocketman AI overhaul][ES-6726] is by periodically
calling a function to update ship and fleet state from AI.cpp.  The function to
update ship state lives in Ship.cpp.

The overall implementation will be similar to [samrocketman AI
overhaul][ES-6726] less other unrelated concepts (for example without carrier
tankers fleet refueling AI behavior since it is not related to battery power).

### Features to be included

- No power gen: Player ship, escorts, fighters, and drones can be powered only
  by batteries.  Equip fighters with battery and they return to carrier for
  recharge.  Other ships will hail for recharge.
- Low power gen: Low powered fighters (small regen and lots of battery) also
  return to carriers for recharge due to speed and efficiency of recharge.
- Ships out of battery will become disabled and can call for help for a
  recharge.  This includes escorts and the player flagship.
- Battery powered fighters and drones will automatically return to carriers to
  recharge.  This includes low powered fighters which have battery and small
  amounts of energy generation.
- Battery powered fighters sharing energy with other ships during recovery
  operations reserve enough energy to be able to return to parent carriers.
  This enables battery powered fighters and drones to be effective when aiding
  in disabled ship recovery.
- Fighters and drones can recover other ships and drones (including battery
  powered fighters and drones).  This is useful when mining in the ember wastes
  with battery powered fighters and drones.
- Carriers will recover their own carried ships which were disabled during
  battle.
- All ships (including carriers) will recover disabled fighters and drones which
  are not carried by them.  This enables fighters associated with destroyed
  carriers to find a new home in a new carrier with empty bays.
- "No suicide pact" for defenseless fighters AI updates
  - Fighters will refuse to launch if they have no weapons and there are enemies
    in the system.
  - Fighters will retreat and re-dock with carrier if they have no weapons and
    enemies enter the system.
  - Minimum 10 second flight time.  Fighters will refuse to deploy if they do
    not have sufficient energy for 10 seconds of flight time overall.  This
    includes worst case scenario of using weapons the entire time.
    - Vulnerable fighters are less vulnerable in battle (like boxwings) because
      they stay docked.
    - Fighters will only deploy if their outfits allow them sufficient flight
      time in battle.

# Drawbacks

Battery power calculations are expensive and have the potential to limit fleet
sizes.  I've done my best to address all performance concerns in [samrocketman
AI overhaul][ES-6726] within the limitations of the current attribute lookup
design of the game engine.  Ideally, a deeper refactor of engine performance
should address engine performance issues aside from this feature.

Performance issues were encountered and addressed via the above method.

# Alternatives

None as far as I know

# Unresolved Questions

This will still be a large PR even when broken up.  I need consensus/agreement
before I am willing to take on this work.  As mentioned earlier low powered
fighters without any of these features leaves a poorer quality gameplay
experience.

The original [samrocketman AI overhaul][ES-6726] has over 20 minor features but
generally 5 or so high level concept features.  My preference for breaking up
that PR is to break up by concept.  This RFC is an example of breaking out the
concept of low powered/battery powered fighters and ships.

Is this acceptable?

[ES-4241]: https://github.com/endless-sky/endless-sky/issues/4241
[ES-6724]: https://github.com/endless-sky/endless-sky/issues/6724
[ES-6726]: https://github.com/endless-sky/endless-sky/pull/6726
[PR-3]: https://github.com/endless-sky/rfcs/pull/3
