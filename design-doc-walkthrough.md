# Voxel GPU Design Document — Plain-English Walkthrough

*A companion to `writeup/block-game-design-document.tex`. This file explains what's
in the design document, why we made the choices we made, and what you should be
ready to defend in the Prof. Edwards meeting.*

---

## 1. What are we actually building?

A hardware-accelerated block game — think a stripped-down Minecraft — running on
the DE1-SoC. The player walks a first-person camera through a small world of
textured cubes and can place/destroy blocks in real time.

The DE1-SoC has two halves:

- **HPS (Hard Processor System):** a dual-core ARM Cortex-A9 running Linux.
- **FPGA fabric:** the reconfigurable logic where we drop custom SystemVerilog.

They're on the same chip and talk to each other through two "bridges" (buses):
a **lightweight** bridge for control registers (slow, 32-bit word transfers) and
a **high-performance** bridge for bulk data. We only use the lightweight bridge
— the data rates we need fit comfortably inside it.

**Output:** A VGA monitor at 640×480 @ 60 Hz, driven by the on-board ADV7123
DAC chip. We actually render internally at **320×240** and just double every
pixel in both X and Y on the way out. That's a 4× reduction in pixel count and
the whole framebuffer fits in on-chip BRAM instead of needing DRAM.

---

## 2. The big architectural decision: what goes on the CPU vs the FPGA?

This is the hardest call in any HW/SW co-design project. Our rule:

- **CPU gets the control-heavy, branch-heavy stuff.** Anything with `if`s and
  pointer chasing and dynamic memory: world state, visibility, MVP matrix math,
  per-quad setup. The ARM is fast and Linux gives us a real programming
  environment.
- **FPGA gets the arithmetic-regular, massively-parallel stuff.** The inner
  loop of a rasterizer is the same four additions repeated 76,800 times per
  frame. That's exactly what custom hardware is good at, and exactly what a CPU
  would waste cycles on (lots of loads/stores, branch misprediction, no SIMD on
  A9).

The per-pixel work ends up being **four adds + four sign checks** in the inner
loop. Everything else (pipelining, Z-test, texture lookup, framebuffer write)
happens in parallel in dedicated hardware. That's why this is a sensible
project to FPGA-accelerate.

---

## 3. The pipeline, end to end

Stand back and squint; the whole system looks like a classic graphics pipeline
split across a kernel boundary.

### 3.1 Input stage (software)

USB keyboard and mouse plug into the HPS. Linux's HID layer decodes the raw
USB reports and surfaces them as `struct input_event` records in
`/dev/input/event*`. A USB "boot protocol" mouse emits 3-byte reports at about
125 Hz: `{buttons, dx, dy}`. A boot keyboard emits 8-byte reports (modifiers +
up to 6 keycodes). We read these with the **evdev** API.

Why this matters: when Prof. Edwards asks "how do you read the mouse," the
answer is: we don't touch USB ourselves — the kernel does the ugly work. We
just open `/dev/input/event*` and get structured events.

### 3.2 Game logic (software)

The world is stored as **16×16×16 chunks of bytes** (block IDs). 4 KB per
chunk. We keep up to ~128 chunks loaded at once in DDR3, so about 512 KB of
world data in memory — negligible for the 1 GB of DDR3 we have.

Why 16³ chunks? It's Minecraft's chunk size, it's a power of two (cheap
indexing), and it fits nicely against our 48-world-unit render distance (3
chunks in each direction).

### 3.3 Visibility — two culling passes (software)

Before we do any expensive math, we throw away geometry that won't be seen.

**Pass 1 — frustum culling per chunk.** For each loaded chunk, test its 8
corners against the 6 planes of the camera frustum. If every corner is on the
outside half-space of any single plane, the chunk is definitely outside the
view. (It might fail the test and still be outside — that's OK. It's
conservative: we never reject a visible chunk, we just sometimes accept one
that turns out not to be visible.)

**Pass 2 — hidden-face culling per block.** For each solid block in a surviving
chunk, check its six neighbors. Emit a quad only for a face where the neighbor
is air. This is what eliminates "interior geometry" in a block game — a wall
10 blocks deep still only contributes 1 face per side, not 10.

Between these two passes, the 500-ish quads we actually rasterize per frame
are a tiny fraction of the millions of faces in the world.

### 3.4 Geometry / quad setup (software)

Now we do the "linear algebra" part. For each surviving quad (4 vertices):

1. **MVP transform.** Multiply each vertex by the combined Model-View-Projection
   matrix. We do this in Q16.16 fixed-point (not float) because the ARM Cortex-A9
   handles fixed-point faster than its VFP unit, and Q16.16 gives ample
   precision for a 320×240 viewport. Result is a 4D homogeneous coordinate
   `(x, y, z, w)`.

2. **Near-plane rejection.** If any vertex has `w ≤ 0.1` (i.e., is behind or
   within 10% of a block-edge from the camera), we throw the whole quad away.
   Why? Because perspective divide `x/w` blows up to infinity as `w → 0`, and
   the projected shape can become non-convex, which would break our
   edge-function test. The artifact is that a face partially behind the camera
   pops out a frame early — barely noticeable.

   The "proper" fix is Sutherland-Hodgman clipping, which splits the quad into
   the visible part. We documented it as future work.

3. **Perspective divide and viewport map.** Divide `(x, y, z)` by `w` to get
   Normalized Device Coordinates in [-1, 1], then scale to screen pixels:
   `screen_x = (x+1)·160`, `screen_y = (1-y)·120`. The `(1-y)` flips Y because
   screen coordinates go downward while NDC goes upward.

4. **Edge coefficients.** For each of the 4 edges of the quad, compute
   `(A, B, C)` such that the linear function `E(x,y) = A·x + B·y + C` is
   non-negative on the interior side of the edge. We do this in Q24.8
   fixed-point (see §6 below for *why* Q24.8 specifically).

5. **Depth and UV gradients.** Depth `z(x,y)` is linear in screen space (proven
   fact of projective geometry — see the note below). We compute `z0`, `dz/dx`,
   `dz/dy` for the whole face, plus the same for `u` and `v` if it's textured.

6. **Bounding box.** Compute the axis-aligned screen-space rectangle
   `[x_min, x_max] × [y_min, y_max]` that encloses the quad. The rasterizer
   will scan only inside this box.

All of this data packs into a **64-byte quad descriptor** (`struct quad_desc`
in the C header), or 128 bytes total if the quad is textured (the extra 64
bytes carry UV origin and gradients).

> **Why is depth linear in screen space?** Under perspective projection,
> `1/w` is exactly linear in screen coordinates. Our depth encoding is
> `z_ndc = a - b/w_eye`, so `z_ndc` is also linear. This is why affine depth
> interpolation is *exact* for planar faces — not an approximation, even though
> affine UV interpolation *is* an approximation.

### 3.5 Submit to hardware (software + kernel)

The user-space game writes the packed quad descriptors to `/dev/voxel_gpu`
using ordinary `write()`. The kernel driver picks those up and does
`iowrite32` in a loop into the memory-mapped **FIFO window** on the FPGA.

Think of the FIFO window as a "mailbox": it's an 8 KB region of the Avalon
bus, and every 32-bit write you do to any address in that window is treated
the same way — "enqueue this word onto the command FIFO." The FPGA's
`avalon_slave` decodes this and pushes to an on-chip BRAM FIFO.

### 3.6 Rasterization (hardware)

The FPGA's `quad_fetch` module pops 16 consecutive 32-bit words (or 32 if
textured) out of the FIFO and reassembles them into a `quad_desc_t` struct.
When it has one ready, it hands it to the `rasterizer` via a standard
valid/ready handshake.

The rasterizer then does the inner loop — for each (x, y) in the bounding box:

- Evaluate the 4 edge functions incrementally (previous value + A or B).
- Test all 4 `≥ 0`. If any is negative, skip this pixel.
- Read the Z-buffer at (x, y).
- Compare stored depth to this pixel's interpolated depth. If we're further,
  skip.
- Either fetch a texel (`TEX=1`: nearest-neighbor lookup into a 16×16 tile) or
  use the flat palette index from the descriptor (`TEX=0`).
- Write the color to the framebuffer and the new depth to the Z-buffer.
- Advance `u`, `v`, and `z` by their per-pixel gradients.

The magic is that edge functions, depth, and UV are all **linear**, so
stepping one pixel just takes *one add per value*. No multiplies in the inner
loop. This is the payoff for doing all the hard math on the ARM upfront.

### 3.7 VGA scanout (hardware)

Completely independent of the rasterizer. A tiny state machine counts
horizontal/vertical pixel positions at 25.175 MHz (VGA pixel clock), and on
every cycle:

1. Reads the framebuffer at the current pixel position. Because the
   framebuffer is 320×240 but VGA is 640×480, we drop the low bit of both
   counters before looking up — each logical pixel appears as a 2×2 block.
2. Looks up the 8-bit palette index in the 256-entry palette BRAM, getting
   back a 24-bit RGB value.
3. Drives that RGB to the ADV7123 DAC pins.

The scanout always reads the **front** framebuffer. The rasterizer always
writes the **back** framebuffer. When the game finishes submitting a frame, it
issues `VGPU_FLIP`, and on the next vertical blank (when the DAC beam is
returning to the top of the screen), we swap which buffer is front. This is
standard double-buffering and gives tear-free output.

---

## 4. Memory layout — what lives where and why

This is the question Prof. Edwards loves asking. Here's the full map.

### 4.1 On-chip FPGA block RAM (M10K), ~512 KB budget

| What | Size | Why |
|---|---|---|
| 2× framebuffers | 154 KB | Double-buffering; each is 320×240×8-bit |
| Z-buffer | 154 KB | Per-pixel depth, 320×240×16-bit |
| Texture tiles | 16 KB | 64 tiles × 16×16 × 8-bit palette index |
| Command FIFO | 8 KB | ~128 quads in flight, decouples CPU from rasterizer |
| Palette | 0.75 KB | 256 entries × 24-bit RGB |
| **Total** | **~332 KB** | **~65% of 512 KB** — comfortable headroom |

**Why on-chip BRAM for everything?** Because DRAM access is variable-latency
(cache misses, row opens, refresh cycles) and our rasterizer wants to hit
memory *every single cycle* at deterministic latency. BRAMs are synchronous,
single-cycle, and we have enough.

**Why not put the framebuffer in DRAM like a real GPU?** Because at 320×240
we don't need to. A real GPU framebuffer is megabytes and must live in DRAM.
Ours is 77 KB and fits in ~8 M10K blocks.

### 4.2 DDR3 (HPS side), 1 GB available

Used only by the software side. World data (~512 KB), frame-local visible
quad list (<64 KB), plus Linux itself (~64 MB for kernel + our game). Less
than 100 MB used out of 1 GB. We aren't even close to a DRAM bandwidth or
capacity concern.

---

## 5. The hardware/software interface

This is the contract between ARM and FPGA. Four things live at fixed Avalon
addresses starting at the lightweight bridge base `0xFF20_0000`:

### 5.1 Control registers (5 of them)

| Offset | Reg | What it does |
|---|---|---|
| 0x00 | CONTROL | 3 bits: enable rasterizer, request flip, trigger clear |
| 0x04 | STATUS | Read-only: busy, FIFO full, FIFO empty, in vblank, FIFO word count |
| 0x08 | FRAME_COUNT | Increments once per completed frame (FPS measurement) |
| 0x0C | PALETTE_ADDR | Which palette entry to program next (0–255) |
| 0x10 | PALETTE_DATA | RGB888 value; writing commits to palette and auto-advances ADDR |

Writing `CONTROL.CLR` triggers a 76,800-cycle linear sweep (~1.5 ms at 50 MHz)
that writes every Z-buffer entry to 0xFFFF (max depth) and every back-buffer
framebuffer entry to palette-index 0 — **in parallel**. Two different BRAMs,
two different write ports, one address counter. We get the FB clear for free.

### 5.2 FIFO window (8 KB, offsets 0x1000–0x2FFF)

Every write to any address in this range is enqueued into the command FIFO.
Software streams 16 (or 32) consecutive 32-bit words per descriptor. If the
FIFO fills up mid-burst, the Avalon slave **asserts `waitrequest`** and stalls
the ARM master until space opens — no data loss.

### 5.3 IOCTLs (5 of them)

For control operations that don't fit cleanly as register writes:

- `VGPU_SET_PALETTE` — one palette entry upload (wraps the two register writes).
- `VGPU_FLIP` — request buffer flip on next vsync (blocking).
- `VGPU_CLEAR_FRAME` — clear Z + back FB (blocking ~1.5 ms).
- `VGPU_GET_STATUS` / `VGPU_GET_FRAME_COUNT` — read back status.

### 5.4 The quad descriptor — 64 bytes, packed

This is the most important data structure in the whole project. Every frame,
we send hundreds of these over the bus.

```
bytes  0–7    : bounding box (x_min, y_min, x_max, y_max, int16 each)
bytes  8–55   : 4 edges × (A, B, C) × 4 bytes each = 48 bytes, Q24.8 signed
bytes 56–61   : z0 (Q1.15), dz/dx (Q1.15), dz/dy (Q1.15)
byte  62      : tex_or_color (texture index OR flat palette index)
byte  63      : flags (TEX, ZTEST, 6 reserved bits)
```

Total: exactly 64 bytes. If `TEX=1`, a second 64-byte block follows with the
UV origin and gradients in Q16.16. Both blocks padded to 64 bytes so *all*
descriptors are a multiple of 64 — makes the rasterizer's pop counter
trivial.

---

## 6. The fixed-point choices and why

Fixed-point vs floating-point is a recurring question with a consistent answer:
**fixed-point is smaller, faster, and deterministic on an FPGA**. The only
question is which format.

### 6.1 Edge coefficients: Q24.8

The edge function `E(x, y) = A·x + B·y + C` has to avoid overflow. For a
full-viewport quad:
- `A = y0 - y1`, up to 239
- `B = x1 - x0`, up to 319
- `C = -(A·x0 + B·y0)`, dominated by products up to ~77,000

So `|E|` can reach about 2·area ≈ 77,000 for a viewport-sized face. That's
more than 2^16 (= 65,536), so Q16.16 (which has 16 integer bits signed = ±32,768)
would silently **overflow**. Q24.8 gives us ±8.4 million of headroom plus 8
bits of sub-pixel fraction. This is the reason we don't just use Q16.16
everywhere.

### 6.2 Depth: Q1.15

Depth after perspective divide lives in [0, 1] nominally. We store it in
16-bit Q1.15 with range [0, 2), which absorbs the slight overshoot past 1 that
bounding-box interpolation can cause. The gradient `dz/dx` per pixel is at
most a few parts per thousand across the screen, well inside Q1.15's ±1 range.

**Gotcha we documented carefully:** depth precision under 1/z-style projection
is non-uniform. Close objects get tons of depth steps per world unit; far
objects get very few. With near-plane = 0.1 and Q1.15, you get ~1 code per
unit at about 55 world units away — which is why we cap render distance at 48
units. Beyond that, you'd start getting z-fighting between nearly-coplanar
faces.

### 6.3 UV: Q16.16

Texture coordinates need fractional precision across long scan lines. Q16.16
gives `2^-16` precision per step; across 320 pixels, accumulated error is
about 0.005 texels — invisible. A narrower format like Q8.8 would accumulate
over one full texel of drift and produce visible texture seams.

The "huge integer range" of Q16.16 (±32K) is overkill for a 16-texel tile;
we just take the low 4 integer bits as the lookup index. The upper bits are
effectively a wrap-around counter, which happens to be exactly what we want
for tiling textures.

---

## 7. The FPGA module hierarchy

A dozen-ish SystemVerilog modules under a single top. Names map one-to-one to
functional blocks in the block diagram:

```
voxel_gpu_top
├── avalon_slave       HPS-facing register file + FIFO write gateway
├── cmd_fifo           8 KB BRAM FIFO (2048 × 32-bit words)
├── quad_fetch         Pops words, reassembles quad_desc structs
├── rasterizer         The inner loop; pipelines one pixel per cycle
│   ├── edge_eval (×4) One per quad edge, incremental adder
│   ├── interp_unit_z  16-bit Q1.15 depth interpolator
│   ├── interp_unit_u  32-bit Q16.16 u interpolator
│   └── interp_unit_v  32-bit Q16.16 v interpolator
├── zbuffer            Dual-port M10K, rast port + clear port
├── texture_unit       Single-port M10K, read-only from rasterizer
├── framebuffer        2× M10K banks + buffer_select mux
├── palette            Dual-clock M10K (write @ 50 MHz, read @ 25.175 MHz)
├── clear_sweep        FSM that drives the 76,800-cycle parallel clear
├── vga_scanout        Timing + pixel-doubling + palette read + DAC drive
└── pll_inst           Platform Designer vendor IP (not our RTL)
```

The design doc gives full port lists (inputs, outputs, widths) for every
authored module. Key design patterns in use:

- **Valid/ready** handshake between `quad_fetch` and `rasterizer` (producer
  says "I have a thing," consumer says "I can take it," they shake when both
  high).
- **Pipelined RMW** on the Z-buffer: read at cycle n, compare at n+1, write
  at n+1. Consecutive pixels address different cells, so no hazard inside a
  quad. At quad boundaries we insert one bubble cycle for coefficient reload.
- **Dual-clock BRAM** for the palette and framebuffer — write side at 50 MHz
  (rasterizer domain), read side at 25.175 MHz (VGA domain). Altera M10Ks
  handle this natively with a two-flop synchronizer around the address.
- **Avalon-MM waitrequest** for back-pressure on the FIFO. No data loss.

---

## 8. Performance budget

Worth internalizing the numbers because Prof. Edwards will ask.

| Quantity | Value |
|---|---|
| FPGA clock | 50 MHz |
| VGA pixel clock | 25.175 MHz (640×480 @ 60 Hz standard) |
| Peak rasterizer throughput | 50 Mpix/s (1 pixel/cycle) |
| Time to fill a full 320×240 screen once | 1.5 ms |
| Frame budget @ 60 fps | 16.6 ms |
| Overdraw budget | ~10× (we can rasterize the whole screen 10 times over) |
| Expected overdraw in a voxel scene | 2–4× |
| Command bandwidth @ 500 quads × 128 B × 60 fps | 3.84 MB/s |
| Lightweight bridge capacity | tens of MB/s |
| Z-clear sweep time | 1.5 ms (parallel FB clear is free) |

All comfortably inside budget. Nothing is the bottleneck.

---

## 9. Milestones

Seven milestones, each producing a runnable system. The team splits 3 ways
(bus/command path, rasterizer core, display path) so milestones 1–3 progress
in parallel after initial scaffolding.

1. **VGA baseline.** FPGA displays a test pattern from a BRAM framebuffer via
   the scanout pipeline. Extends lab 3's `vga_ball`.
2. **Linux driver.** `voxel_gpu.ko`, `/dev/voxel_gpu`, five ioctls, `write()`,
   platform-bind via device-tree.
3. **Software renderer.** Full C pipeline including software rasterization.
   Proves geometry/culling code before HW integration.
4. **FPGA rasterizer in simulation.** Verilator/ModelSim against hardcoded
   quad descriptors. Add Z-buffer, verify occlusion.
5. **Integration.** Hardware rasterizer driven by software pipeline through
   the driver. Flat-shaded cubes at ≥30 fps.
6. **Interaction.** First-person camera, block place/destroy with per-chunk
   face-list regeneration.
7. **Textures.** Enable TEX path, `$readmemh` upload, per-pixel UV lookup.

Stretch goals if time: procedural generation, simple lighting, skybox,
IRQ-driven frame pacing instead of polling.

---

## 10. What to be ready to defend in the meeting

Prof. Edwards will probe specific decisions. Here's what he's likely to
ask and the short version of the answer:

- **Why quads instead of triangles?** Every block face is a rectangle; one
  quad rasterizer avoids 2× tessellation overhead; the four-edge-function test
  handles convex quads correctly.

- **Why 320×240 not 640×480?** Fits in on-chip BRAM (eliminates SDRAM
  controller). 2×2 pixel doubling gets the output to standard VGA timing with
  zero extra logic.

- **Why 50 MHz?** Default LW bridge clock on Cyclone V, meets M10K timing
  with margin, and we only need 50 Mpix/s peak for 10× overdraw at 60 fps.

- **Why affine UV (not perspective-correct)?** Saves one divider per pixel.
  At 320×240 the visible warp is mild and the style ("PS1 look") is acceptable
  in a block game. Upgrade path is documented: add 1/w, u/w, v/w to the
  descriptor, add a reciprocal unit.

- **How does near-plane handling work?** Reject the whole quad if any vertex
  has w ≤ 0.1. Cheap (4 comparisons) and the artifact (faces popping out a
  frame early) is barely visible. Sutherland-Hodgman clipping is the
  documented future upgrade.

- **What clears the back framebuffer?** The `CLR` bit drives a dual-write
  sweep: same address counter writes both Z (→ 0xFFFF) and back FB (→ palette
  index 0) in parallel. 1.5 ms total. Critical because without a skybox,
  uncovered pixels would otherwise show stale contents from two frames ago.

- **How do flat-shaded quads specify their color?** Byte 62 of the descriptor
  (`tex_or_color`) is interpreted based on the TEX flag: texture ID when
  TEX=1, 8-bit palette index when TEX=0. Same byte, different interpretation
  — no descriptor-size penalty.

- **How does the FIFO handle overflow?** Avalon-MM `waitrequest`: the slave
  stalls the master (ARM) until space opens. No data loss. Software can
  optionally poll `STATUS.FFL` to pace writes if it wants to avoid stalling.

- **How does vsync-locked flipping work?** `VGPU_FLIP` sets a bit latched
  through a two-flop synchronizer into the VGA clock domain. The actual
  buffer swap happens on the next rising edge of `in_vblank`, guaranteeing
  no tearing. The ioctl is blocking and returns after `FRAME_COUNT`
  increments.

- **What's the texture file format?** ASCII hex, 16,384 lines, 2 hex digits
  per line (palette index per texel). Indexed tile-major then row-major
  within a tile. Loaded by `$readmemh("textures.hex", tex_ram)` at synthesis.

- **What's a chunk and why that size?** 16×16×16 bytes of block IDs. Power
  of two (cheap indexing), matches a natural "render distance unit" (3
  chunks = 48 world units = our depth precision cap).

---

## 11. Known limitations we've documented

Transparent about what v1 doesn't do:

- **No transparency / blending.** All quads are opaque. Would need depth
  sorting + alpha compositing.
- **No anti-aliasing.** Edge function is a hard ≥0 test. Sub-pixel precision
  is there in Q24.8, but we just threshold.
- **No perspective-correct texturing.** Affine only. Visible warping at
  steep angles.
- **No dynamic texture upload.** Textures are baked in at synthesis.
- **No IRQ-driven frame pacing.** Polling only. Fine at 60 fps (tens of
  reads per frame). IRQ is a stretch goal.
- **Render distance capped at 48 world units.** Beyond that, Q1.15 depth
  precision degrades to <1 step per world unit and you get z-fighting.

Every one of these has a documented upgrade path in the TeX file.

---

## 12. How this document tracks the assignment rubric

Prof. Edwards's rubric called out specific things he wants to see. Here's the
crosswalk:

| Rubric requirement | Where we address it |
|---|---|
| Top-level block diagram with the custom peripheral as one block | §2.1, Figure 1 |
| Internal structure of the custom peripheral | §2.2, Figure 2 |
| USB protocol explanation | §2.3 block 1 (HID boot report format) |
| Register map (addresses, read/write semantics) | §5.1 |
| Bit widths on connections | Figures 1, 2 and module ports in §6.2 |
| Memory sizes | §4.1 table, block subtitles in Figure 2 |
| Handshake protocols for complex connections | §6.3 |
| C header content (types, function declarations) | §5.4 (full `voxel_gpu.h`) |
| Verilog module interface definitions | §6.2 |
| Module instantiation hierarchy | §6.1 tree |
| File formats for anything read/written | §5.2 (texture hex file) |

Every box on the rubric has an explicit answer.

---

*End of walkthrough. If anything is unclear, grep the TeX source for the
relevant section name; the definitive wording is there.*
