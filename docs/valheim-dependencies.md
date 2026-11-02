# Valheim Code Dependency

Everything this mod relies on from Valheim, so a game update can be checked quickly. `check-mods.ps1` in the
`.decompiles` repo reads the **Game code** tables and flags any entry the game changed. See "Valheim updates" in
`docs/DEVELOPMENT.md` for the whole check.

**Checked against Valheim:** 1.0.21 (2026-11-02)

Update this file in the same commit as any code that patches, calls or relies on something new in the game, and
update the version line above each time the mod passes the Valheim update check.

## Game code

The first column holds the game class or member in backticks, as `Class` or `Class.Member`. Nested types use their
parent's name, e.g. `Player.PlacementStatus`. Only the first column is read by the script; the rest is for people.

### Harmony patches

What the mod patches. A changed signature stops the patch applying; changed logic can make it wrong.

| Game code | Patch | What it does | If the game changes it |
|---|---|---|---|

### Private members reached by reflection

Fields, properties and methods read or called through `AccessTools`, `Traverse` or `System.Reflection`. These fail
at runtime, not compile time, if renamed.

| Game code | Kind | Used for |
|---|---|---|

### Public game code called or read

Methods, fields and components used directly. A rename or signature change breaks the build, which is easy to spot;
a change in behaviour isn't, so list the ones whose behaviour the mod depends on.

| Game code | Used for |
|---|---|

### Behaviour the mod assumes

Things the game does that the mod counts on without calling them, e.g. "only the owner runs
`Pickable.UpdateRespawn`". These break silently, so name the code where the behaviour lives.

| Game code | Assumption |
|---|---|

## Prefabs and assets

Not game code, so no script can check these. They come from the game's asset bundles: vanilla prefabs the mod
clones or reads, child objects it finds by name, materials, shader properties and layer names. Test them in game
after each update.

| Prefab or asset | Used for | Breaks if |
|---|---|---|
