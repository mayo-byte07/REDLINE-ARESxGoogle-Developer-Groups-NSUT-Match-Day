# Team 1 — Garchomp / Salamence / Volcarona

This file documents the Team 1 configuration visible in the provided arena screenshot. The lineup is not the earlier Garchomp / Scizor / Gengar proposal; the screenshot-confirmed team is **Garchomp, Salamence, and Volcarona**, in that order.

## Team configuration

| Slot | Pokémon | Ability | Item | Nature | Intended role | Moves shown in the arena |
|---|---|---|---|---|---|---|
| 1 | Garchomp | Rough Skin | Life Orb | Jolly | Physical attacker and setup | Earthquake, Dragon Claw, Rock Slide, Swords Dance |
| 2 | Salamence | Intimidate | Life Orb | Naive | Mixed coverage and recovery | Draco Meteor, Flamethrower, Earthquake, Roost |
| 3 | Volcarona | Flame Body | Heavy-Duty Boots | Timid | Special setup sweeper | Fiery Dance, Bug Buzz, Giga Drain, Quiver Dance |

The moves, abilities, items, natures, species and slot ordering above are transcribed from the screenshot.

## EV-spread note

The screenshot shows preset labels beginning with **Physical Sweeper** for Garchomp and Salamence, and **Special Sweeper** for Volcarona. The labels and their exact EV allocations are truncated in the screenshot. Therefore, the EV values in [`team.txt`](team.txt) are conventional approximations of those presets, not a verified export of the portal's hidden preset values. If exact EV values matter for the event, open **Customize Team** and verify each preset in the live arena.

## Strategy overview

- **Garchomp** provides physical damage and a Swords Dance setup option.
- **Salamence** combines physical coverage with Draco Meteor and Flamethrower, with Roost for recovery. The selected Naive nature and Physical Sweeper preset are preserved from the screenshot even though this is a mixed-move set.
- **Volcarona** uses Quiver Dance to set up for a special sweep, with Heavy-Duty Boots as its selected item.

This is a proposed battle plan, not proof that the lineup is optimal. Test it against varied opponents and record match outcomes before reporting a win rate.

## Arena setup

1. Select Garchomp, Salamence, and Volcarona in slots 1–3.
2. Match the shown ability, item, nature and move choices wherever they are available in the arena's curated menus.
3. Verify the EV presets in the live UI because the screenshot clips their full labels.
4. Upload `redline_ai.py` separately in the arena's ML algorithm upload control.

`team.txt` is a human-readable Showdown-style record; do not assume the website imports this text file automatically.
