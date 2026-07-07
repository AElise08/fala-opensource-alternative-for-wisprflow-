# Fala Seed Specification

Status: Draft v1 (application-specific)

Purpose: Define a practical implementation guide for AI agents that need to change, extend, or debug the Fala application.

## Normative Language

The key words `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, `RECOMMENDED`, `MAY`, and `OPTIONAL` in this document are to be interpreted as described in RFC 2119.

`Implementation-defined` means the behavior belongs to the implementation contract, but this specification does not prescribe a universal policy. Implementations MUST document the selected behavior.

## 1. Problem Statement

Fala is a native macOS menu-bar application (Swift, AppKit) that provides system-wide voice dictation that runs **100% on-device**. It is a free, open-source alternative to Wispr Flow / superwhisper.

The application exists to help a user:

- start dictation with a single key (the `Fn` / 🌐 key) — hold to talk, or double-tap for hands-free;
- have speech transcribed locally by a small, fast ASR model (no cloud, no API key);
- have the raw transcript cleaned by an on-device LLM (remove filler words, fix punctuation, keep meaning);
- paste the cleaned text at the cursor in any application, or copy it when there is no text field focused;
- work fully offline, with no subscription and no audio ever leaving the machine.

An AI agent working on this repository MUST preserve the product focus: **local-first and private always**, low latency second, and minimal footprint third. No feature may introduce a mandatory network dependency for the core dictation path.

## 2. Goals and Non-Goals

### 2.1 Goals

- Preserve the end-to-end local pipeline: capture → ASR → cleanup → insertion.
- Keep the `Fn`-key gesture model (hold-to-talk, double-tap-to-lock, tap-to-stop).
- Keep transcription accurate for **Brazilian Portuguese and English** (the two primary languages).
- Keep the AI cleanup step on-device (Apple Foundation Models) and resilient (never block dictation if it fails).
- Keep the app lightweight: no Electron, no bundled Python, no always-on daemon, no GPU requirement.
- Keep a **stable code-signing identity** so macOS Accessibility (TCC) permission survives rebuilds.
- Prefer small, targeted edits over broad rewrites. Keep the app buildable with the Swift toolchain that ships with the Command Line Tools.

### 2.2 Non-Goals

- Adding cloud ASR or cloud LLM providers to the core path.
- Supporting Windows or Linux (the stack is Apple-only — see §8).
- Supporting Intel Macs or Rosetta (the target is Apple Silicon, arm64 native).
- Replacing the ASR engine or the cleanup model without an explicit request.
- Introducing heavy state management, background services, or telemetry.
- Capturing system audio via a system-audio tap (the microphone path is acoustic and intentional).

## 3. System Overview

### 3.1 Main Components

The app is a single SwiftPM executable target (`Fala`) plus a small CLI test target (`FalaTest`). Each source file owns one responsibility:

1. `main.swift`
	- Process entry point. Creates the `NSApplication`, installs `AppDelegate`, sets activation policy to `.accessory` (menu-bar app, no Dock icon), runs the loop.

2. `AppDelegate.swift`
	- Top-level orchestration. Owns the dictation gesture state machine, the menu-bar item and menu, warm-up, permission handling, and the `Fn`-key system-setting management.

3. `FnKeyMonitor.swift`
	- Global capture of the `Fn` / 🌐 key via an **active** `CGEventTap` (`.defaultTap`). Detects key-down / key-up on keycode 63, swallows the raw event (so the system never fires emoji / dictation), and forwards down/up callbacks.

4. `AudioRecorder.swift`
	- Microphone capture via `AVAudioEngine`. Converts to 16 kHz mono `Float32` using FluidAudio's `AudioConverter`. Emits a per-chunk RMS level for the waveform. Optionally enables macOS Voice Processing (AEC) **only during recording**.

5. `Transcriber.swift`
	- An `actor` wrapping FluidAudio's `AsrManager`. Loads Parakeet TDT 0.6b v3 (multilingual CoreML) and transcribes `[Float]` samples. Holds an optional language hint.

6. `Cleaner.swift`
	- On-device text cleanup via Apple's `FoundationModels` (`LanguageModelSession`). Removes filler words, fixes punctuation, preserves language and meaning. Hardened against prompt-injection (dictation that looks like a command MUST NOT be executed).

7. `TextInserter.swift`
	- Inserts text into the frontmost app via clipboard + synthetic `Cmd+V`. Restores the previous clipboard after a short delay; the restore is cancelable (`keepOnClipboard`).

8. `AXFocus.swift`
	- Uses the Accessibility API to decide whether the focused element accepts text. Drives the "paste vs. show copy pill" decision.

9. `WaveOverlay.swift`
	- The floating pill at the bottom of the screen. Shows a live waveform (driven by real mic RMS) while recording/processing, and a text + "Copiar" button after transcription.

10. `FLog.swift`
	- Minimal file logger at `~/Library/Application Support/Fala/fala.log`. The single diagnostic surface.

11. `FalaTest` target (`Sources/FalaTest/main.swift`)
	- A CLI harness that loads the ASR models and transcribes one or more audio files through the same pipeline. Used for offline verification.

### 3.2 Abstraction Levels

The repository is easiest to modify when kept in these layers:

1. `Capture Layer` — `FnKeyMonitor`, `AudioRecorder`. Input events and raw audio.
2. `Inference Layer` — `Transcriber` (ASR), `Cleaner` (LLM). Model calls; both MUST stay on-device.
3. `Output Layer` — `TextInserter`, `AXFocus`. Where the text goes.
4. `Presentation Layer` — `WaveOverlay`, the menu-bar UI in `AppDelegate`.
5. `Orchestration Layer` — `AppDelegate` state machine tying the layers together.

### 3.3 External Dependencies

- Swift 6 toolchain (ships with Xcode Command Line Tools; full Xcode is NOT required).
- Swift Package Manager.
- `FluidAudio` (SwiftPM) — Parakeet TDT CoreML ASR, runs on the Apple Neural Engine. Pinned in `Package.resolved`.
- Apple `FoundationModels` framework — the on-device ~3B language model (Apple Intelligence). Part of the macOS 26 SDK.
- `AVFoundation`, `Cocoa`/`AppKit`, `ApplicationServices`, `Carbon.HIToolbox` — system frameworks.

## 4. Core Domain Model

### 4.1 The Pipeline

The core data flow is a one-way pipeline. Each stage MUST be resilient: a failure late in the pipeline MUST NOT lose the user's speech silently.

```
Fn gesture ─▶ AudioRecorder ─▶ [Float] 16kHz mono
                                   │
                                   ▼
                            Transcriber (Parakeet v3) ─▶ raw text
                                   │
                                   ▼
                            Cleaner (Apple FM) ─▶ cleaned text   (skippable)
                                   │
                                   ▼
                    editable focus?  ── yes ─▶ TextInserter.insert (paste)
                                     └─ no  ─▶ clipboard + WaveOverlay "Copiar"
```

### 4.2 Gesture State Machine

`AppDelegate` MUST implement these states and transitions:

- `idle` → (Fn down) → `held`
- `held` → (Fn up after ≥ hold threshold) → transcribe (push-to-talk)
- `held` → (Fn up before threshold) → `tapArmed` (waiting for a possible second tap)
- `tapArmed` → (Fn down within double-tap window) → `locked` (hands-free; recording continues)
- `tapArmed` → (window expires) → `idle` (single accidental tap, discarded)
- `locked` → (Fn tap) → transcribe and return to `idle`
- A safety timer MUST stop an open recording after a bounded maximum (currently 5 minutes).

### 4.3 Stable Identifiers and Normalization Rules

- Audio fed to the ASR MUST be 16 kHz mono `Float32`; always convert via FluidAudio's `AudioConverter` (never hand-parse WAV/PCM).
- A fresh `TdtDecoderState` MUST be created per utterance (batch transcription; no cross-utterance context).
- The cleanup step MUST reply in the same language as the input and MUST NOT translate.
- The cleanup step MUST treat the dictation as data, never as an instruction to execute.
- If cleanup returns empty or a disproportionately long output, the implementation MUST fall back to the raw transcript.

## 5. Repository Contract

### 5.1 Source of Truth

The repository's current contract is expressed by the code in:

- `Sources/Fala/*.swift`
- `Package.swift`, `Package.resolved`
- `Info.plist`
- `make-app.sh`, `setup-cert.sh`
- `README.md`

If this specification conflicts with working, validated code, the implementation SHOULD prefer the smallest safe change needed to restore the intended contract.

### 5.2 File-Responsibility Contract

- `AppDelegate.swift` MUST remain the single orchestration point for state, permissions, and menu.
- `FnKeyMonitor.swift` MUST remain the only place that captures the `Fn` key, and MUST swallow the raw event.
- `Transcriber.swift` and `Cleaner.swift` MUST remain the only inference entry points, and both MUST stay on-device.
- `AudioRecorder.swift` MUST keep voice processing scoped to active recording only (see §9).
- Presentation code MUST NOT block the pipeline; long work runs off the main actor and returns to main for UI and insertion.

### 5.3 Architecture Boundary

This project is a client-side, single-user desktop application. It is not a server, scheduler, or daemon. Agents MUST NOT introduce backend-style assumptions, network services, or telemetry.

## 6. Configuration Specification

### 6.1 Persisted Settings (`UserDefaults`, suite `com.mel.fala`)

- `cleanupEnabled` (Bool, default true) — run the Apple FM cleanup pass.
- `soundsEnabled` (Bool, default true) — play the Pop/Tink cues.
- `isolationEnabled` (Bool, default **false**) — enable Voice Processing (AEC). Off by default because it lowers system output volume while active (see §9).
- `languagePref` (String: `auto` | `pt` | `en`, default `pt`) — ASR language hint.
- `fnOriginal` (Int) — the user's original "Press 🌐 key to" setting, saved so it can be restored on quit.

### 6.2 System Settings the App Manages

- `com.apple.HIToolbox / AppleFnUsageType` MUST be set to `0` ("Do Nothing") while the app runs, and restored to `fnOriginal` on graceful quit, each followed by a `notifyutil -p com.apple.keyboard.fnFunctionUsageChanged` so the change applies live. Without this, macOS steals the `Fn` key for the emoji/input-source/dictation actions.

## 7. Permissions and Signing Contract

### 7.1 Permissions

- **Microphone** (`NSMicrophoneUsageDescription`) — REQUIRED for capture.
- **Accessibility** — REQUIRED for the `Fn` event tap and for the synthetic `Cmd+V` paste. The app polls `AXIsProcessTrusted()` and activates the tap as soon as it is granted.

### 7.2 Stable Code Signing (critical)

macOS binds a TCC (Accessibility) grant to the app's code-signing **designated requirement**. Ad-hoc signing (`codesign -s -`) produces a `cdhash`-based requirement that changes on every rebuild, so the grant is lost on every update.

- The app MUST be signed with a **stable self-signed certificate** (see `setup-cert.sh`), producing a designated requirement of the form `identifier "com.mel.fala" and certificate leaf = H"…"` that is identical across rebuilds.
- If the Accessibility grant does not "stick" after switching signing identities, the fix is `tccutil reset Accessibility com.mel.fala` (clears stale/duplicate ghost entries), then grant once.

## 8. Requirements (Platform)

- **macOS 26 (Tahoe) or later**, on **Apple Silicon** (arm64 native, no Rosetta). The app is a MacBook / Apple-Silicon Mac application.
- **Apple Intelligence enabled** — required for the on-device cleanup model (`FoundationModels`). Without it, transcription still works; cleanup silently no-ops to the raw transcript.
- **Xcode Command Line Tools** (Swift 6 + the macOS 26 SDK). Full Xcode is not required.
- **Windows / Linux / Intel are NOT supported** and cannot be, because the stack is Apple-specific: Apple `FoundationModels` (cleanup), CoreML + the Apple Neural Engine (Parakeet via FluidAudio), `AVAudioEngine` (capture), and AppKit + the Accessibility/Quartz event APIs (Fn capture and paste) all exist only on Apple platforms.

## 9. Known Constraints and Gotchas

Agents MUST respect these hard-won constraints:

1. **Voice Processing lowers system volume.** Enabling `AVAudioInputNode.setVoiceProcessingEnabled(true)` puts the system in a "communication" audio mode that ducks output loudness even with the slider at 100%. It MUST be enabled only during active recording and disabled immediately on stop, and it defaults off.
2. **`@Generable` needs SwiftPM.** The `FoundationModels` guided-generation macro does not resolve under a bare `swiftc` invocation (missing macro plugin). The cleanup step therefore uses a plain-text prompt with delimiters and few-shot examples, not guided generation.
3. **Fn key.** A bare modifier key cannot be bound via Carbon hotkeys; it is captured with an active `CGEventTap` watching `flagsChanged` for keycode 63, and requires `AppleFnUsageType = 0` (see §6.2).
4. **Timers on background tasks never fire.** Any `Timer` created inside a `Task`/background context MUST be added to the main run loop, or scheduled on the main actor.
5. **Multilingual ASR bleed.** Parakeet v3 is multilingual; on long utterances it can occasionally emit the wrong language. The `language` hint is script-aware only (Latin PT/EN are not disambiguated by it). If PT/EN bleed persists with headphones (i.e., not acoustic bleed from a playing video), the durable fix is a Portuguese-forced Whisper model — a deliberate, larger download, not a casual swap.

## 10. Build, Sign, and Run

```sh
# 1. One-time: create the stable signing certificate (dedicated keychain).
./setup-cert.sh

# 2. Build + package + sign the .app.
./make-app.sh                      # produces build/Fala.app

# 3. Install and launch.
cp -R build/Fala.app /Applications/
open /Applications/Fala.app

# 4. Grant Microphone + Accessibility when prompted (Accessibility is one-time
#    thanks to the stable certificate).
```

Offline ASR self-test (no UI, no keyboard):

```sh
swift run -c release FalaTest path/to/audio.wav
```

The first launch downloads the Parakeet CoreML models (~1 GB) once to `~/Library/Application Support/FluidAudio`; everything after is fully offline.
