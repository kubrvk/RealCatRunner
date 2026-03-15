# Real Cat Runner

> **Endless runner — mobile** — Unreal Engine 5.7 · C++ · Android · Solo Development  
> [ArtStation](https://www.artstation.com/kubrik)

---

## Overview

Real Cat Runner is an endless auto-runner built in Unreal Engine 5.7 using C++, targeting Android. The player controls a cat moving at continuously increasing speed through a procedurally generated track. The game loop is built on three pillars: responsive single-tap and swipe touch controls, a procedural level generation system that constructs the track ahead of the player in real-time, and a mobile optimization pipeline designed to maintain 60 fps on mid-range Android hardware while delivering high-quality visuals.

The primary technical challenges were building a procedural chunk-based world that streams seamlessly at high velocities, implementing a touch input model that feels precise at increasing game speeds, and sustaining visual fidelity within aggressive mobile performance budgets. All gameplay systems, procedural generation, character controller, UI, and assets were developed by a single developer.

---

## Engine & Technical Stack

| Layer | Technology |
|---|---|
| Engine | Unreal Engine 5.7 |
| Primary Language | C++ (gameplay, generation, systems) |
| Platform | Android (primary) |
| Input | Custom touch input component |
| Physics | Chaos — lane collision, obstacle interaction |
| Rendering | Mobile forward renderer, scalable quality tiers |
| Procedural Gen | Chunk-based streaming, seeded spawn tables |
| Animation | UE5 Animation Blueprint + `UAnimInstance` C++ subclass |
| Build | Android SDK/NDK, Gradle, UE5 Android packaging |
| 3D Pipeline | ZBrush → Maya → Substance Painter → UE5 |

---

## Architecture Overview

```
RealCatRunner/
├── Source/
│   ├── Core/
│   │   ├── RCRCharacter.h/.cpp                # Player cat character, lane movement
│   │   ├── RCRGameMode.h/.cpp                 # Session, speed scaling, score, game state
│   │   └── RCRPlayerController.h/.cpp         # Touch input routing, HUD init
│   ├── Systems/
│   │   ├── TouchInputSystem/                   # Raw touch → lane/jump/slide intent
│   │   ├── LaneMovementSystem/                 # Auto-run, lane switching, speed ramp
│   │   ├── ProceduralGenSystem/               # Chunk spawner, obstacle placement, pooling
│   │   ├── ObstacleSystem/                     # Obstacle types, collision, difficulty scaling
│   │   ├── CollectibleSystem/                  # Coin/power-up placement, magnetism, effects
│   │   ├── PowerUpSystem/                      # Active power-up logic, duration, stacking
│   │   ├── ScoreSystem/                        # Distance score, coin score, multiplier
│   │   ├── DifficultySystem/                   # Speed curve, obstacle density ramp
│   │   ├── CameraSystem/                       # Chase camera, speed-reactive FOV
│   │   └── PerformanceSystem/                  # Adaptive quality, thermal management
│   ├── Chunks/
│   │   ├── ChunkBase/                          # Base chunk actor, spawn interface
│   │   └── ChunkData/                          # Biome data assets, obstacle configs
│   └── UI/
│       ├── HUD/                                # Score, coins, distance, power-up timer
│       ├── TouchOverlay/                        # Swipe zone feedback
│       └── Menus/                              # Main menu, game over, shop, settings
```

---

## Core Systems: Technical Detail

### 1. Auto-Run & Lane Movement System

The player character moves forward automatically at a speed controlled entirely by `URCRGameMode`. Lateral movement is constrained to discrete lanes — the player's only control axes are lane-switch (left/right), jump, and slide.

**Forward Movement:**
- `ULaneMovementSystem` drives the character forward each tick via direct position offset — not physics-based forward movement.
- `CurrentSpeed` is a replicated float on `ARCRGameMode`, incremented by the difficulty curve over session time.
- Forward delta: `NewLocation = CurrentLocation + (ForwardVector * CurrentSpeed * DeltaTime)`.
- Speed is applied to the character position, not via `UCharacterMovementComponent::MaxWalkSpeed` — this avoids CMC pathfinding overhead and gives precise control over the forward axis without physics interference.

**Lane System:**
- Track divided into `LaneCount` (default: 3) discrete lanes, each defined by a `float LaneOffset` from center.
- `CurrentLane` (`int32`, range 0 to LaneCount-1) tracks active lane.
- Lane switch: target lane offset computed; character X position lerped from current to target over `LaneSwitchDuration` via `FMath::FInterpTo`.
- Lane switch blocked during: mid-air, slide, obstacle collision recovery.

```cpp
void ULaneMovementSystem::SwitchLane(int32 Direction)
{
    int32 TargetLane = FMath::Clamp(CurrentLane + Direction, 0, LaneCount - 1);
    if (TargetLane == CurrentLane || bLaneSwitchLocked) return;

    CurrentLane = TargetLane;
    TargetLaneOffset = LaneOffsets[CurrentLane];
    bLaneSwitching = true;
}

void ULaneMovementSystem::TickLaneSwitch(float DeltaTime)
{
    if (!bLaneSwitching) return;

    FVector Location = CharacterOwner->GetActorLocation();
    Location.Y = FMath::FInterpTo(Location.Y, TargetLaneOffset, DeltaTime, LaneSwitchSpeed);
    CharacterOwner->SetActorLocation(Location);

    if (FMath::Abs(Location.Y - TargetLaneOffset) < LaneSnapThreshold)
    {
        Location.Y = TargetLaneOffset;
        CharacterOwner->SetActorLocation(Location);
        bLaneSwitching = false;
    }
}
```

**Jump:**
- Simple vertical impulse via `UCharacterMovementComponent::AddImpulse`; gravity returns character to track.
- Jump height configurable; double-jump available as a power-up state.
- Landing detection via `OnLanded` callback; brief landing animation blend on return.

**Slide:**
- Capsule half-height reduced on slide start; restored on slide end.
- Slide duration fixed; early exit available if swipe-up received before duration expires.
- Slide plays dedicated animation blend — low-body pose with maintained forward motion.

---

### 2. Touch Input System

Single-finger swipe and tap gestures map to all player actions. The system is implemented as a custom `UTouchInputComponent` that intercepts raw touch events before UE's default input processing.

**Gesture Recognition:**

| Gesture | Action | Detection |
|---|---|---|
| Swipe Left | Lane switch left | Horizontal delta > threshold, leftward |
| Swipe Right | Lane switch right | Horizontal delta > threshold, rightward |
| Swipe Up | Jump | Vertical delta > threshold, upward |
| Swipe Down | Slide | Vertical delta > threshold, downward |
| Tap | Jump (alt) | Duration < `TapMaxDuration`, displacement < `TapMaxDrift` |

**Swipe Detection Pipeline:**
- `TouchBegin`: record `StartPosition`, `StartTime`.
- `TouchEnd`: compute `Delta = EndPosition - StartPosition`, `Duration = EndTime - StartTime`.
- If `Delta.Size() > SwipeMinDistance` and `Duration < SwipeMaxDuration`: classify dominant axis (X vs Y) and sign → map to intent.
- Diagonal swipes: resolved by `FMath::Abs(Delta.X) > FMath::Abs(Delta.Y)` — dominant axis wins.

**Input Buffering:**
- All gesture intents buffered for `InputBufferWindow` (default: 5 frames).
- Buffer consumed on first valid execution frame — essential for jump inputs slightly before landing and lane switches during lane-switch completion.

```cpp
ETouchIntent UTouchInputSystem::ClassifySwipe(const FVector2D& StartPos, const FVector2D& EndPos, float Duration)
{
    FVector2D Delta = EndPos - StartPos;
    if (Delta.Size() < SwipeMinDistance || Duration > SwipeMaxDuration)
        return ETouchIntent::None;

    if (FMath::Abs(Delta.X) > FMath::Abs(Delta.Y))
        return Delta.X > 0 ? ETouchIntent::LaneRight : ETouchIntent::LaneLeft;
    else
        return Delta.Y < 0 ? ETouchIntent::Jump : ETouchIntent::Slide;
}
```

---

### 3. Procedural Generation System

The world is generated at runtime as a continuous stream of chunks ahead of the player. `UProceduralGenSystem` manages the chunk lifecycle: spawn, active window, and despawn/pool.

**Chunk Architecture:**
- `AChunkBase` is the base class for all track segments — a fixed-length actor containing: ground mesh(es), obstacle spawn points, collectible spawn points, and decoration placement points.
- Chunk length is standardized (`ChunkLength` constant) to simplify the spawn lookahead calculation.
- Chunks are authored as prefab-like actors with tagged spawn sockets; obstacle and collectible placement resolved at runtime, not baked.

**Streaming Pipeline:**
```
[Active Chunk Window]
  Chunk N-1 (behind player, queued for pool return)
  Chunk N   (current player position)
  Chunk N+1 (ahead — populated)
  Chunk N+2 (lookahead — being populated)
  Chunk N+3 (just spawned — empty, populating)
```
- `SpawnLookahead` (default: 3 chunks): number of chunks pre-generated ahead of player.
- Each frame, `UProceduralGenSystem` evaluates player progress within `CurrentChunk`; when player crosses `ChunkTriggerThreshold` within the chunk, a new chunk is spawned and populated at the front.
- Chunks behind the player beyond `DespawnDistance` are returned to the object pool.

**Object Pool:**
- `UChunkPool` maintains a `TArray<AChunkBase*>` free list per chunk type.
- On pool request: if free list non-empty, dequeue, reset transform, re-enable; else spawn new instance.
- On pool return: disable actor, clear all spawned obstacles/collectibles, enqueue to free list.
- Pool eliminates per-chunk `SpawnActor` / `DestroyActor` calls during gameplay — critical for mobile GC pressure.

```cpp
AChunkBase* UChunkPool::AcquireChunk(TSubclassOf<AChunkBase> ChunkClass)
{
    TArray<AChunkBase*>& Pool = ChunkPools.FindOrAdd(ChunkClass);

    if (Pool.Num() > 0)
    {
        AChunkBase* Chunk = Pool.Pop();
        Chunk->SetActorHiddenInGame(false);
        Chunk->SetActorEnableCollision(true);
        return Chunk;
    }

    return GetWorld()->SpawnActor<AChunkBase>(ChunkClass);
}

void UChunkPool::ReturnChunk(AChunkBase* Chunk)
{
    Chunk->ResetChunk(); // Clear obstacles, collectibles
    Chunk->SetActorHiddenInGame(true);
    Chunk->SetActorEnableCollision(false);
    ChunkPools.FindOrAdd(Chunk->GetClass()).Add(Chunk);
}
```

**Obstacle Placement:**
- Each chunk type has an `UObstaclePlacementDataAsset`: a list of `FObstacleSpawnConfig` entries per spawn socket tag — each config specifies obstacle class, placement probability, and difficulty tier range.
- On chunk acquisition, `UProceduralGenSystem::PopulateChunk` iterates spawn sockets and rolls against probability weighted by current `DifficultyTier`.
- Lane exclusivity: obstacle placement validates that no impassable combination is generated — at least one lane must always be clear per obstacle group (enforced via `FLaneMask` bitmask evaluation).

**Biome System:**
- Chunk type selection weighted by current biome. `FBiomeConfig` data asset defines: chunk class weights, obstacle set, decoration mesh set, material parameter overrides, and ambient Niagara system.
- Biome transitions occur at distance thresholds defined in `FBiomeSequenceConfig`; blend handled via per-chunk `UMaterialParameterCollection` lerp over `BiomeTransitionLength` chunks.

---

### 4. Obstacle System

Obstacles are `AObstacleBase` subclasses placed by the procedural system. Each has: a collision profile, a `FObstacleConfig` data asset reference, and an optional movement component for dynamic variants.

**Obstacle Categories:**

| Type | Lane Behavior | Player Response |
|---|---|---|
| Ground Block | 1–2 lanes blocked | Jump over or lane switch |
| Overhead Bar | Full width, low | Slide under |
| Side Wall | 1 lane blocked | Lane switch |
| Moving Block | Oscillates between lanes | Timed lane switch or jump |
| Barrier Gate | 2 lanes blocked | Single open lane |

**Collision Resolution:**
- `AObstacleBase` uses a `UBoxComponent` with `ECC_GameTraceChannel_Obstacle`.
- On player overlap: `URCRGameMode::OnPlayerHitObstacle` called → triggers stumble animation, brief speed reduction, and if no shield power-up active, triggers run-end sequence.
- Near-miss detection: separate larger trigger volume around each obstacle; near-miss within threshold awards score bonus.

**Dynamic Obstacles:**
- Moving obstacles use a `UInterpToMovementComponent` with configurable waypoints and speed.
- Speed of moving obstacles scales with `DifficultyTier` — same config, faster movement at higher difficulty.

---

### 5. Difficulty & Speed System

`UDifficultySystem` governs the game's progressive challenge increase over session distance.

**Speed Curve:**
- `CurrentSpeed` initialized at `BaseSpeed`; increases via `UCurveFloat SpeedCurve` sampled against `SessionDistance`.
- Speed curve authored to ramp quickly early (hook player), then plateau with occasional burst events.
- `MaxSpeed` cap prevents inputs from becoming physically impossible to respond to on touch.

**Difficulty Tier:**
- `DifficultyTier` (`int32`) increments at distance thresholds defined in `TArray<float> TierThresholds`.
- Tier governs: obstacle density multiplier, obstacle type weights (harder types unlock at higher tiers), moving obstacle speed, collectible gap distances.
- Obstacle placement probability: `AdjustedProbability = BaseProbability * DifficultyMultiplierCurve(DifficultyTier)`.

**Score System:**
- `ScoreSystem` tracks: `DistanceScore` (continuous, scales with speed), `CoinScore` (per-coin flat value), `ComboMultiplier` (increments on consecutive near-misses, resets on hit).
- Final score: `(DistanceScore + CoinScore) * ComboMultiplier`.
- High score persisted via `USaveGame` slot.

---

### 6. Collectible & Power-Up System

**Collectibles:**
- `ACoinActor` and `APowerUpActor` placed on chunk populate pass.
- Coins arranged in lane-following arc patterns defined in `FCoinPatternConfig` data assets — straight runs, arcs, zigzag between lanes.
- Pattern selection weighted by difficulty tier — higher tiers introduce reward patterns that require skillful lane changes to collect fully.

**Magnet Power-Up:**
- On activation: all `ACoinActor` instances within `MagnetRadius` receive a `MoveToPlayer` task via `UInterpToMovementComponent` override.
- Radius checked each tick; newly spawned coins within radius are auto-collected.

**Active Power-Ups:**

| Power-Up | Effect | Duration |
|---|---|---|
| Magnet | Auto-collects nearby coins | Timed |
| Shield | Absorbs one obstacle hit | Single-use |
| Score Multiplier | ×2 score output | Timed |
| Speed Boost | Temporary speed surge + invulnerability | Timed |
| Double Jump | Enables second mid-air jump | Timed |

- `UPowerUpSystem` manages active effects as `TArray<FActivePowerUp>` — each with `EPowerUpType`, `RemainingDuration`, and applied effect reference.
- Power-ups processed per tick: duration decremented, expired effects deactivated and removed.
- Stacking: same power-up type refreshes duration rather than stacking multiplicatively.

---

### 7. Camera System

**Chase Camera:**
- `ARCRCameraActor` maintains a fixed offset behind and above the character.
- Position updated via `FMath::VInterpTo` each tick — lag tuned for runner pacing (tight enough to feel responsive, loose enough to avoid jitter).
- Forward offset slightly ahead of character gives player visibility of upcoming obstacles — lookahead distance scales with `CurrentSpeed`.

**Speed-Reactive FOV:**
- Camera FOV widens as `CurrentSpeed` increases: `CurrentFOV = FMath::FInterpTo(CurrentFOV, BaseFOV + (SpeedFOVScale * NormalizedSpeed), DeltaTime, FOVInterpSpeed)`.
- Communicates speed increase to player viscerally without UI.

**Death Camera:**
- On run end: camera briefly holds position, then slowly pulls back and rises for a "survey the scene" beat before game over UI appears.
- Implemented as a `UCameraSequence` Sequencer track — triggered from `URCRGameMode::OnRunEnd`.

---

### 8. Mobile Performance & Optimization

Target: **60 fps sustained on mid-range Android hardware** (Snapdragon 6-series, Mali-G57 equivalent) over a 30-minute session without thermal throttling.

**Rendering Budget:**
- Mobile forward renderer — no deferred shading pipeline.
- Draw call target: < 120 per frame (runner camera sees a narrow frustum — frustum culling is aggressive).
- Static chunk geometry merged via `UHierarchicalInstancedStaticMeshComponent` for repeating tile meshes — one draw call per mesh type regardless of instance count.
- Texture budget: character max 1024×1024 ASTC; environment tiles 512×512 ASTC.
- Dynamic shadows: cast only from character; environment uses baked lightmaps on static chunk components.
- Particle cap: Niagara `MaxParticleCount` set per-system; off-screen emitters culled via scalability settings.

**Object Pooling Impact:**
- Chunk pool eliminates `SpawnActor` / `DestroyActor` GC pressure during gameplay.
- Obstacle and coin actors similarly pooled — `UObstaclePool` and `UCoinPool` follow identical pattern to `UChunkPool`.
- Pool pre-warmed at session start during loading screen — no pool misses during active gameplay.

**Tick Optimization:**
- Chunk despawn evaluation: checked at fixed 0.1s interval via `FTimerHandle`, not per-frame.
- Score update: broadcast via `OnScoreChanged` delegate — UI updates event-driven, not polled.
- Difficulty evaluation: checked on `OnChunkEntered` event, not continuous tick.
- Only `ULaneMovementSystem`, `UTouchInputSystem`, and camera tick at full frame rate.

**Adaptive Quality:**
- `UPerformanceSystem` monitors rolling frame time average.
- On sustained frame budget overrun: reduces shadow distance, particle counts, post-process quality one tier.
- On sustained recovery: restores one tier.
- Prevents thermal throttle spiral on extended sessions.

**Memory:**
- Chunk assets use `TSoftObjectPtr` — async loaded per biome on first transition, not all at boot.
- `FStreamableManager` handles async load; gameplay not gated on load completion — fallback chunk type used if target asset not yet loaded.
- Per-biome asset group unloaded on biome exit if not in the lookahead window.

**Android-Specific:**
- Portrait orientation locked.
- `r.MobileHDR=0` — standard forward renderer.
- `r.Shadow.CSM.MaxCascades=1` on mobile device profile.
- ASTC texture compression for Adreno/Mali; ETC2 fallback via App Bundle split.
- Haptic feedback on coin collection and obstacle hit via `FAndroidApplication::Vibrate`.

---

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
| Developer count | 1 (solo) |
| Engine | Unreal Engine 5.7 |
| Languages | C++ |
| Platform | Android (Google Play) |
| Generation | Chunk-based procedural streaming with object pooling |
| Obstacle types | 5 categories with variants |
| Power-ups | 5 types |
| 3D Assets | All original |
| Gameplay systems | 10 discrete systems (see above) |
| Development tools | UE5 Editor, Android Studio, ZBrush, Maya, Substance Painter |

---

## Related Projects

| Project | Description |
|---|---|
| [Royal Jump](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) | Mobile precision platformer; touch controls, physics movement — UE5.7 |
| [TIME SOUL](https://store.steampowered.com/app/2928270/TIME_SOUL) | Souls-like action platformer; parkour, time-as-resource — UE5.1 |
| [U.N. Owen Was Her](https://store.steampowered.com/app/3420540/UN_Owen_Was_Her) | Third-person horror; AI, bullet-hell boss — UE5.3 |
| [Olympus of the Heavens](https://store.steampowered.com/app/3358020/Olympus_of_the_Heavens) | Isometric co-op ARPG; 12 bosses, Steam co-op — UE5.3 |
| [Blood Garden](https://kubrik.itch.io/bloodgarden) | Souls-like melee combat; stamina, parry, elemental — UE5.4 |
| [ArtStation Portfolio](https://www.artstation.com/kubrik) | 3D modeling — characters, creatures, props, environments |

---

## Developer

**Kubrik** — Developer & 3D Artist  
9 years web development · 7 years 3D modeling · 5 years Unreal Engine C++  
5 shipped commercial games as sole developer.

[ArtStation](https://www.artstation.com/kubrik) · [Google Play](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) · [Steam](https://store.steampowered.com/search/?developer=Kubrik)

---

*All code, art, design, and marketing assets produced by a single developer. No third-party gameplay code or purchased asset packs used in core systems.*
