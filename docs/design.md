# QuanCartography: design (brief, draft)

Status: draft, 2 Nov 2026. Nothing built. Valheim 1.0.21.

**Decided (2 Nov 2026):**
- **Every pin is shared and owned by the server.** There are no private pins: everybody's pins sync to everybody.
- **Pins close together are combined**, not duplicated, using a configurable distance.
- **Pins sync in near real time** to everyone online, and a player who joins gets the full, current set.
- **Anyone can edit or delete any pin**, as on a shared paper map. The server logs who did what, and deleted pins go
  to a bin an admin can restore from.
- **Exploration radius per vehicle is a config setting**, enforced by the server (see Exploration radius).
- **Map display matches BetterMap's:** extra pin rows in the legend, auto-pins filed under the right icon, and each
  player's hidden icons kept through a relog.
- **The vanilla cartography table is ignored.** It still works for anyone who uses it, but we don't hook into it or
  rely on it. Everything goes through the server.
- **Combining starts at 15 m** (ServerSideMap's figure), and the **first pin's name wins**. Both are tunable.
- **Pins can be renamed**, which vanilla can't do (see Renaming pins).
- **Death markers are shared**, so others can help with corpse runs, and are **tied to the tombstone** (see Death
  markers).
- **The minimap follows the big map's filters**: an icon hidden on one is hidden on the other.
- **Players keep vanilla's "hide my position" option**, but a server setting can force positions on, and it
  **defaults to forced on**: everyone's position is shared.
- **Sailing waypoints** are in scope (see Waypoints).
- **Radar and minimap features are still open.**

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
  deleted on the server and comes back for everyone on the next join.
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
6. **Migrate what we have** without losing anyone's pins (see Launch and migration).

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

**Anyone can change any pin.** Every player can rename, tick or delete any pin, whoever made it. The server logs each change with who made it, and a deleted pin goes to a bin for a set time (for example 7 days) so an
admin can restore it.

**Changes are requests with a receipt.**
- Add, rename, tick and delete each go to the server as a request. The server applies it, logs it, and
  broadcasts the change to everyone online within a second.
- The client keeps every request it hasn't had a receipt for and resends it, including after a disconnect or on next
  login. A pin can't silently fail to go up.
- Hand-made pins are sent the moment they're placed, then updated when named, so closing the map mid-name can't
  lose one.

**Combining.**
- When a new pin lands within the combine distance of an existing pin of the **same icon**, it's merged into that
  pin instead of added. Different icons never combine, so a portal next to a copper vein keeps both.
- The combine distance is a server setting: **15 m to start**, with optional per-icon overrides later (for example
  tighter for portals, wider for ore).
- When pins combine, **the first pin's name is kept.**
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

## Exploration radius

- Set in the mod's config file, one value per way of travelling. Gale's shared profile carries the file to every
  player, and the server also enforces its own values (admin-locked config sync, as our other mods do), so a player
  who edits their local copy can't explore further than everyone else.
- Worked out every frame from what the player is in or on, not on a boarding event. That avoids the
  Expanded_Boat_Minimap bug.
- **Every value starts at 100 m**, vanilla's on-foot radius (`Minimap.m_exploreRadius`), and is tuned upwards in
  play. The shipped defaults are an open decision for release. A rough direction from earlier discussion: boats
  scaling up with size (karve, then longship, then drakkar the widest).
- Entries: on foot, raft, karve, longship, drakkar. Carts, mounts and other vehicle mods can get entries later.

## Map display (parity with BetterMap)

Whatever we add must behave like BetterMap does today:
- **Extra pin types** each get a row in the map's pin legend, with an icon.
- **Auto-pins** are filed under the right icon for what was found (copper under its ore icon, a crypt under
  dungeons, and so on), using a mapping from each resource or location to an icon, grouped by biome as BetterMap's
  per-biome lists are.
- **Hiding is per player and persists.** Right-clicking a legend row hides that icon type on the map and minimap,
  and the choice survives a relog, so each player keeps their map as cluttered or as clean as they like. Hiding
  never touches the shared pins themselves.
- **Who decides what gets auto-pinned:** the server, because auto-pins are shared. Each player then chooses what
  they see. (BetterMap lets each player choose what gets pinned, which only works because its pins are private.)
- Creature, tame and vehicle icons, kept death markers, icon scale and smooth zoom carry over too.

## Renaming pins

Vanilla can name a pin only when it's placed. We add a rename: a gesture on an existing pin (for example
double-click, or a modifier with click; to settle in design) opens vanilla's own pin name box with the current name
filled in. Confirming sends a rename request to the server by the pin's ID, and everyone sees the new name.

## Death markers

- **Shared** like every other pin.
- **Cleared when the tombstone goes.** When a death creates a tombstone, the marker's pin ID is saved on the
  tombstone itself (saved data on the tombstone, which survives restarts, unlike an object ID). When the tombstone is
  emptied, by "take all" or item by item, the game removes it (`TombStone.UpdateDespawn`), and we remove its marker
  at the same moment. It works whoever empties it and wherever it has floated or been moved to.
- **Self-repairing:** the server also checks now and then for death markers whose tombstone no longer exists (for
  example removed by an admin or another mod) and clears them.
- **Deaths with no tombstone.** The game only makes a tombstone if you were carrying something and the world's
  "keep inventory" option is off (`Player.CreateTombStone`), but it adds a death marker every time
  (`Player.OnDeath`). A server setting, **Death markers only with a tombstone**, skips the marker when there's no
  tombstone. Off, it behaves like vanilla and the marker stays until someone deletes it.

## Minimap

- **Same filters as the big map.** Hiding an icon type on the map's legend hides it on the minimap too, and the
  other way round, kept through a relog. Each player sees the same amount of clutter in both places.
- **What vanilla already has** (from `Minimap`): the minimap shows fog of war (the same fog texture as the big map),
  a wind direction arrow, and it already zooms with the map zoom keys, between tight limits. Fog rarely shows simply
  because the minimap can't zoom out past the area you've just uncovered.
- **Our additions:** a wider zoom range, so fog and more of the surroundings come into view, with a scale readout in
  fixed steps (World of Warcraft style, for example 32 m, 64 m, 128 m). Wind stays.
- Look-and-feel options (size, round or square, north-up or rotating) are an open decision: we'll look at what's
  possible and judge how cluttered it gets.

## Player positions

Vanilla's "hide my position" toggle stays, but a server setting, **Force position sharing**, overrides it. It
defaults to **on**: every player's position shows on everyone's map. Server owners who prefer vanilla behaviour turn
it off.

## Waypoints

A sailing target that stays in view. Vanilla's map ping (middle click) shows "Quandru: ping" on the map and in the
world for a few seconds, then it's gone. A waypoint is the same idea but **persistent until someone removes it**.

- **A waypoint pin type.** Labelled with who set it, e.g. "Quandru's waypoint".
- **Placing one (proposed):** pick the waypoint icon in the map's pin type selector, then double-click the map, the
  same way as any other pin. One change from normal pins: a waypoint skips the name box, so laying out a route is
  just a string of double-clicks. It's numbered automatically ("Quandru's waypoint 3"). Pick another icon to go back
  to normal pins.
- **Routes.** Waypoints you place in a row form a route, numbered in the order placed.
- **Reached means removed.** When you come within a set distance of a waypoint (a setting, to tune for boat speed),
  it disappears. Sail from one to the next and the route clears itself behind you.
- **Distance on screen.** Each waypoint's marker in the 3D view shows its distance, and the next one on your route
  (or the nearest, if you're not following a route) is highlighted, so you can always turn towards it.
- **As many as you like.** No per-player limit: they manage themselves because they expire.
- **Short-lived.** Waypoints are only for the journey in hand, so they expire: all are cleared on a server restart,
  and each also expires after a set time (1 hour to start). Both are settings.
- **Shown in the 3D view, not just on the map:** a marker over the spot with its name and distance, which slides to
  the edge of the screen with an arrow when the spot is behind you or off screen, so you can steer by it.
- **Local by default; sharing is a server option.** Shared waypoints show for everyone. Anyone can delete one, and
  people will clear them as they finish with them, since nobody wants a cluttered view.
- Each player can switch the in-view markers off.
- **Is it doable?** Yes. Projecting a world position onto the screen and clamping it to the edges is a standard
  technique, and vanilla already draws ping text at a world position. No existing mod we know of does it.

## Vanilla cartography table

Ignored, as most map mods do. Anyone can still use it, but QuanCartography doesn't hook into it or depend on it.
One side effect worth knowing: reading a table adds its pins to that player's map like any other saved pin, so they
join the shared map and get combined with what's already there. That's harmless, and no different from someone
placing the same pins by hand.

## Launch and migration

Today's pins are spread over three places: ServerSideMap's server file (incomplete, because not everyone has been
uploading), and each player's own PC (anything ServerSideMap never sent). QuanCartography gathers both:

1. **Import the server file.** On first start, the server reads ServerSideMap's `.explored` file: fog and pins.
2. **Collect each player's local pins.** The first time each player joins with QuanCartography, their client uploads
   every pin saved on their PC. Those are exactly the pins ServerSideMap never sent, because ServerSideMap stops
   saving a pin locally once it has uploaded it. A player who joins weeks later still brings theirs in.
3. **Combine.** Everything goes through the normal combine rule, so the same copper vein marked by three players
   ends up as one pin.

Optional, before switching over: each player turns `EnableMarkerShare` on in ServerSideMap's config and types
`/convertpins` (the plain form, not `ignorelocaldupes`, which skips any pin within 15 m of a server pin of **any**
icon, so a portal beside a copper pin would be lost). That gets more pins onto the server early, but step 2 covers
anyone who doesn't.

**A local copy as well.** ServerSideMap's `/downloadpins` converts the server's pins back into local pins, so a player
can keep their own copy before the switch (handled separately). Also back up the server's
`<world>.mod.serversidemap.explored` file before switching: the importer reads it directly.

## Where I'd like your input

Open, to come back to when we build:
- **Radar:** what it shows (creatures, players, boats, carts, your tames, which overlaps QuanCompanions D4.5) and any
  range limits.
- **Minimap look and feel:** size, round or square, north-up or rotating, zoom steps.
- **Auto-pin list:** what gets pinned automatically.
- **Radius defaults** for release.
- **How renames are triggered** from the map.
- **Shared waypoints when reached:** with sharing on, does reaching a waypoint remove it for everyone, or only for
  the player who reached it?

Everything else in the earlier list has been decided (see the top of this doc).
