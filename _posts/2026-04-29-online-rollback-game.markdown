---
layout: post
title: "Rollback Netcode from Scratch: Key Concepts and Implementation"
date: 2026-04-29
categories: [Game Development, Networking, C++]
tags: [Rollback Netcode, Online Multiplayer, Determinism, Photon, SDL3]
---

<details>
  <summary style="cursor: pointer; font-weight: bold;">Table of Contents</summary>
  <ul>
    <li><a href="#the-project-windjammers-online">The Project: Windjammers Online</a></li>
    <li><a href="#why-lockstep-falls-short">Why Lockstep Falls Short</a></li>
    <li><a href="#rollback-rewind-and-replay">Rollback: Rewind and Replay</a></li>
    <li><a href="#determinism">Determinism</a>
      <ul>
        <li><a href="#the-floating-point-problem">The Floating-Point Problem</a></li>
        <li><a href="#other-common-sources-of-non-determinism">Other Common Sources of Non-Determinism</a></li>
      </ul>
    </li>
    <li><a href="#the-two-model-pattern">The Two-Model Pattern</a></li>
    <li><a href="#the-input-buffer">The Input Buffer</a></li>
    <li><a href="#desync-detection-with-checksums">Desync Detection with Checksums</a></li>
    <li><a href="#architecture-mvc-and-simulation-purity">Architecture: MVC and Simulation Purity</a></li>
    <li><a href="#notable-bugs-and-fixes">Notable Bugs and Fixes</a></li>
    <li><a href="#key-takeaways">Key Takeaways</a></li>
  </ul>
</details>

## The Project: Windjammers Online

The project is a simplified online clone of Windjammers — a two-player competitive disc-throwing game. Both players share a 1280×720 court and take turns throwing a disc into the opponent's goal. The center section of the goal scores **5 points**, the outer sections score **3 points**. First to **12 points** wins. Disc speed escalates with each successful throw, starting at 600 px/s and capping at 1200 px/s.

**Tech stack:** C++23, SDL3 for rendering, a custom 2D physics engine, Photon ExitGames SDK for networking, ImGui for the lobby UI. The simulation runs on a fixed 60 FPS timestep.

The focus of this post is the netcode layer — specifically the rollback implementation, why each piece exists, and the bugs that surface when building it.

---

## Why Lockstep Falls Short

The baseline approach to online multiplayer is **lockstep**: both clients must exchange inputs before simulating each frame.

```
1. Capture local input for frame N
2. Send it to the remote client
3. Wait for the remote client's input for frame N
4. Simulate frame N
5. Repeat
```

Step 3 is the problem. At 60 FPS, a 100ms round-trip stalls the simulation for 6 frames. Players feel this immediately — the game runs correctly but the controls feel unresponsive.

The common mitigation is **input delay**: both players commit to inputs N frames in the future, buying time for packets to arrive before they are needed. Fighting games like Street Fighter IV used this approach. It reduces stuttering but trades it for artificial latency; anything above 3–4 frames of delay becomes perceptible.

Rollback eliminates both problems by never waiting.

---

## Rollback: Rewind and Replay

The core principle of rollback netcode:

> Run the simulation at full speed using a **speculative input** for the remote player. If the real input arrives and differs from the speculation, **rewind** to the last confirmed state and **replay** the simulation forward with the correct input.

Three sub-problems follow from this:

- **Speculation**: When no remote input is available, assume the remote player is repeating their last known input. This is correct the large majority of the time — direction changes are infrequent at the frame level.
- **Rewinding**: A snapshot of the full game state must be stored at the last confirmed frame. This snapshot is what gets restored on rollback.
- **Replaying**: The simulation must be re-runnable from any snapshot, frame by frame, purely from inputs — no hidden state, no side effects.

When the replay produces identical results on both clients, the two simulations converge to the same state. A correct rollback is invisible to the player if the speculative inputs were close; if they were wrong, the correction completes within a single render frame.

---

## Determinism

Rollback only works if both clients produce **identical simulation results** from the same inputs. Any divergence — called a **desync** — causes the two simulations to silently drift apart. Typical symptoms: players phasing through each other, the disc position disagreeing between screens, scores diverging.

Determinism must hold across different machines, operating systems, and compiler versions.

### The Floating-Point Problem

The most dangerous source of non-determinism is floating-point arithmetic. The IEEE 754 standard governs `float` and `double` behavior, but allows compilers to use **fused multiply-add (FMA)** instructions — computing `a*b+c` in a single step with higher intermediate precision than IEEE 754 requires. One machine may use FMA for a given expression; another may not. The results are both valid IEEE 754, but they differ.

The fix is to enforce strict IEEE 754 compliance across every compiled target that runs inside the simulation:

```cmake
if(MSVC)
    add_compile_options(/fp:strict)
else()
    add_compile_options(-ffp-contract=off -msse2 -mfpmath=sse)
endif()
```

This flag must be applied to **every library in the simulation** — including third-party physics engines. A single library compiled without it can produce diverging body positions that compound frame over frame.

### Other Common Sources of Non-Determinism

- **`std::unordered_map` / `std::unordered_set`**: iteration order is unspecified. Any game logic that iterates these containers may process elements in a different order between runs or machines.
- **Unseeded RNG**: `rand()` produces different values per client. If randomness is required, both clients must agree on a seed at connection time.
- **System clock inside the simulation tick**: wall clock values are obviously different per machine. Timers must be tracked as frame counters, not real time.
- **Uninitialized memory**: padding bytes in structs can vary by compiler. Checksumming a struct with uninitialized padding will produce different results. Zero-initialize all game state structs.
- **`std::sort` on equal elements**: the standard only guarantees a sorted result, not a stable one. Use `std::stable_sort` or ensure comparators are fully deterministic.

The general rule: anything inside a simulation tick that depends on something other than the input sequence is a determinism bug.

---

## The Two-Model Pattern

Supporting rollback requires maintaining two parallel copies of the game simulation:

| Model | Role |
|---|---|
| **Confirm model** | Snapshot of the last frame both players' inputs are known. Never advances speculatively. |
| **Speculative model** | The live simulation, always at the current frame. Rendered every frame. |

The confirm model is the rollback target. The speculative model is what the player sees.

Per-frame flow:

1. Advance the speculative model using the latest available inputs (speculative for the remote player if their packet hasn't arrived)
2. If a received remote input **differs** from what was speculated, trigger a rollback:
   - Deep-copy the confirm model into the speculative model (rewind)
   - Re-simulate from confirm frame to present using correct inputs (replay)
3. When a frame is confirmed (both inputs are known and checksums match), advance the confirm model one step

```cpp
void RollbackAndResimulate() {
    current_game_model.RollbackFrom(confirm_game_model);

    for (int i = 0; i < delta_frames; ++i) {
        current_game_model.set_inputs(inputs_at[confirm_frame + i]);
        current_game_model.Tick();
    }
}
```

`RollbackFrom` must be a complete deep copy: every physics body position, velocity, game phase, score, timer, and collider state. Any field omitted from the copy will hold a stale value after every rollback.

---

## The Input Buffer

A circular buffer stores the last N frames of inputs for both players — 64 frames (roughly one second at 60 FPS) is sufficient for realistic network latency.

The critical operation is **speculative propagation**. When a remote input packet arrives for frame 45 while the simulation is at frame 50, the received input is stored at index 45 and then copied forward to indices 46–50, overwriting whatever was speculated there:

```
Frame: 43  44  45  46  47  48  49  50  (current)
Old:    →   →   →   →   →   →   →   →  (speculation held forward)
New:    ·   ·  [A]  A   A   A   A   A  (received for frame 45, propagated forward)
```

A rollback is only triggered if any of the newly received values **differ** from what was already stored. If the speculation matched reality — which is common when the player holds a direction for multiple frames — there is nothing to correct and the simulation continues undisturbed.

This means resimulation cost is proportional to how often predictions are wrong, not to how often packets arrive.

---

## Desync Detection with Checksums

Determinism bugs cause silent divergence. Without active detection, the game simply runs incorrectly on one or both clients with no indication of when or why the drift began.

The solution is to checksum the full game state on every confirmed frame and compare between clients. The implementation uses **Adler-32**, a lightweight incremental checksum. Values are accumulated in a fixed, deterministic order:

```
Checksum 0: Player 0 position.x, position.y, velocity.x, velocity.y
Checksum 1: Player 1 position.x, position.y, velocity.x, velocity.y
Checksum 2: Disc position, velocity + game phase + scores + frame timers
```

Only Player 0 (the "master") sends confirmation packets — each containing the frame number and these three 32-bit checksums. Player 1 computes the same checksums locally at that frame and compares. A mismatch is logged immediately:

```
DESYNC at frame 1823: local=0xA3F2C1B0 master=0xD7E4920F
```

Splitting the checksum by object means a mismatch isolates the source: if checksum 2 is wrong but 0 and 1 match, the disc or game state diverged, not player positions. This significantly narrows the debugging surface.

Checksums should be implemented before gameplay is built. Desyncs found early are isolated to a small number of recently added lines of code; desyncs found late can be anywhere.

---

## Architecture: MVC and Simulation Purity

Rollback requires ticking the simulation multiple times per render frame — once for the speculative advance, potentially many more times during re-simulation. This is only possible if the simulation has **no rendering or audio side effects**.

The project uses a strict MVC separation:

- **Model** (`GameModel`): pure game logic and physics. No rendering calls, no I/O. Can be ticked any number of times per frame.
- **View** (`GameView`): reads from the model to produce one rendered frame. Runs once per render frame.
- **Controller** (`GameOnlineController`): drives the model (ticking it as many times as needed), then triggers the view.

Placing rendering calls inside simulation logic — for example, spawning a particle effect inside a collision handler — causes those calls to fire during every re-simulation tick. The result is either corrupted rendering or a crash. The model must remain a pure state machine.

The same constraint applies to audio. Sound triggers should be collected as events during simulation ticks and played back once per render frame after all ticks complete.

---

## Notable Bugs and Fixes

**Physics engine not covered by `/fp:strict`.** The game logic was deterministic but bodies were diverging. The physics engine was compiled as a separate CMake target and did not inherit the strict floating-point flag from the top-level project. Adding `/fp:strict` explicitly to the physics engine target resolved the desync. Every library that runs inside the simulation tick needs this flag applied directly.

**Incomplete deep copy in `RollbackFrom`.** The initial implementation of `RollbackFrom` copied body positions and velocities but not the physics overlap state — which colliders are currently touching. After a rollback, collision triggers were firing twice: once from the stale pre-rollback overlap state, and once from the correct re-simulated state. The fix was extending `PhysicsWorld::RollBackFrom` to copy all collider overlap records in addition to body state.

**Input arriving for already-confirmed frames.** Network packets can arrive significantly out of order. When a packet carrying an input for a frame already past the confirm index was processed, it triggered a rollback on the confirm model rather than being discarded. This corrupted the confirm snapshot. The fix was to reject any incoming input with a frame number at or below `last_confirm_frame_`.

**Edge-detected input re-triggering during re-simulation.** The action button (throw/catch) uses edge detection — it fires only on the frame the button transitions from released to pressed. During re-simulation, the edge detection was being recomputed from raw button state rather than replayed from the stored input. This caused the action to trigger again on frames where it had already fired. The fix was storing the edge-detected boolean in the input buffer directly, so re-simulation replays the result rather than recomputing it.

---

## Key Takeaways

1. **Lockstep stalls; rollback speculates.** The fundamental difference is whether the simulation waits for network data or runs ahead and corrects.
2. **Rollback = snapshot + replay.** A confirmed state snapshot enables rewinding; a pure deterministic simulation enables replaying.
3. **Determinism is mandatory and fragile.** Apply IEEE 754 strict flags to every library in the simulation. Any external dependency breaks the guarantee.
4. **Two models are required.** The confirm model provides the rollback target. The speculative model is what gets rendered.
5. **Propagate inputs forward; roll back only on mismatch.** Rollbacks are proportional to prediction error, not to packet frequency.
6. **Checksums are non-optional for debugging.** Implement them before gameplay. A desync found in isolation is a ten-minute fix; one found after weeks of development can take days.
7. **The model must be side-effect free.** No rendering, no audio, no I/O inside simulation ticks. MVC enforces this separation cleanly.
