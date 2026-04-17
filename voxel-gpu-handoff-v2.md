# Voxel GPU Project — Session Handoff Document (v2)

**Date:** April 17, 2026
**Course:** CSEE 4840 Embedded System Design, Spring 2026 (Columbia)
**Team:** Josh Bernheisel (jcb2301), Wesley Maa (wm2505), Mihir Joshi (mnj2122)
**Platform:** DE1-SoC (Intel Cyclone V, ARM Cortex-A9 + FPGA fabric)
**Project name:** Hardware-accelerated block game, working module name `voxel_gpu`

This is the second handoff document, superseding v1. The design document has been through three review-and-revision passes and is submission-ready. This document captures:
- Final design decisions (with rationales)
- All six remaining specification gaps the implementing team should resolve
- Complete Platform Designer / Quartus build state
- Module decomposition, team split, implementation plan
- Known traps, decisions that were made silently during revision, and why each is OK

Read this whole document before taking implementation actions.

---

## 1. Project overview

Hardware-accelerated 3D block game on the DE1-SoC:

- **ARM Cortex-A9 (Linux)** handles world state, visibility culling, camera math, per-quad setup.
- **FPGA fabric** implements a fixed-function quadrilateral rasterizer with z-buffering, texturing, and VGA output.
- **Linux kernel driver** mediates via the HPS-to-FPGA lightweight bridge.

### Design decisions (finalized)

1. **Quads, not triangles.** Block faces are rectangles. Four-edge-function test handles perspective-projected convex quads correctly. Avoids doubling primitive count.
2. **320×240 internal resolution, 2×2 upscaled to 640×480.** Fits in on-chip BRAM; no SDRAM controller needed; deterministic single-cycle memory access.
3. **8bpp palette-indexed framebuffer + 256-entry RGB888 palette.** Half the framebuffer memory vs 16bpp; scanout adds one pipeline stage for palette lookup with no throughput cost.
4. **16-bit Z-buffer, Q1.15 format.** 3×10⁻⁵ precision, limited by perspective divide concentration near z=1 (see depth precision budget below).
5. **Double-buffered, flip on vsync.**
6. **Near-plane `w_near = 0.1` world units.** Any quad with vertex behind this plane is rejected. Game logic caps render distance at ~48 world units (3 chunk widths). This combination keeps depth precision usable across the visible range.
7. **Affine UV interpolation** (no perspective correction). "PS1 look" at steep angles, acceptable for block-game content at 320×240.
8. **Polling-only frame pacing** for v1 (no IRQ wiring). Polling cost is negligible at 60fps.
9. **Textures baked into BRAM via `$readmemh`** at synthesis. No runtime texture upload in v1.
10. **Avalon `waitrequest` backpressure for FIFO full** (no silent drop). More robust, slightly more hardware complexity for Person A.
11. **Fixed-point formats:**
    - MVP matrix and post-divide screen coords: Q16.16 on ARM (intermediate only)
    - Edge coefficients in descriptor: signed **Q24.8** (overflow argument: edge function reaches ±77,000 for viewport-sized quads, beyond Q16.16's ±32,768 int range)
    - Depth z0, dz_dx, dz_dy: signed **Q1.15** (range [0,2) for z0 absorbs fixed-point rounding past 1.0)
    - UV u0, v0, du_dx, dv_dx, du_dy, dv_dy: signed **Q16.16** (fractional precision needed to avoid drift across scan line; only low 4 integer bits indexed mod 16 for 16×16 tile)

---

## 2. Design document — FINAL

**Status: Submitted or submitting April 17, 2026.**

File: current LaTeX source and compiled 13-page PDF. Structure follows Edwards' sample doc: Introduction, System Block Diagram, Algorithms, Resource Budgets, Hardware/Software Interface, Milestones.

### Distinguishing features of the final version

- **Format rationale paragraphs** for Q24.8 (overflow), Q1.15 (precision), and Q16.16 UV (drift analysis).
- **Depth precision budget** quantifying Z-resolution vs z_eye; this is what motivates the 48-unit render cap.
- **Near-plane rejection** explicitly documented with Sutherland-Hodgman as upgrade path.
- **BRAM port arrangement** paragraph explaining how Z-buffer RMW, framebuffer double-buffering, and texture reads all sustain 1 pixel/cycle.
- **Frame clear** (not just Z-clear) on CLR, justified by the skybox-absence argument.
- **Per-face list regeneration** (not "meshing") on block changes, including immediate neighbors across the modified face.

### Changes made during review that are worth knowing

These are silent decisions made by revision passes. Implementing team should actively accept or reject each:

| Change | From | To | Impact |
|---|---|---|---|
| Near-plane distance | 0.05 | 0.1 | Better depth precision; can't get quite as close to blocks. **Accept.** |
| FFL semantics | silent drop | waitrequest backpressure | Person A's slave must implement waitrequest handshaking. **Accept** (more robust). |
| `texture_id` field | 6-bit tex index only | `tex_or_color` (dual-use: tex index or flat palette index) | Enables flat-shaded quads, was a real hole. **Accept**, mandatory. |
| `VGPU_CLEAR_Z` ioctl | Z-only clear | `VGPU_CLEAR_FRAME` (clears Z + back framebuffer in parallel) | Prevents frame-N-2 ghost pixels with no skybox. **Accept**, mandatory. |
| DDR3 chunk budget | 32 chunks | 128 chunks | Needed for 3-chunk render distance. **Accept**, cosmetic. |

---

## 3. Remaining specification gaps (TO RESOLVE DURING IMPLEMENTATION)

The design doc is submittable but has six small gaps that an implementer will hit. Decide and document:

### 3.1 FLP behavior when EN=0
Spec says EN=0 is "idle, safe to reconfigure" and FLP flips on next vsync. Does setting FLP while EN=0 still cause a flip? **Recommended answer: yes.** FLP is independent of EN; scanout is always running even when rasterizer is disabled. Document in Person A's slave code.

### 3.2 `flags.ZTEST=0` semantics
Spec says "always pass" but doesn't say whether the Z-buffer is still *written*. Two options:
- (a) Skip z-test but still z-write (depth updates).
- (b) Skip both z-test and z-write (depth untouched).

**Recommended answer: (b)**, because ZTEST=0 is primarily for HUD/skybox-style overlays that shouldn't pollute depth. Person B to confirm when writing the rasterizer.

### 3.3 Reserved bits policy
`flags[7:2]` and CONTROL[31:3] are reserved. Policy should be: software writes 0, hardware ignores. Document in the driver and hardware spec. Standard practice.

### 3.4 Z-buffer clear value
Spec says `0xFFFF`. In Q1.15 that reads as 1.99997, past the valid [0,1] depth range. It works (anything closer beats it) but strictly should be `0x7FFF` (= 1.0 in Q1.15). Pick one and document. **Recommended: keep `0xFFFF`** — simpler for the CLR sweep logic, since you're just writing all-ones.

### 3.5 Render distance vs load distance
Spec caps *render* at 3 chunk widths (~48 world units, ~113 chunks in a sphere). DDR3 budget is 128 chunks. The prefetch margin is tight — you want load distance > render distance. **Recommended: bump chunk budget to 256 during implementation** (1 MB DDR3, still trivial), set render distance = 3 chunks and load distance = 4 chunks.

### 3.6 Rasterizer pipeline depth in pseudocode
The pseudocode in §3.4 reads as single-cycle read-compare-write. The port arrangement paragraph explains it's actually read at cycle n, conditional write at cycle n+1. An implementer should treat the pseudocode as logical behavior and the text paragraph as physical timing. Person B should write the SystemVerilog with an explicit 2-stage pipeline for Z-buffer access.

---

## 4. Authoritative hardware/software interface

Copy exact values from design doc §5 when implementing. Key fields:

### Base address
`0xFF20_0000` (HPS lightweight bridge base + peripheral offset 0).

### Control register map (32-bit words, byte offsets)

| Offset | Name | R/W | Notes |
|---|---|---|---|
| 0x00 | CONTROL | R/W | `[0]=EN [1]=FLP [2]=CLR`, all others reserved |
| 0x04 | STATUS | R | `[0]=BSY [1]=FFL [2]=FEM [3]=VSY [15:4]=FIFO_WORDS` |
| 0x08 | FRAME_COUNT | R | free-running |
| 0x0C | PALETTE_ADDR | W | index [7:0] |
| 0x10 | PALETTE_DATA | W | `[23:16]=R [15:8]=G [7:0]=B`, auto-increments ADDR |
| 0x1000–0x2FFF | FIFO_WINDOW | W | 8 KB, word-streamed descriptor data |

**Backpressure:** writes to FIFO_WINDOW when full stall via Avalon `waitrequest`. No data drop.

### Quad descriptor (64 bytes, 16 words)

| Offset | Size | Field | Format |
|---|---|---|---|
| 0 | 8 | x_min, y_min, x_max, y_max | 4× int16 |
| 8 | 48 | edge0..3 (A, B, C) | 12× int32 Q24.8 |
| 56 | 6 | z0, dz_dx, dz_dy | uint16 Q1.15 + 2× int16 Q1.15 |
| 62 | 1 | `tex_or_color` | uint8 (TEX=1: tex index 0-63; TEX=0: palette index 0-255) |
| 63 | 1 | `flags` | `[0]=TEX [1]=ZTEST` |

### Textured second block (64 bytes, only when TEX=1)

| Offset | Size | Field | Format |
|---|---|---|---|
| 0 | 24 | u0, v0, du_dx, dv_dx, du_dy, dv_dy | 6× int32 Q16.16 |
| 24 | 40 | reserved (write 0) | — |

### IOCTLs

| Name | Arg | Function |
|---|---|---|
| `VGPU_SET_PALETTE` | `{idx, r, g, b}` | upload one palette entry |
| `VGPU_FLIP` | none | flip on vsync (blocking) |
| `VGPU_CLEAR_FRAME` | none | clear Z-buffer AND back framebuffer in parallel (blocking) |
| `VGPU_GET_STATUS` | uint32_t* | read STATUS |
| `VGPU_GET_FRAME_COUNT` | uint32_t* | read FRAME_COUNT |

### Per-frame sequence

```
1. poll input, update game state
2. ioctl(fd, VGPU_CLEAR_FRAME)
3. cull, transform → descriptors
4. write(fd, buf, flat*64 + textured*128)
5. ioctl(fd, VGPU_FLIP)
```

---

## 5. Resource budget

**Cyclone V M10K total: ~496 KB (design doc overstates as 512 KB, utilization headroom makes this immaterial).**

| Resource | Size |
|---|---|
| Framebuffer ×2 (320×240 × 8bpp) | 153.6 KB |
| Z-buffer (320×240 × 16-bit) | 153.6 KB |
| Palette (256 × 24-bit) | 0.75 KB |
| Texture tiles (64 × 16×16 × 8bpp) | 16 KB |
| Command FIFO | 8 KB |
| **Total** | **~332 KB (~67%)** |

Performance: 50 MHz × 1 pix/cycle = 50 Mpix/s. A 320×240 frame takes 1.5 ms peak; 10× overdraw budget at 60fps. Command buffer bandwidth worst case 3.84 MB/s, well under lightweight bridge's tens of MB/s.

---

## 6. Module decomposition

```
rtl/
├── voxel_gpu.sv         — top-level (currently a stub: readdata=0, VGA=0)
├── avalon_slave.sv      — MM interface, control regs, waitrequest backpressure
├── cmd_fifo.sv          — 8 KB BRAM FIFO
├── quad_fetch.sv        — pops 16 (or 32) words, assembles descriptor
├── rasterizer.sv        — FSM: IDLE → LOAD → SCAN → DONE
├── edge_eval.sv         — 4× combinational edge function + sign check
├── interp_unit.sv       — Z, UV incremental interpolation
├── texture_unit.sv      — texture BRAM + nearest-neighbor fetch
├── zbuffer.sv           — dual-port M10K + compare/write
├── framebuffer.sv       — 2× 8bpp BRAM + flip logic (vsync-gated)
├── palette.sv           — 256 × 24-bit BRAM, ARM-writable
├── vga_timing.sv        — 640×480@60 hsync/vsync counters
└── vga_scanout.sv       — 3-stage pipe: FB read → palette → DAC, 2×2 doubling
```

### Team split (3 people)

- **Person A (bus + command path)**: `avalon_slave`, `cmd_fifo`, `quad_fetch`, top-level wiring, kernel driver, Platform Designer maintenance. Must implement Avalon waitrequest.
- **Person B (rasterizer core)**: `rasterizer`, `edge_eval`, `interp_unit`, `zbuffer`. Testbench-heavy. Can work in simulation for weeks before integration.
- **Person C (display path)**: `framebuffer`, `palette`, `vga_timing`, `vga_scanout`, `texture_unit`. Can start immediately by extending lab 3's vga_ball to a real framebuffer.

### Three key cross-module `interface` blocks to define first

Before any real RTL, write ~30 lines of SystemVerilog `interface` definitions:

1. `quad_desc_if` — `quad_fetch` → `rasterizer`, parsed descriptor + valid/ready handshake.
2. `fb_write_if` — `rasterizer` → `framebuffer`, carries (x, y, color_idx, we).
3. `zbuf_if` — `rasterizer` ↔ `zbuffer`, carries (x, y, z_new, we) plus z_pass back.

Three people work in parallel against these contracts. Do this in a single 30-minute pairing session with all three present.

---

## 7. Platform Designer + Quartus state — DONE

Scaffolding is complete. FPGA bitstream compiles. Do NOT redo this work.

### Current state

- Project dir: `voxel-gpu-hw` (copied from `lab3-hw`).
- `voxel_gpu.sv` stub exists with correct ports: `clk`, `reset`, Avalon slave (`address[12:0]`, `chipselect`, `write`, `writedata[31:0]`, `byteenable[3:0]`, `readdata[31:0]`), VGA conduit (8 signals).
- Stub drives `readdata=0` and all `VGA_*=0`. Bus plumbing verified; no logic yet.
- Platform Designer component `voxel_gpu` created with correct interface classifications (clock, reset, avalon_slave_0, vga conduit with lowercase signal types `r/g/b/clk/hs/vs/blank_n/sync_n`).
- `voxel_gpu_hw.tcl` has device tree assignments: `csee4840`, `voxel_gpu`, `voxel`.
- Instance `voxel_gpu_0` connected: `clock`→`clk_0.clk`, `reset`→`clk_0.clk_reset`, `avalon_slave_0`→`hps_0.h2f_lw_axi_master` (base `0x0`, physical `0xFF20_0000`), `vga` conduit exported as `vga`.
- `soc_system_top.sv` already has the VGA pin connections (inherited from lab 3, names match because export is still `vga`).
- `make qsys` ✓, `make quartus` ✓. Bitstream at `output_files/soc_system.sof`.

### CRITICAL GOTCHA

**Every time the Platform Designer Component Editor re-runs "Finish", `voxel_gpu_hw.tcl` is regenerated and the three `set_module_assignment` lines are lost.** You MUST re-add them after every interface change:

```tcl
set_module_assignment embeddedsw.dts.vendor "csee4840"
set_module_assignment embeddedsw.dts.name "voxel_gpu"
set_module_assignment embeddedsw.dts.group "voxel"
```

Lab 3 handout warns about this. Automate it in a Makefile post-step if you can.

### Address port width

Currently 13 bits (32 KB span, using 12 KB). If you ever expand FIFO_WINDOW past 32 KB, you must widen the port and regenerate. Recommend bumping to 16 bits at first interface change. Zero hardware cost.

---

## 8. Pending hardware bring-up — NOT DONE

Finish these before writing real SystemVerilog:

### 8.1 Generate .rbf and .dtb

```bash
make rbf                  # SOF → RBF
# NOTE: dtb generation requires sourcing EDS environment first:
/tools/intel/intelFPGA/21.1/embedded/embedded_command_shell.sh
# In sub-shell:
cd <project-dir>
make dtb                  # → soc_system.dtb
```

Earlier session failed at `make dtb` because EDS wasn't sourced. That's the fix.

### 8.2 Copy to SD card

`output_files/soc_system.rbf` and `soc_system.dtb` → SD card boot partition. Either mount card on workstation or scp to board and `mount /dev/mmcblk0p1 /mnt`.

### 8.3 Boot and verify

```bash
ls /proc/device-tree/sopc@0/bridge@0xc0000000/
cat /proc/device-tree/sopc@0/bridge@0xc0000000/voxel_gpu@*/compatible
# → csee4840,voxel_gpu-1.0
```

### 8.4 Optional u-boot smoke test

Before Linux boots, drop to u-boot prompt, load FPGA from SD, poke registers with `mw.l 0xff200000 1` and `md.l 0xff200004 1`. Stub won't respond meaningfully, but writes should not hang the bus.

---

## 9. Kernel driver plan

Template: lab 3's `vga_ball.c`.

### Platform binding

```c
static const struct of_device_id voxel_gpu_of_match[] = {
    { .compatible = "csee4840,voxel_gpu-1.0" },
    {},
};
```

### ioctl definitions (shared header for kernel + userspace)

```c
#define VGPU_IOC_MAGIC 'V'
#define VGPU_SET_PALETTE      _IOW(VGPU_IOC_MAGIC, 0, struct vgpu_palette_entry)
#define VGPU_FLIP             _IO(VGPU_IOC_MAGIC, 1)
#define VGPU_CLEAR_FRAME      _IO(VGPU_IOC_MAGIC, 2)   // renamed from CLEAR_Z
#define VGPU_GET_STATUS       _IOR(VGPU_IOC_MAGIC, 3, uint32_t)
#define VGPU_GET_FRAME_COUNT  _IOR(VGPU_IOC_MAGIC, 4, uint32_t)

struct vgpu_palette_entry {
    uint8_t idx, r, g, b;
};
```

### Operations

- `open`/`release`: trivial.
- `write(buf, count)`: copy_from_user to kernel buffer, `iowrite32` each word to FIFO_WINDOW. `count` must be multiple of 4 (word boundary); descriptor boundary (64 or 128 bytes) is caller's responsibility.
- `unlocked_ioctl`: switch on command.

### Blocking semantics

- `VGPU_FLIP`: poll STATUS.VSY in a loop with `usleep_range`. More correct: wait on queue woken by IRQ (not in v1).
- `VGPU_CLEAR_FRAME`: set CONTROL.CLR, poll until auto-clears (BSY/CLR handshake from §3.4).

### ioremap region size

At least 0x3000 bytes to cover FIFO_WINDOW through 0x2FFF. Use the size from the device tree `reg` property.

---

## 10. Software pipeline plan

### World

- Block IDs: `uint8_t`, 0=air, 1..255=solid.
- Chunks: 16×16×16 = 4096 bytes.
- Load up to 256 chunks (~1 MB DDR3), render 3-chunk radius.

### Input (evdev)

- `/dev/input/event0` keyboard, `/dev/input/event1` mouse. Non-blocking per-frame reads.
- WASD + space/shift movement, mouse for yaw/pitch, buttons for place/destroy.

### Visibility

1. Frustum cull chunks (8 corners × 6 planes).
2. Hidden-face cull blocks (6 neighbors each).
3. Emit one quad per exposed face.

### Transform (per quad)

1. MVP multiply (Q16.16 intermediate).
2. **Near-plane test**: reject if any `w ≤ 0.1`.
3. Perspective divide.
4. Viewport map to 320×240.
5. Edge coefs: `A = round((y0-y1)·256)`, etc. → Q24.8.
6. Depth plane (Q1.15).
7. UV gradients (Q16.16) if textured.
8. Clamp bbox to [0,319]×[0,239].

### Driver interaction

- Startup: open `/dev/voxel_gpu`, upload 256 palette entries via `VGPU_SET_PALETTE`.
- Per frame: `VGPU_CLEAR_FRAME`, `write()` descriptors, `VGPU_FLIP`.

---

## 11. Build commands

```bash
# First-time FPGA build from the voxel-gpu-hw directory:
make qsys-clean && make qsys && make quartus && make rbf
# (EDS shell required for dtb)
make dtb

# When iterating on RTL only (no interface change):
make qsys-clean && make qsys && make quartus && make rbf

# When interface changes (new ports, widened ports, new conduits):
#   1. qsys-edit soc_system.qsys  (GUI)
#   2. Edit voxel_gpu component, re-analyze, fix Signal Types
#   3. Finish → Yes Save
#   4. RE-ADD set_module_assignment lines to voxel_gpu_hw.tcl
#   5. F5 in Platform Designer
#   6. File → Save
#   7. make qsys-clean && make qsys && make quartus && make rbf
```

---

## 12. Milestones (from §6 of design doc)

1. **VGA + framebuffer baseline.** Test pattern from 320×240 BRAM, 2×2 upscale to 640×480. ARM pokes pixels via bridge.
2. **Linux device driver.** Platform driver + `/dev/voxel_gpu` + 5 ioctls. Userspace test uploads palette.
3. **Software renderer prototype.** Full C pipeline with software rasterization to framebuffer through driver.
4. **FPGA quad rasterizer.** SystemVerilog rasterizer + edge_eval + interp_unit + zbuffer. Verilator/ModelSim verification.
5. **Integration.** User-space game → driver → FPGA rasterizer. Flat-shaded cubes at ≥30 fps.
6. **Camera + interaction.** Evdev input, block place/destroy, per-chunk face-list regeneration.
7. **Texturing.** `$readmemh` texture BRAM, TEX flag path, UV interpolation, nearest-neighbor.

**Stretch (post-M7):** procedural world gen, flat-shaded lighting from fixed sun, skybox quad, IRQ-driven frame pacing.

---

## 13. Technical things the team should know

### Verified correct (sanity-checked during reviews)

- Affine z-interpolation is exact (unlike UV), because `z_ndc = a - b/w` is linear in screen space since `1/w` is linear in screen space under perspective projection. No precision worries for z beyond what Q1.15 already documents.
- Edge coefficient overflow bound: `|E|_max ≈ 2·area ≈ 77,000` for viewport-sized quads. Q24.8 gives ±8.4M integer range, 23 bits of headroom vs 17 needed.
- Descriptor offsets sum to 64 bytes exactly.
- FIFO_WORDS 12 bits covers 0-2048 words (8 KB FIFO).
- VGA hcount[9:1] covers 0-639 visible horizontal span.
- Double-buffer flip ordering: FLIP at end of frame N blocks on rasterizer drain + vsync, so frame N+1's CLEAR_FRAME runs against idle hardware on the just-swapped back buffer.

### Known weaknesses (documented, acceptable for v1)

- **Z-fighting past ~55 world units** due to Q1.15 + near=0.1. Render cap at ~48 units avoids it.
- **Affine UV warping** ("PS1 look") on steep-angle quads. Acceptable at 320×240.
- **Near-plane rejection** causes faces to pop out of view one frame early when camera approaches them. Visible but tolerable.
- **Polling frame pacing** uses negligible CPU but isn't as clean as IRQ-driven.

### Upgrade paths (all clean, none require redoing hardware)

- Sutherland-Hodgman near-plane clipping → replaces rejection with up-to-5-vertex polygon.
- Perspective-correct UV → add 1/w, u/w, v/w to descriptor, add reciprocal unit (Newton-Raphson or LUT), divide per pixel.
- 24-bit reciprocal-z Z-buffer → uniform precision across depth.
- IRQ on FEM → wire `f2h_irq0`, wait on kernel waitqueue.
- Greedy meshing → cuts visible quad count significantly.
- Runtime texture upload → second write window analogous to palette.

---

## 14. Chat session summary (for future agents)

Over three sessions, this project went from a proposal draft to a submitted design document with a working FPGA scaffolding. Key events:

**Session 1:** Convinced user to keep quads (not triangles). Settled on 320×240 + 8bpp palette. Drafted design doc in chat, compiled to LaTeX/PDF with TikZ.

**Session 2:** Walked through Platform Designer component creation step-by-step. Troubleshooted auto-classification errors (VGA signals mis-tagged as Avalon readdata). Full Quartus build completed successfully.

**Session 3:** Three review passes on the design doc. Fixed: truncated STATUS row, missing near-plane value, ambiguous chunk re-meshing, missing arrow widths, Z-buffer hazard reasoning, UV format justification, flat-color descriptor hole, framebuffer clear absence, edge coefficient rounding, CCW winding ambiguity, chunk radii math, DDR3 budget. Near-plane silently moved 0.05 → 0.1 with justification. Document is now submission-ready.

**Where to resume:** user is submitting the design doc. Next phases in order: finish hardware bring-up (§8), write kernel driver (§9), begin milestone 1 (VGA + framebuffer). Platform Designer scaffolding done; implementation is the next phase.

**Things that burned time and should NOT be redone:**
- Debating triangles vs quads (settled: quads).
- Debating resolution and color depth (settled: 320×240 + 8bpp palette).
- Debating silent-drop vs waitrequest (settled: waitrequest).
- Debating near-plane value (settled: 0.1).
- Platform Designer GUI walkthrough (done).

**Trust level for the working state:** high. Every fix in §3 of the design doc has been reviewed by 2+ passes. The six remaining spec gaps in §3 of this handoff are the only loose threads, and each has a recommended answer.
