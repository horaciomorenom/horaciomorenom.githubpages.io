# Technical Outline: GPU-Accelerated Browser Simulation with CindyJS

## Purpose

This document explains the architecture of a GPU-accelerated interactive physics simulation running entirely in the browser. The pattern generalizes to any cellular automaton or pixel-level iterative simulation (e.g., reaction-diffusion, fluid dynamics, Conway's Game of Life, wave equations).

---

## Core Idea: Textures as State, Shaders as Compute

The fundamental trick is to treat a GPU texture (an image) as a 2D array of state values, and use fragment shaders — code that normally just colors pixels — as a massively parallel compute engine. Every pixel's update rule runs simultaneously on the GPU.
co
**Data flow each frame:**
```
texture "state" (t)  →  fragment shader (update rule)  →  texture "state" (t+1)  →  render to screen
```

This is called **ping-pong rendering** or **render-to-texture**. The same texture is read as input and written as output each pass.

---

## Library: CindyJS + CindyGL

CindyJS is a mathematical visualization library. CindyGL is its WebGL extension, which compiles CindyScript expressions into GLSL fragment shaders transparently. The developer writes high-level math; the library handles all WebGL boilerplate.

**Why CindyJS instead of raw WebGL:**
- No explicit shader code required
- Built-in functions for texture sampling, random numbers, math
- Event system (mousedown, mouseup, draw loop) handled automatically
- `colorplot` is the key primitive: it runs a given expression for every pixel in a region

**Tradeoff:** CindyJS is less flexible than raw WebGL/Three.js/WebGPU and has limited documentation. For production use, the same pattern can be implemented with raw WebGL or libraries like `regl`, `twgl`, or Three.js with RenderTargets.

---

## HTML Structure

```html
<body style="margin:0;padding:0;overflow:hidden;">
  <div id="CSCanvas"></div>

  <script id="csinit"  type="text/x-cindyscript"> ... </script>
  <script id="csdraw"  type="text/x-cindyscript"> ... </script>
  <script id="csmousedown" type="text/x-cindyscript"> ... </script>
  <script id="csmouseup"   type="text/x-cindyscript"> ... </script>

  <script type="text/javascript">
    CindyJS({ scripts: "cs*", autoplay: true, ports: [...] });
  </script>
</body>
```

- `type="text/x-cindyscript"` prevents the browser from executing these as JavaScript. CindyJS reads and compiles them.
- The `id` of each script block maps to a lifecycle event: `csinit` runs once on startup, `csdraw` runs every animation frame, `csmousedown`/`csmouseup` on pointer events.
- `scripts: "cs*"` tells CindyJS to collect all script tags whose `id` starts with `cs`.
- `autoplay: true` starts the draw loop immediately.
- `ports` defines the canvas size and coordinate system. `visibleRect: [0, 0, W, H]` sets pixel coordinates so that (0,0) is bottom-left and (W,H) is top-right.

---

## Initialization (`csinit`)

```
createimage("state", W, H);
colorplot(L, R, "state", (randomint(2), 0, 0));
```

- `createimage` allocates a GPU texture of size W×H. It is named "state" and referenced by that string.
- `colorplot(L, R, target, expr)` runs `expr` for every pixel in the rectangle from point L to point R and writes the result (an RGB triple) into `target`. Here it writes a random 0 or 1 into the red channel to initialize random spin states.
- Only the red channel is used for state storage. Green and blue are set to 0 and ignored.

**Defining functions in CindyScript:**
```
get(x, y) := imagergb(L, R, "state", (mod(x,W), mod(y,H)))_1;
```
- `imagergb` samples the texture at a given coordinate and returns an RGB triple.
- `_1` extracts the red channel (1-indexed).
- `mod(x,W)` wraps coordinates, implementing periodic boundary conditions for free.

---

## State Encoding

State values must be encoded as colors (floats in [0,1]) since GPU textures store colors, not arbitrary data.

| Physical value | Stored value | Encoding |
|---|---|---|
| spin down (−1) | 0.0 in red channel | `get(x,y)` returns 0 |
| spin up (+1) | 1.0 in red channel | `get(x,y)` returns 1 |

To convert stored value `s ∈ {0,1}` to spin `σ ∈ {−1,+1}`: use `(2*s - 1)`.

For simulations with more state variables (e.g., velocity + density), allocate multiple images or pack multiple values into R, G, B, A channels.

---

## Update Rule: Metropolis Algorithm

The update rule is the heart of the simulation. It must be a **pure function of the current pixel position and its neighbors' states** — no mutable global state, no sequential dependency.

```
energy(x, y, spin) :=
  - (2*spin-1)*(2*get(x, y+1)-1)
  - (2*spin-1)*(2*get(x, y-1)-1)
  - (2*spin-1)*(2*get(x+1, y)-1)
  - (2*spin-1)*(2*get(x-1, y)-1)
  - h * (2*spin-1);
```

This computes the local interaction energy: a spin contributes negative energy (favorable) when aligned with each of its 4 neighbors, and interacts with external field `h`.

```
newstate(x, y) := (
  current  = get(x, y);
  proposal = randomint(2);
  currentEnergy  = energy(x, y, current);
  proposalEnergy = energy(x, y, proposal);
  if (random() < exp(-beta * (proposalEnergy - currentEnergy)), proposal, current)
);
```

This is the Metropolis criterion: always accept if energy decreases; accept with probability `exp(-β·ΔE)` if energy increases. At high β (low temperature), uphill moves are rare → ordered phases. At low β (high temperature), all moves accepted → disordered phases.

---

## GPU Parallelism and the Checkerboard Problem

**The fundamental constraint of GPU compute:** all pixels execute simultaneously. A pixel cannot see updates made by its neighbors in the same pass, only the state from the previous pass. This means strict Metropolis detailed balance is violated if every pixel updates every frame.

**The solution used here — random partial updates:**
```
colorplot(L, R, "state",
  if (random() < invrate, (newstate(#.x, #.y), 0, 0), (get(#.x, #.y), 0, 0))
);
```
Only a fraction `invrate = 0.05` (5%) of pixels attempt an update each pass. The rest copy their current state unchanged. With 20 passes per frame (`repeat(20, ...)`), this approximates sequential Metropolis while remaining GPU-parallel.

**Alternative approaches for other simulations:**
- **Checkerboard decomposition**: Update red squares one pass, black squares the next (like a chess board). Neighbors of red squares are always black (not updated this pass), so reads are safe. This gives exact detailed balance at 50% update rate.
- **Double buffering**: Maintain two textures (`state_A`, `state_B`). Read from A, write to B, then swap. Standard for fluid simulations.
- **Async compute (WebGPU)**: True compute shaders with memory barriers. Not available in WebGL.

---

## Rendering Pass

After updating state, a separate `colorplot` renders to screen:
```
colorplot(if (imagergb("state", #)_1 > 0.5, (1,0,0), (0,0,1)));
```
This reads the state texture and maps spin-up → red, spin-down → blue. The update and render passes are intentionally separate so the rendering can apply any visual mapping without affecting simulation state.

---

## Interactive Parameter Control

Mouse position is mapped to physical parameters during drag:
```
beta = max([0, min([1, (mouse().x - border)/(W - 2*border)])]) * (betamax - betamin) + betamin;
```
This is a standard linear remap with clamping: `param = clamp((mouse - min) / (max - min)) * (paramMax - paramMin) + paramMin`.

Snapping to critical values:
```
beta = if (abs(beta - betacrit) < betasnap, betacrit, beta);
```
This makes it easy to land exactly at the phase transition without precise mouse control.

---

## Generalizing to Other Simulations

To implement a different simulation with this pattern, replace:

1. **State encoding** — decide what physical quantities to store and how to pack them into texture channels.
2. **`get(x,y)`** — define how to read state from the texture with appropriate boundary conditions (periodic, Dirichlet, Neumann).
3. **`newstate(x,y)`** — the local update rule. Must depend only on the current pixel and a fixed neighborhood. Examples:
   - *Game of Life*: count neighbors, apply birth/survival rules
   - *Reaction-diffusion*: compute Laplacian from 4-neighborhood, apply Gray-Scott equations
   - *Wave equation*: store current and previous displacement in two channels, apply finite difference stencil
4. **Update strategy** — choose partial-update, checkerboard, or double-buffering based on whether neighbors' simultaneous updates cause artifacts.
5. **Render pass** — map state values to colors independently of simulation logic.
6. **Interactive controls** — map pointer/keyboard input to physical parameters using the linear remap pattern.

---

## Key CindyGL Primitives Reference

| Function | Description |
|---|---|
| `createimage(name, W, H)` | Allocate a named GPU texture |
| `colorplot(L, R, target, expr)` | Run `expr` at every pixel in rect [L,R], write to `target` |
| `colorplot(expr)` | Run `expr` at every pixel, render to screen |
| `imagergb(L, R, name, pos)` | Sample texture `name` at coordinate `pos` within rect [L,R] |
| `imagergb(name, #)` | Sample texture at current pixel position (screen coords) |
| `#.x`, `#.y` | Current pixel coordinates inside `colorplot` |
| `randomint(n)` | GPU-side random integer in [0, n-1] |
| `random()` | GPU-side random float in [0, 1) |
| `screenbounds()` | Returns corner points of the visible canvas |

---

## Minimal Template

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://cindyjs.org/dist/latest/Cindy.js"></script>
  <script src="https://cindyjs.org/dist/latest/CindyGL.js"></script>
</head>
<body style="margin:0;padding:0;overflow:hidden;">
  <div id="CSCanvas"></div>

  <script id="csinit" type="text/x-cindyscript">
    use("CindyGL");
    corners = screenbounds();
    W = dist(corners_1, corners_2);
    H = dist(corners_1, corners_4);
    L = [0,0]; R = [W,0];

    createimage("state", W, H);
    // Initialize state:
    colorplot(L, R, "state", (INITIAL_VALUE, 0, 0));

    // Define read helper with periodic BC:
    get(x, y) := imagergb(L, R, "state", (mod(x,W), mod(y,H)))_1;

    // Define update rule:
    newstate(x, y) := ( /* ... your physics here ... */ );
  </script>

  <script id="csdraw" type="text/x-cindyscript">
    // Update state (partial random update to handle GPU parallelism):
    repeat(20,
      colorplot(L, R, "state",
        if(random() < 0.05, (newstate(#.x, #.y), 0, 0), (get(#.x, #.y), 0, 0))
      )
    );
    // Render to screen:
    colorplot( /* map state to color */ );
  </script>

  <script type="text/javascript">
    CindyJS({
      scripts: "cs*",
      autoplay: true,
      ports: [{ id: "CSCanvas", width: window.innerWidth, height: window.innerHeight,
                transform: [{ visibleRect: [0, 0, window.innerWidth, window.innerHeight] }] }]
    });
  </script>
</body>
</html>
```
