# RubiksCube-VisualSolver

**A 3-Dimensional Interactive Rubik's Cube Simulator with Reverse-History State Reversion**

> A browser-based 3D Rubik's Cube that scrambles and solves itself by reversing its move history — built with Three.js.

---

## Abstract

This project presents a browser-based, real-time 3D simulation of the 3×3×3 Rubik's Cube puzzle, implementing a complete visualization pipeline from geometric state representation to animated group-action playback. The system employs a **reverse-history algorithm** for state reversion: every face rotation applied to the cube is recorded as an abstract group element; the "solve" operation constructs the inverse word by reversing the recorded sequence and negating each generator, thereby returning the configuration to its identity state. The visual front-end leverages Three.js with physically-based rendering (PBR), ACES Filmic Tone Mapping, and spherical linear interpolation (slerp) for smooth rotational animations. A finite state machine governs user interaction concurrency, enforcing mutual exclusion between input and animation phases.

---

## 1. Problem Statement

### 1.1 The Rubik's Cube Group

The 3×3×3 Rubik's Cube represents a permutation puzzle whose state space comprises approximately:

$$\frac{8! \times 12! \times 3^8 \times 2^{12}}{12} \approx 4.325 \times 10^{19}$$

distinct configurations. Each configuration is reachable via sequences of face rotations, where each face turn constitutes a generator of the Rubik's Cube Group — a subgroup of the symmetric group acting on the 54 coloured facelets (or equivalently, on the 26 visible cubies).

### 1.2 Computational Challenges

Existing solvers fall into two broad categories:

- **Algorithmic optimal solvers** (Kociemba's Two-Phase Algorithm, Korf's IDA* search) that compute near-optimal or optimal solutions but operate as black boxes with minimal visual feedback
- **Static simulators** that permit manual face twisting without demonstrating the principled inverse-solve process

The challenge addressed herein is threefold:

1. **Model** the cube as a composition of 27 individual cubies with independent spatial positions and orientations, rendered in 3D with realistic materials
2. **Support** arbitrary face rotations with real-time animation, providing continuous visual feedback during each group action
3. **Demonstrate** a trivially correct solve strategy — the reverse-history algorithm — that illustrates the fundamental group-theoretic property that any word in the generators admits an inverse word

---

## 2. Algorithmic Approach

### 2.1 Reverse-History State Reversion

The core algorithmic contribution is the **reverse-history solver**, which exploits the algebraic structure of the Rubik's Cube Group:

**Definition.** Let $G = \langle R, L, U, D, F, B \rangle$ be the Rubik's Cube Group, where each generator represents a 90° clockwise rotation of the corresponding face. A *move* is an element $m \in \{R, L, U, D, F, B, R^{-1}, L^{-1}, U^{-1}, D^{-1}, F^{-1}, B^{-1}\}$.

**Definition.** A *scramble* is a word $w = m_1 m_2 \cdots m_n \in G$. The *scrambled state* is $S = w \cdot I$, where $I$ is the identity (solved) state and $\cdot$ denotes the group action on the cube.

**Proposition.** The inverse word $w^{-1} = m_n^{-1} m_{n-1}^{-1} \cdots m_1^{-1}$ satisfies $w^{-1} \cdot (w \cdot I) = I$.

**Proof.** By associativity of the group action: $w^{-1} \cdot (w \cdot I) = (w^{-1} w) \cdot I = I \cdot I = I$.

The solver computes $w^{-1}$ in $O(n)$ time by:

1. Maintaining an ordered log $H = [m_1, m_2, \ldots, m_n]$ of all moves applied
2. On solve: constructing $H_{\text{rev}} = [m_n^{-1}, m_{n-1}^{-1}, \ldots, m_1^{-1}]$ where each $m_i = (\text{face}, \text{dir})$ and $m_i^{-1} = (\text{face}, -\text{dir})$
3. Applying $H_{\text{rev}}$ in sequence

This guarantees correct reversion for **any** scramble of **any** length, with no search, heuristics, or lookup tables required.

### 2.2 Scramble Generation

The scramble function produces a pseudo-random word $w$ of configurable length $L \in \{12, 20, 25, 30\}$ subject to the constraint that no generator immediately repeats: $m_i \neq m_{i+1}$ for all $i < L$. This prevents trivial cancellations ($F F = F^2$) and ensures a more thorough mixing of the state space.

**Algorithm 1: Scramble Generation**

```
Input:  Length L
Output: Sequence M of L moves

1.  M ← []
2.  last ← ""
3.  for i ← 1 to L do
4.      repeat
5.          f ← uniform_random({R, L, U, D, F, B})
6.      until f ≠ last
7.      dir ← uniform_random({+1, -1})
8.      M.append({face: f, dir: dir})
9.      last ← f
10. return M
```

### 2.3 Animation via Spherical Linear Interpolation

Each face rotation is animated by constructing a pivot transformation that interpolates between the identity rotation and the target 90° rotation about the face's axis. The interpolation uses **spherical linear interpolation (slerp)** on the unit quaternion manifold $S^3$:

$$\text{slerp}(q_0, q_1; t) = \frac{\sin((1-t)\theta)}{\sin\theta} q_0 + \frac{\sin(t\theta)}{\sin\theta} q_1$$

where $\cos\theta = q_0 \cdot q_1$ and $t \in [0,1]$.

To improve perceptual quality, $t$ is mapped through a **cubic ease-in-out** function:

$$f(t) = \begin{cases}
4t^3 & \text{if } t < 0.5 \\
1 - \frac{(2 - 2t)^3}{2} & \text{otherwise}
\end{cases}$$

This yields $C^1$ continuity with zero velocity at the endpoints, creating a natural deceleration/acceleration profile.

---

## 3. System Architecture

### 3.1 State Machine

The application is governed by a two-state finite state machine:

```
                ┌─────────────┐
                │             │
    ┌──────────►│    IDLE     │◄──────────┐
    │           │             │           │
    │           └──────┬──────┘           │
    │                  │                  │
    │     Scramble /   │   Animation      │
    │     Solve / Face │   completes      │
    │     Button       │   (queue empty)  │
    │                  │                  │
    │           ┌──────▼──────┐           │
    │           │             │           │
    └───────────┤  ANIMATING  ├───────────┘
                │             │
                └─────────────┘
```

All user inputs are gated: no action is accepted unless the current state is `IDLE`. This prevents race conditions and ensures queue integrity.

### 3.2 Move Processing Pipeline

```
User Input (click | keypress)
        │
        ▼
  ┌─────────────┐
  │  Input      │  Validates state == IDLE
  │  Validator  │  Rejects if ANIMATING
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   Queue     │  push({face, dir}) to moveQueue
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  tick()     │  if !isAnimating && queue non-empty:
  │             │      dequeue → update UI → startAnimation()
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  Animation  │  slerp pivot group (requestAnimationFrame)
  │  Engine     │  on complete: snap cubies → scene.attach()
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  Callback   │  isAnimating = false → tick() (recursive)
  └─────────────┘
```

### 3.3 Data Structures

| Structure | Type | Description |
|---|---|---|
| `cubies` | `Array<THREE.Group>` | 27 cubie objects, each with a 0.94³ box body and up to 3 coloured sticker planes |
| `moveQueue` | `Array<{face, dir}>` | FIFO queue of pending moves |
| `moveHistory` | `Array<{face, dir}>` | Append-only log of completed moves (for reverse-solve) |
| `moveCount` | `ℕ` | Total moves executed |
| `totalQueued` | `ℕ` | Total moves in current batch (for progress tracking) |
| `currentAnim` | `{pivot, cubies, sq, eq, start, dur, cb} \| null` | Active animation state (singleton) |
| `state` | `IDLE \| ANIMATING` | Finite state machine register |

### 3.4 Face Rotation Encoding

Each face is encoded as a triple $(\text{axis}, \text{dir}, \text{check})$:

```
R:  axis = (1,0,0),  dir = +1,  check: p.x > +0.5
L:  axis = (1,0,0),  dir = -1,  check: p.x < -0.5
U:  axis = (0,1,0),  dir = +1,  check: p.y > +0.5
D:  axis = (0,1,0),  dir = -1,  check: p.y < -0.5
F:  axis = (0,0,1),  dir = -1,  check: p.z > +0.5
B:  axis = (0,0,1),  dir = +1,  check: p.z < -0.5
```

The `check` predicate partitions the 27 cubies into a face (9 cubies) and the complement (18 cubies + the centre). The `axis × dir` pair, combined with the move direction (±1), determines the rotation quaternion:

$$\text{angle} = \text{face.dir} \times \text{move.dir} \times \frac{\pi}{2}$$

---

## 4. Complexity Analysis

### 4.1 Time Complexity

| Operation | Complexity | Reasoning |
|---|---|---|
| Scramble generation | $O(L)$ | $L$ iterations of constant-time random selection |
| Cubie selection per move | $O(27) = O(1)$ | Linear scan over 27 cubies with predicate check |
| Animation update (per frame) | $O(k)$ | $k$ cubies on the rotating face ($k \in \{0, 9\}$), constant-time slerp |
| Solve construction | $O(n)$ | Reverse $n$-element history, negate each direction |
| Complete solve | $O(n \times 27) \approx O(n)$ | $n$ moves, each selecting $O(27)$ cubies |
| Reset (rebuild cube) | $O(27)$ | Destroy and recreate 27 meshes |

### 4.2 Space Complexity

| Component | Complexity | Details |
|---|---|---|
| Cubie storage | $O(27)$ | 27 `THREE.Group` objects with meshes and materials |
| Move history | $O(n)$ | $n$ recorded moves |
| Move queue | $O(n)$ | Pending moves (at most $n$) |
| Animation state | $O(1)$ | Singleton; at most one active animation |
| Notation strip | $O(8)$ | Fixed-size circular buffer |
| Total | $O(n)$ | Dominated by move history during scramble–solve cycle |

---

## 5. Implementation Details

### 5.1 Rendering Pipeline

- **Renderer**: `WebGLRenderer` with ACES Filmic Tone Mapping at exposure 2.2, providing cinema-grade colour rendering with extended dynamic range
- **Pixel ratio**: clamped to `min(window.devicePixelRatio, 2)` for performance on high-DPI displays
- **Lighting**: Three-point system — key light (warm, 4.5 intensity, upper-right), fill light (cool, 1.2 intensity, upper-left), rim light (cool, 0.7 intensity, lower-left) — plus ambient fill at 0.25 intensity
- **Cubie construction**: Each cubie is a `BoxGeometry(0.94, 0.94, 0.94)` body with `PlaneGeometry(0.78, 0.78)` stickers offset at 0.471 units from the centre (sticker gap: 0.029 units)

### 5.2 Sticker Material Model

Stickers use `MeshStandardMaterial` with:

- **Roughness**: 0.05 (near-mirror finish)
- **Metalness**: 0.0 (dielectric surface)
- **Emissive**: Equal to the sticker colour at intensity 0.08 — a subtle self-illumination that simulates light bouncing from adjacent faces

### 5.3 Theme System

A dark/light theme toggle is implemented entirely via CSS custom properties with a single `body.light` class toggle. All colour values are derived from `var(--*)` references, enabling instantaneous theme switching without JavaScript style injection. The preference is persisted to `localStorage` under the key `cube-theme`.

### 5.4 Camera and Interaction

The scene uses a perspective camera (36° FOV, distance 5.5) with `OrbitControls` providing:

- Drag-to-orbit with damping (factor 0.07)
- Zoom range: [4, 12] units
- **Idle auto-rotation**: Activates after 4 seconds of inactivity; pauses on any user interaction (mousedown, touchstart) and resumes after a further 4-second idle period; never activates during animations (checked via state predicate)

### 5.5 Responsive Layout

The sidebar collapses at breakpoints of 900px and 640px, adapting the UI for tablet and mobile viewports. The celebration canvas and renderer resize on `window.resize` events.

---

## 6. Results and Usage

### 6.1 Running the Application

```bash
# Open directly (no server required — single self-contained file)
open index.html
```

### 6.2 Controls Reference

| Control | Action |
|---|---|
| **Scramble** (S) | Generate and apply $L$ random moves |
| **Solve** (↵) | Reverse the scramble via inverse-word construction |
| **Reset** (R) | Destroy and rebuild cube to identity state |
| **Face buttons** (U/R/F/D/L/B) | Apply a single generator (clockwise or counter-clockwise) |
| **Speed slider** | Adjust animation duration: $t_{\text{dur}} = 0.28 / s$ where $s \in [0.2, 2.0]$ |
| **Scramble length** | $L \in \{12, 20, 25, 30\}$ |

### 6.3 Display Panels

| Panel | Function |
|---|---|
| **Status** | Current FSM state as human-readable string |
| **Moves** | $n$: number of moves executed in current session |
| **Time** | Solve duration in seconds (wall-clock) |
| **Progress** | Bar + $k/n$ fraction indicating queue consumption |
| **Last Moves** | Circular buffer of the 8 most recent moves (live-highlighted) |
| **Scramble** | Full scramble word, grouped 5 per line for readability |

---

## 7. Comparison with Prior Work

| Feature | Typical Simulators | This Work |
|---|---|---|
| **Solve algorithm** | None or Kociemba (black-box) | Reverse-history (transparent, trivially correct) |
| **Tone mapping** | Reinhard or linear | ACES Filmic (exposure 2.2) |
| **Design language** | Varied | Apple Liquid Glass (frosted glass, blur 48px, saturate 1.6) |
| **Animation easing** | Linear or none | Cubic ease-in-out (C¹ continuous) |
| **State machine** | Usually implicit | Explicit two-state FSM with queue |
| **Build step** | Often required (npm/webpack) | Zero — single HTML file, ES modules via importmap |

---

## 8. Future Work

- **Speed-differentiated animation**: Apply faster $t_{\text{dur}}$ during scramble than during solve, enhancing visual rhythm
- **History export**: Copy the scramble word to clipboard as standard notation
- **Optimal-solver integration**: Add an optional mode using Kociemba's algorithm for comparison with the reverse-history approach
- **Sound synthesis**: Haptic-audio feedback synchronized with rotation completion
- **Touch gesture recognition**: Swipe-to-rotate on mobile (currently relies on face buttons)
- **Undo/Redo**: Stack-based move history with forward/backward traversal
- **Custom colour palettes**: User-configurable sticker colour mapping
- **PWA deployment**: Service-worker-based offline support

---

## 9. References

1. Kociemba, H. (1992). *Closed-Form Algorithm for Solving the Rubik's Cube*. Mathematical Institute, TU Darmstadt.
2. Korf, R. E. (1997). *Finding Optimal Solutions to Rubik's Cube Using Pattern Databases*. AAAI/IAAI, 700–705.
3. Shoemake, K. (1985). *Animating Rotation with Quaternion Curves*. ACM SIGGRAPH, 19(3), 245–254.
4. Joy, K. I. (2002). *On the Mathematics of Animation: Easing Functions*. UC Davis Graphics Papers.
5. Three.js Contributors (2024). *Three.js Documentation*. https://threejs.org/docs/

---

## Project Structure

```
rubiks-cube-visual-solver/
├── index.html              # Standalone application (HTML + CSS + JS)
└── README.md             # This document
```

**License**: MIT  
**Dependencies**: Three.js 0.163.0 (loaded via CDN importmap)  
**Browser Support**: Chrome, Firefox, Safari, Edge (ES module + WebGL compliant)
