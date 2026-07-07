# Fala — Seed 

This repository ships a **seed**: a single specification file, [`seed.md`](seed.md), that a capable AI coding agent can use to build **Fala** from scratch.

There is **no source code here by design.** The seed is the source of truth — the implementation is meant to be *regenerated* from it, not versioned.

## What is Fala

Fala is a free, 100% local, open-source alternative to Wispr Flow for macOS. Hold the **Fn** (🌐) key, speak, release: your words are transcribed on-device (Parakeet v3 on the Apple Neural Engine), cleaned up by Apple's on-device Foundation Model (fillers removed, punctuation fixed), and pasted at your cursor. No cloud, no subscription, no audio ever leaves your Mac.

## How to use this seed

Give [`seed.md`](seed.md) to your AI coding agent and ask it to implement Fala. The seed defines the architecture, the pipeline, the file-responsibility contract, the platform requirements, and the hard-won gotchas (code-signing for permission persistence, the Fn-key capture, the voice-processing volume trap, and more).

## Requirements of the app it builds

- **macOS 26 (Tahoe) or later** on an **Apple Silicon Mac** (arm64 native).
- **Apple Intelligence enabled** (for the on-device cleanup model).
- **Xcode Command Line Tools** to build.

> Windows, Linux, and Intel Macs are **not** supported — the stack is Apple-only (Apple Foundation Models, CoreML / Apple Neural Engine, AppKit). See `seed.md` §8.

## License

MIT.
