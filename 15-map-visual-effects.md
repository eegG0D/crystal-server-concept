# Map Visual Effects

Per-map ambient effects: fire / lightning damage zones, weather particles (Leaves, Rain, Snow, Fog…), light setting (Dawn/Day/Evening/Night), tinted overlays. All set in the Map Info editor; the client renders them client-side based on map flags.

## What it does

Three layers of effects:

1. **Damage zones.** `MapInfo.Fire` and `MapInfo.Lightning` are bool flags; when true, the server periodically deals area damage to players on the map using `FireDamage` / `LightningDamage` values. Visual: client renders falling embers / sparks particle effects.
2. **Weather particles.** `MapInfo.WeatherParticles` is an enum (`None / Leaves / Rain / Snow / Fog / FireyLeaves / Petals`) read by the client and applied as a sprite-emitter overlay on top of the map render.
3. **Lighting.** `MapInfo.Light` is a `LightSetting` enum (`Dawn / Day / Evening / Night / Snow / Lightning`) that drives the client's color grading and ambient lighting math.

## Where to find it

- **Map editor:** Server.exe → Database → Map. Each map row has Fire / Lightning checkboxes, FireDamage / LightningDamage ushorts, Weather dropdown, Light dropdown.
- **Map info storage:** `Server\MirDatabase\MapInfo.cs` — these fields are serialized in the map block of `Server.MirDB`.
- **Server-side damage tick:** `Server\MirEnvir\Map.cs` — `Process()` runs the Fire/Lightning damage at the configured rate.
- **Client rendering:** `Client\MirScenes\GameScene.cs` — `Draw()` reads `CurrentMap.Light` and `CurrentMap.WeatherParticles`, dispatches to `Settings.Effect` particle emitters.

## How to configure (per map)

| Field | What it does |
|---|---|
| **Light** | `Dawn` / `Day` / `Evening` / `Night` / `Snow` / `Lightning`. Affects ambient color + particle palette. |
| **WeatherParticles** | Visual-only emitter. Doesn't deal damage. |
| **Fire (bool)** | Periodic area fire damage to all players on the map. |
| **FireDamage (ushort)** | Damage value per tick. |
| **Lightning (bool)** | Periodic area lightning damage. |
| **LightningDamage (ushort)** | Damage per strike. |
| **Music (ushort)** | Background music track index (drives the client's MP3 / WAV select). |
| **NoTeleport / NoEscape / NoRecall etc.** | Map flags that gate teleport mechanics. |

## How it works under the hood

- Server-side fire/lightning is in `Map.Process()`:
  ```csharp
  if (Info.Fire && Envir.Time >= FireTime) {
    foreach (var player in Players) {
      // pick random spot near each player
      // queue DelayedAction → RangeDamage
    }
    FireTime = Envir.Time + Random.Next(3000, 15000);
  }
  ```
  Lightning is identical with `Lightning` and `LightningTime`.
- The `WeatherParticles` enum value rides on the `S.ObjectMap` / map-info packet to the client. The client's emitter system uses that value to pick which particle library to spawn.
- The `Light` setting drives the client's color blend math during `Draw()`.

## How to make a "haunted forest" map (example)

1. Open Map editor on your forest map.
2. Set `Light = Night`.
3. Set `WeatherParticles = Fog` (or `FireyLeaves` for a dramatic look).
4. Set `Music = X` (pick a creepy track index).
5. (Optional) `Fire = true`, `FireDamage = 5` → small ambient damage adds tension.
6. Save DB. Restart server. Players entering the map see the darkened, foggy ambience and take light fire damage.

## Common pitfalls

- **Fire/Lightning damage stacks with `NoFight` flag.** A `NoFight` map with `Fire=true` still damages players — `NoFight` only blocks player-to-player combat. Don't set `Fire=true` on safe-zone maps.
- **Lightning + Fire on the same map** is allowed but feels chaotic. Pick one.
- **WeatherParticles changes the framerate** on slow GPUs. Test with the heaviest emitter (`Rain`) on your minimum-spec target before shipping.
- **Light = Night during siege** makes archers nearly invisible. Document this in the conquest design or use `Lightning` (lighter darkness + periodic flashes) instead.

## Quick smoke test

1. Pick a test map, set `WeatherParticles = Snow`, save.
2. Restart server, walk onto the map.
3. Snow should fall.
4. Toggle `Lightning = true, LightningDamage = 10`, save, restart.
5. Walk onto the map for 30s — should take a couple of small lightning hits with on-screen sparks.
