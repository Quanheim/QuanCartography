# QuanCartography: design (brief, draft)

Status: draft, 2 Nov 2026. Nothing built. Valheim 1.0.21.

**Decided (2 Nov 2026):**
- **Every pin is shared and owned by the server.** There are no private pins: everybody's pins sync to everybody.
- **Pins close together are combined**, not duplicated, using a configurable distance.
- **Pins sync in near real time** to everyone online, and a player who joins gets the full, current set.

## Purpose

One map mod for the Quanheim server, replacing three:

| Mod | What we keep from it | What goes wrong with it |
|---|---|---|
| **BetterMap 1.1.0** (xtavim) | Creature, boat and cart icons; auto-pins for resources, dungeons, portals and traders; kept death markers; pin icon scale; smooth zoom. | Nothing major reported, but its auto-pins are per player, so on a shared map they'd duplicate. |
| **Expanded_Boat_Minimap 1.0.0** (lunarbin) | A bigger exploration radius on boats, per boat type. | The radius often doesn't update until you get off and back on the boat. Client-only, no server control. |
| **ServerSideMap 1.3.14** (Mydayyy) | Shared fog of war, saved on the server. This part works well. | **Shared pins are unreliable:** pins that never reach other players, and duplicates. See below. |

## Why ServerSideMap's pins fail

Read from its source (`PinSync.cs`, `InitialPinSync.cs`, `UtilityPin.cs`, `ExplorationDatabase.cs`, `SaveWorld.cs`,
`Store.cs`), checked against Valheim 1.0.21's `Minimap`.

### Pins that never reach other players

This matches what we see: Quan places a pin, can still see it every session, but Andy never gets it. In every case
below the pin stays saved on the creator's PC only, so the creator has no sign anything went wrong.

1. **Sharing is opt-in per player, and off by default.** `EnableMarkerShare` defaults to `false` in each player's
   own config. A player with it off silently neither uploads nor downloads pins, while everyone else carries on.
2. **Only one action uploads a pin:** confirming the pin's name box (`Minimap.OnPinTextEntered`). Never uploaded:
   - pins the game adds itself: vegvisir and boss locations, Hildir's maps and other discoveries
     (`Minimap.DiscoverLocation`);
   - death markers;
   - pins added by other mods, such as BetterMap's auto-pins and portal pins;
   - a hand-made pin whose name box is closed without confirming because the map closed under it
     (`Minimap.SetMapMode` clears the name box), for example when you're hit, die or teleport.
3. **No retry and no confirmation.** A pin is sent once. If the message is lost, or the client wasn't flagged as
   sharing yet (it only starts after receiving the server's pin list on join), nothing notices.
4. **The server only writes pins to disk when the world saves.** A crash or forced restart loses every pin added
   since the last save. Worse, once a pin is uploaded the creator's own copy is marked "don't save locally", so a
   pin the server loses disappears for its creator too.

### Duplicates and pins that won't delete

- **A shared pin has no ID and no owner**, only name, type, position and a ticked flag.
- **Every upload is stored as a new pin.** Nothing checks for one already there, so the same pin arriving twice
  (two players marking one copper vein, or `/convertpins`) is stored twice.
- **Deletes and ticks match on exact name, type and position.** A pin that differs by a rounding error can't be
  deleted on the server and comes back for everyone on the next join. Renames aren't synced at all.
- **Pins and fog share one save file** (`.explored`), which has been wiped to zero bytes once. A bad save takes the
  map with it.

## Goals

1. **Shared fog of war**, at least as good as ServerSideMap's, saved on the server.
2. **Every pin on the server, reliably, for everyone** (see Pins below).
3. **Exploration radius per way of travelling:** on foot, each boat type, carts and mounts. It takes effect the
   moment you board, change vehicle or get off, and the server sets the values.
4. **BetterMap's extras:** creature, tame and vehicle icons on map and minimap, auto-pins for resources, portals and
   traders, more pin types, kept death markers.
5. **Minimap and radar improvements:** see questions.
6. **Migrate what we have:** import the server's ServerSideMap `.explored` file (fog and pins) and every player's
   local pins, combining duplicates on the way in, so nobody loses anything.

## Out of scope (for now)

- Crossplay or console clients: every player runs the modpack.
- Anything the chunk loader shows on the map. Its observer pins belong to QuanChunkLoader (one job per mod).

## Pins

**One server list, one source of truth.**
- Every pin has a server-assigned ID, plus who created it, when, and who last changed it.
- Clients never keep pins of their own. A client's map shows exactly the server's list.
- **Every saved pin is shared,** whatever made it: placed by hand, discovered by the game (vegvisirs, bosses,
  Hildir's maps), or added by our auto-pins. Temporary pins (pings, shouts, your bed, raids) stay as vanilla has
  them.
- Players with the mod installed are always in. There's no per-player opt-out.

**Changes are requests with a receipt.**
- Add, rename, change icon, tick and delete each go to the server as a request. The server applies it, logs it, and
  broadcasts the change to everyone online within a second.
- The client keeps every request it hasn't had a receipt for and resends it, including after a disconnect or on next
  login. A pin can't silently fail to go up.
- Hand-made pins are sent the moment they're placed, then updated when named, so closing the map mid-name can't
  lose one.

**Combining.**
- When a new pin lands within the combine distance of an existing pin of the **same icon**, it's merged into that
  pin instead of added. Different icons never combine, so a portal next to a copper vein keeps both.
- The combine distance is a server setting, with a default and optional per-icon overrides (for example tighter for
  portals, wider for ore).
- The same rule runs over the import, and an admin command can re-run it over the whole map.

**Joining and staying in sync.**
- On join, the client gets the full list and a revision number.
- Each change carries the next revision. A client that sees a gap, or whose periodic check doesn't match the
  server's, re-downloads the list. So two players online together stay in step, and someone who joins later gets
  everything.
- `/pins` shows the client's pin count, revision and any unsent requests, so "is it synced?" has an answer.

**Saving.**
- Pins live in their **own** file, separate from fog. The server writes it within a few seconds of any change, not
  only at world save, as a temporary file renamed into place, with rolling backups.
- Deleted pins go to a bin for a set time (for example 7 days), so a mistaken or malicious delete can be undone by an
  admin.

## Exploration radius

Worked out every frame from what the player is in or on, not on a boarding event. That avoids the
Expanded_Boat_Minimap bug.

## Where I'd like your input

1. **Combine distance.** What default? ServerSideMap used 15 m for its duplicate check. Any icons that need their
   own distance?
2. **Names when combining.** When a pin is merged into an existing one with a different name, keep the existing
   name, keep the newer one, or join both ("Copper / big vein")?
3. **Who can edit or delete a pin?** With every pin shared, I'd suggest anyone can, with the server logging who and
   the bin as a safety net. Or only the creator and admins?
4. **Death markers.** Shared like everything else (useful for corpse runs with friends), or the one exception that
   stays personal?
5. **Radius values.** On foot, raft, karve, longship, drakkar, cart, and mounted (lox, horse mods). Any other vehicle
   mods on the server that need entries?
6. **Radar and minimap.** What should it show: creatures (hostile or tame), other players, boats, carts, your tames
   (overlaps QuanCompanions D4.5)? Any limits, for example other players only within a range? And what cosmetic
   changes did you have in mind: shape, size, zoom levels, a north lock, a compass ring?
7. **Auto-pin list.** Start from BetterMap's defaults (valuable ores, dungeons, vegvisirs, traders, portals) or your
   own list?
8. **The vanilla cartography table.** With everything shared, it has nothing left to do. Leave it as decoration, or
   disable reading and writing to it?
9. **Player positions.** Vanilla lets players hide their position on the map. Keep that choice as it is?
