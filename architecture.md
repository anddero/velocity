# Velocity — Technical Architecture

> Technical design for Velocity — a minimalist first-person tunnel racer for Android.
> Companion document to [spec.md](spec.md).

---

## Table of Contents

1. [Tech Stack Recommendation](#1-tech-stack-recommendation)
2. [Alternatives Considered](#2-alternatives-considered)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Rendering Pipeline](#4-rendering-pipeline)
5. [Game Loop Design](#5-game-loop-design)
6. [Data Model](#6-data-model)
7. [Procedural Generation](#7-procedural-generation)
8. [Collision Detection](#8-collision-detection)
9. [Input Handling](#9-input-handling)
10. [Extensibility Hooks](#10-extensibility-hooks)
11. [Open Questions & Decisions](#11-open-questions--decisions)

---

## 1. Tech Stack Recommendation

### Primary choice: **Kotlin + Android OpenGL ES 2.0 (raw)**

| Layer | Technology |
|---|---|
| Language | Kotlin (JVM, targeting Android) |
| Graphics | OpenGL ES 2.0 via `android.opengl.GLSurfaceView` |
| Math | Custom lightweight vec3/mat4 utilities *or* `android.opengl.Matrix` |
| Build | Gradle (Kotlin DSL) with Android Gradle Plugin |
| Min SDK | API 26 (Android 8.0) |
| Input | `android.hardware.SensorManager` (accelerometer) |
| Persistence | `SharedPreferences` / Jetpack DataStore |
| Audio (future) | `SoundPool` (SFX) + `MediaPlayer` or Oboe NDK (music) |

### Why raw OpenGL ES?

1. **The game is geometrically trivial.** The entire world is flat-shaded rectangles and cubes. No skeletal animation, no physics engine, no scene graph, no sprite batching. A full engine is overhead with no payoff.
2. **Total control over the render loop.** We need buttery 60 fps with minimal GC pressure. Writing the GL code directly means zero abstraction tax.
3. **Tiny APK.** No engine runtime to bundle. The entire app will be well under 5 MB for Phase 1.
4. **Learning investment is bounded.** The rendering code is ~2 shaders (flat color + future lit) and a straightforward VBO pipeline. Not a large surface area.
5. **Android-native integration is seamless.** Sensors, lifecycle, permissions, UI overlays (menus) — all standard Android APIs, no bridging layers.

---

## 2. Alternatives Considered

| Engine / Approach | Pros | Cons | Verdict |
|---|---|---|---|
| **libGDX** | Cross-platform; mature 2D/3D; built-in `ShapeRenderer`, math, input abstraction; large community. | Extra ~8 MB runtime; abstractions we won't use (sprites, tilemaps, UI); Java-centric API (Kotlin interop fine but idiomatic friction); would still need custom tunnel mesh generation. | **Strong runner-up.** Reconsider if iOS port becomes a goal or if the project scope balloons. |
| **Godot 4 (GDScript / Kotlin)** | Visual editor; built-in physics, particles, audio; export to Android. | GDScript is a separate language to maintain; Kotlin/JVM bindings are community-maintained and less stable; export APK is 30–40 MB minimum; opaque render pipeline makes custom mesh generation awkward. | **Over-engineered** for this scope. |
| **Unity** | Industry standard; huge asset ecosystem; C# is productive. | Massive runtime (~50+ MB APK); C# not Kotlin; overkill for flat rectangles; licensing considerations. | **Not appropriate** for a minimalist Android-native project. |
| **Unreal Engine** | Best-in-class rendering. | Absurdly heavy for this use case. | **No.** |
| **Vulkan (direct)** | Maximum GPU control; modern API. | Enormous boilerplate for pipeline setup; not justified by the rendering complexity; API 24+ but driver quality varies on low-end devices. | **Premature optimization.** Could be a Phase 3 "port the renderer" exercise. |
| **Jetpack Compose Canvas / Custom View** | Pure Kotlin, no GL. | 2D only; no perspective projection; can't do 3D tunnel without reinventing rasterization; performance ceiling. | **Unsuitable** for 3D. |

### Recommendation path

```
Phase 1:  Kotlin + raw OpenGL ES 2.0
Phase 2+: If cross-platform needed → migrate renderer behind an interface, drop in libGDX or KorGE backend.
          If advanced rendering needed → upgrade shaders to ES 3.0 / Vulkan.
```

---

## 3. High-Level Architecture

### 3.1 Module Map

```
┌─────────────────────────────────────────────────────────┐
│                      Android App                        │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────────┐ │
│  │  Title /  │  │  HUD /   │  │   GLSurfaceView        │ │
│  │  Menu UI  │  │ Overlay  │  │   (full-screen)        │ │
│  └────┬─────┘  └────┬─────┘  └───────────┬────────────┘ │
│       │              │                    │              │
│       ▼              ▼                    ▼              │
│  ┌──────────────────────────────────────────────────┐   │
│  │                 GameEngine                        │   │
│  │                                                   │   │
│  │  ┌────────────┐ ┌────────────┐ ┌──────────────┐  │   │
│  │  │  GameLoop   │ │  Renderer  │ │ InputManager │  │   │
│  │  │ (update +   │ │ (GL draw)  │ │ (sensor/     │  │   │
│  │  │  physics)   │ │            │ │  touch)      │  │   │
│  │  └──────┬─────┘ └──────┬─────┘ └──────┬───────┘  │   │
│  │         │               │              │          │   │
│  │         ▼               ▼              ▼          │   │
│  │  ┌─────────────────────────────────────────────┐  │   │
│  │  │              GameState                      │  │   │
│  │  │  - PlayerState (θ, speed, alive)            │  │   │
│  │  │  - TunnelBuffer (ring buffer of chunks)     │  │   │
│  │  │  - ScoreState (distance, highscore)         │  │   │
│  │  │  - DifficultyState (current params)         │  │   │
│  │  └─────────────────────────────────────────────┘  │   │
│  │         ▲                                         │   │
│  │         │                                         │   │
│  │  ┌──────┴──────┐  ┌──────────────┐               │   │
│  │  │ Tunnel      │  │ Collision    │               │   │
│  │  │ Generator   │  │ System       │               │   │
│  │  └─────────────┘  └──────────────┘               │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐                     │
│  │ ScoreManager │  │ AssetManager │  (future: audio,    │
│  │ (persist)    │  │ (future)     │   textures, maps)   │
│  └──────────────┘  └──────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Module Responsibilities

| Module | Responsibility |
|---|---|
| **GameLoop** | Fixed-timestep update tick (physics, generation, collision). Decoupled from render frame rate. Owns the master clock. |
| **Renderer** | Reads `GameState` snapshot each frame. Builds/uploads GL buffers. Executes draw calls. Owns shaders and projection matrix. |
| **InputManager** | Listens to accelerometer and touch events. Produces a normalized lateral input value (−1.0 … +1.0) consumed by `GameLoop`. |
| **GameState** | Central, mutable state container. All game-relevant data lives here. Single source of truth. |
| **TunnelGenerator** | Produces `TunnelChunk` objects ahead of the camera. Implements the procedural algorithm (and later, reads from map files). |
| **CollisionSystem** | Each tick, tests the player's position against obstacles in the current and next segment. Reports hit/miss. |
| **ScoreManager** | Reads distance from `GameState`, persists high score. |
| **AssetManager** | Future: loads textures, audio, map files. In Phase 1, there are no external assets — geometry is generated. |

### 3.3 Threading Model

```
Main Thread (UI)         GL Thread (render)          Sensor Thread (input)
─────────────────        ──────────────────          ─────────────────────
 Activity lifecycle       onDrawFrame() {             onSensorChanged() {
 Menu / HUD overlays        gameLoop.tick(dt)            inputManager.update(event)
                             renderer.draw(state)      }
                          }
```

- `GLSurfaceView` provides a dedicated GL thread. Both `GameLoop.tick()` and `Renderer.draw()` run on this thread sequentially per frame — simple, no cross-thread sync needed for game state.
- `InputManager` receives sensor callbacks on the sensor thread; it writes to an `AtomicReference<Float>` (or similar lock-free slot) that `GameLoop` reads each tick.
- Main thread handles Android UI (menus, pause overlay, HUD via `View` overlay on top of `GLSurfaceView`).

---

## 4. Rendering Pipeline

### 4.1 Coordinate System

- **World space:** right-handed. +Z = forward (into the tunnel), +Y = up, +X = right.
- **Camera** sits at a fixed world Z and moves forward each frame (or equivalently, the world scrolls backward — implementation detail).
- Tunnel segments are generated in world Z coordinates ahead of the camera.

### 4.2 Shaders (Phase 1)

Two shaders total:

#### Flat Color Shader (tunnel panels & obstacles)

```glsl
// Vertex
uniform mat4 uMVPMatrix;
attribute vec4 aPosition;

void main() {
    gl_Position = uMVPMatrix * aPosition;
}

// Fragment
precision mediump float;
uniform vec4 uColor;

void main() {
    gl_FragColor = uColor;
}
```

#### (Optional) Depth-Fade Shader

Same as above but mixes `uColor` toward a fog color based on `gl_Position.z / uFarPlane`. Gives a nice depth-fading effect cheaply.

### 4.3 Geometry Pipeline

```
TunnelGenerator                Renderer
─────────────                  ────────
produces TunnelChunk ────►  reads chunk vertex data
  (CPU vertex arrays)          uploads to VBO (if dirty)
                               binds VBO → draw call per panel color
```

- Each `TunnelChunk` owns a pre-built `FloatArray` of vertex positions for its panels.
- The `Renderer` maintains a pool of GL VBO handles. When a chunk enters draw range, its vertices are uploaded. When it exits, the VBO is recycled.
- **Draw call batching:** all panels of the same color are concatenated into one VBO → one `glDrawArrays` call. With 2 colors (tunnel + obstacles), that's ≈ 2–4 draw calls per frame. Trivially fast.

### 4.4 "Tunnel Rotation" Visual

The tilt input produces a **roll angle φ** applied to the View matrix:

```
ViewMatrix = LookAt(cameraPos, cameraPos + forward, up)
           × RotateZ(φ)   // ← this is the "tilt" visual
```

The entire world appears to rotate around the camera. This is exactly the SpeedX 3D effect. Internally, the player's angular position θ changes, and φ is a smoothed visual representation of θ (or its derivative — see [§9 Input](#9-input-handling)).

### 4.5 Draw Distance & Culling

- Generate and draw ~100–150 segments ahead.
- Segments behind the camera (`z < camera.z - margin`) are not drawn (simple Z cull, not even needing frustum math).
- Fog/fade hides the far-end pop-in.

---

## 5. Game Loop Design

### 5.1 Fixed-Timestep Update

```
const val TICK_RATE = 120          // physics ticks per second
const val TICK_DT   = 1.0 / 120   // seconds per tick

var accumulator = 0.0

fun onDrawFrame(gl: GL10) {
    val now = System.nanoTime()
    val frameDt = (now - lastFrameTime) / 1_000_000_000.0
    lastFrameTime = now
    accumulator += frameDt

    // Fixed-step update (may run 0, 1, or 2 times per frame)
    while (accumulator >= TICK_DT) {
        gameLoop.update(TICK_DT)
        accumulator -= TICK_DT
    }

    // Render with interpolation factor for smooth visuals
    val alpha = accumulator / TICK_DT
    renderer.draw(gameState, alpha)
}
```

- 120 Hz tick rate gives precise collision detection at high speeds.
- Render interpolates between previous and current state for visual smoothness at any frame rate.

### 5.2 Update Order

Each `gameLoop.update(dt)`:

1. **Read input** — get lateral value from `InputManager`.
2. **Move player** — update θ based on input and dt.
3. **Advance camera** — move forward by `speed × dt`.
4. **Update difficulty** — adjust speed, obstacle density based on distance.
5. **Generate terrain** — if camera is within N segments of the last generated chunk, produce more.
6. **Recycle terrain** — discard chunks far behind the camera.
7. **Collision check** — test player against obstacles in current segment(s).
8. **Update score** — accumulate distance.
9. **State transitions** — if collision → DEAD; etc.

---

## 6. Data Model

### 6.1 Core Types

```kotlin
/** A single ring of the tunnel — one polygon cross-section at a given depth. */
data class TunnelRing(
    val centerZ: Float,                     // world Z position
    val center: Vec3,                       // world position of ring center (for curves)
    val forward: Vec3,                      // tangent direction (for curves)
    val up: Vec3,                           // local up (for banking)
    val radius: Float,                      // distance from center to vertices
    val sides: Int,                         // polygon side count (e.g. 8)
    // Vertex positions are computed: center + rotate(radius, i * 2π/sides) in the ring's local plane
)

/** A rectangular panel connecting two adjacent rings on one side. */
data class TunnelPanel(
    val ringIndex: Int,                     // index of the "near" ring
    val sideIndex: Int,                     // which side of the polygon (0 until sides)
    // Four corner vertices are derived from ring[ringIndex] and ring[ringIndex+1]
)

/** A chunk of tunnel — a batch of rings and the obstacles within them. */
class TunnelChunk(
    val rings: List<TunnelRing>,            // e.g. 20–40 rings per chunk
    val obstacles: List<Obstacle>,
) {
    // Pre-built vertex data for the renderer
    lateinit var panelVertices: FloatArray
    lateinit var obstacleVertices: FloatArray
    var gpuBufferHandle: Int = 0            // GL VBO name (0 = not uploaded)
    var dirty: Boolean = true               // needs re-upload
}

/** An obstacle sitting on the tunnel wall. */
data class Obstacle(
    val segmentIndex: Int,                  // which ring it's near
    val sideIndex: Int,                     // which panel it sits on
    val localOffset: Float,                 // 0..1 position along the panel width
    val depthOffset: Float,                 // 0..1 position along the panel depth (between rings)
    val size: Vec3,                         // half-extents
    val type: ObstacleType = ObstacleType.STATIC_CUBE,
)

enum class ObstacleType {
    STATIC_CUBE,
    // Future:
    // MOVING_CUBE, ROTATING_CUBE, RAMP, ...
}
```

### 6.2 Ring Buffer / Chunk Queue

```
                    camera
                      │
   ◄── retired ───    ▼    ─── generated ahead ──►
  [chunk][chunk]  [CHUNK][CHUNK][CHUNK][CHUNK][chunk][chunk]
                  ▲ draw range                       ▲
              draw start                         draw end
              (camera - margin)                  (camera + drawDist)
```

- The `TunnelBuffer` holds an `ArrayDeque<TunnelChunk>`.
- When the camera advances past a chunk's end, the chunk is removed from the front, its VBO recycled, and the object returned to a pool.
- When the camera approaches the last generated chunk, `TunnelGenerator` produces new chunks appended to the back.
- Typical buffer size: 8–12 chunks × 30 rings = 240–360 rings alive at any time.

### 6.3 Player State

```kotlin
data class PlayerState(
    var theta: Float = 0f,          // angular position around the tunnel (radians)
    var radialOffset: Float = 0f,   // distance from wall (0 = on surface; >0 = jumping/flying) — Phase 2
    var speed: Float = BASE_SPEED,  // forward speed (world units / second)
    var distance: Float = 0f,       // total distance traveled (score)
    var alive: Boolean = true,
)
```

### 6.4 Game State

```kotlin
class GameState {
    val player = PlayerState()
    val tunnelBuffer = TunnelBuffer()
    val difficulty = DifficultyState()
    var phase: GamePhase = GamePhase.TITLE

    enum class GamePhase { TITLE, PLAYING, PAUSED, DEAD }
}
```

---

## 7. Procedural Generation

### 7.1 Algorithm Overview

```
generateChunk(prevChunkEnd: TunnelRing, difficulty: DifficultyState): TunnelChunk {

    1. Determine chunk parameters from difficulty:
       - ringCount (e.g. 30)
       - curvature: random yaw/pitch deltas, smoothed with a spline
       - obstacleDensity: expected obstacles per ring

    2. Generate rings:
       for i in 0 until ringCount:
           - Advance Z by segmentDepth
           - Apply curvature: rotate the forward vector incrementally
           - Compute center = prevCenter + forward * segmentDepth
           - Compute up via slerp / Frenet frame
           - Create TunnelRing(center, forward, up, radius, sides)

    3. Place obstacles:
       for each ring (with probability = obstacleDensity):
           - Pick a random side index (0 until sides)
           - Check that the side is not too close to recently placed obstacles (min gap)
           - Create Obstacle(sideIndex, size, ...)

    4. Build vertex arrays:
       - For each pair of adjacent rings, emit 4 vertices per panel (2 tris × 3 or 1 quad)
       - For each obstacle, emit a cube (24 vertices / 36 indices or 12 tris)

    5. Return TunnelChunk(rings, obstacles, panelVertices, obstacleVertices)
}
```

### 7.2 Curvature Generation

- Maintain a **heading** (yaw, pitch) that evolves smoothly.
- Each chunk samples a random target yaw/pitch delta from a range scaled by difficulty.
- Interpolate toward the target using cosine easing over the chunk length.
- This produces gentle S-curves that look organic.
- The heading does **not** affect gameplay (see spec §2.1) — the camera auto-follows the tunnel center.

### 7.3 Difficulty Curve

```kotlin
data class DifficultyState(
    var level: Float = 0f,   // 0.0 = easiest, 1.0 = hardest plateau
) {
    val speed get()           = lerp(BASE_SPEED, MAX_SPEED, level)
    val obstacleDensity get() = lerp(MIN_DENSITY, MAX_DENSITY, level)
    val gapWidth get()        = lerp(MAX_GAP, MIN_GAP, level)
    val curvatureRange get()  = lerp(MIN_CURVE, MAX_CURVE, level)

    fun advance(dt: Float) {
        level = min(1f, level + RAMP_RATE * dt)
    }
}
```

### 7.4 Seeded RNG

- The generator uses `kotlin.random.Random(seed)`.
- In infinite mode, seed is random per run.
- In pre-made maps (Phase 2), the seed is fixed (or generation is replaced by file loading).
- Seed-based RNG allows replaying / sharing specific tunnel configurations.

---

## 8. Collision Detection

### 8.1 Approach

The collision space is essentially **2D** — the player's angular position θ on the tunnel ring versus the obstacle's angular span.

```
  Ring cross-section (top view):
        ___
      /  |  \        ← obstacle on side 2
     |   █   |
     |       |
      \ ___ /
         ▲
      player at θ
```

### 8.2 Algorithm (per tick)

```
1. Determine which ring(s) the player currently overlaps
   (based on camera Z vs ring Z positions — typically 1 or 2 rings).

2. For each obstacle in those rings:
   a. Compute obstacle angular span:
      - center angle = sideIndex * (2π / sides) + localOffset * (2π / sides)
      - half-width angle ≈ atan(obstacleSize.x / radius)
      - span = [center - halfWidth, center + halfWidth]

   b. Compute player angular span:
      - center = player.theta
      - half-width ≈ atan(playerRadius / tunnelRadius)  (player has a small implicit size)
      - span = [theta - halfWidth, theta + halfWidth]

   c. Check angular overlap (accounting for wraparound at 2π).

   d. Check depth overlap:
      - player Z vs obstacle Z ± obstacle half-depth.

   e. (Phase 2) Check radial overlap:
      - player radialOffset vs obstacle radial extent.

3. If any obstacle overlaps → collision → DEAD.
```

### 8.3 Performance

- Only 1–3 rings are checked per tick (the ones near the camera).
- Each ring has at most a handful of obstacles.
- Total: ~5–15 AABB-like checks per tick. Negligible cost.

---

## 9. Input Handling

### 9.1 Accelerometer Pipeline

```
Raw sensor event
      │
      ▼
Low-pass filter (α ≈ 0.15)
      │
      ▼
Deadzone clamp (|x| < 0.5 → 0)
      │
      ▼
Normalize to [-1, +1] range
      │
      ▼
Sensitivity multiplier (user setting)
      │
      ▼
Store in AtomicReference<Float>  ← read by GameLoop each tick
```

### 9.2 Mapping to θ

```kotlin
fun updatePlayer(input: Float, dt: Float) {
    val angularSpeed = input * MAX_ANGULAR_SPEED  // radians/sec
    player.theta += angularSpeed * dt
    player.theta = player.theta.mod(2f * PI)      // wrap around
}
```

### 9.3 Visual Roll

The visual roll angle φ displayed by the renderer is **not** the same as θ. It represents the *rate of change* — a tilt to the side while moving:

```kotlin
val targetRollAngle = -input * MAX_VISUAL_ROLL   // e.g. ±25°
visualRoll = lerp(visualRoll, targetRollAngle, ROLL_SMOOTH * dt)
```

This gives the aileron-bank feel described in the spec.

---

## 10. Extensibility Hooks

How the architecture accommodates each future feature from [spec.md §7](spec.md#7-future-features-phase-2):

### 10.1 Terrain Flattening → `TunnelRing` morph

`TunnelRing` stores a `morphFactor: Float` (0 = closed polygon, 1 = flat plane). Vertex computation interpolates between circular placement and flat-line placement:

```kotlin
fun vertexPosition(sideIndex: Int): Vec3 {
    val closedPos = circularPosition(sideIndex, radius, sides)
    val flatPos   = linearPosition(sideIndex, width, sides)
    return lerp(closedPos, flatPos, morphFactor)
}
```

The rest of the pipeline (rendering, collision) works unchanged because it consumes vertex positions, not semantic "tunnel" concepts.

### 10.2 Sophisticated Obstacles → `Obstacle` polymorphism

`ObstacleType` is already an enum. Extend to a sealed class hierarchy:

```kotlin
sealed class Obstacle {
    abstract fun update(dt: Float)       // no-op for static
    abstract fun meshData(): FloatArray  // each type provides its own geometry
    abstract fun collisionBounds(): ...  // each type provides its own bounds
}

class StaticCube(...) : Obstacle() { ... }
class MovingCube(val path: ...) : Obstacle() { ... }
class RotatingCube(val axis: Vec3, val speed: Float) : Obstacle() { ... }
```

### 10.3 Jump / Fly → `radialOffset` in `PlayerState`

Already present as a placeholder (`radialOffset = 0f`). When jumping:

- `radialOffset` increases (away from surface).
- Gravity pulls it back toward 0.
- Collision check gains a radial dimension (already noted in §8.2e).

### 10.4 Speed Effects → modifier system

```kotlin
data class SpeedModifier(
    val multiplier: Float,  // e.g. 1.5 for boost, 0.5 for slow
    val remaining: Float,   // seconds left
)

// GameLoop applies active modifiers to player.speed each tick.
```

Visual effects (FOV warp, blur) read `player.speed` and scale accordingly.

### 10.5 Pre-Created Maps → `MapSource` interface

```kotlin
interface MapSource {
    fun nextChunk(prevEnd: TunnelRing): TunnelChunk?
    fun hasMore(): Boolean
}

class ProceduralMapSource(seed: Long, difficulty: DifficultyState) : MapSource { ... }
class FileMapSource(mapData: MapData) : MapSource { ... }
```

`TunnelGenerator` delegates to a `MapSource`. Swapping between procedural and file-based is transparent to the rest of the engine.

### 10.6 Lighting & Particles → shader swap + particle system

- The renderer already uses a shader program abstraction. Add a `LitColorShader` that accepts normal data and light uniforms.
- Particle system: a simple GPU particle emitter using a separate VBO with point sprites or small quads, updated per frame.

### 10.7 Sound → `AudioManager` module

A new top-level module, same pattern as `ScoreManager`:

```kotlin
object AudioManager {
    fun playOneShot(sfx: SfxId) { ... }
    fun setMusicTrack(track: MusicId) { ... }
    fun setSpeedPitch(speed: Float) { ... }
}
```

Backed by `SoundPool` (short SFX) and `MediaPlayer` or Oboe (streaming music).

### 10.8 Color & Textures → material system

Replace `uniform vec4 uColor` with a `Material` concept:

```kotlin
data class Material(
    val color: Vec4,
    val textureHandle: Int? = null,   // GL texture name, or null for solid color
)
```

The shader checks whether a texture is bound; if so, samples it using UV coords added to the vertex data.

---

## 11. Open Questions & Decisions

| # | Question | Options | Recommendation |
|---|---|---|---|
| Q1 | **Polygon side count for tunnel cross-section** | 6 (hex), 8 (oct), 10, 12 | Start with **8** (octagon). Looks good, low vertex count. Make it a constant so it's trivially changeable. |
| Q2 | **Segment depth (distance between rings)** | 0.5 – 2.0 world units | Start with **1.0**. Shorter = smoother curves but more geometry. Profile and adjust. |
| Q3 | **Coordinate convention: scroll world or move camera?** | (a) Camera moves forward, world is static. (b) World scrolls backward, camera is at Z=0. | **(a) Camera moves forward.** More intuitive for world-space collision math and generation. |
| Q4 | **HUD implementation** | (a) Android `View` overlay on top of `GLSurfaceView`. (b) GL-rendered text (bitmap font). | **(a) View overlay** for MVP — trivially easy with `TextView`. Switch to GL text if overlay causes perf issues (unlikely). |
| Q5 | **Audio library for Phase 2** | `SoundPool` + `MediaPlayer`, Oboe (NDK/C++), or a library like FMOD. | **`SoundPool` + `MediaPlayer`** for simplicity. Upgrade to Oboe if latency matters. |
| Q6 | **ECS (Entity-Component-System) pattern?** | (a) Adopt a lightweight ECS (e.g. Fleks, Artemis). (b) Keep the current object-oriented model. | **(b) Stay OO for Phase 1.** The entity count is tiny (~300 obstacles alive). If Phase 2 adds thousands of entities (particles, complex AI), revisit. |
| Q7 | **Testing strategy** | Unit tests for math/generation/collision; manual play-testing for feel. | **Unit test the deterministic parts** (generation with seed, collision geometry). Renderer is tested by eye. |
| Q8 | **Android target SDK** | 34 (latest stable as of 2026) | **Target SDK 35**, min SDK 26. |
| Q9 | **Build structure** | Single module or multi-module Gradle? | **Single module** for Phase 1. Extract `engine` module when/if it grows large. |

---

## Appendix A — Dependency Budget (Phase 1)

| Dependency | Purpose | Size impact |
|---|---|---|
| Kotlin stdlib | Language runtime | Already included by AGP |
| AndroidX Core | Lifecycle, compat | ~200 KB |
| (none else) | — | — |

**Total added dependency: ≈ 0.** The game engine is pure Kotlin + Android SDK.

---

## Appendix B — Rough File / Package Structure

```
app/
├── src/main/
│   ├── java/com/velocity/
│   │   ├── VelocityActivity.kt          // Entry point, GLSurfaceView setup
│   │   ├── engine/
│   │   │   ├── GameLoop.kt              // Fixed-timestep update logic
│   │   │   ├── GameState.kt             // Central state container
│   │   │   ├── PlayerState.kt
│   │   │   ├── DifficultyState.kt
│   │   │   └── GamePhase.kt
│   │   ├── tunnel/
│   │   │   ├── TunnelRing.kt
│   │   │   ├── TunnelChunk.kt
│   │   │   ├── TunnelBuffer.kt
│   │   │   ├── TunnelGenerator.kt
│   │   │   ├── Obstacle.kt
│   │   │   └── MapSource.kt             // Interface (procedural impl only in Phase 1)
│   │   ├── render/
│   │   │   ├── VelocityRenderer.kt      // GLSurfaceView.Renderer impl
│   │   │   ├── ShaderProgram.kt         // Compile/link helper
│   │   │   ├── Shaders.kt              // GLSL source strings
│   │   │   ├── MeshBuffer.kt           // VBO pool & upload
│   │   │   └── Camera.kt               // View/projection matrices
│   │   ├── input/
│   │   │   ├── InputManager.kt          // Accelerometer + touch
│   │   │   └── InputConfig.kt           // Deadzone, sensitivity
│   │   ├── collision/
│   │   │   └── CollisionSystem.kt
│   │   ├── score/
│   │   │   └── ScoreManager.kt          // Persist high score
│   │   └── math/
│   │       ├── Vec3.kt
│   │       ├── Mat4.kt
│   │       └── MathUtils.kt            // lerp, mod, clamp, etc.
│   ├── res/
│   │   ├── layout/
│   │   │   └── activity_main.xml        // GLSurfaceView + HUD overlay
│   │   └── values/
│   │       ├── strings.xml
│   │       └── styles.xml               // Fullscreen, no action bar
│   └── AndroidManifest.xml
├── build.gradle.kts
└── settings.gradle.kts
```

---

*Document version: 0.1 — Initial draft.*
*See also: [spec.md](spec.md)*

