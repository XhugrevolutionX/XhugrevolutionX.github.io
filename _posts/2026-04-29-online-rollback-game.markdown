---
layout: post
title: "Rollback Netcode from Scratch: What You Actually Need to Understand"
date: 2026-04-29
categories: [Game Development, Networking, C++]
tags: [Rollback Netcode, Online Multiplayer, Determinism, Photon, SDL3]
---

<details>
  <summary style="cursor: pointer; font-weight: bold;">Table of Contents</summary>
  <ul>
    <li><a href="#what-this-post-is">What This Post Is</a></li>
    <li><a href="#the-project-windjammers-online">The Project: Windjammers Online</a></li>
    <li><a href="#concept-1-why-lockstep-feels-bad">Concept 1: Why Lockstep Feels Bad</a></li>
    <li><a href="#concept-2-rollback-is-just-rewind-and-replay">Concept 2: Rollback Is Just Rewind and Replay</a></li>
    <li><a href="#concept-3-your-game-must-be-deterministic">Concept 3: Your Game Must Be Deterministic</a>
      <ul>
        <li><a href="#the-floating-point-problem">The Floating-Point Problem</a></li>
        <li><a href="#other-things-that-will-break-determinism">Other Things That Will Break Determinism</a></li>
      </ul>
    </li>
    <li><a href="#concept-4-the-two-model-pattern">Concept 4: The Two-Model Pattern</a></li>
    <li><a href="#concept-5-the-input-buffer">Concept 5: The Input Buffer</a></li>
    <li><a href="#concept-6-desync-detection-with-checksums">Concept 6: Desync Detection with Checksums</a></li>
    <li><a href="#architecture-why-mvc-matters-here">Architecture: Why MVC Matters Here</a></li>
    <li><a href="#problems-i-ran-into">Problems I Ran Into</a></li>
    <li><a href="#key-takeaways">Key Takeaways</a></li>
  </ul>
</details>

## What This Post Is

This is a write-up of my second-year networking assignment at SAE: building an online multiplayer game using **rollback netcode** from scratch. I'm writing this specifically for students who will do the same assignment — to explain the concepts that took me the longest to click, the mistakes I made, and the things I wish someone had told me before I started.

The game itself is secondary. The point is understanding **how rollback works and why it has to work that way**.

---

## The Project: Windjammers Online

The game is a simplified clone of Windjammers — two players on a 1280×720 court throwing a disc at each other's goal. Center goal = 5 points, outer goal = 3 points. First to 12 wins. Disc speed increases with each throw (600 → 1200 px/s). It runs at a fixed 60 FPS.

The tech stack: C++23, SDL3 for rendering, a custom 2D physics engine, Photon ExitGames SDK for networking, ImGui for the lobby UI.

None of that is what matters for this post. What matters is the netcode layer on top of it.

---

## Concept 1: Why Lockstep Feels Bad

Before understanding rollback, you need to understand what it replaces: **lockstep**.

In lockstep, both clients must have each other's inputs before they can advance the game. The sequence is:

1. Capture my input for frame N
2. Send it to the remote player
3. Wait for their input for frame N
4. Simulate frame N
5. Repeat

The problem is step 3. Even at 50ms ping (fast), you're waiting half a frame at 60 FPS. At 100ms — a realistic cross-country connection — you're stalling for 6 frames. Players feel it immediately. The game is "correct" but it feels like you're playing through mud.

The naive fix — adding input delay — moves the stall to a fixed offset. Your input takes effect 3 frames later. Technically it still waits, but now it feels like lag rather than stuttering. Fighting games like Street Fighter IV used this. It's still bad for anything below ~4 frames delay.

**Rollback fixes this by not waiting at all.**

---

## Concept 2: Rollback Is Just Rewind and Replay

Here is the core idea, stated as plainly as possible:

> Run the game at full speed using a **guess** for the remote player's input. If the guess was wrong when the real input arrives, **rewind** to the last known-good state and **replay** forward with the correct input.

That's it. The rest of the implementation is just answering: how do you guess, how do you rewind, and how do you make sure the replay produces the same result on both machines.

**Guessing** is simple: assume the remote player is doing whatever they did last frame. In practice this is correct the vast majority of the time — players don't change direction every single frame.

**Rewinding** means you need to store a snapshot of the game state at the last confirmed frame.

**Replaying** means you need to be able to re-simulate the game from that snapshot up to the present, frame by frame.

If the replay produces the same result on both machines, both players' screens converge to identical state. The rollback is completely invisible if the guess was close — and even if it wasn't, the correction happens faster than a single monitor refresh.

---

## Concept 3: Your Game Must Be Deterministic

This is the part that will cause you the most pain.

**Determinism** means: given the same starting state and the same sequence of inputs, the simulation produces exactly the same result every time — on every machine, every OS, every CPU.

If this condition fails, the rewind-and-replay produces different results on each client. The two games diverge. This is called a **desync**, and it usually manifests as players phasing through each other, the disc teleporting, or the score being different on the two screens.

### The Floating-Point Problem

The most common source of non-determinism is floating-point arithmetic. The IEEE 754 standard defines how `float` and `double` behave, but it leaves room for compilers to optimize expressions using **fused multiply-add (FMA)** instructions — computing `a*b+c` in a single operation with higher intermediate precision. This is fine for graphics, but it means two machines can produce slightly different values for the same computation if one uses FMA and the other doesn't.

The fix: force strict IEEE 754 compliance in your build system.

```cmake
if(MSVC)
    add_compile_options(/fp:strict)
else()
    add_compile_options(-ffp-contract=off -msse2 -mfpmath=sse)
endif()
```

This must be applied to **every library** that runs inside the simulation, including your physics engine. I spent time chasing a desync that turned out to be the physics engine being compiled without this flag — the game logic was deterministic but the body positions were diverging by tiny amounts that accumulated over time.

### Other Things That Will Break Determinism

- **`std::unordered_map` and `std::unordered_set`** — iteration order is not guaranteed. If your game logic iterates over either, the order can differ between runs or machines.
- **`rand()` or any non-seeded RNG** — random values will differ per client. If you need randomness, seed with a value both clients agree on (e.g. a hash of both players' IDs negotiated at connection time).
- **Reading system time inside the simulation** — wall clock values are obviously different per machine. Keep time-tracking outside the simulation tick.
- **Uninitialized memory** — if a struct has padding bytes that vary by compiler, checksums over that struct will differ. Zero-initialize everything in your game state.
- **`std::sort` on equal elements** — the standard only guarantees a valid sort, not a stable one. Use `std::stable_sort` or make your comparison fully deterministic.

The pattern for all of these: anything that depends on something other than the input sequence is a determinism bug.

---

## Concept 4: The Two-Model Pattern

To support rollback, you need two copies of your game simulation running in parallel:

| Model | What it represents |
|---|---|
| **Confirm model** | The last state both players agreed on — only advances when a frame is confirmed |
| **Speculative model** | The "live" simulation — always at the current frame, may be ahead of confirmation |

The confirm model is your snapshot for rewinding. The speculative model is what you render.

Every frame, the flow is:

1. Advance the speculative model (using the remote player's guessed input if their real one hasn't arrived yet)
2. If a new remote input arrived and it **differs** from what was guessed, trigger a rollback:
   - Copy confirm model → speculative model (rewind)
   - Re-simulate every frame from confirm up to present (replay)
3. When a frame is confirmed (both inputs are known), advance the confirm model one step

The re-simulation in step 2 looks like this:

```cpp
void RollbackAndResimulate() {
    // Rewind
    current_game_model.RollbackFrom(confirm_game_model);

    // Replay
    for (int i = 0; i < delta_frames; ++i) {
        current_game_model.set_inputs(inputs_at[confirm_frame + i]);
        current_game_model.Tick();
    }
}
```

This runs entirely within a single render frame. The player never sees a "rewind" — they just see the corrected state, which may look like the opponent snapping to a new position. At low latency this snap is sub-frame and invisible.

**Important:** `RollbackFrom` must be a true deep copy of all mutable state — every physics body position, velocity, game phase, score, timer. If you forget to copy something, that value will be wrong after every rollback.

---

## Concept 5: The Input Buffer

You need to store the last N frames of inputs for both players. I used a 64-frame buffer (about 1 second at 60 FPS — more than enough for any realistic latency).

The key operation is **speculative propagation**: when a remote input arrives for frame 45 while you're currently on frame 50, you don't just store it at index 45. You copy it forward to indices 46 through 50 as well, because your current speculation for those frames was based on whatever you last received. Then you check whether any of those frames differ from what was already stored — if they do, mark dirty and trigger a rollback.

```
Frame: 43  44  45  46  47  48  49  50 (current)
Old:    →   →   →   →   →   →   →   →  (last known input, propagated forward)
New:    ·   ·  [A]  A   A   A   A   A  (received input for frame 45, propagated)
```

If the received input `[A]` differs from what was speculated at frame 45 (and therefore at 46-50), a rollback happens. If it matches, nothing to do — the speculation was correct.

This means rollbacks only happen when predictions are wrong, not on every received packet.

---

## Concept 6: Desync Detection with Checksums

Even with all the determinism precautions, you want a way to **detect** desyncs during development. Without detection, the game just silently diverges and you have no idea when it happened or why.

The approach: after confirming a frame, compute a checksum over the entire game state and compare it between clients.

I used **Adler-32**, a simple incremental checksum. You add values to it in a fixed, deterministic order:

```
Checksum 0: Player 0 position.x, position.y, velocity.x, velocity.y
Checksum 1: Player 1 position.x, position.y, velocity.x, velocity.y
Checksum 2: Disc position, velocity + game phase + scores + timers
```

Only one client (Player 0, the "master") sends confirmations — a packet containing the frame number and these three 32-bit checksums. Player 1 computes the same checksums locally at that frame and compares. A mismatch means desync:

```
DESYNC at frame 1823: local=0xA3F2C1B0 master=0xD7E4920F
```

Which checksum mismatches tells you *what* diverged. If checksum 2 is wrong but 0 and 1 are right, the desync is in the disc or the game state, not the player positions. This narrows down the bug significantly.

During development this caught: the missing `/fp:strict` flag on the physics engine, an `unordered_map` with non-deterministic iteration, and an uninitialized timer field in the game state struct. None of those would have been findable without checksums.

---

## Architecture: Why MVC Matters Here

You'll notice that rollback requires running the game simulation multiple times per frame (once for the speculative step, potentially many more for re-simulation). This is only manageable if the simulation is **completely separated from rendering**.

The architecture that makes this possible is MVC:

- **Model**: pure game logic and physics. No rendering calls, no I/O. Can be ticked multiple times per frame without side effects.
- **View**: reads from the model to draw the current state. Runs once per render frame.
- **Controller**: drives both — captures input, calls `model.Tick()` as many times as needed, then lets the view render.

If you put rendering calls inside your game logic (e.g. `DrawHealthBar()` inside `Player::TakeDamage()`), every re-simulation will fire draw calls, which will corrupt your frame or crash. Keep the model pure.

The same applies to sound. Don't play sounds inside simulation ticks. Collect events from the simulation into a list, then play them once per render frame after all ticks are done.

---

## Problems I Ran Into

**Desync from the physics engine.** The game logic was deterministic but the simulation was diverging. The bug was that the physics engine (compiled as a separate library) wasn't inheriting the `/fp:strict` flag. The fix was explicitly adding the flag to the physics engine's CMake target. Lesson: the determinism flags must cover every library that runs inside the simulation tick.

**Rollback copying the wrong state.** Early on, `RollbackFrom()` didn't copy the physics overlap state (which colliders are currently touching). After a rollback, collisions were firing twice — once from the old overlap state that was left stale, once from the re-simulation. The fix was making `PhysicsWorld::RollBackFrom()` a complete copy of all body state *and* all collider overlap records.

**Input arriving for already-confirmed frames.** Network packets can arrive out of order or very late. When an input arrived for a frame that was already confirmed, the code was triggering a rollback on the confirm model, which corrupted it. The fix was to ignore any input for a frame number at or below `last_confirm_frame_`.

**Action button triggering multiple times after rollback.** The action button (throw/catch) was edge-detected — it only fires on the frame the button is first pressed. After a rollback and re-simulation, the re-simulation was replaying those frames and re-triggering the edge detection. The fix was storing the edge-detected result in the input buffer, not re-computing it during re-simulation. If the input said "action = true at frame 40", that value was used as-is during replay, not re-derived from button state.

---

## Key Takeaways

If you take one thing from each section:

1. **Lockstep** stalls the game to wait for the network. Rollback never waits — it guesses and corrects.
2. **Rollback = snapshot + replay.** Store a confirmed state, re-simulate when the guess was wrong.
3. **Determinism is mandatory and fragile.** Enable `/fp:strict` or equivalent on every compiled target in your simulation. Checksums will save you hours of debugging.
4. **Two models, not one.** The confirm model is your rollback snapshot. The speculative model is what you render.
5. **Propagate speculative inputs forward.** Only roll back when a received input *differs* from the speculation, not on every packet.
6. **MVC isn't optional.** Your simulation must be re-entrant — tickable multiple times per frame with no rendering side effects.

The implementation isn't conceptually complicated, but the details will catch you if you're not methodical. Get checksums working early, before you have any gameplay to test. An hour of desync debugging without checksums is worth ten minutes with them.
