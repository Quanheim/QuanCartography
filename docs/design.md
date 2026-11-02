# QuanCartography: design (brief, draft)

Status: first draft, 2 Nov 2026. Nothing agreed or built. Valheim 1.0.21.

## Purpose

One map mod for the Quanheim server, replacing three:

| Mod | What we keep from it | What goes wrong with it |
|---|---|---|
| **BetterMap 1.1.0** (xtavim) | Creature, boat and cart icons; auto-pins for resources, dungeons, portals and traders; kept death markers; pin icon scale; smooth zoom. | Nothing major reported. Its auto-pins are per player, so on a shared map they'd duplicate. |
| **Expanded_Boat_Minimap 1.0.0** (lunarbin) | A bigger exploration radius on boats, per boat type. | The radius often doesn't update until you get off and back on the boat. Client-only, no server control. |
| **ServerSideMap 1.3.14** (Mydayyy) | Shared fog of war, saved on the server. This part works well. | **Shared pins are unreliable:** duplicates, and pins that go missing. See below. |

## Why ServerSideMap's pins fail

From its source (`PinSync.cs`, `UtilityPin.cs`, `ExplorationDatabase.cs`):
- **A shared pin has no ID and no owner.** It's only name, type, position and a checked flag.
- **The server matches pins by exact equality** of name, type and position. Deleting or ticking a pin only works if all three match exactly, so a pin whose position drifted by a float rounding, or that was renamed, can't be deleted on the server. It comes back for everyone on the next join.
- **Every add is stored as-is.** Nothing stops the same pin arriving twice (a pin made while sharing was toggled, `/convertpins`, two players marking the same thing), so duplicates pile up.
- **Renaming a pin isn't synced at all.**
- **Pins and fog share one save file** (`.explored`), which its own page says has been wiped to zero bytes once. A bad pin save takes the map with it.

## Goals

1. **Shared fog of war** that works at least as well as ServerSideMap's, saved on the server.
2. **Shared pins that never duplicate or go missing.** Every pin has a server-assigned ID and an owner. Add, rename, move, tick and delete are all by ID. The server is the only source of truth, and clients rebuild from it on join.
3. **Exploration radius per way of travelling:** on foot, each boat type, carts and mounts. It takes effect the moment you board, change vehicle or get off, and the server sets the values.
4. **BetterMap's extras:** creature, tame and vehicle icons on map and minimap, auto-pins for resources, portals and traders, more pin types, kept death markers.
5. **Minimap and radar improvements:** see questions.
6. **Migrate what we have:** import the server's existing ServerSideMap `.explored` file, fog and pins, so nobody loses their map, de-duplicating pins on the way in.

## Out of scope (for now)

- Crossplay or console clients: every player runs the modpack.
- Anything the chunk loader shows on the map. Its observer pins belong to QuanChunkLoader (one job per mod).

## How it would work (outline)

- **Everyone must have it:** a server part (storage, IDs, broadcasting) plus a client part (map UI, radius, icons). Version-checked like our other mods.
- **Fog:** clients send newly explored map squares; the server merges them (union) and broadcasts. On join the client gets the server's full map. Stored in its own save file beside the world.
- **Pins:** the client sends a request (add, rename, move, tick, delete); the server applies it, assigns or checks the ID, and broadcasts the result. Pins are stored in their **own** file, separate from fog, written safely (temporary file then rename) with a rolling backup.
- **Auto-pins:** created once on the server for everyone, merged by type and distance, so two players finding the same copper vein make one pin. A deleted auto-pin is remembered and not recreated.
- **Exploration radius:** worked out each frame from what the player is in or on, not on a boarding event. That avoids the Expanded_Boat_Minimap bug.

## Where I'd like your input

1. **Who sees which pins?** Everything shared, or a choice per pin (shared or private), with private as the vanilla default? Should players be able to share with a group rather than everyone?
2. **Who can delete or edit a shared pin?** Only its owner (plus admins)? Or anyone, with the server logging who did it?
3. **Which pins come over from ServerSideMap?** All of them, with duplicates merged? Or start clean and only keep fog?
4. **Radius values.** On foot, raft, karve, longship, drakkar, cart, and mounted (lox, horse mods). Any other vehicle mods on the server, such as extra ships, that need entries?
5. **Radar and minimap.** What should it show: creatures (hostile/tame), other players, boats, carts, your tames (overlaps QuanCompanions D4.5)? Any limits, for example other players only within a range? And what cosmetic changes did you have in mind: shape, size, zoom levels, a north lock, a compass ring?
6. **Auto-pin list.** Start from BetterMap's defaults (valuable ores, dungeons, vegvisirs, traders, portals) or your own list?
7. **The vanilla cartography table.** Leave it working alongside, disable it, or make it a place to choose which of your private pins to share?
8. **Player positions.** Vanilla lets players hide their position on the map. Keep that choice as it is?
