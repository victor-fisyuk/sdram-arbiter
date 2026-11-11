# Pipelined Multi-Client SDRAM Arbiter

An SDRAM arbiter, controller and display path for FPGA GPUs, written in SystemVerilog for the Lattice ECP5
on the [Radiona ULX3S](https://radiona.org/ulx3s/) board.

> **Status:** the source code will be published here once it is ready to be open-sourced.
> This README describes the project in the meantime.

## What it is

A memory subsystem for an FPGA graphics pipeline: several clients share one 16-bit SDR SDRAM
(32 MB, 4 banks) while a display controller scans a framebuffer out as digital video (through the
ULX3S GPDI port). The arbiter is built to keep the SDRAM busy when many clients work on different banks
at the same time, and to give scanout top priority on the bank it reads from.

### Arbiter

- **Per-bank client election.** Eight fixed clients (display, framebuffer, depth buffer, texture, two clear
  streams, two host ports) can use any bank. Each bank elects its owner every cycle, so different clients
  own different banks at once and one bank's row change overlaps another bank's data transfer.
- **Anti-starvation preempt.** Election is by priority; when a bank's second-place client has kept losing
  for about 32 cycles, it is promoted for one access.
- **Full-rate page-hit streaming.** A feed-forward pipeline (every stage boundary registered) with burst
  length 1 lets a client stream page hits to any column of an open row at one access per clock with no
  arbiter-added bubble; a single client's page-hit stream is detected a cycle early so it skips the
  per-bank round-robin.
- **Independent ingress.** All clients can be accepted in the same cycle (per-client FIFOs, no shared
  allocator).
- **Read-after-write ordering without a CAM.** Each client's accesses to one bank issue in order, so a
  client's read after its own write to the same address is ordered by construction (one owner per bank,
  in-order issue). Ordering between clients is left to the write fences below.
- **Access timing in the arbiter.** It tracks every bank's open row and enforces the row-access and
  read/write turnaround timing itself; the controller only runs the init sequence and refresh, drives the
  pins and captures read data.
- **Display first.** Scanout has top priority in its bank's election and reserves that bank while it
  streams, so its row normally stays open (the anti-starvation preempt can still let another client in);
  an 8-line prefetch FIFO gives slack. A guaranteed share of the command bus is still open work.
- **Cross-client write fences.** Every writing client exposes `writes_pending`, so a renderer can drain its
  writes before a page flip, or make sure a texture upload has landed before it is sampled.

### Display path

640×480 at 60 Hz digital video (timing set by an input, not hard-wired), double-buffered page flip on
vblank, configurable line pitch, a dual-clock line buffer, an optional on-screen FPS counter, and an
optional live SDRAM performance panel (per-client wait share and access rate, per-bank read/write share and
row-miss ratio, command-bus and refresh use).

## Hardware and tools

- Board: Radiona ULX3S (Lattice ECP5-85F) with its on-board 32 MB SDRAM (W9825G6KH-6, 4 × 8192 rows × 512
  columns × 16 bit).
- Render/SDRAM clock: 130 MHz, verified on hardware.
- Arbiter size: about 6,200 LUT4 and 3,700 flip-flops (under 8 % of the ECP5-85F), no block RAM, no DSP;
  the per-client request FIFOs are in about 90 small LUT-RAM cells.
- Fully open toolchain: Yosys with yosys-slang, nextpnr-ecp5; Verilator for simulation, with a
  protocol-checking SDRAM model.

## Demo: galaxy splat renderer

[![SDRAM Arbiter for FPGA - Galaxy Demo (ULX3S)](https://img.youtube.com/vi/KMo5h-kqXBI/hqdefault.jpg)](https://www.youtube.com/watch?v=KMo5h-kqXBI)

A standalone demo renders a rotating spiral galaxy of 24,576 depth-tested stars at 640×480, generated on
the FPGA with no PC attached. Every frame clears the colour and depth buffers, streams the star records,
and does a depth read-modify-write and a colour write for every visible star, while the display scans out
the other buffer — each SDRAM bank serving a different client at once:

| Bank | Client | Traffic |
|---|---|---|
| 0 / 1 | display, framebuffer, framebuffer clear | scanout of one buffer; clear and random colour writes into the other |
| 2 | depth buffer, depth buffer clear | clear and random depth read-modify-write |
| 3 | texture | sequential star-record stream |

On the ULX3S at 130 MHz it holds a steady 60 frames per second (the display refresh rate; the page flip
waits for vblank), with the live SDRAM performance panel on screen.

## Demo: fractal flame

[![SDRAM Arbiter for FPGA - Fractal Flame Demo (ULX3S)](https://img.youtube.com/vi/guGGuJXExok/hqdefault.jpg)](https://www.youtube.com/watch?v=guGGuJXExok)

A second standalone demo renders a fractal flame — an iterated function system of four affine maps with nonlinear
variations — generated on the FPGA with no PC attached. Every frame the FPGA computes 69,632 points; each one
that lands on screen is added to a 640×480 histogram in SDRAM with one read-modify-write at an effectively random
address. Outside the bright cores there is little reuse and no spatial locality, so a cache would rarely hit. The
histogram is spread over all four banks in 512-pixel blocks (one SDRAM row each), and four read-modify-write
engines, one per bank, overlap their row misses. A tone-map pass then decays the histogram and turns it into a
glowing RGB565 image (log density, automatic exposure) while the display scans out the other buffer. Eight
presets take turns: each holds for 30 seconds, then morphs into the next over 5 seconds.

The demo borrows the arbiter's fixed client ports whatever their names:

| Bank | Client | Traffic |
|---|---|---|
| 0 | depth buffer, display, framebuffer clear | histogram read-modify-write; scanout, or tone-mapped writes (swapping with bank 1 every frame) |
| 1 | framebuffer, display, framebuffer clear | histogram read-modify-write; scanout, or tone-mapped writes (swapping with bank 0 every frame) |
| 2 | host framebuffer access | histogram read-modify-write |
| 3 | texture, host texture upload | histogram read-modify-write (reads on one client, writes on the other) |
| 0–3 | depth buffer clear | zeroes the histogram once at start-up |

On the ULX3S at 130 MHz it holds a steady 60 frames per second, with the live SDRAM performance panel on
screen.

## Demo: voxel terrain

[![SDRAM Arbiter for FPGA - Voxel Terrain Demo (ULX3S)](https://img.youtube.com/vi/57LyhwzPfSM/hqdefault.jpg)](https://www.youtube.com/watch?v=57LyhwzPfSM)

A third standalone demo flies a camera over a voxel landscape, generated on the FPGA with no PC attached. At
start-up the FPGA computes a 2048×2048 height-and-colour map (value noise with water, sand, grass, forest, rock
and snow bands and baked sun shading) and writes all 8 MB of it into SDRAM. Every frame a column ray-marcher
draws the landscape at 640×480: each screen column steps front to back through the map in 444 samples, with
distance haze and a sky gradient. A fixed autopilot turns gently, follows the terrain and swings between low
valley runs and high overviews, with no input.

Where the other two demos scatter read-modify-writes, this one only reads its map: every frame makes 284,160 map reads
along the rays of the screen columns, at addresses that depend only on the camera, never on data already read,
so they can be issued far ahead of the data. The map is stored as 32×16-cell tiles, one tile per SDRAM row, in a
checkerboard over two banks, so about one read in ten opens a new row. One in-order read port per bank and a
256-entry reorder buffer keep many reads in flight, so one bank's row changes overlap the other bank's reads.
A write cannot start while read data is still on the SDRAM data bus, so under a steady stream of reads the writes
would rarely get a turn; the demo therefore batches its traffic itself: each group of 32 columns is rendered into
one on-chip column buffer, then written out line by line before the next group starts, so the map reads and the
pixel writes run in separate phases.

Like the flame demo, it borrows the arbiter's fixed client ports whatever their names:

| Bank | Client | Traffic |
|---|---|---|
| 0 / 1 | display, framebuffer | scanout of one buffer; line-by-line pixel writes into the other |
| 2 | depth buffer | map reads (one colour of the checkerboard) |
| 3 | texture | map reads (the other colour) |
| 2 / 3 | host texture upload | writes the map once at start-up |

On the ULX3S at 130 MHz it holds a steady 60 frames per second, with the live SDRAM performance panel on
screen.

## License

To be decided when the source is published.

---

Lattice and ECP5 are trademarks of Lattice Semiconductor Corporation. Other names may be trademarks of their
respective owners. This project is not affiliated with or endorsed by any of them.