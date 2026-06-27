# SuperConvolver

A convolution reverb with a GPU-rendered UI, built on [Pulp](https://github.com/danielraffel/pulp). Signed and notarized for macOS (Apple Silicon).

> Temporary distribution repo for sharing preview builds. The source lives in the Pulp repo (`examples/super-convolver`).

![SuperConvolver editor — impulse-response waveform, output spectrum, Mix / Size / Gain / Rooms, Engine (CPU/GPU) and Bypass, with a live GPU status line](screenshot.png)

Running as a CLAP plugin in REAPER:

![SuperConvolver loaded in REAPER](reaper.png)


## What this is — and an honest take on GPU audio

SuperConvolver's audio runs on the **CPU by default**. There's an opt-in **GPU engine** for one specific job that the GPU is genuinely good at. The honest reality of audio on the GPU is that it's **not a free speedup** — and this plugin is built to show exactly where it helps and where it doesn't.

The catch is the round-trip: moving a block of audio to the GPU, running a kernel, and reading the result back costs hundreds of microseconds of fixed overhead per block. For ordinary DSP — a single reverb, an EQ, a compressor — that overhead is *larger than just doing the work on the CPU*, so the CPU wins, full stop. The GPU only pulls ahead when there's enough **parallel** work in one block to amortize that round-trip: many FFTs, many filters, big matrix multiplies.

So SuperConvolver puts the GPU on the one job that fits: **many rooms at once**. Switch **Engine → GPU** and raise **Rooms**, and it convolves your signal against many distinct impulse responses — each panned to its own stereo position — **in one batched GPU submit per block**. This is irreducible work (the rooms can't be collapsed into one summed IR because each is panned differently), and it's exactly the regime where the GPU earns its keep.

### Reading the numbers

A quick glossary, because the units matter:

- **µs** = microseconds = millionths of a second. 1,000 µs = 1 millisecond (ms).
- **block** = the small chunk of audio a host hands a plugin at a time. Here a block is **512 samples**, and at a 48 kHz sample rate that block represents **10,666 µs (≈10.7 ms)** of sound.
- **real-time budget** = the time you have to process one block before you fall behind and the audio glitches. It equals the block's own duration — **10,666 µs** here. Finish under it and you're fine; go over it and you get dropouts.
- **µs/block** = how long the engine actually takes to process one block. **Lower is better.** Anything above the real-time budget means the audio can't keep up.
- **rooms** = the number of distinct, separately-panned impulse responses being convolved at once. More rooms = more parallel work.

So when a row below says "17,448 µs (over budget)," it means the CPU needed 17,448 µs to process a block that has to finish in 10,666 µs — it can't, and the audio breaks up. "3,402 µs" on the GPU means the same block finished comfortably inside the budget.

### Where the GPU wins (many panned rooms)

Measured on Apple Silicon / Metal, block 512 @ 48 kHz. Real-time budget = 10,666 µs/block. **Lower µs/block is better.**

| IR | Rooms | CPU µs/block | GPU µs/block | speedup |
|----|------:|-------------:|-------------:|---------|
| 0.25 s | 256 | 10,027 | 1,752 | **5.7×** |
| 0.50 s | 128 | 8,608 | 1,782 | **4.8×** |
| 0.50 s | 256 | 17,448 (**over** the real-time budget) | 3,402 | **5.1× — the CPU can't keep up; the GPU stays smooth** |
| 1.00 s | 128 | 15,490 (**over** budget) | 3,660 | **4.2×** |

At 0.5 s / 256 rooms and 1.0 s / 128 rooms the CPU simply can't finish a block in time — it overruns the real-time budget and the audio breaks up. The GPU does the same work in a third of the budget. That's the point: not "a bit faster," but **possible at all** in real time.

### Where the GPU does *not* help (and we won't pretend otherwise)

A single convolution — one IR, the normal case — is **faster on the CPU at every length**, because the round-trip dominates:

| IR length | CPU µs/block | GPU µs/block | winner |
|-----------|-------------:|-------------:|--------|
| 0.10 s | 33 | 863 | **CPU** |
| 1.00 s | 212 | 1,169 | **CPU** |
| 4.00 s | 767 | 1,956 | **CPU** |

So single-IR convolution stays on the CPU — the GPU engine is there for the many-rooms case only. The crossover is around 16 rooms; below that, use the CPU. And there's a hard ceiling: past the GPU's storage-buffer binding limit (128 MiB on this device — e.g. 1.0 s × 256 rooms) the GPU engine **refuses the job rather than fake it**, and the CPU engine stays available.

This is the same conclusion a lot of audio developers have reached: the GPU is the wrong tool for a single cheap effect, and the right tool for a few specific massively-parallel problems. SuperConvolver is a demonstration of one of the latter.

### Live stats — see it for yourself

The editor's status line reports what the audio engine is actually doing, in real time:

> *Audio: GPU · Metal · 64 rooms · 128 blocks, 2 misses · 4467 µs/block (84% of real-time)*

Reading it left to right:

- **Audio: GPU** — which engine is carrying the *sound* right now (the picture is always GPU-drawn; this tells you whether the **audio** is on the CPU or the GPU).
- **Metal** — the GPU graphics/compute backend in use. It's **Metal** on Apple Silicon; on other platforms it would read D3D12 or Vulkan.
- **64 rooms** — how many separately-panned impulse responses are being convolved every block. The more rooms, the more parallel work the GPU is doing.
- **128 blocks** — how many audio blocks the GPU worker has finished and delivered so far. It just counts up while audio plays; it's proof the GPU is actually producing sound, not sitting idle.
- **2 misses** — blocks the GPU couldn't deliver in time, so the plugin filled them another way (no dropout — it stays seamless). A couple of misses at startup is normal while the engine warms up; a steadily climbing number means the GPU is falling behind for your settings.
- **4467 µs/block** — the real measured cost of one block on the GPU path, in microseconds (millionths of a second), **including** the CPU↔GPU round-trip. Lower is better.
- **84% of real-time** — the headroom gauge: that µs/block as a percentage of how long the plugin actually has to process a block *on the machine you're running it on*. It's derived live from your device's measured speed and the current sample rate, so it answers the practical question — **how much room is left?** 84% means roughly 16% to spare; add rooms and it climbs toward 100%, and once it would pass 100% you're out of real-time budget and should use fewer rooms (or the CPU). This is the single number that tells you what *your* GPU can handle.

These are measured values, not decorations — the µs/block figure is timed on the worker thread that drives the GPU, the same way the benchmark numbers above are produced, and the percentage is computed against your device's own real-time budget.

### How we verified it really runs on the GPU

Three independent checks, all reproducible from the source repo:

1. **Hardware** — under a sustained GPU-only convolution load, the Mac's GPU utilization counter rises to ~60%, while at rest it reads 0%.
2. **Correctness** — the GPU output matches an independent CPU reference bit-for-bit (cross-correlation = 1.0). The GPU path emits **silence** if it can't run, so a non-working GPU would fail this test outright — it passes.
3. **Timing** — the GPU finishes the 256-room block in 3,402 µs; the same work takes the CPU 17,448 µs. You cannot get the GPU's number on a CPU.

## ⚠️ Apple Silicon native — run your host natively (not Rosetta)

These builds are **arm64 (Apple Silicon) only**. A host running under **Rosetta (x86_64)** cannot load an arm64-only plugin — it shows up as *"couldn't be opened"* / not found. If a plugin won't load:

1. Quit your DAW.
2. In **Applications**, right-click the DAW → **Get Info** → **uncheck "Open using Rosetta"**.
3. Relaunch. (Ableton Live 11.3+, Logic, REAPER, Bitwig all run natively on Apple Silicon.)

If it still won't load, run **SuperConvolver Diagnostics** (below) and send back the report.

## Download

Grab the latest [release](../../releases/latest) — a single installer:

- **`SuperConvolver-1.0.5.pkg`** — one notarized installer. Its **Customize** pane lets you pick any of:
  - **Audio Unit (AU)** → Logic Pro, GarageBand
  - **VST3** → most DAWs
  - **CLAP** → REAPER, Bitwig
  - **Standalone app** → `SuperConvolver.app` in Applications (no DAW needed)
  - **Diagnostics helper** → if a plugin won't load, run it; it writes a report **.zip to your Desktop** (system info, per-format install/sign/quarantine status, **executable architecture**, `auval`). Send that zip back.

Signed with a Developer ID and notarized by Apple, so it opens without Gatekeeper warnings.

## Controls

- **Mix** — dry/wet.
- **Size** — reverb length (0.05–4 s). Morphs the IR live (rebuilt off the audio thread, swapped in lock-free).
- **Gain** — output trim.
- **Rooms** — how many distinct panned IRs to convolve at once (1–256). On the GPU engine this is the knob that makes the GPU worth using.
- **Engine** — CPU (default) or GPU. CPU is the right choice for one room; GPU for many.
- **Bypass**.

Correct under any host block size (internal re-blocking). The GPU engine runs at a fixed reported latency so host plugin-delay compensation stays exact when you switch engines.

The UI — a live impulse-response waveform tinted by the output spectrum, a log-frequency spectrum display, and the controls — is drawn through Pulp's Skia/Dawn GPU surface and themed with Pulp's *Ink & Signal* design language.

## Requirements

macOS on Apple Silicon (arm64). Run your DAW natively (not under Rosetta).

## Feedback

It's a preview — expect rough edges. Issues and notes welcome.
