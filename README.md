# SuperConvolver

A convolution reverb with a GPU-rendered UI, built on [Pulp](https://github.com/danielraffel/pulp). Signed and notarized for macOS (Apple Silicon).

> Temporary distribution repo for sharing preview builds. The source lives in the Pulp repo (`examples/super-convolver`).

## Download

Grab the latest [release](../../releases/latest):

- **`SuperConvolver-1.0.0.dmg`** — the standalone app. Drag to Applications and double-click. No DAW needed.
- **`SuperConvolver-1.0.0.pkg`** — the plugin installer. Installs the **AU**, **VST3**, and **CLAP** formats (the installer's *Customize* pane lets you pick which). For Logic/GarageBand (AU), most DAWs (VST3), REAPER/Bitwig (CLAP).

Both are signed with a Developer ID and notarized by Apple, so they open without Gatekeeper warnings.

## What it is

- **Convolution reverb** — your signal convolved with a decaying impulse response. The **Size** knob morphs the IR live (rebuilt off the audio thread, swapped in lock-free).
- **GPU-rendered UI** — a live impulse-response waveform tinted by the output spectrum, a log-frequency spectrum display, and Mix / Size / Gain + Bypass, all drawn through Pulp's Skia/Dawn GPU surface.
- **Mix** dry/wet, **Size** reverb length (0.05–4 s), **Gain** output trim, **Bypass**.

## Requirements

macOS on Apple Silicon (arm64).

## Feedback

It's a preview — expect rough edges. Issues and notes welcome.
