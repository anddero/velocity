# Velocity — Backlog

> Incremental task list for building the MVP (Phase 1).
> Derived from [spec.md](spec.md) and [architecture.md](architecture.md).

---

## 0 · Project Setup

- [x] Initialize Android project with Gradle (Kotlin DSL), min SDK 26, target SDK 35
- [x] Configure landscape-only orientation and fullscreen (no action bar) in manifest and styles
- [x] Create `VelocityActivity` with a `GLSurfaceView` as the main content view
- [x] Verify the app launches to a blank GL screen on a device or emulator

## 1 · Math Utilities

- [ ] Implement `Vec3` (add, subtract, scale, normalize, dot, cross, length)
- [ ] Implement `Mat4` (identity, multiply, translate, rotate, perspective, lookAt)
- [ ] Implement `MathUtils` (lerp, clamp, mod, toRadians, toDegrees)
- [ ] Write unit tests for math utilities

## 2 · Shader Infrastructure

- [ ] Create `ShaderProgram` helper (compile vertex/fragment, link, uniform/attribute accessors)
- [ ] Define flat-color vertex and fragment shader source strings in `Shaders.kt`
- [ ] Verify shaders compile and link successfully on GL thread startup

## 3 · Basic Rendering

- [ ] Implement `Camera` (perspective projection matrix, view matrix with position and forward)
- [ ] Implement `VelocityRenderer` (`GLSurfaceView.Renderer`) with clear color and camera setup
- [ ] Render a single hardcoded triangle or quad to confirm the pipeline works end-to-end

## 4 · Tunnel Geometry

- [ ] Implement `TunnelRing` — compute vertex positions for an N-sided polygon ring at a given center/forward/up
- [ ] Implement panel vertex generation between two adjacent rings (4 vertices per panel, 2 triangles)
- [ ] Implement `TunnelChunk` — build a straight sequence of rings and pack panel vertices into a `FloatArray`
- [ ] Render a single hardcoded straight tunnel chunk to verify geometry

## 5 · VBO Management

- [ ] Implement `MeshBuffer` — VBO pool with create, upload, bind, and recycle operations
- [ ] Upload tunnel chunk vertex data to a VBO and draw using `glDrawArrays`
- [ ] Confirm stable frame rate with a few hundred panels on screen

## 6 · Tunnel Buffer & Scrolling

- [ ] Implement `TunnelBuffer` (ArrayDeque of chunks with append/retire logic)
- [ ] Generate multiple chunks in a row to form a long straight tunnel
- [ ] Implement camera forward movement (advance Z each frame at constant speed)
- [ ] Cull / skip drawing chunks behind the camera
- [ ] Generate new chunks ahead of the camera as it advances
- [ ] Recycle chunks and VBOs that fall behind the camera

## 7 · Depth Fade / Fog

- [ ] Implement depth-fade fragment shader variant (mix toward background color based on Z)
- [ ] Apply depth fade to tunnel panels so far-end segments smoothly disappear

## 8 · Tunnel Curvature

- [ ] Implement smooth heading (yaw/pitch) evolution across a chunk using cosine interpolation
- [ ] Generate curved tunnel chunks with rings whose centers follow the curved path
- [ ] Ensure the camera auto-follows the tunnel center through curves

## 9 · Panel Coloring

- [ ] Alternate panel shades (e.g. odd/even side index or ring index) for visual depth
- [ ] Set a distinct accent color for the tunnel against a dark background

## 10 · Obstacle Generation

- [ ] Implement `Obstacle` data class with segment index, side index, offset, and size
- [ ] Generate random obstacles within a chunk based on a density parameter
- [ ] Build cube vertex data for each obstacle and append to the chunk's obstacle vertex array
- [ ] Render obstacles in a contrasting color

## 11 · Game State & Phase Machine

- [ ] Implement `GameState`, `PlayerState`, `DifficultyState`, and `GamePhase` enum
- [ ] Implement state transitions: TITLE → PLAYING → DEAD → TITLE, and PLAYING ↔ PAUSED
- [ ] Wire the renderer to only advance the game world when phase is PLAYING

## 12 · Game Loop (Fixed Timestep)

- [ ] Implement fixed-timestep accumulator in `onDrawFrame` (120 Hz tick)
- [ ] Implement `GameLoop.update(dt)` with the correct update order (input → move → generate → collide → score)
- [ ] Implement render-state interpolation for smooth visuals between ticks

## 13 · Input — Accelerometer

- [ ] Implement `InputManager` with accelerometer sensor listener
- [ ] Apply low-pass filter, deadzone, and normalization to raw sensor values
- [ ] Expose a thread-safe lateral input value (−1.0 … +1.0) via `AtomicReference`

## 14 · Input — Touch Fallback

- [ ] Implement touch drag input as a fallback (drag left/right from screen center)
- [ ] Map drag distance to the same −1.0 … +1.0 lateral value
- [ ] Allow switching between accelerometer and touch in a config/settings flag

## 15 · Player Lateral Movement

- [ ] Map lateral input to angular velocity (dθ/dt) and update `player.theta` each tick
- [ ] Apply visual roll (rotate view matrix around Z) based on input, with smoothing
- [ ] Verify the tunnel appears to rotate when tilting / dragging

## 16 · Collision Detection

- [ ] Implement angular-overlap check between player θ span and obstacle angular span
- [ ] Implement depth-overlap check between player Z and obstacle Z extent
- [ ] Handle wraparound at 2π for angular checks
- [ ] Transition to DEAD on collision
- [ ] Write unit tests for collision edge cases (wraparound, near-miss, direct hit)

## 17 · Difficulty Progression

- [ ] Implement `DifficultyState` with level ramp over time (0.0 → 1.0)
- [ ] Drive forward speed from difficulty level
- [ ] Drive obstacle density and gap width from difficulty level
- [ ] Drive tunnel curvature range from difficulty level
- [ ] Tune ramp rate and plateau values for playability

## 18 · Scoring

- [ ] Accumulate distance traveled each tick as the score
- [ ] Implement `ScoreManager` — persist high score via `SharedPreferences`
- [ ] Load high score on app start and save on run end

## 19 · HUD Overlay

- [ ] Create an Android `View` overlay on top of `GLSurfaceView` (FrameLayout)
- [ ] Display running distance counter (top-center) during PLAYING
- [ ] Optionally display speed multiplier (top-right) during PLAYING
- [ ] Hide HUD elements when not in PLAYING phase

## 20 · Title Screen

- [ ] Design a simple title screen layout ("Velocity" title, "Tap to Start" prompt)
- [ ] Use the spinning triangle as a loading/splash visual on the title screen
- [ ] Show the title overlay when phase is TITLE
- [ ] Transition to PLAYING on tap
- [ ] Add an "Exit" button that closes the app

## 21 · Death Screen

- [ ] Design a death screen overlay (run score, high score, "Restart" and "Exit" buttons)
- [ ] Show the death overlay when phase is DEAD
- [ ] Screen flash or brief visual effect on collision
- [ ] "Restart" resets game state and transitions to PLAYING
- [ ] "Exit" closes the app

## 22 · Pause Screen

- [ ] Design a pause overlay ("Paused", "Resume" and "Exit" buttons)
- [ ] Pause on tap during gameplay or Android back button press
- [ ] Resume on "Resume" button tap
- [ ] "Exit" closes the app
- [ ] Pause rendering/updates when the app goes to background (`onPause`/`onResume`)

## 23 · Seeded RNG & Reproducibility

- [ ] Wire the tunnel generator to use a seeded `kotlin.random.Random`
- [ ] Use a random seed per run in infinite mode
- [ ] Verify that the same seed produces the same tunnel

## 24 · Performance Profiling & Polish

- [ ] Profile frame rate on a mid-range device — target stable 60 fps
- [ ] Minimize GC allocations in the game loop (object pooling, pre-allocated arrays)
- [ ] Verify draw call count stays minimal (2–4 per frame)
- [ ] Check RAM usage stays under 100 MB
- [ ] Verify APK size is under 15 MB

## 25 · Final MVP Polish

- [ ] Fade-in effect on game start
- [ ] Tune tunnel radius, segment depth, polygon side count for best visual feel
- [ ] Tune player angular speed and visual roll angle for satisfying controls
- [ ] Tune obstacle sizing and spacing for fair gameplay
- [ ] Playtest and iterate on difficulty curve
- [ ] Verify lifecycle handling (rotate, background, kill — no crashes)

---

*Backlog version: 0.1 — Initial draft.*
*See also: [spec.md](spec.md) · [architecture.md](architecture.md)*

