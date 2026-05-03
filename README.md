# Real Cat Runner
<img align="left" width="50%" src="https://github.com/kubrvk/RealCatRunner/blob/main/Content/RealCatRunner/images/catbanner.jpg"/>
<h3> <a href="https://play.google.com/store/apps/details?id=com.Kubrick.RealCatRunner"><img src="https://img.shields.io/badge/Google_Play:-com.Kubrick.RealCatRunner-000000?style=flat-square&logo=google-play&logoColor=white&labelColor=000000" height="25"/> </a></h3>

![](https://img.shields.io/badge/Mobile-0c5299?style=) ![](https://img.shields.io/badge/Platformer-759651?style=) ![](https://img.shields.io/badge/Touch--Controls-635196?style=) ![Blueprint](https://img.shields.io/badge/Blueprint-00599C?style=logo=c%2B%2B&logoColor=white)  ![Blueprint](https://img.shields.io/badge/Unreal_Engine_5.7-0E1128?style=for-the-badges&logo=unrealengine&logoColor=white)  ![Blueprint](https://img.shields.io/badge/Status-Shipped-success?style=for-the-badges) 
<br>
An endless auto-runner targeting Android. Control a cat through a procedurally generated track at ever-increasing speed. Built on responsive tap/swipe controls, real-time procedural generation, and a mobile optimization pipeline targeting 60fps on mid-range hardware.

The primary technical challenges were building a procedural chunk-based world that streams seamlessly at high velocities, implementing a touch input model that feels precise at increasing game speeds, and sustaining visual fidelity within aggressive mobile performance budgets. All gameplay systems, procedural generation, character controller, UI, and assets were developed by a single developer.
<br clear="left"/>
<p align="center">
<img src="https://play-lh.googleusercontent.com/B27x_iAvinUCyKGBYyf5LtYsGcOQljUr6QMmDSTr0dUqz8uA-85uiog5_a0ewqCNjNQLvxXK435CdQmiYbzvJZc=w5120-h2880-rw" width="25%"/><img src="https://play-lh.googleusercontent.com/ymVE0MT-UvHPTSJgTrDo7i4AD3LaTvskVnMftjXqGbMRnpP_Qjp7JQaZrXmDdGLx8h2n5bc-3AIxGiwVZLPydQ=w2560-h1440-rw" width="25%"/><img src="https://play-lh.googleusercontent.com/70-K5h4mLOzhAaSiRZGKvFQplZtzXJbw25KjFQIv-GXNk2Boi-9HRrifJ8HZmercMh8j6wEKqZvJ3aa-dAOxUQk=w2560-h1440-rw" width="25%"/><img src="https://play-lh.googleusercontent.com/ART04ZE12TzpDkqGrabdj_l5Y6j65fGf1LCI_B4IMde-brk2PXNpnhbqPOvmV4aoLN7P_HiNbxoSChg8oEqf3Xk=w2560-h1440-rw" width="25%"/>
</p>

---

## Technical Detail

| Layer | Technology |
|---|---|
| Engine | Unreal Engine 5.7 |
| Primary Language | Blueprint (gameplay, generation, systems) |
| Platform | Android (primary) |
| Input | Custom touch input component |
| Physics | Chaos , lane collision, obstacle interaction |
| Rendering | Mobile forward renderer, scalable quality tiers |
| Procedural Gen | Chunk-based streaming, seeded spawn tables |
| Animation | UE5 Animation Blueprint + `UAnimInstance` Blueprint subclass |
| Build | Android SDK/NDK, Gradle, UE5 Android packaging |
| 3D Pipeline | Blender, UE5 |

---


## 1. Auto-Run & Lane Movement System
<img src="https://play-lh.googleusercontent.com/B27x_iAvinUCyKGBYyf5LtYsGcOQljUr6QMmDSTr0dUqz8uA-85uiog5_a0ewqCNjNQLvxXK435CdQmiYbzvJZc=w5120-h2880-rw" width="100%"/>

> Blueprint-based implementation in Unreal Engine

The player character moves forward automatically at a speed controlled entirely by `BP_GameMode`. Lateral movement is constrained to discrete lanes — the player's only control axes are lane-switch (left/right), jump, and slide.

### Forward Movement

- `BP_LaneMovementSystem` drives the character forward each tick via direct position offset — not physics-based movement.
- `CurrentSpeed` is a replicated float variable on `BP_GameMode`, incremented over session time via a difficulty curve.
- Forward delta is computed inside the **Event Tick** node:
  ```
  NewLocation = CurrentLocation + (ForwardVector × CurrentSpeed × DeltaTime)
  ```
- Speed is applied directly to the character's world position using **Set Actor Location**, bypassing the **Character Movement Component's** `Max Walk Speed`. This avoids CMC pathfinding overhead and gives precise control over the forward axis without physics interference.

### Lane System

- The track is divided into `LaneCount` (default: `3`) discrete lanes, each defined by a `LaneOffset` float from center.
- `CurrentLane` (integer, range `0` to `LaneCount - 1`) is stored as a variable on the player Blueprint and tracks the active lane.
- On lane switch: the target lane offset is computed and the character's X position is interpolated from current to target over `LaneSwitchDuration` using the **FInterp To** node.
- Lane switches are blocked during: mid-air state, slide state, and obstacle collision recovery (managed via a boolean flag `bCanSwitchLane`).

### Jump

- A vertical impulse is applied via **Launch Character** node; gravity returns the character to the track.
- Jump height is configurable via a `JumpImpulseStrength` float variable.
- Double-jump is available as a power-up state tracked by `bDoubleJumpEnabled`.
- Landing detection is handled via the **On Landed** event; a brief landing animation blend is triggered on return.

### Slide

- On slide start: capsule half-height is reduced via **Set Capsule Half Height**.
- On slide end: capsule half-height is restored to its default value.
- Slide duration is fixed via `SlideDuration` float; early exit is available if a swipe-up gesture is received before the duration expires.
- A dedicated animation blend is played — low-body pose with maintained forward motion — driven by a **Blend Pose by Bool** node in the Animation Blueprint.

---

## 2. Touch Input System

Single-finger swipe and tap gestures map to all player actions. Gesture recognition is implemented inside `BP_TouchInputComponent` (an Actor Component Blueprint) that processes raw touch events from **Event Begin/End Touch** before passing intents downstream.

### Gesture Map

| Gesture | Action | Detection Condition |
|---|---|---|
| Swipe Left | Lane switch left | Horizontal delta > threshold, leftward |
| Swipe Right | Lane switch right | Horizontal delta > threshold, rightward |
| Swipe Up | Jump | Vertical delta > threshold, upward |
| Swipe Down | Slide | Vertical delta > threshold, downward |
| Tap | Jump (alt) | Duration < `TapMaxDuration`, displacement < `TapMaxDrift` |

### Swipe Detection Pipeline

1. **Event Begin Touch** → store `StartPosition` and `StartTime` as local variables.
2. **Event End Touch** → compute:
   - `Delta = EndPosition − StartPosition`
   - `Duration = EndTime − StartTime`
3. If `Vector Length (Delta) > SwipeMinDistance` AND `Duration < SwipeMaxDuration`:
   - Classify dominant axis: compare `Abs(Delta.X)` vs `Abs(Delta.Y)` via **Select** node — dominant axis wins.
   - Map axis + sign to gesture intent and fire the corresponding dispatcher.

### Input Buffering

- All gesture intents are buffered for `InputBufferWindow` (default: `5` frames) using a **Circular Buffer** array variable.
- Buffer is consumed on the first valid execution frame — essential for jump inputs slightly before landing and lane switches initiated during an in-progress lane transition.

---

## 3. Procedural Generation System

The world is generated at runtime as a continuous stream of chunks ahead of the player. `BP_ProceduralGenSystem` (a Game Instance Subsystem Blueprint equivalent, implemented as a persistent Actor) manages the chunk lifecycle: spawn, active window, and despawn/pool.

### Chunk Architecture

- `BP_ChunkBase` is the parent Blueprint class for all track segments — a fixed-length actor containing:
  - Ground Static Mesh components
  - Obstacle spawn point Scene Components (tagged by name)
  - Collectible spawn point Scene Components
  - Decoration placement Scene Components
- Chunk length is standardized via a `ChunkLength` constant to simplify lookahead calculation.
- Chunks are set up as parent Blueprints with tagged Child Actor sockets; obstacle and collectible placement is resolved at runtime via `PopulateChunk` — not baked into the chunk.

### Streaming Pipeline

```
[Active Chunk Window]
  Chunk N-1  →  behind player, queued for pool return
  Chunk N    →  current player position
  Chunk N+1  →  ahead — fully populated
  Chunk N+2  →  lookahead — being populated
  Chunk N+3  →  just spawned — empty, populating
```

- `SpawnLookahead` (default: `3` chunks) defines how many chunks are pre-generated ahead of the player.
- Each tick, `BP_ProceduralGenSystem` evaluates player progress within `CurrentChunk`. When the player crosses `ChunkTriggerThreshold` inside the chunk (checked via a **Float >=** comparison), a new chunk is spawned and populated at the front.
- Chunks behind the player beyond `DespawnDistance` are returned to the object pool.

### Object Pool

- `BP_ChunkPool` maintains a **TArray of BP_ChunkBase references** as a free list, organized per chunk type.
- **On pool request:** if free list is non-empty, dequeue the first element, reset its transform via **Set Actor Transform**, and re-enable it via **Set Actor Hidden in Game (false)** + **Set Actor Enable Collision (true)**.
- **On pool return:** hide the actor, clear all spawned obstacles/collectibles via a loop + **Destroy Actor**, and enqueue it back to the free list.
- The pool eliminates **Spawn Actor from Class** / **Destroy Actor** calls during active gameplay — critical for reducing GC pressure on mobile.

### Obstacle Placement

- Each chunk type references an `DA_ObstaclePlacement` Data Asset containing an array of `FObstacleSpawnConfig` structs per spawn socket tag. Each config specifies: obstacle Blueprint class, spawn probability, and difficulty tier range.
- On chunk acquisition, `PopulateChunk` iterates each spawn socket and rolls against probability weighted by the current `DifficultyTier` using a **Random Float** node vs adjusted probability.
- **Lane exclusivity:** obstacle placement validates via an `LaneMask` bitmask (integer bitwise operations) that no impassable combination is generated — at least one lane must always be clear per obstacle group.

### Biome System

- Chunk type selection is weighted by the current biome via a **Weighted Random** selection from `DA_BiomeConfig`.
- `DA_BiomeConfig` Data Asset defines: chunk class weights, obstacle set, decoration mesh set, Material Parameter Collection overrides, and ambient Niagara system reference.
- Biome transitions occur at distance thresholds defined in `DA_BiomeSequenceConfig`.
- Blend is handled per-chunk via a **Set Scalar Parameter Value (MPC)** lerp node across `BiomeTransitionLength` chunks.

---

## 4. Obstacle System

Obstacles are child Blueprints of `BP_ObstacleBase` placed by the procedural system. Each has: a collision Box Component, a `DA_ObstacleConfig` Data Asset reference, and an optional movement component for dynamic variants.

### Obstacle Categories

| Type | Lane Behavior | Player Response |
|---|---|---|
| Ground Block | 1–2 lanes blocked | Jump over or lane switch |
| Overhead Bar | Full width, low | Slide under |
| Side Wall | 1 lane blocked | Lane switch |
| Moving Block | Oscillates between lanes | Timed lane switch or jump |
| Barrier Gate | 2 lanes blocked | Single open lane |

### Collision Resolution

- `BP_ObstacleBase` uses a **Box Collision** component with a custom `Obstacle` collision profile.
- On player overlap: **On Component Begin Overlap** fires → calls `OnPlayerHitObstacle` on `BP_GameMode` via **Get Game Mode** + **Cast**.
- `OnPlayerHitObstacle` triggers: stumble animation montage, brief speed reduction, and (if no shield power-up active) the run-end sequence.
- **Near-miss detection:** a separate, larger **Box Collision** trigger surrounds each obstacle. Entry + exit without a hit awards a score bonus via the **Score System**.

### Dynamic Obstacles

- Moving obstacles use an **Interp To Movement Component** with configurable waypoints and speed defined in the Data Asset.
- Movement speed of dynamic obstacles scales with `DifficultyTier` — same Blueprint, faster movement via a multiplied speed input at higher difficulties.

---

## 5. Difficulty & Speed System

`BP_DifficultySystem` (an Actor Component on `BP_GameMode`) governs progressive challenge increase over session distance.

### Speed Curve

- `CurrentSpeed` is initialized at `BaseSpeed` and increases by sampling a **Curve Float asset** (`C_SpeedCurve`) against `SessionDistance` using the **Get Float Value** node each tick.
- The speed curve ramps quickly early (to hook the player), then plateaus with occasional burst events.
- `MaxSpeed` is clamped via a **Clamp (Float)** node — prevents inputs from becoming physically impossible to respond to on touch.

### Difficulty Tier

- `DifficultyTier` (integer) increments when `SessionDistance` crosses thresholds defined in a `TierThresholds` float array, evaluated on the **On Chunk Entered** custom event.
- Tier governs: obstacle density multiplier, obstacle type weights (harder types unlock at higher tiers), moving obstacle speed, and collectible gap distances.
- Adjusted probability:
  ```
  AdjustedProbability = BaseProbability × DifficultyMultiplierCurve(DifficultyTier)
  ```
  Multiplier is sampled from a **Curve Float** asset via `DifficultyTier` as input.

### Score System

- `BP_ScoreSystem` (Actor Component) tracks:
  - `DistanceScore` — continuous, scaled with speed each tick
  - `CoinScore` — flat value added per coin collected
  - `ComboMultiplier` — increments on consecutive near-misses, resets on obstacle hit
- Final score formula:
  ```
  FinalScore = (DistanceScore + CoinScore) × ComboMultiplier
  ```
- High score persisted via a **Save Game Blueprint** (`BP_SaveGame`) and **Save Game to Slot** node.

---

## 6. Collectible & Power-Up System

### Collectibles

- `BP_Coin` and `BP_PowerUp` are placed during the chunk populate pass.
- Coins are arranged in lane-following arc patterns defined in `DA_CoinPattern` Data Assets — straight runs, arcs, zigzag between lanes.
- Pattern selection is weighted by `DifficultyTier` — higher tiers introduce reward patterns that require skillful lane changes to collect fully.

### Magnet Power-Up

- On activation: all `BP_Coin` actors within `MagnetRadius` are found via **Get All Actors of Class** + **Distance** check.
- Each coin within radius receives a move-to-player command driven by an **Interp To Movement Component** override.
- Each tick, newly spawned coins within radius are auto-collected via the same radius check.

### Active Power-Ups

| Power-Up | Effect | Duration |
|---|---|---|
| Magnet | Auto-collects nearby coins | Timed |
| Shield | Absorbs one obstacle hit | Single-use |
| Score Multiplier | ×2 score output | Timed |
| Speed Boost | Temporary speed surge + invulnerability | Timed |
| Double Jump | Enables second mid-air jump | Timed |

- `BP_PowerUpSystem` (Actor Component) manages active effects as an array of `FActivePowerUp` structs — each storing `PowerUpType` (Enum), `RemainingDuration` (float), and an active effect flag.
- Power-ups are processed each tick: duration is decremented via `DeltaTime`, expired effects are deactivated and removed from the array via **Remove Index**.
- **Stacking rule:** activating the same power-up type refreshes its `RemainingDuration` rather than stacking multiplicatively — handled via an **Array Find** check before adding.

---

## 7. Camera System

### Chase Camera

- `BP_RunnerCamera` (a Camera Actor Blueprint) maintains a fixed offset behind and above the character, stored as `CameraOffset` (Vector variable).
- Position is updated via **VInterp To** each tick — lag is tuned for runner pacing (tight enough to feel responsive, loose enough to avoid jitter).
- The forward offset is slightly ahead of the character to give the player visibility of upcoming obstacles; lookahead distance scales with `CurrentSpeed`.

### Speed-Reactive FOV

- Camera FOV widens as `CurrentSpeed` increases, computed each tick:
  ```
  CurrentFOV = FInterp To(CurrentFOV, BaseFOV + (SpeedFOVScale × NormalizedSpeed), DeltaTime, FOVInterpSpeed)
  ```
  Applied via **Set Field of View** on the Camera Component.
- Communicates speed increase viscerally without any UI element.

### Death Camera

- On run end: camera briefly holds position, then slowly pulls back and rises for a "survey the scene" beat before the game over UI appears.
- Implemented as a **Level Sequence** triggered from the `OnRunEnd` custom event in `BP_GameMode` via **Play Level Sequence** node.

---

## 8. Mobile Performance & Optimization

**Target: 60 fps sustained on mid-range Android hardware** (Snapdragon 6-series, Mali-G57 equivalent) over a 30-minute session without thermal throttling.

### Rendering Budget

- Mobile Forward Renderer — no deferred shading pipeline (`r.MobileHDR=0`).
- Draw call target: **< 120 per frame** (runner camera sees a narrow frustum — frustum culling is aggressive).
- Repeating tile meshes use **Hierarchical Instanced Static Mesh Components (HISM)** — one draw call per mesh type regardless of instance count.
- Texture budget: character max `1024×1024` ASTC; environment tiles `512×512` ASTC.
- Dynamic shadows: cast only from the character; environment uses baked lightmaps on static chunk components.
- Particle cap: Niagara `Max Particle Count` set per-system; off-screen emitters culled via scalability settings.

### Object Pooling Impact

- Chunk pool eliminates **Spawn Actor** / **Destroy Actor** GC pressure during gameplay.
- Obstacle and coin actors are similarly pooled — `BP_ObstaclePool` and `BP_CoinPool` follow the identical pattern used by `BP_ChunkPool`.
- All pools are pre-warmed at session start during the loading screen — no pool misses occur during active gameplay.

### Tick Optimization

| System | Tick Strategy |
|---|---|
| Chunk despawn evaluation | **Set Timer by Function Name** at 0.1s interval — not per-frame |
| Score UI update | **OnScoreChanged** Event Dispatcher — UI updates are event-driven, not polled |
| Difficulty evaluation | Fired on **OnChunkEntered** custom event — not continuous tick |
| Lane Movement, Touch Input, Camera | Full frame-rate tick via **Event Tick** |

### Adaptive Quality

- `BP_PerformanceSystem` monitors a rolling frame time average using a float array sampled each tick.
- On sustained frame budget overrun: reduces shadow distance, particle counts, and post-process quality by one tier via **Execute Console Command** nodes.
- On sustained recovery: restores one tier.
- Prevents thermal throttle spiral on extended sessions.

### Memory Management

- Chunk assets use **Soft Object References** (`TSoftObjectPtr` equivalent) — async loaded per biome on first transition, not all at boot.
- **Async Load Asset** node handles loading; gameplay is not gated on load completion — a fallback chunk type is used if the target asset is not yet loaded.
- Per-biome asset groups are unloaded on biome exit if not in the lookahead window via **Unload Primary Asset**.

### Android-Specific Settings

- Portrait orientation locked in **Project Settings → Supported Orientations**.
- `r.MobileHDR=0` — standard forward renderer.
- `r.Shadow.CSM.MaxCascades=1` set in the mobile device profile.
- ASTC texture compression for Adreno/Mali; ETC2 fallback via App Bundle split configured in **Android Build Settings**.
- Haptic feedback on coin collection and obstacle hit via the **Play Haptic Effect** node.

## Build & Packaging

| Setting | Value |
|---|---|
| Minimum SDK | API 26 (Android 8.0) |
| Target SDK | API 34 (Android 14) |
| ABI | arm64-v8a primary, armeabi-v7a fallback |
| Texture Format | ASTC primary, ETC2 fallback (App Bundle split) |
| Orientation | Portrait locked |
| HDR | Disabled (`r.MobileHDR=0`) |
| Shadow Cascades | 1 (mobile device profile) |

---

## Performance Targets

| Metric | Target | Approach |
|---|---|---|
| Frame rate | 60 fps sustained | < 120 draw calls, HISM, forward renderer |
| RAM | < 180MB | Async streaming, object pooling, soft refs |
| APK size | < 120MB | App Bundle, ASTC split, asset compression |
| Thermal stability | 30-min session | Adaptive quality, tick reduction, pool warmup |
| Level load | < 1.5s | Pool pre-warm at load screen, minimal hard deps |

---

## Development Scope

| Category | Detail |
|---|---|
| Developer count | 1 |
| Engine | Unreal Engine 5.7 |
| Languages | Blueprint |
| Platform | Android (Google Play) |
| Generation | Chunk-based procedural streaming with object pooling |
| Obstacle types | 5 categories with variants |
| Power-ups | 5 types |
| 3D Assets | All original |
| Gameplay systems | 10 discrete systems (see above) |
| Development tools | UE5 Editor, Android Studio, Blender, Substance Painter |

---

## Related Projects

| Project | Description |
|---|---|
| [Royal Jump](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) | Mobile precision platformer; touch controls, physics movement , UE5.7 |
| [TIME SOUL](https://store.steampowered.com/app/2928270/TIME_SOUL) | Souls-like action platformer; parkour, time-as-resource , UE5.1 |
| [U.N. Owen Was Her](https://store.steampowered.com/app/3420540/UN_Owen_Was_Her) | Third-person horror; AI, bullet-hell boss , UE5.3 |
| [Olympus of the Heavens](https://store.steampowered.com/app/3358020/Olympus_of_the_Heavens) | Isometric co-op ARPG; 12 bosses, Steam co-op , UE5.3 |
| [Blood Garden](https://kubrik.itch.io/bloodgarden) | Souls-like melee combat; stamina, parry, elemental , UE5.4 |
| [ArtStation Portfolio](https://www.artstation.com/kubrik) | 3D modeling , characters, creatures, props, environments |

---

## Developer

**Kubrik** , Developer & 3D Artist  
[ArtStation](https://www.artstation.com/kubrik) · [Google Play](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) · [Steam](https://store.steampowered.com/search/?developer=Kubrik)

---

*All code, art, design, and marketing assets produced by a single developer. No third-party gameplay code or purchased asset packs used in core systems.*
