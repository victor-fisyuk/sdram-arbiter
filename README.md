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
- Fully open toolchain: Yosys with yosys-slang, nextpnr-ecp5; Verilator for simulation, with a
  protocol-checking SDRAM model.

## License

To be decided when the source is published.

---

Lattice and ECP5 are trademarks of Lattice Semiconductor Corporation. Other names may be trademarks of their
respective owners. This project is not affiliated with or endorsed by any of them.