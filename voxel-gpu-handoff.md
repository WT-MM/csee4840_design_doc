# Voxel GPU Project — Session Handoff Document

**Date of session:** April 16, 2026
**Course:** CSEE 4840 Embedded System Design, Spring 2026 (Columbia)
**Team:** Josh Bernheisel (jcb2301), Wesley Maa (wm2505), Mihir Joshi (mnj2122)
**Platform:** DE1-SoC (Intel Cyclone V, ARM Cortex-A9 + FPGA fabric)
**Project name:** Hardware-accelerated block game, working module name `voxel_gpu`

This document captures the full state of the project after a design-day working session. It is written for an AI agent who will help implement the project over the coming weeks. It is intentionally detailed — err on the side of reading all of it before taking action.

---

## 1. Project overview

The goal is a hardware-accelerated 3D block game (Minecraft-style) running on the DE1-SoC. Split of responsibility:

- **ARM Cortex-A9 (running Linux):** all control-heavy work — world representation, visibility, camera math, per-quad setup.
- **FPGA fabric:** fixed-function quadrilateral rasterizer with z-buffering, texture lookup, VGA output.
- **Linux kernel driver:** mediates between user-space game and FPGA peripheral through the HPS-to-FPGA lightweight bridge.

**Key design decisions made this session (do not relitigate without strong reason):**

1. **Quads, not triangles.** Every block face is a rectangle; greedy meshing produces rectangles. Quads avoid doubling the primitive count and avoid splitting work. Four-edge-function rasterizer handles perspective-projected convex quads correctly.

2. **320×240 internal resolution, 2×2 upscaled to 640×480 VGA.** Fits in on-chip BRAM. Deterministic single-cycle memory access. No SDRAM controller headaches.

3. **8bpp palette-indexed framebuffer, 16-bit Z-buffer.** Palette is 256 entries of RGB888. Saves ~150KB vs 16bpp, and block-game aesthetics don't need more than 256 colors at a time. Scanout adds one pipeline stage for the palette lookup; throughput unaffected.

4. **Double-buffered framebuffer.** Flip on vsync.

5. **Fixed-point everywhere.** Q16.16 for transforms and edge coefficients; Q4.12 for UV and depth gradients; Q0.16 for depth values.

---

## 2. Design document

**Status: COMPLETE. Due April 17, 2026 (submitted or about to be).**

Files:
- `/mnt/user-data/outputs/block-game-design-document.pdf` — 10-page compiled PDF with TikZ block diagram and scanout pipeline figure.
- `/mnt/user-data/outputs/block-game-design-document.tex` — LaTeX source. Rebuild with `pdflatex design.tex` run twice.

Structure follows Professor Edwards' sample doc: Introduction, System Block Diagram (with figure), Algorithms, Resource Budgets, Hardware/Software Interface.

---

## 3. Hardware/software interface — authoritative register map

This is the contract between hardware and software. Section 5 of the design doc has it all; here it is in compact form.

**Base address:** `0xFF20_0000` (lightweight bridge), peripheral offset `0x0`.
**Address port width:** 13 bits word-addressed (32 KB span — using 12 KB, leaves room).
**writedata/readdata width:** 32 bits.

### Control registers

| Offset | Name | R/W | Description |
|--------|------|-----|-------------|
| 0x00 | CONTROL | R/W | bit0=EN, bit1=FLP, bit2=IEN, bit3=CLR |
| 0x04 | STATUS | R | bit0=BSY, bit1=FFL, bit2=FEM, bit3=VSY, [19:4]=FIFO_COUNT |
| 0x08 | FRAME_COUNT | R | 32-bit free-running frame counter |
| 0x0C | PALETTE_ADDR | W | palette entry index [7:0] |
| 0x10 | PALETTE_DATA | W | [23:16]=R [15:8]=G [7:0]=B; auto-increments PALETTE_ADDR |
| 0x1000–0x2FFF | FIFO_WINDOW | W | 8 KB memory-mapped window into command FIFO |

### CONTROL bits
- EN: 1 = rasterizer processes FIFO, 0 = idle
- FLP: write 1 to request front/back flip on next vsync; auto-clears
- IEN: 1 = assert IRQ on FIFO-empty (frame done)
- CLR: write 1 to clear Z-buffer; auto-clears when done

### STATUS bits
- BSY: rasterizer actively processing a quad
- FFL: FIFO full; writes will be dropped
- FEM: FIFO empty
- VSY: currently in vblank (safe to flip)
- FIFO_COUNT: number of quad descriptors currently queued

### Quad descriptor format (64 bytes, written to FIFO_WINDOW as 16 consecutive 32-bit little-endian words)

| Offset | Size | Field | Format | Meaning |
|--------|------|-------|--------|---------|
| 0 | 2 | x_min | int16 | bbox left |
| 2 | 2 | y_min | int16 | bbox top |
| 4 | 2 | x_max | int16 | bbox right |
| 6 | 2 | y_max | int16 | bbox bottom |
| 8 | 4 | edge0_A | int32 Q16.16 | |
| 12 | 4 | edge0_B | int32 Q16.16 | |
| 16 | 4 | edge0_C | int32 Q16.16 | |
| 20-52 | 3×12 | edge1/2/3 ABC | int32 Q16.16 each | (A,B,C) per edge |
| 56 | 2 | z0 | uint16 Q0.16 | depth at (x_min, y_min) |
| 58 | 2 | dz_dx | int16 Q4.12 | |
| 60 | 2 | dz_dy | int16 Q4.12 | |
| 62 | 1 | texture_id | uint8 | 0-63 |
| 63 | 1 | flags | uint8 | bit0=TEX, bit1=ZTEST, rest reserved |

Textured quads append a second 64-byte block with UV origin and gradients (u0, v0, du_dx, dv_dx, du_dy, dv_dy). Flat-shaded quads omit the second block.

Edge function: `E(x,y) = A*x + B*y + C`, non-negative inside the quad for CCW winding. Pixel is inside when all four E_i >= 0.

### IOCTLs

| Name | Arg | Function |
|------|-----|----------|
| VGPU_SET_PALETTE | `struct {uint8_t idx,r,g,b;}` | upload one palette entry |
| VGPU_FLIP | none | flip on next vsync (blocks) |
| VGPU_CLEAR_Z | none | clear Z-buffer (blocks) |
| VGPU_GET_STATUS | `uint32_t*` (out) | read STATUS |
| VGPU_GET_FRAME_COUNT | `uint32_t*` (out) | read FRAME_COUNT |

Quad descriptor writes happen via ordinary `write()` on `/dev/voxel_gpu`, which the driver forwards to FIFO_WINDOW using `iowrite32`.

### Per-frame software sequence

```
for each frame:
  1. poll input, update game state
  2. ioctl(fd, VGPU_CLEAR_Z)
  3. cull chunks, cull faces, transform quads -> quad_desc[]
  4. write(fd, quad_desc, n * 64)
  5. ioctl(fd, VGPU_FLIP)
```

---

## 4. Resource budget (BRAM)

Cyclone V M10K total: ~512 KB / 4 Mbit.

| Resource | Size |
|----------|------|
| Framebuffer ×2 (320×240, 8bpp) | 153.6 KB |
| Z-buffer (320×240, 16-bit) | 153.6 KB |
| Palette (256 × 24-bit) | 0.75 KB |
| Texture tiles (64 × 16×16 × 8bpp) | 16 KB |
| Command FIFO | 8 KB |
| **Total** | **~332 KB (~65%)** |

Comfortable headroom. Expansion options if needed: more textures, deeper FIFO, triple-buffering.

---

## 5. Module decomposition (planned SystemVerilog)

```
rtl/
├── voxel_gpu.sv         — top-level peripheral (currently a stub)
├── avalon_slave.sv      — Avalon MM interface, control regs
├── cmd_fifo.sv          — 8 KB command FIFO (wraps BRAM)
├── quad_fetch.sv        — pops 16 words from FIFO, assembles descriptor
├── rasterizer.sv        — FSM: IDLE → LOAD → SCAN → DONE
├── edge_eval.sv         — 4× combinational edge evaluators
├── interp_unit.sv       — Z and UV incremental interpolation
├── texture_unit.sv      — texture BRAM + nearest-neighbor UV fetch
├── zbuffer.sv           — Z-buffer BRAM + compare/write
├── framebuffer.sv       — double-buf 8bpp BRAM + flip logic
├── palette.sv           — 256×24-bit RGB BRAM
├── vga_timing.sv        — hsync/vsync counters (standard 640×480@60)
└── vga_scanout.sv       — 3-stage pipeline: FB read → palette lookup → DAC
```

### Team split (3 people)

- **Person A (bus + command path):** avalon_slave, cmd_fifo, quad_fetch, top-level wiring, kernel driver, Platform Designer maintenance.
- **Person B (rasterizer core):** rasterizer, edge_eval, interp_unit. Testbench-heavy. Can work entirely in simulation with synthetic quads for weeks before integration.
- **Person C (display path):** framebuffer, palette, vga_timing, vga_scanout, texture_unit, zbuffer BRAM wrappers. Can start immediately by extending lab 3 vga_ball into a working framebuffer display — that's basically milestone 1.

### Three key cross-module interfaces to define first (SystemVerilog `interface` blocks, ~30 lines total)

1. `quad_desc_if` — between quad_fetch and rasterizer. Carries the parsed descriptor + valid/ready handshake.
2. `fb_write_if` — rasterizer → framebuffer. Carries (x, y, color_index, write_enable).
3. `zbuf_if` — rasterizer ↔ zbuffer. Carries (x, y, z_new) plus z_pass back.

Define these first so all three people have a contract to work against.

---

## 6. Platform Designer setup — COMPLETED THIS SESSION

The FPGA project is scaffolded. Starting point was `lab3-hw`, copied to a new project directory.

### Current state

- `voxel_gpu.sv` stub exists at project root. Has correct port list:
  - `clk`, `reset`
  - Avalon slave: `address[12:0]`, `chipselect`, `write`, `writedata[31:0]`, `byteenable[3:0]`, `readdata[31:0]`
  - VGA conduit: `VGA_R[7:0]`, `VGA_G[7:0]`, `VGA_B[7:0]`, `VGA_CLK`, `VGA_HS`, `VGA_VS`, `VGA_BLANK_n`, `VGA_SYNC_n`
  - Stub logic drives `readdata = 0` and all `VGA_*` to 0.

- Platform Designer component `voxel_gpu` created with:
  - `clock` interface → `clk`
  - `reset` interface → `reset`, Associated Clock = `clock`
  - `avalon_slave_0` interface, Associated Clock = `clock`, Associated Reset = `reset`, Address Units = WORDS, Read Wait = 1
  - `vga` conduit with 8 signals, Signal Types `r`, `g`, `b`, `clk`, `hs`, `vs`, `blank_n`, `sync_n`

- `voxel_gpu_hw.tcl` has device tree assignment lines:
  ```tcl
  set_module_assignment embeddedsw.dts.vendor "csee4840"
  set_module_assignment embeddedsw.dts.name "voxel_gpu"
  set_module_assignment embeddedsw.dts.group "voxel"
  ```

- Instance `voxel_gpu_0` added to system, connected:
  - clock → clk_0.clk
  - reset → clk_0.clk_reset
  - avalon_slave_0 → hps_0.h2f_lw_axi_master (base `0x0000_0000`, which is `0xFF20_0000` from ARM's view)
  - vga conduit exported as `vga`

- `soc_system_top.sv` already had VGA pin connections from lab 3 (reused because export name `vga` matches lab 3's). No edits needed there.

- `make qsys` completed successfully.
- `make quartus` completed successfully. Bitstream at `output_files/soc_system.sof`.
- Fmax of `clock_50_1` needs verification in `output_files/soc_system.sta.rpt` (should be >> 50 MHz for the stub; must stay > 50 MHz as real logic is added).

### KNOWN GOTCHA — device tree tcl is overwritten

Every time the Component Editor re-runs Finish (i.e., after any interface change), `voxel_gpu_hw.tcl` is regenerated and the three `set_module_assignment` lines are lost. Re-add them manually each time. Lab 3 handout warns about this too.

---

## 7. Pending steps to finish hardware bring-up

These were not completed in the session. They should be done before writing real SystemVerilog.

### 7.1 Generate `.rbf` and `.dtb`

```
# Convert bitstream to raw binary format for SD card loading
make rbf

# Generate device tree blob (requires EDS environment)
/tools/intel/intelFPGA/21.1/embedded/embedded_command_shell.sh
# In the sub-shell:
cd <project-dir>
make dtb
```

`make dtb` failed during the session with "sopc2dts not found" because `embedded_command_shell.sh` had not been sourced. This is the fix.

### 7.2 Copy to SD card

Copy `output_files/soc_system.rbf` and `soc_system.dtb` to the boot partition of the SD card. Either:
- Remove the card, mount on workstation, copy, `sync`, unmount.
- Or boot the board, `mount /dev/mmcblk0p1 /mnt`, `scp` from workstation.

### 7.3 Boot and verify

```
# Verify device tree shows the peripheral
ls /proc/device-tree/sopc@0/bridge@0xc0000000/
cat /proc/device-tree/sopc@0/bridge@0xc0000000/voxel_gpu@*/compatible
# Should output: csee4840,voxel_gpu-1.0
```

### 7.4 Optional: verify Avalon path from u-boot

Before booting Linux, during u-boot autoboot prompt, hit a key to enter u-boot shell. Load the FPGA:
```
fatload mmc 0:1 $fpgadata soc_system.rbf
fpga load 0 $fpgadata $filesize
run bridge_enable_handoff
```
Then poke registers:
```
mw.l ff200000 1      # write 1 to CONTROL (sets EN)
md.l ff200004 1      # read STATUS
```
Stub will not respond meaningfully, but writes should not hang the bus.

---

## 8. Kernel driver plan

After bring-up, the next layer. Template is lab 3's `vga_ball.c`. Key changes:

### Compatible string match

```c
static const struct of_device_id voxel_gpu_of_match[] = {
    { .compatible = "csee4840,voxel_gpu-1.0" },
    {},
};
```

### Register access

Mapped region is at least 0x3000 bytes (header says 0x1000 but full FIFO_WINDOW needs 0x3000). Use `ioremap` from `of_iomap` result; access with `iowrite32` / `ioread32`.

### Character device

Expose `/dev/voxel_gpu`. File ops:
- `open`, `release`: trivial
- `write(buf, count)`: copy from user to a kernel staging buffer, then `iowrite32` each 4-byte word to FIFO_WINDOW. Count must be a multiple of 64 (quad descriptor size). Validate.
- `unlocked_ioctl`: switch on command, handle the five defined ioctls.

### IOCTL definitions (put in a shared header for kernel + user-space)

```c
#define VGPU_IOC_MAGIC 'V'
#define VGPU_SET_PALETTE     _IOW(VGPU_IOC_MAGIC, 0, struct vgpu_palette_entry)
#define VGPU_FLIP            _IO(VGPU_IOC_MAGIC, 1)
#define VGPU_CLEAR_Z         _IO(VGPU_IOC_MAGIC, 2)
#define VGPU_GET_STATUS      _IOR(VGPU_IOC_MAGIC, 3, uint32_t)
#define VGPU_GET_FRAME_COUNT _IOR(VGPU_IOC_MAGIC, 4, uint32_t)

struct vgpu_palette_entry {
    uint8_t idx, r, g, b;
};
```

### Blocking behavior

VGPU_FLIP and VGPU_CLEAR_Z should block until done. Simplest: poll STATUS in a loop (with `usleep_range`); more correct: wait on a waitqueue woken by the IRQ handler when FEM goes high. Start with polling; add IRQ later if needed.

---

## 9. Software pipeline plan (user-space)

Runs as a single-threaded C program on the ARM under Linux.

### World representation

- Block IDs: `uint8_t`. 0 = air, 1..255 = solid block types.
- Chunks: 16×16×16 = 4096 bytes each.
- World: 3D array of chunks. Up to 32 chunks loaded at once = ~128 KB.

### Input

- `/dev/input/event0` (keyboard), `/dev/input/event1` (mouse) — use evdev API.
- Non-blocking reads each frame; accumulate deltas.
- Keyboard: WASD for movement, space/shift for vertical.
- Mouse: relative motion for yaw/pitch, buttons for place/destroy.

### Visibility

1. View-frustum cull chunks (test 8 corners vs 6 planes).
2. For each visible chunk, for each solid block, check 6 neighbors; emit quad for each face where neighbor is air.

### Transform

1. Build model-view-projection matrix (4×4, Q16.16).
2. For each quad's 4 vertices: multiply by MVP → perspective divide → viewport map to 320×240.
3. Compute edge coefficients: `A = y0 - y1, B = x1 - x0, C = -(A*x0 + B*y0)`.
4. Compute depth plane: solve for dz_dx, dz_dy from 3 vertices with known z.
5. Compute UV gradients (for textured quads).
6. Clamp bbox to screen bounds.
7. Pack into 64-byte (or 128-byte if textured) descriptor.

### Driver calls

- Open `/dev/voxel_gpu` once at startup.
- At startup: upload palette via 256 × `VGPU_SET_PALETTE` ioctls.
- Each frame: `VGPU_CLEAR_Z`, `write()` descriptors, `VGPU_FLIP`.

---

## 10. Build commands reference

From project root (inside EDS shell where needed):

```
make qsys-clean       # delete generated interconnect
make qsys             # regenerate interconnect from .qsys
make quartus          # full Quartus compile → .sof
make rbf              # convert .sof → .rbf for SD card
make dtb              # generate device tree blob (needs EDS env)
```

Iteration cycle when editing only SystemVerilog internals (no interface change):
```
make qsys-clean && make qsys && make quartus && make rbf
```

When interface changes (new registers, widened ports, new conduits):
1. `qsys-edit soc_system.qsys` (GUI)
2. Select voxel_gpu under Project → Edit
3. Re-analyze synthesis files
4. Check/fix Signals & Interfaces tab
5. Finish → Yes, Save
6. **Re-add the three `set_module_assignment` lines to `voxel_gpu_hw.tcl`**
7. Press F5 in Platform Designer (Refresh System)
8. File → Save
9. `make qsys-clean && make qsys && make quartus && make rbf`

---

## 11. First milestones to work toward (after design doc submitted)

From design doc §6 of the proposal, in order:

1. **VGA + framebuffer baseline.** FPGA VGA timing controller displays a test pattern from a 640×480 pixel buffer (or 320×240 upscaled). ARM writes colored pixels through the driver.

2. **Linux device driver.** Implement the kernel module, expose `/dev/voxel_gpu`, support writes from user space.

3. **Software renderer prototype.** Full C pipeline running in user space doing software rasterization to the framebuffer via the driver.

4. **FPGA quad rasterizer.** SystemVerilog implementation with 4-edge-function test. Verify with hardcoded quad descriptors via Verilator/ModelSim. Add Z-buffer.

5. **Integration.** Wire user-space game to FPGA rasterizer through driver. Render small static scene of colored cubes using HW acceleration.

6. **Camera and interaction.** First-person camera with mouse/keyboard; block place/destroy.

7. **Texturing.** Per-pixel texture lookup from BRAM.

---

## 12. Style/quality expectations (from sample design doc)

- Block diagram: ~10 blocks, detailed but not overwhelming. Label every communication pathway with its protocol.
- Algorithm section: pseudocode is fine, but clear structure. Describe both hardware and software algorithms. For hardware, explain why the structure maps well to hardware.
- Resource budgets: quantified, in bits or bytes. Mention BRAM as the key constraint.
- HW/SW interface: bit-level for control registers (TMS9918A style); struct-table format for command descriptors; both are appropriate and expected.

---

## 13. Open questions / things to revisit

1. **Address port width.** Currently 13 bits. Enough for 0x3000 of registers + FIFO. Flagged during session: widening to 16 bits later is an interface change → annoying. Worth widening now if revisiting Platform Designer.

2. **Texture upload.** Design doc doesn't specify a mechanism for the ARM to upload textures to BRAM. Options: (a) bake textures into BRAM at synthesis via `$readmemh`, (b) add a second write window similar to palette, (c) dedicated ioctl with its own register range. For v1, option (a) is simplest — hardcode textures.

3. **Interrupt handling.** The IEN control bit is defined but the IRQ wiring from the FPGA to the HPS isn't set up. For v1, polling is fine. Add later if needed.

4. **Chunk management.** How does the game decide which chunks are loaded? For v1: fixed small world (say 4×4×2 chunks), all loaded always. Procedural generation can come later.

5. **Perspective-correct texturing.** Current plan is affine UV interpolation (no perspective correction). This will look wavy on quads at steep angles — acceptable for block games, especially at 320×240. Perspective-correct interpolation requires a per-pixel divide; not planning it for v1.

---

## 14. Chat session summary

The session covered:

1. Pushback on the user's proposal to use triangles instead of quads — convinced them to stay with quads for correctness + efficiency reasons.
2. Design decisions: resolution, color depth, memory layout, descriptor format.
3. Explanation of palette-indexed rendering (8bpp + CLUT) and why it's fast.
4. Drafting the design doc in chat, then compiling to LaTeX + PDF with a TikZ block diagram.
5. Walking through Platform Designer component creation step-by-step, troubleshooting auto-classification errors ("writeresponsevalid_n appearing 5 times" was the auto-guesser mis-tagging VGA signals — fix was moving them to a conduit and setting correct Signal Types).
6. Full Quartus compile completed successfully.
7. `make dtb` failed because `embedded_command_shell.sh` was never sourced — left as next step.

**Where to resume:** user is about to submit the design doc. After that, continue with hardware bring-up (§7 above), then kernel driver (§8), then first VGA milestone (§11 step 1). The scaffolding is done; implementation is the next phase.
