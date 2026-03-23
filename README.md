# rubiks-cube-cracker

A virtual Rubik's Cube with an optimal solver written in C++ and rendered with OpenGL.

The optimal solver (Korf's algorithm) solves any scrambled cube in **≤20 moves**, guaranteed. It's more performant than other tested optimal solvers, but deep scrambles can take time — so there's also a Thistlethwaite solver that solves any cube **near-instantly** in at most 46 moves.

---

## Building & Running

See [BUILDING.md](BUILDING.md). Developed on Linux (g++), also tested on Windows via MinGW (64-bit).

---

## Controls

### Face Twists
| Key | Move |
|-----|------|
| `U` `L` `F` `R` `B` `D` | Clockwise twist of that face |
| `SHIFT` + above | Counter-clockwise (prime) |
| `ALT` + above | 180° (double) twist |

### Cube Rotation
| Key | Action |
|-----|--------|
| `↑` `↓` `←` `→` | X / Y rotations |
| `Z` | Z rotation (`SHIFT+Z` for prime) |

### Slice Moves
`M`, `E`, `S` for their respective slices.

### Solvers & Misc
| Key | Action |
|-----|--------|
| `F1` | Thistlethwaite (fast, ≤46 moves) |
| `F2` | Korf optimal (≤20 moves, slow for hard scrambles) |
| `F5` | Apply a 100-move scramble |

---

## Rubik's Cube Notation

Standard notation throughout: `U L F R B D` are 90° clockwise twists of the up, left, front, right, back, and down faces. Append `'` for counter-clockwise, `2` for 180°. That gives 18 total moves.

Cubies are the small pieces: **corner cubies** have 3 stickers, **edge cubies** have 2, and **center cubies** are fixed. More on notation [here](https://ruwix.com/the-rubiks-cube/notation/).

---

## Renderer

Written from scratch in OpenGL — no shortcuts taken.

- **Quaternions + SLERP** for face rotation animations. Euler angles cause gimbal lock and ugly snapping; quaternions interpolate smoothly between any two orientations, which is exactly what you want when animating a face mid-twist.
- **Phong reflection model** — custom per-fragment lighting with a single distance light and specular materials. You can see this clearly when rotating the whole cube; the highlights shift realistically as the angle changes. Most hobby renderers skip this and just flat-shade everything.
- **Procedurally-generated stickers** with intentional imperfections (smudges). A perfectly clean sticker looks fake. The noise makes it feel like a cube that's actually been used.
- **Levitation effect** inspired by original EverQuest characters. Purely aesthetic — the cube hovers with a subtle bob. Fair warning: might cause motion sickness.

---

## Optimal Solver (Korf's Algorithm)

Solves any cube in ≤20 moves using **IDA\*** (iterative-deepening A\*).

### Why IDA\* and not BFS?

A plain breadth-first search would work in theory — it's guaranteed optimal — but the memory requirements are astronomical. The Rubik's Cube has ~43 quintillion states. IDDFS gives you the same optimality guarantee as BFS but explores depth-by-depth using almost no memory, since you're just doing DFS repeatedly with a tighter depth limit each time.

Raw IDDFS is still too slow though. The branching factor is ~13 even after pruning obviously redundant moves (like `F F'` which cancels out, or `F B` which equals `B F`). Without guidance, you'd be searching for thousands of years. That's where A\* comes in — it uses a heuristic to estimate how far any given state is from solved, and cuts off branches that are provably going to overshoot the current depth limit.

### Pattern Databases (the Heuristic)

The heuristic needs to be fast to compute and must never overestimate (otherwise A\* loses its optimality guarantee). Korf's solution: precompute the exact number of moves to solve sub-problems and store them in lookup tables. At runtime, look up the current cube state and get a guaranteed lower bound on moves remaining.

| Database | What it stores | Size |
|----------|----------------|------|
| Corner positions | All 8! × 3⁷ = 88,179,840 corner states | ~42 MB |
| 7-edge set A | 12P7 × 2⁷ states, ≤10 moves | ~244 MB |
| 7-edge set B | Same, different 7 edges | ~244 MB |
| Edge permutations | All 12! / 2 arrangements | ~228 MB |

**Why 7-edge databases instead of Korf's original 6?** More edges per database = tighter lower bounds = fewer nodes explored during IDA\*. The jump from 6 to 7 edges is a massive speed improvement. Going to 8 would push each database to ~2.4 GB — not worth it for most machines.

**Why a separate edge permutation database?** The 7-edge databases track both position and orientation of a subset of edges. The permutation DB covers all 12 edges but ignores orientation. Together they give a stronger combined heuristic than either alone.

### Indexing (Why This is Faster Than Other Solvers)

Given a scrambled cube, you need to convert its state into an index into one of these databases. The naive way — scan through and count — is O(n²) per lookup. This implementation uses a linear algorithm for converting permutations to [Lehmer codes](https://en.wikipedia.org/wiki/Lehmer_code), which runs in O(n). At the scale of IDA\* (millions of lookups per second), this is the single biggest performance differentiator compared to other optimal solvers.

---

## Quick Solver (Thistlethwaite's Algorithm)

Solves any scramble in ≤46 moves, effectively instantaneously. Also uses IDA\* with pattern databases, but with a key difference: these databases give the **exact** move count to reach the next group (not just a lower bound). The tradeoff is you get a suboptimal solution, but you get it in milliseconds instead of hours.

The core idea: instead of solving the whole cube at once, break it into four stages. Each stage restricts the move set further, making the remaining problem easier to reason about.

| Group | Allowed Moves | DB Size | Max Twists |
|-------|--------------|---------|------------|
| G0 → G1 | All 18 moves | 2,048 | 7 |
| G1 → G2 | No F/B quarter turns | 1,082,565 | 10 |
| G2 → G3 | No L/R quarter turns | 352,800 | 14 |
| G3 → Solved | Only half turns (U2 F2 L2 R2 B2 D2) | 663,552 | 15 |

**G0→G1 — Orient all edges.** Edge pieces have two orientations (flipped or not). The insight: edge pieces can't be flipped if you never use F/B quarter turns. So get all 12 edges correctly oriented first, and you permanently unlock that constraint for the rest of the solve.

**G1→G2 — Orient corners + place E-slice edges.** With F/B quarter turns gone, orient all corners and move the FR, FL, BL, BR edges into the E slice. Corners can now never be mis-oriented again.

**G2→G3 — Pair up corners within tetrads.** This deviates from Thistlethwaite's original. Instead of computing tetrad twists (complex and error-prone), this uses Stefan Pochmann's technique: pair corners `{ULB, URF}`, `{DLF, DRB}`, etc. within their tetrads, and ensure even corner parity. Even corner parity forces even edge parity too — which means the final stage is always solvable with half turns alone. Also positions M- and S-slice edges.

**G3→Solved — Pure half turns.** Only 6 moves allowed. The cube is constrained enough that this group is tiny and solves instantly.

**Why ≤46 and not Thistlethwaite's original ≤52?** The Group 3 implementation differs — Pochmann's pairing approach finds shorter paths through that stage than the original tetrad twist method.

---

## Optimal Solver Benchmarks

10 random 100-move scrambles on a Core i7 Sandy Bridge (2011), code v2.2.0. Newer releases are faster.

Scramble length doesn't directly correlate with solve difficulty — what matters is how many moves the *optimal* solution takes. God's number is 20, meaning no cube position requires more than 20 moves to solve. Most scrambles land in the 17–18 move range. The one 19-move solution below took 24+ hours — that's near the hardest possible case for this algorithm.

| Solution | Moves | Time (s) | Time (hrs) |
|----------|-------|----------|------------|
| R B U2 L2 U F2 R2 F2 U F' U2 R2 F' U' F2 L B L' | 18 | 24,152 | 6.71 |
| B' L2 R' F U' B' R F' D2 B U2 D R U F B' L B U2 | 19 | 88,412 | 24.56 |
| U2 B' U F B U' R F' R2 F' U' R B' L D' L2 R U2 | 18 | 25,116 | 6.98 |
| R F U' D' B2 R B2 D2 F' R' B U2 F R D2 R2 U2 D' | 18 | 9,788 | 2.72 |
| U2 B L R' F2 U2 L' F2 R U' L2 F' L R' U L' R2 D' | 18 | 25,470 | 7.08 |
| B L U' L' U' B R U' B' U' L2 U' F R F B R U | 18 | 14,478 | 4.02 |
| D' F2 U' R2 U L U2 B' L' R' U2 B' R' F2 U' F B2 R' | 18 | 38,683 | 10.75 |
| L' R2 D L' R2 B' D' F' R B R' U2 D' B2 L' F2 R' | 17 | 2,212 | 0.61 |
| D' B2 L F2 R' B2 L D2 F2 U F' U' R F' D L' U L2 | 18 | 37,914 | 10.53 |
| F D2 F' B D F L F U' F2 L2 B' L B R' F2 L' R' | 18 | 37,511 | 10.42 |
