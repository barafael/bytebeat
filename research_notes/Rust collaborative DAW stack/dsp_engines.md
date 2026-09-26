# Rust DSP, synthesis, and audio-graph engines for a miniature DAW (native + WASM)

Method note: Version numbers, release dates, licenses and download counts come from the crates.io API (queried 2026-09-26). Design and feature claims come from the READMEs, changelogs, design docs, Cargo.toml files and source files in the published crate tarballs (static.crates.io) for the versions listed. "Last commit" dates come from shallow `git clone`s of the default branches on 2026-09-26. The GitHub REST API (stars, issue counts) was not reachable from this environment, so star and issue counts are missing.

## fundsp: version, graph notation, Net/Sequencer, DSP inventory, no_std/WASM, SIMD, license, activity, DAW suitability

### Takeaway
fundsp 0.23.0 (released 2026-01-07, MIT OR Apache-2.0, single maintainer) has the richest DSP vocabulary in the Rust ecosystem. It provides a compact algebraic graph notation, a runtime-editable `Net` with a real-time-safe frontend/backend split (`commit()`), a `Sequencer` for timed events, `no_std` support, and explicit SIMD through the `wide` crate. It is the best choice for the instrument and effect DSP layer. It is weaker as the whole DAW engine: it has no transport or musical clock, no looping in `Sequencer`, no feedback loops in `Net`, no real compressor (only limiters), and no audio I/O layer of its own.

### Cited Findings
- **Version and cadence:** Latest release is 0.23.0 (2026-01-07). 0.22.0 came out the same day, 0.21.0 on 2026-01-03, and 0.20.0 on 2024-10-04. Totals: 208,433 downloads, 38,573 in the last 90 days. License is MIT OR Apache-2.0. The crate uses Rust edition 2024 — [crates.io fundsp](https://crates.io/crates/fundsp/versions)
- **Maintainer activity:** The most recent commit on `master` is dated 2026-03-03 (message "Update."). The listed author is SamiPerttu only — [GitHub SamiPerttu/fundsp](https://github.com/SamiPerttu/fundsp/commits/master)
- **Changes in 0.23:** `Sequencer` can now take inputs, there is a new `convolve` opcode (FFT convolution via `fft_convolver`), and resampling has quality settings. **0.22** added `resample_fir` (FIR sinc via the `resampler` crate) and renamed the preludes to `prelude64`/`prelude32`. **0.21** added PolyBLEP oscillators (`poly_saw`, `poly_square`, `poly_pulse`), `biquad_bank()`, and a builder notation such as `sine().phase(0.0)` and `noise().seed(1)` — [CHANGES.md](https://github.com/SamiPerttu/fundsp/blob/master/CHANGES.md)
- **Changes in 0.20:** `Net` gained a family of non-panicking edit methods (`can_pipe`, `set_source`, `fade_in`, `ids`, `contains`, `NetError`/`Net::error`). A cycle now sets an error condition instead of panicking. 0.19 added `Net::crossfade` for click-free node replacement — [CHANGES.md](https://github.com/SamiPerttu/fundsp/blob/master/CHANGES.md)
- **Rewrite in 0.18:** All samples became `f32` (the 64-bit prelude keeps 64-bit internal state behind a 32-bit interface). This release added explicit SIMD in block processing via the `wide` crate and made `no_std` available by disabling the `std` feature — [CHANGES.md](https://github.com/SamiPerttu/fundsp/blob/master/CHANGES.md)
- **Two component systems:** `AudioNode` is statically dispatched, inlined and stack-allocated, with input/output arity fixed at compile time. `AudioUnit` is dynamic (object-safe, heap-allocated), with arity fixed after construction. They convert into each other via `An` and `unit` — [README](https://github.com/SamiPerttu/fundsp#readme)
- **Graph operators:** `A >> B` (pipe), `A | B` (stack), `A & B` (bus: sum with identical connectivity), `A ^ B` (branch), `A + B` / `A - B` / `A * B` (mix, difference, ring-mod), `!A` (thru), `-A` (negate). Every operator also has a function form (`pipe`, `sum`, `product`, …) with iterator variants (`pipei`, `sumi`, `sumf`, …) — [README, Graph Notation](https://github.com/SamiPerttu/fundsp#graph-notation)
- **Block processing:** Blocks are always 64 samples (`MAX_BUFFER_SIZE`). Block processing "enables explicit SIMD support". `BlockRateAdapter` calls `process` automatically. `allocate()` preallocates memory before a unit is sent to a real-time context, and the `Net` and `Sequencer` frontends call it automatically — [README](https://github.com/SamiPerttu/fundsp#readme)
- **Net frontend/backend:** `net.backend()` returns a real-time-safe backend to send to the audio thread. The frontend then edits (`replace`, `crossfade`, even `net = net >> peak_hz(...)`) and calls `net.commit()`. "Using dynamic networks incurs some overhead so it is an especially good idea to use block processing." Dynamic `Net` overhead is the `Box<dyn AudioUnit>` call; beyond that it is "roughly as efficient" as static — [README, Net section](https://github.com/SamiPerttu/fundsp#readme)
- **Backend mechanism:** The backend is a `thingbuf` MPSC channel. The frontend sends whole `Net` versions or `Setting`s, and the backend returns old `Net`/`Box<dyn AudioUnit>` objects "for deallocation back to the frontend", so nothing is freed on the audio thread — [src/realnet.rs](https://github.com/SamiPerttu/fundsp/blob/master/src/realnet.rs)
- **Sequencer:**
  - "mixes together nodes dynamically".
  - Events are added with `push(start, end, Fade, fade_in, fade_out, unit)` or `push_relative`, with times in seconds. Each call returns an `EventId` that can be used later with `edit`/`edit_relative`.
  - `ReplayMode` controls whether past events replay after a reset.
  - The sequencer splits into frontend and backend "for use as a dynamic mixer", and the backend can be placed inside a `Net`.

  Sources: [README, Sequencer](https://github.com/SamiPerttu/fundsp#sequencer); [src/sequencer.rs](https://github.com/SamiPerttu/fundsp/blob/master/src/sequencer.rs)
- **Other real-time control:** `shared()`/`var()` atomics (for example `noise() * (var(&amp) >> follow(0.1))`), the `timer` opcode for stream time, `Setting` listeners, and `snoop`/`monitor`/`meter` for visualization — [README, Multithreading and Real-Time Control](https://github.com/SamiPerttu/fundsp#more-on-multithreading-and-real-time-control)
- **Prelude opcodes in 0.23.0** (from `pub fn` definitions in the prelude):
  - Oscillators: `sine`, `saw`/`square`/`triangle` (bandlimited), `poly_saw`/`poly_square`/`poly_pulse` (PolyBLEP), `dsf_saw`/`dsf_square`, `soft_saw`, `hammond`, `organ`, `pulse`, `ramp`, `pluck` (Karplus-Strong), `wavetable`.
  - Noise: `noise`/`white`/`pink`/`brown`/`mls`, plus `lorenz`/`rossler`.
  - Samples and resampling: `playwave`/`playwave_at` (sample playback), `resample`/`resample_fir`.
  - Filters:
    - Standard biquads: `lowpass`/`highpass`/`bandpass`/`notch`/`peak`/`bell`/`lowshelf`/`highshelf` (with `_hz`/`_q`), `butterpass`, `lowrez`/`bandrez`.
    - Moog ladder: `moog`/`moog_hz`/`moog_q`.
    - Nonlinear variants: `dlowpass`/`dhighpass`/`dbell`/`dresonator` ("dirty") and `flowpass`/`fhighpass`/`fbell`/`fresonator` (feedback biquads).
    - Other: `resonator`, `allpass`, `allpole`, `lowpole`/`highpole`, `dcblock`, `pinkpass`, `morph`, `biquad`, `biquad_bank`, `fir`.
  - Effects: `reverb_stereo`, `reverb2_stereo`, `reverb3_stereo`, `reverb4_stereo`, `fdn`/`fdn2`, `chorus`, `flanger`, `phaser`, `delay`, `tap`/`tap_linear`/`multitap`, `feedback`/`feedback2`, `limiter`/`limiter_stereo`, `shape`/`clip` (waveshaping distortion with `Tanh`, `Clip`, `Adaptive` shapes), `oversample`, `convolve`, `pan`/`panner`/`rotate`.
  - Envelopes and utilities: `adsr_live`, `envelope`/`lfo`, `follow`/`afollow`, `declick`, `resynth` (FFT).

  Source: [src/prelude.rs](https://github.com/SamiPerttu/fundsp/blob/master/src/prelude.rs)
- **Drum synthesis:** The `sound` module ships drum and sound presets, including `bassdrum`, `snaredrum`, `cymbal`, `risset_glissando` and `pebbles` — [src/sound.rs](https://github.com/SamiPerttu/fundsp/blob/master/src/sound.rs)
- **Maintainer's FUTURE list** (not yet done):
  - Engine gaps: "Compressor without lookahead", "Support feedback loops in `Net`", "Looping in `Sequencer`", "Real-time safe sound server that uses `cpal`", "Time stretching / pitch shifting algorithm", making `Granular` real-time safe.
  - Quality gaps: "Improve basic effects implemented in graph notation such as `chorus`, `flanger` and `phaser`" and "Improve or replace the drum sounds".

  Source: [FUTURE.md](https://github.com/SamiPerttu/fundsp/blob/master/FUTURE.md)
- **no_std:** Disable default features (`std`, `files`, `fft`). `alloc` is still required. File I/O (Symphonia) and the convolution engine are unavailable in `no_std` — [README, no_std Support](https://github.com/SamiPerttu/fundsp#no_std-support)
- **Dependencies:** `wide` 1.1.1 (default-features off), `thingbuf`, `microfft`, `resampler` (no_std), `libm`, `hashbrown`, `funutd`, `numeric-array`; optional `symphonia` 0.5.5 and `fft-convolver` 0.3.0. There is no `wasm32`-specific code. Denormal flushing (in `denormal.rs`) is gated to `x86`/`x86_64` only — [Cargo.toml / src/denormal.rs](https://github.com/SamiPerttu/fundsp/blob/master/src/denormal.rs)
- **Ecosystem built on fundsp:**
  - `bevy_fundsp` 0.4.0 (last updated 2023-08-17, stale) and `bevy_procedural_audio` 0.5.0 (2025-05-16).
  - `midi_fundsp` 0.8.0 (2026-05-04, live MIDI synths).
  - `cochlea-synth` 0.7.0 (2026-08-11, "Instrument trait over fundsp, preset library").
  - `quiver-dsp` 0.3.3 (2026-08-12).
  - `insta-fun` 2.4.1 (snapshot testing of fundsp units).
  - `knyst` advertises fundsp interop.

  Source: [crates.io search "fundsp"](https://crates.io/search?q=fundsp)

### Inferences
- fundsp should compile to `wasm32-unknown-unknown` with `default-features = false`, plus `std` if wanted, but without `files`. The only platform-specific code is x86 denormal handling, and all dependencies are pure Rust. No official WASM example or CI target was found, so a spike to confirm this is prudent.
- SIMD on WASM: `wide` has `simd128` code paths (see the WASM SIMD section below). fundsp's block processing should therefore vectorize in the browser if built with `-C target-feature=+simd128`. Without the flag it falls back to scalar.
- **As a DAW backbone,** fundsp is excellent for instrument voices (drum synths, subtractive synths), per-track effect chains, and bytebeat post-processing. `Net` + `commit()` fits "user edits track chain → hot-swap graph". It lacks engine-level concerns, so a host layer is needed: device I/O (cpal), a sample-accurate transport/step clock, a mixer bus model, compressor/sidechain, and sample streaming. `Sequencer` timing is in seconds, with no tempo, looping or musical clock. A thin custom scheduler that triggers events at sample offsets inside 64-sample blocks is still needed.
- The bus factor is 1 (a single maintainer). Release bursts are irregular (a 15-month gap between 0.20 and 0.21), and breaking renames are common between 0.x versions.

### Gaps
- No benchmark numbers from the fundsp author were found (the README makes only qualitative efficiency claims).
- GitHub stars and open-issue count were unavailable (API blocked).
- No verified public demo of fundsp running inside an AudioWorklet was found. Searches returned generic Rust/WASM AudioWorklet examples only.
- Whether `Sequencer` start times are resolved at sample accuracy inside a block, or only at block granularity, was not confirmed from source.

## Firewheel and bevy_seedling: design, node model, scheduling, WASM, maturity

### Takeaway
Firewheel 0.14.0 (2026-09-05, MIT OR Apache-2.0, by Billy Messenger/BillyDM) is an actively developed, real-time-correct, DAG audio-graph engine with backends for desktop, mobile and WASM (including a multi-threaded AudioWorklet backend). It has seconds, sample and musical-transport clocks and scheduled events. It explicitly says it is *not* a DAW engine, ships few effects, and is being folded into Bevy. It is a strong candidate as the host/graph/transport layer, with fundsp-style DSP inside custom nodes.

### Cited Findings
- **Releases:** firewheel 0.14.0 (2026-09-05), 0.13.0 (2026-08-28), 0.12.1 (2026-07-16), 0.12.0 (2026-07-09). Created 2022-10-23. 52,406 downloads total, 15,426 in the last 90 days. The repository is now github.com/BillyDM/firewheel (also mirrored on Codeberg) — [crates.io firewheel](https://crates.io/crates/firewheel/versions); last commit 2026-09-05 "bump version to 0.14" — [GitHub BillyDM/firewheel](https://github.com/BillyDM/firewheel)
- **Future direction:** "Firewheel is currently planned to be upstreamed into the Bevy game engine where it will become the default audio engine". The core "will still be available to use outside of Bevy without any other Bevy dependencies (except for the very lightweight bevy_platform dependency)" — [README](https://github.com/BillyDM/firewheel#readme)
- **Key features:**
  - Modular backends (Windows, Mac, Linux, Android, iOS, WebAssembly).
  - "any directed, acyclic graph with support for both one-to-many and many-to-one connections".
  - A custom node API and an "optional data-driven parameter API that is friendly to ECS's".
  - Silence optimizations and Symphonium file loading.
  - Stream fault tolerance, "Properly respects realtime constraints (no mutexes!)", and `no_std` compatibility.

  Source: [README](https://github.com/BillyDM/firewheel#readme)
- **Non-goal: DAW engine.** "it does NOT aim to be a complete DAW … the needs of game audio engines and DAW audio engines are in conflict". Per the design doc, a DAW ties state to the transport and may discard conflicting parameter events, while Firewheel guarantees user parameter events are never discarded. Other non-goals:
  - MIDI and parameter events at graph level (both now partly possible via `ProcStore`).
  - Built-in synthesizer instruments.
  - Multi-threaded graph processing.
  - VST/VST3/LV2/AU hosting.

  Sources: [README](https://github.com/BillyDM/firewheel#readme); [DESIGN_DOC.md](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)
- **Goal checklist:** Unchecked "later goals" as of 0.14: echo, HRTF, equalizer, compressor, CLAP hosting, C bindings. Done: sampler sequencing, Doppler pitch shifting on the sampler, delay compensation, convolution, and lowpass/highpass/bandpass filters — [DESIGN_DOC.md](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)
- **Built-in nodes in firewheel-nodes 0.14.0:** `beep_test`, `convolution`, `delay_compensation`, `fast_filters` (lowpass/highpass/bandpass), `freeverb`, `mix`, `noise_generator` (white, pink), `peak_meter`, `sampler`, `spatial_basic`, `stereo_to_mono`, `svf`, `triple_buffer`, `volume`, `volume_pan` — [crates.io firewheel-nodes](https://crates.io/crates/firewheel-nodes)
- **Cargo features:** `musical_transport`, `scheduled_events`, `midi_events`, `node_profiling`, `wasm-bindgen`, `rtaudio`, `cpal`, `unsafe_flush_denormals_to_zero`, `serde`, and per-node flags (`freeverb_node`, `svf_node`, `convolution_node`, `sampler_node`, …) — [firewheel Cargo.toml on crates.io](https://crates.io/crates/firewheel/0.14.0)
- **Engine lifecycle:**
  - The context compiles the graph into a schedule and sends it to the executor over a real-time-safe channel.
  - The user calls `update()` periodically; this flushes queued events as a group so that same-cycle events land in the same process cycle.
  - Any graph change is recompiled into a new schedule.

  Source: [DESIGN_DOC.md, Engine Lifecycle](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)
- **Clocks:**
  - The seconds clock is read from the OS audio API where possible and accounts for underflows.
  - The sample clock counts processed samples and does not account for underflows.
  - The musical clock (`MusicalTransport`) is started, paused and stopped manually and counts beats as `f64`.
  - Events are scheduled with `EventInstant::AtClockSeconds` or at a sample time.

  Sources: [DESIGN_DOC.md, Clocks and Events](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md); [firewheel-core clock.rs](https://github.com/BillyDM/firewheel/blob/main/crates/firewheel-core/src/clock.rs)
- **Silence optimization:** Each buffer carries a silence flag (`ProcInfo::in_silence_mask`), and nodes can skip processing — [DESIGN_DOC.md](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)
- **WASM design rules:** No C/system-library dependencies (so CLAP hosting is disabled on WASM), no file I/O, no spawned threads, no blocking — [DESIGN_DOC.md, WebAssembly Considerations](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)
- **firewheel-web-audio 0.5.0** (2026-08-09), "A multi-threaded wasm32-unknown-unknown Web Audio backend":
  - Stereo inputs and outputs only.
  - Requires a nightly toolchain plus `build-std`, and the `+atomics,+bulk-memory,+mutable-globals` target features.
  - The page must be served over HTTPS with `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp` (or `credentialless`, which "may not work on Safari").

  Source: [crates.io firewheel-web-audio](https://crates.io/crates/firewheel-web-audio)
- **bevy_seedling** 0.8.0 (2026-08-09), "A sprouting integration of the Firewheel audio engine": 29,027 downloads. Its features include `effects = ["firewheel/all_nodes"]` and optional HRTF via `firewheel-ircam-hrtf` — [crates.io bevy_seedling](https://crates.io/crates/bevy_seedling)

### Inferences
- **As host layer:** Firewheel is the most complete "engine shell" in Rust today: graph compilation, real-time-safe messaging, a musical transport, scheduled events, a sampler, device fault tolerance, and a WASM AudioWorklet backend. A mini-DAW could use it as the host and implement synths, drum machines, the bytebeat voice and effects as custom Firewheel nodes wrapping fundsp `AudioUnit`s.
- **Risks:**
  - Breaking 0.x releases arrive roughly monthly (0.12 → 0.14 in two months).
  - The project is moving into Bevy, which may reshape its public API.
  - The DAW non-goal: transport-authoritative automation would be the app's own responsibility.
  - The effect set is small: no compressor, delay/echo or EQ yet.
- The multi-threaded WASM backend needs nightly Rust and cross-origin isolation headers. This is a deployment constraint for a collaborative web app (for example, third-party embeds and some CDNs).

### Gaps
- No independent benchmarks of Firewheel graph overhead were found.
- It was not verified whether scheduled events are applied at exact sample offsets inside a process block. The clock API supports sample times, but intra-block splitting was not read in source.
- BillyDM's blog (for Meadowlark/Firewheel status posts) was blocked by the egress proxy. Only post titles were visible in search, for example "Why I'm Taking a Break from Meadowlark" ([search listing](https://billydm.github.io/blog/why-im-taking-a-break-from-meadowlark/)).

## knyst: design and status

### Takeaway
knyst 0.5.1 (2024-11-25, MIT OR Apache-2.0) has an attractive design: a dynamic graph with sample-accurate scheduling, nested graphs, feedback edges and fundsp interop. It is effectively dormant: there have been no commits since the 0.5.1 release (22 months), recent downloads are around 131, and the README warns the API "can vary wildly". It is not recommended as a foundation.

### Cited Findings
- **Status:** Latest version 0.5.1 (2024-11-25), preceded by 0.5.0 (2024-01-18). 13,837 downloads total, 131 in the last 90 days — [crates.io knyst](https://crates.io/crates/knyst/versions). The last commit on `main` is 2024-11-25 "Release 0.5.1" — [GitHub ErikNatanael/knyst](https://github.com/ErikNatanael/knyst)
- **Scope:** "Knyst is a real time audio synthesis framework … main target use case is desktop multi-threaded environments … Embedded platforms are currently not supported". It also warns: "Knyst is not stable. Knyst's API is still developing and can vary wildly between releases." — [README](https://github.com/ErikNatanael/knyst#readme)
- **Features:**
  - "real time changes to the audio graph via an async compatible interface" and "sample accurate node and parameter change scheduling".
  - Interop with dasp and fundsp.
  - Graphs as nodes, editable at any depth.
  - "feedback connections to get a 1 block delayed output".
  - Inner graphs with different block sizes.
  - `Gen` trait / `#[impl_gen]` macro for custom nodes.

  Source: [README](https://github.com/ErikNatanael/knyst#readme)
- **Roadmap (unrealized):** Automatic parameter interpolation, GUI generation, "musical time scheduling", parallel processing, automatic SRC, and `no_std` — [README, Roadmap](https://github.com/ErikNatanael/knyst#readme)
- **Default features:** `cpal`, `jack`, `assert_no_alloc` (debug detection of audio-thread allocation) — [knyst Cargo.toml](https://crates.io/crates/knyst/0.5.1)

### Inferences
- knyst's design ideas (sample-accurate scheduling, nested graphs, 1-block feedback) are worth copying. Its maintenance state and desktop/multithread focus make it a poor bet for a WASM-first project.

### Gaps
- WASM compatibility of knyst was not verified. The README does not mention WASM, and the default features include `jack`.
- No community comparison threads (Reddit or the Rust users forum) comparing fundsp, knyst and Firewheel were found via search. Search returned only repository pages and a link list ([WeirdConstructor's Rust Audio Link Collection](https://gist.github.com/WeirdConstructor/276f7e0555b2dbe83614268b59a7a998)).

## glicol and glicol_synth: engine, graph model, WASM/AudioWorklet deployment, library usability

### Takeaway
glicol_synth 0.13.5 (2024-04-23, MIT) is a usable, compact graph engine: a fork of `dasp_graph` with const-generic block size, per-node messages and many music nodes. Glicol's web deployment (Rust→WASM inside an `AudioWorkletProcessor`, with a SharedArrayBuffer ring buffer) is a proven reference architecture. Development has slowed (no release since April 2024, last commit January 2025), and it depends on old unmaintained crates (`fasteval`, `petgraph` 0.6).

### Cited Findings
- **Status:** glicol_synth and glicol 0.13.5 (2024-04-23), preceded by 0.13.4 (2024-02-18). crates.io lists the license as "non-standard" (license-file). The LICENSE file is the MIT License, © 2020-present Qichao Lan — [crates.io glicol_synth](https://crates.io/crates/glicol_synth); [LICENSE](https://github.com/chaosprint/glicol/blob/main/LICENSE). The last commit on `main` is 2025-01-23 (a README typo fix) — [GitHub chaosprint/glicol](https://github.com/chaosprint/glicol)
- **Origins:** "`glicol_synth` begins with a fork of the `dasp_graph` crate". Changes include:
  - const generics for a customisable buffer size;
  - input map by node id;
  - real-time messages to nodes;
  - a higher-level `AudioContext` API (for example `AudioContextBuilder::<16>::new().sr(44100)`, `context.connect(...)`, `context.send_msg(node, Message::SetToNumber(0, 100.))`, `next_block()`).

  Source: [glicol_synth README](https://github.com/chaosprint/glicol/tree/main/rs/synth)
- **Node inventory in 0.13.5:**
  - Oscillators: sin, saw, squ, tri.
  - Filters: onepole, rlpf, rhpf, apfmsgain.
  - Effects: plate, reverb, pan, balance.
  - Delays: delayms, delayn.
  - Envelopes: adsr, envperc.
  - Sampling: sampler, psampler.
  - Sequencing: seq, choose, speed, arrange.
  - Signals: const, imp, noise, phasor, points.
  - Compound drum/synth nodes: bd, hh, sn, sawsynth, squsynth, trisynth.
  - Dynamic nodes: `meta` (Rhai scripts) and `eval` (per-sample expression via `fasteval` compiled instructions).

  Source: [glicol_synth src/node](https://github.com/chaosprint/glicol/tree/main/rs/synth/src/node)
- **Dependencies:** `dasp_*` 0.11, `petgraph` 0.6, `fasteval` 0.2.4, `rhai` 1.12 (behind `node-dynamic`). An `expr` node using `evalexpr` exists in source but is commented out — [glicol_synth Cargo.toml / dynamic/mod.rs](https://github.com/chaosprint/glicol/tree/main/rs/synth)
- **Web deployment:**
  - `glicol-engine.js` defines `class GlicolEngine extends AudioWorkletProcessor`.
  - The main thread calls `audioWorklet.addModule("glicol-engine.js")` and creates an `AudioWorkletNode`.
  - The code references padenot's `ringbuf.js` (a SharedArrayBuffer ring buffer).
  - The WASM crate (`rs/wasm`, `glicol-wasm`) uses `wasm-bindgen` with features `use-samples`, `use-meta`.

  Sources: [glicol js/src](https://github.com/chaosprint/glicol/tree/main/js/src); [rs/wasm/Cargo.toml](https://github.com/chaosprint/glicol/tree/main/rs/wasm)

### Inferences
- glicol_synth is usable as a library, and its `eval` node is effectively a per-sample expression instrument (bytebeat-like). The coarse node set (no Moog filter, compressor, chorus or distortion) and slowing maintenance argue for using it as a reference design rather than as a dependency.
- The architecture — Rust DSP compiled to WASM, running in an AudioWorkletProcessor, with control via `postMessage` or a SAB ring buffer — is the right template for the browser build regardless of which engine is chosen.

### Gaps
- No performance measurements of glicol_synth were found.
- Whether glicol's `dasp_graph`-derived processing allocates on the audio thread during graph edits was not checked.

## dasp: sample/frame/signal/interpolation, maintenance status

### Takeaway
dasp 0.11.0 has not had a release since 2020-05-29. The repo still gets occasional commits (the latest is 2025-09-09) and it remains a heavily used transitive dependency. Treat it as stable-but-frozen utility code (sample-format conversion, `Frame`, interpolation), not as an engine.

### Cited Findings
- **Release history:** `dasp` 0.11.0 is the only listed version (2020-05-29). `dasp_signal`/`dasp_interpolate` are also 0.11.0 (2020-05-29), and `dasp_graph` 0.11.0 is from 2020-07-16. License is MIT OR Apache-2.0. `dasp` shows 5.12M downloads total and about 1.14M in the last 90 days (dasp_signal/interpolate about 6.5M total each) — [crates.io dasp](https://crates.io/crates/dasp); [crates.io dasp_graph](https://crates.io/crates/dasp_graph)
- **Last commit:** 2025-09-09, "fix: improve sqrt implementations for f32 and f64 without std (#192)" — [GitHub RustAudio/dasp](https://github.com/RustAudio/dasp)
- **no_std:** The README's `no_std` section notes some crates require nightly to build in a `no_std` context — [dasp README](https://github.com/RustAudio/dasp#no_std)
- **Downstream use:** glicol_synth depends on `dasp_interpolate`, `dasp_ring_buffer`, `dasp_signal`, `dasp_slice` 0.11 — [glicol_synth Cargo.toml](https://crates.io/crates/glicol_synth/0.13.5)

### Inferences
- The high recent download counts most likely come from transitive use (for example, sample-format conversion in the playback stack), not new direct adoption. The crate is safe to use for sample-type conversions but should not be relied on for new features.

### Gaps
- Could not confirm via the GitHub API whether RustAudio considers dasp officially maintained or archived.

## web-audio-api-rs: a native engine mirroring the browser API

### Takeaway
web-audio-api 1.7.0 (2026-08-08, MIT) is an actively maintained, spec-tracking pure-Rust implementation of the Web Audio API. It includes a Rust `AudioWorkletProcessor` trait for custom nodes. This makes "same graph code natively and in the browser" possible *at the API-shape level*: native Rust uses this crate, and the browser uses the real Web Audio API. Its node set is Web Audio's (BiquadFilter, DynamicsCompressor, Convolver, Delay, WaveShaper, Oscillator, AudioBufferSource), which covers a mini-DAW's basic effects. Its WASM path is marked experimental, and the default 128-frame render size can crackle on ALSA.

### Cited Findings
- **Releases:** 1.7.0 (2026-08-08), 1.6.0 (2026-06-20), 1.5.0 (2026-05-23), 1.4.0 (2026-05-18). License MIT. 165,970 downloads total, 22,797 in the last 90 days — [crates.io web-audio-api](https://crates.io/crates/web-audio-api/versions). Last commit 2026-09-05 (PR #645) — [GitHub orottier/web-audio-api-rs](https://github.com/orottier/web-audio-api-rs)
- **Purpose:** "A pure Rust implementation of the Web Audio API, for use in non-browser contexts". Deviations from the spec: snake_case names, getters/setters instead of attributes, namespacing, and inheritance modelled with traits — [README](https://github.com/orottier/web-audio-api-rs#readme)
- **Spec conformance:** NodeJS bindings (`ircam-ismm/node-web-audio-api`) let the project run the official WPT webaudio test harness and track a compliance score — [README](https://github.com/orottier/web-audio-api-rs#readme)
- **Backends:** `cpal` (default: ALSA/WASAPI/CoreAudio/Oboe), `cpal-jack`, `cpal-pipewire`, `cpal-asio`, and `cubeb` (experimental). "Using the library on Linux with the ALSA backend might lead to unexpected cranky sound with the default render size (i.e. 128 frames)". The suggested workaround is the `Playback` latency hint (1024 frames), or JACK for low latency — [README](https://github.com/orottier/web-audio-api-rs#readme)
- **Browser:** "We can go full circle and pipe the Rust WebAudio output back into the browser via cpal's wasm-bindgen backend … Warning: experimental!" — [README, Targeting the browser](https://github.com/orottier/web-audio-api-rs#targeting-the-browser)
- **Node modules in 1.7.0:**
  - Sources: `audio_buffer_source`, `constant_source`, `oscillator`, `media_element_source`, `media_stream_source`/`destination`.
  - Processing: `biquad_filter`, `iir_filter`, `convolver`, `delay`, `dynamics_compressor`, `gain`, `waveshaper`.
  - Routing, spatial and analysis: `panner`, `stereo_panner`, `channel_merger`/`splitter`, `analyser`, `script_processor`.
  - Engine: render quantum `RENDER_QUANTUM_SIZE = 128`, plus an `AudioWorkletNode` with a Rust `AudioWorkletProcessor` trait (constructor on render thread, message-port handler).

  Source: [src/node, src/worklet.rs](https://github.com/orottier/web-audio-api-rs/tree/main/src)

### Inferences
- **Pros:**
  - Mature spec semantics: AudioParam automation (`setValueAtTime`, ramps) and sample-accurate `start(when)` on sources — standard Web Audio features.
  - A compressor exists.
  - Browser parity is possible by writing an abstraction over "Web Audio API" with two implementations.
- **Cons:**
  - Web Audio's built-in node set is limited: no Moog/ladder filter, chorus, phaser or bytebeat, so custom synths must be AudioWorklet processors anyway.
  - Graph mutation is the Web Audio "connect/disconnect + fire-and-forget source nodes" model, not a compiled DAG.
  - On the web you would use the *browser's* Web Audio rather than this crate, which means two implementations whose behaviour must be kept in sync.
- It is a good fit if the product wants the browser build to rely on native browser nodes (smaller WASM, browser-optimized convolver and compressor). It is a weaker fit if the goal is bit-identical rendering on native and web.

### Gaps
- No CPU-performance comparison between web-audio-api-rs and browser engines was found.
- The current WPT compliance score was not retrieved.

## Other engines and building blocks (kira, oddio, rodio, synthrs, twang, SoundFont/SFZ, biquad, HexoDSP/synfx-dsp, dsp-chain, vult, Meadowlark)

### Takeaway
Among the "others", kira (very active, with effects, a clock system and a mixer) is the most relevant. rustysynth (MIT) is the practical SoundFont option, and biquad is a good small filter crate. HexoDSP/synfx-dsp have a rich modular node set and a JIT, but are GPL-3.0 and dormant since January 2024. oddio, twang and dsp-chain are stale. synthrs is not on crates.io, there are no sfizz Rust bindings on crates.io, and "vult" on crates.io is an unrelated finance crate.

### Cited Findings
- **kira** 0.12.5 was released 2026-09-26 (0.12.4 on 2026-08-27, 0.12.3 on 2026-08-09). MIT OR Apache-2.0, 936,112 downloads total, about 144k in the last 90 days — [crates.io kira](https://crates.io/crates/kira/versions)
  - "tweens for smoothly adjusting properties of sounds, a flexible mixer for applying effects to audio, a clock system for precisely timing audio events, and spatial audio support". Clocks tick at seconds-per-tick or ticks-per-minute and can start sounds on a given tick — [README](https://github.com/tesselode/kira#readme); [src/clock.rs](https://github.com/tesselode/kira/blob/main/src/clock.rs)
  - Built-in effects: compressor, delay, distortion, eq_filter, filter, panning_control, reverb, volume_control — [src/effect](https://github.com/tesselode/kira/tree/main/src/effect)
  - `AudioManagerSettings::internal_buffer_size` defaults to 128 — [src/manager/settings.rs](https://github.com/tesselode/kira/blob/main/src/manager/settings.rs)
  - On WASM, "Static sounds cannot be loaded from files" and "Streaming sounds are not supported because they make heavy use of threads". On `wasm32` it uses cpal with the `wasm-bindgen` feature — [README](https://github.com/tesselode/kira#readme)
- **oddio** 0.7.4 (2023-10-15). The last commit (2023-10-15) is "Bump version", so it is stale — [crates.io oddio](https://crates.io/crates/oddio); [GitHub Ralith/oddio](https://github.com/Ralith/oddio)
- **rodio** 0.22.2 (2026-03-05), MIT OR Apache-2.0. A playback/recording library with 11.8M downloads — [crates.io rodio](https://crates.io/crates/rodio)
- **awedio** 0.8.0 (2026-06-12), "low-overhead and adaptable audio playback library" — [crates.io awedio](https://crates.io/crates/awedio)
- **synthrs** is not published on crates.io (the API returns "crate `synthrs` does not exist"). On GitHub it is a "Toy synthesiser library in Rust"; its last commit (2026-06-16) is a dependabot merge — [GitHub gyng/synthrs](https://github.com/gyng/synthrs)
- **twang** 0.9.0 (2022-10-23), Apache-2.0 OR BSL-1.0 OR MIT, "pure Rust advanced audio synthesis". No release in about 4 years — [crates.io twang](https://crates.io/crates/twang)
- **SoundFont options:**
  - rustysynth 1.3.6 (2025-08-10, MIT, "SoundFont MIDI synthesizer written in pure Rust"); last commit 2026-05-17 — [crates.io rustysynth](https://crates.io/crates/rustysynth); [GitHub sinshu/rustysynth](https://github.com/sinshu/rustysynth)
  - Fork `rustysynth-ext` 1.4.0 (2026-09-09, adds sf3) — [crates.io rustysynth-ext](https://crates.io/crates/rustysynth-ext)
  - oxisynth 0.1.0 (2025-05-25, **LGPL-2.1**) — [crates.io oxisynth](https://crates.io/crates/oxisynth)
- **SFZ:**
  - sofiza 0.3.1 (2022-09-30) is only an SFZ *parser* — [crates.io sofiza](https://crates.io/crates/sofiza)
  - No `sfizz` or `sfizz-sys` crate exists on crates.io (the API returns "does not exist"), and a crates.io search for "sfizz" returned nothing relevant — [crates.io search](https://crates.io/search?q=sfizz)
- **biquad** 0.6.0 (2026-03-22), MIT OR Apache-2.0, 132,837 downloads in the last 90 days — [crates.io biquad](https://crates.io/crates/biquad)
- **HexoDSP** 0.2.2 and **synfx-dsp** 0.5.6 were both last released 2024-01-04. Both are **GPL-3.0-or-later**; hexodsp has 15 recent downloads — [crates.io hexodsp](https://crates.io/crates/hexodsp); [crates.io synfx-dsp](https://crates.io/crates/synfx-dsp)
  - HexoDSP features: "Runtime changeable DSP graph", serialization, monitoring, and an optional "JIT compiled custom DSP code".
  - Its nodes include a sample player, bandlimited/vector-phase/FM-formant oscillators, `FVaFilt` (Moog, EDP Wasp, Korg MS20), `PVerb` (Dattorro plate reverb), `TSeq` (tracker/pattern sequencer) and `Code` (JIT node).

  Source: [HexoDSP README](https://github.com/WeirdConstructor/HexoDSP#readme)
  - The JIT is `synfx-dsp-jit` 0.6.2 (2024-01-04, GPL-3.0-or-later) — [crates.io synfx-dsp-jit](https://crates.io/crates/synfx-dsp-jit)
- **dsp-chain** 0.13.1 (2016-06-08) is abandoned — [crates.io dsp-chain](https://crates.io/crates/dsp-chain)
- **"vult"** on crates.io is "Core library for Vult Finance integrations" (0.1.0), unrelated to the Vult DSP language — [crates.io vult](https://crates.io/crates/vult)
- **Meadowlark:** `dropseed` ("The DAW audio graph engine used in Meadowlark") is only a 0.0.0 placeholder from 2022-06-14, GPL-3.0 — [crates.io dropseed](https://crates.io/crates/dropseed)
- **creek** 1.2.3 (2025-09-22, MIT OR Apache-2.0), "Realtime-safe disk streaming to/from audio files" (Meadowlark org, Codeberg) — [crates.io creek](https://crates.io/crates/creek)
- **cpal** 0.18.2 (2026-08-16, Apache-2.0) offers WASM backends:
  - `wasm-bindgen` (Web Audio API backend, stable Rust 1.85).
  - `audioworklet` (nightly; "running audio on a dedicated thread"; requires `+atomics,+bulk-memory,+mutable-globals`, `-Zbuild-std` and cross-origin headers for SharedArrayBuffer).

  Source: [cpal README](https://github.com/RustAudio/cpal#readme)

### Inferences
- **kira** is optimized for game-style fire-and-forget sounds and tweens. Its clock-tick start times and effect set (compressor, delay, distortion, EQ, reverb) are useful, but it does not expose an arbitrary DSP graph, and WASM sample loading must come from memory.
- **HexoDSP/synfx-dsp:** GPL-3.0 makes them unsuitable if the DAW is to be permissively licensed. They are useful as algorithm references (Dattorro plate, virtual-analog filters, tracker sequencer).
- **rustysynth** is the practical permissive General MIDI / SoundFont voice for a mini-DAW. It is pure Rust, so it should compile to WASM (not verified).

### Gaps
- No Rust sfizz (SFZ player) bindings could be located on crates.io. A GitHub-only binding may exist but was not found.
- "twang" and "synthrs" WASM status was not checked, since both are toy or stale.

## Resampling and time-stretching (rubato, rubberband, signalsmith-stretch, pure-Rust stretchers)

### Takeaway
rubato 5.0.0 (2026-08-10, MIT OR Apache-2.0) is the standard real-time-safe resampler, but its hand-written SIMD covers only x86_64/aarch64, so WASM runs its scalar path. It also went through four major versions in 2026. For time-stretch there are two routes. The first is C++ wrappers: `signalsmith-stretch` (MIT, bindgen + cc) or the brand-new `rubberband` bindings (GPL-2.0+). The second is young pure-Rust crates: `timestretch` (MIT, real-time, EDM-oriented), `pitch_shift` (phase vocoder) and `wsola`. fundsp itself only has resampling; time-stretch is on its FUTURE list.

### Cited Findings
- **rubato releases:** 5.0.0 (2026-08-10), 4.0.0 (2026-07-09), 3.0.0 (2026-05-20), 2.0.0 (2026-04-01). 11.8M downloads total and about 4.5M in the last 90 days — [crates.io rubato](https://crates.io/crates/rubato/versions)
- **rubato design:**
  - Asynchronous sinc resamplers with ratios adjustable on the fly, plus synchronous FFT resamplers.
  - I/O goes through `audioadapter` traits (`audioadapter` 5.0.0, 2026-07-31).
  - "designed with real-time safety in mind, avoiding allocations during processing". Logging allocates a `String` and should be avoided in real-time use.

  Source: [rubato README](https://github.com/HEnquist/rubato#readme)
- **rubato SIMD:** "The asynchronous sinc resampler supports SIMD on x86_64 and on aarch64" (AVX, then SSE3, then Neon, detected at runtime), with a scalar fallback. The FFT resampler relies on RustFFT's SIMD — [rubato README, SIMD acceleration](https://github.com/HEnquist/rubato#simd-acceleration)
- **rubberband** (Rust bindings) 0.1.5 was released 2026-09-18; the crate was created 2026-09-09. License **GPL-2.0-or-later**. It supports Rubber Band v4.0.0, builds the library from source via `rubberband-sys` and bindgen, and requires Clang 9+ — [crates.io rubberband](https://crates.io/crates/rubberband); [README](https://github.com/jefflongo/rubberband-rs#readme)
- **signalsmith-stretch** 0.1.3 (2025-09-18, MIT) wraps the C++ Signalsmith Stretch library, with bindgen and cc build-dependencies. It has 91,301 downloads — [crates.io signalsmith-stretch](https://crates.io/crates/signalsmith-stretch); [GitHub colinmarc/signalsmith-stretch-rs](https://github.com/colinmarc/signalsmith-stretch-rs). An alternative binding is `ssstretch` 0.1.0 (2025-03-01, MIT) — [crates.io ssstretch](https://crates.io/crates/ssstretch)
- **timestretch** 0.15.0 (2026-09-02, MIT), preceded by 0.14.0 (2026-08-26) and 0.13.0 (2026-08-20). "Pure Rust audio time-stretching library optimized for electronic dance music":
  - Real-time `EngineProcessor::process` is "infallible, allocation-free, lock-free on the audio thread".
  - Pipeline delay is 12.7 ms in keylock mode.
  - The only DSP dependency is `rustfft`.

  Sources: [crates.io timestretch](https://crates.io/crates/timestretch/versions); [README](https://github.com/robmorgan/timestretch-rs#readme)
- **Other pure-Rust options:**
  - `pitch_shift` 2.1.0 (2026-04-20, MIT, phase vocoder) — [crates.io pitch_shift](https://crates.io/crates/pitch_shift)
  - `wsola` 0.1.0 (2026-07-13, "pure Rust, no C") — [crates.io search](https://crates.io/search?q=time%20stretch)
  - `rodio-wsola` 0.2.0 (2026-07-20) — [crates.io search](https://crates.io/search?q=time%20stretch)
- **fundsp:** 0.22/0.23 added `resample_fir` and quality settings. "Time stretching / pitch shifting algorithm" is still a FUTURE item — [CHANGES.md](https://github.com/SamiPerttu/fundsp/blob/master/CHANGES.md); [FUTURE.md](https://github.com/SamiPerttu/fundsp/blob/master/FUTURE.md)
- **Firewheel's sampler** supports "Doppler stretching (pitch shifting)", meaning varispeed — [DESIGN_DOC.md](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)

### Inferences
- **WASM feasibility:** The C++ wrappers (`signalsmith-stretch`, `rubberband`) need a C/C++ toolchain targeting wasm32 (for example clang with wasi-sdk, or Emscripten). Mixing them with `wasm-bindgen`'s `wasm32-unknown-unknown` target is typically painful, and Firewheel's design doc explicitly avoids C dependencies for WASM. Pure-Rust stretchers are far easier to ship in the browser.
- **License:** rubberband's GPL-2.0+ is a blocker for permissive or closed distribution unless a commercial Rubber Band licence is bought. signalsmith-stretch (MIT) is the permissive high-quality choice.
- rubato's frequent majors (four in 2026) mean the version should be pinned. In the browser it runs scalar, which is fine for offline sample-rate conversion at import time.

### Gaps
- No verified reports of `signalsmith-stretch` Rust bindings compiled to `wasm32-unknown-unknown` were found.
- No quality comparisons of timestretch, pitch_shift and wsola were found. All are young (2026) crates.

## Bytebeat: Rust crates and safe, fast evaluation of user formulas on native and WASM

### Takeaway
There is no maintained, reusable Rust bytebeat evaluation library:
- chaosprint's `bytebeat` 0.4.0 (2023) is a CLI with a tiny shunting-yard evaluator.
- balt-dev's `bytebeat-rs` uses LLVM JIT, which is not WASM-capable.

Generic evaluators have problems too:
- `evalexpr` switched to **AGPL-3.0-only** in v12 (2024-10-17), which is a license trap.
- `meval` and `fasteval` are unmaintained.

The robust design is a custom tiny language: parse the formula on the control thread into a validated AST, then compile it to either (a) a register/stack bytecode run by a small allocation-free VM in the audio thread (portable to WASM), or (b) native code via Cranelift on desktop, or a generated WASM module via `wasm-encoder` in the browser. The browser reference implementation (dollchan composer) simply uses JS `new Function` inside an AudioWorklet.

### Cited Findings
- **bytebeat (chaosprint):** `bytebeat` 0.4.0 (2023-12-18), last commit 2023-12-18. A CLI/TUI (cpal, ratatui). Example usage is `bytebeat "((t >> 10) & 42) * t" --sr 8000`, and output is computed as `(result % 256) as f32 / 255.0 * 2.0 - 1.0`. The source has `tokenize`, `infix_to_postfix` and `eval_postfix` functions returning `Option<u32>` — [crates.io bytebeat](https://crates.io/crates/bytebeat); [GitHub chaosprint/bytebeat-rs](https://github.com/chaosprint/bytebeat-rs)
- **bytebeat-rs (balt-dev):** `bytebeat-rs` 0.2.0 (2025-02-24, MIT) depends on `inkwell` + `llvm-sys`. `bytebeat-cli` 0.2.2 (2025-02-24) is "An LLVM-powered program to JIT-compile bytebeats" and needs "LLVM v180 installed" — [crates.io bytebeat-cli](https://crates.io/crates/bytebeat-cli); [GitHub balt-dev/bytebeat-rs](https://github.com/balt-dev/bytebeat-rs)
- **Expression evaluator crates:**
  - `evalexpr` 13.1.0 (2025-11-26). The license changed from MIT (through 11.x) to **AGPL-3.0-only** starting 12.0.0 (2024-10-17) — [crates.io evalexpr versions](https://crates.io/crates/evalexpr/versions)
  - `meval` 0.2.0 (2018-09-30) — [crates.io meval](https://crates.io/crates/meval)
  - `fasteval` 0.2.4 (2020-01-25, MIT; used by glicol's `eval` node) — [crates.io fasteval](https://crates.io/crates/fasteval)
  - `exmex` 0.21.0 (2026-05-23, MIT OR Apache-2.0, "fast, simple, and extendable mathematical expression evaluator") — [crates.io exmex](https://crates.io/crates/exmex)
  - `rhai` 1.26.1 (2026-09-10, MIT OR Apache-2.0, embedded scripting) — [crates.io rhai](https://crates.io/crates/rhai)
  - `tinyexpr` 0.1.1 (2016) — [crates.io tinyexpr](https://crates.io/crates/tinyexpr)
- **Code generation crates:**
  - `cranelift-jit` 0.136.1 (2026-09-24, Apache-2.0 WITH LLVM-exception) — [crates.io cranelift-jit](https://crates.io/crates/cranelift-jit)
  - `wasm-encoder` 0.259.0 (2026-09-10, Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT), "A low-level WebAssembly encoder" — [crates.io wasm-encoder](https://crates.io/crates/wasm-encoder)
- **Prior art in audio JITs:** HexoDSP's `Code` node / `synfx-dsp-jit` (GPL-3.0) JIT-compiles custom DSP code at runtime for "native speed" (Cranelift-based) — [HexoDSP README](https://github.com/WeirdConstructor/HexoDSP#readme); [crates.io synfx-dsp-jit](https://crates.io/crates/synfx-dsp-jit)
- **Glicol's evaluation approach:** The `eval` node pre-compiles an expression to `fasteval::Instruction`s and evaluates it per sample inside `process()`. The `meta` node runs Rhai scripts — [glicol_synth dynamic/eval.rs](https://github.com/chaosprint/glicol/blob/main/rs/synth/src/node/dynamic/eval.rs)
- **Browser reference (dollchan composer):** The composer's `audioProcessor extends AudioWorkletProcessor`. It compiles formulas with `new Function(...Object.getOwnPropertyNames(Math), 'int', 'window', 't', 'return 0,\n' + code)`, binding the Math functions as bare names. It keeps the old function if compilation fails — [bytebeat-composer src/audio-processor.mjs](https://github.com/SthephanShinkufag/bytebeat-composer/blob/master/src/audio-processor.mjs)
- **Formula modes** in that composer: Bytebeat (unsigned 8-bit), Signed bytebeat, Floatbeat (−1..1), and Funcbeat (code returns a function of time in seconds) — [bytebeat-composer README](https://github.com/SthephanShinkufag/bytebeat-composer/blob/master/README.md)
- **JIT inside WASM:** Wingo's "just-in-time code generation within webassembly" covers generating new WASM modules at runtime and instantiating them from the host (a JIT-in-WASM technique) — [wingolog 2022](https://www.wingolog.org/archives/2022/08/18/just-in-time-code-generation-within-webassembly)

### Inferences
- **Recommended design:**
  1. Define a restricted bytebeat grammar: integer and float ops, bitwise ops, ternary, a few math functions, `t`, and user parameters. No loops, no allocation, and bounded AST depth.
  2. Parse and validate on the UI/control thread and type-check int versus float semantics.
  3. Compile to a flat, preallocated bytecode array (stack or register VM).
  4. Send it to the audio thread through the engine's lock-free channel (for example as a fundsp `Setting`, a Firewheel event, or a `Net` node replacement with crossfade). Evaluate per sample, or vectorized over a 64-sample block.

  This path is safe (no code execution, no panics if the evaluator uses wrapping arithmetic and defines division by zero), deterministic across native and WASM, and identical in both builds.
- **Speed-ups:**
  - Native: Cranelift JIT of the same IR gives native speed (as HexoDSP demonstrates). It cannot run on `wasm32-unknown-unknown`.
  - Browser: emit a tiny WASM module with `wasm-encoder` and instantiate it via JS (`WebAssembly.Module`/`Instance`) so the engine calls it through an imported function table. Or, like dollchan, compile to JS `new Function` in a separate AudioWorklet.
  - A bytecode VM over 64-sample blocks is likely fast enough for a handful of bytebeat voices, so JIT is an optimization, not a requirement.
- **Semantic trap:** Bytebeat semantics come from C/JS. JS bitwise ops coerce to int32 and `/` is a float divide, while C uses unsigned integer division. Pick one convention explicitly, most likely the JS/dollchan convention for compatibility with shared formulas.
- **Avoid** evalexpr ≥12 (AGPL) in a permissively licensed or closed product. `fasteval` and `meval` are unmaintained. `exmex` is permissive and maintained but float-oriented (bitwise operators would need custom operator definitions — unverified).

### Gaps
- No maintained, reusable Rust crate specifically for bytebeat evaluation (VM or JIT) was found.
- Whether `exmex` supports custom integer/bitwise operators suitable for bytebeat was not verified.
- It was not verified whether `WebAssembly.Module` synchronous compilation is permitted inside `AudioWorkletGlobalScope` in all browsers (relevant for JIT-in-worklet).

## SIMD on WASM (simd128) and browser DSP performance

### Takeaway
Fixed-width WASM SIMD (simd128) is supported in all major browsers: Chrome 91, Firefox 89, Safari 16.4. Relaxed SIMD is not yet in Safari. Threads/atomics (needed for SharedArrayBuffer multithreaded audio backends) are available in Chrome 74, Firefox 79 and Safari 15.2. Rust must be built with `-C target-feature=+simd128`, and code must use `core::arch::wasm32` or a portable crate like `wide` (which fundsp uses and which has simd128 paths). Crates with hand-written x86/ARM intrinsics only (like rubato's sinc resampler) fall back to scalar in the browser.

### Cited Findings
- **Browser support** (from MDN browser-compat-data):
  - `webassembly.fixed-width-SIMD`: Chrome 91, Firefox 89, Safari 16.4 (iOS and Edge mirror these).
  - `relaxed-SIMD`: Chrome 114, Firefox 146, Safari "preview".
  - `threads-and-atomics`: Chrome 74, Firefox 79, Safari 15.2.

  Source: [MDN browser-compat-data fixed-width-SIMD.json](https://github.com/mdn/browser-compat-data/blob/main/webassembly/fixed-width-SIMD.json); [relaxed-SIMD.json](https://github.com/mdn/browser-compat-data/blob/main/webassembly/relaxed-SIMD.json); [threads-and-atomics.json](https://github.com/mdn/browser-compat-data/blob/main/webassembly/threads-and-atomics.json)
- **`wide`:** 1.7.1 (2026-09-14; Zlib OR Apache-2.0 OR MIT). Its `f32x4` implementation has `target_feature="simd128"` code paths (42 occurrences in `f32x4_.rs`) — [crates.io wide](https://crates.io/crates/wide). fundsp depends on `wide` 1.1.1 for explicit block SIMD — [fundsp CHANGES 0.18](https://github.com/SamiPerttu/fundsp/blob/master/CHANGES.md)
- **Enabling simd128 in Rust:** Use `rustflags = ["-C", "target-feature=+simd128"]`. A January 2026 article reports 1.7–4.5× speedups from SIMD128 optimization of browser audio processing (DeepFilterNet3 noise suppression workload) — [zenn.dev article (2026-01-23)](https://zenn.dev/fitness_densuke/articles/2026-01-23-wasm-simd-optimization) (secondary source; the article itself could not be fetched, the claim is from its search snippet)
- **Blog claims (practitioner blog, not a benchmark study):** "WebAssembly runs at 80-95% of native speed for number-crunching code like DSP". A 128-sample render quantum at 44.1 kHz gives about 2.9 ms per callback — [Joel Löf, Web Audio API for Real-Time DSP](https://joellof.com/blog/web-audio-api-real-time-dsp/)
- **Prior art for Rust + WASM SIMD FM synthesis in an AudioWorklet** (content not fetched: domain blocked) — [Casey Primozic, "FM Synthesis in the Browser with Rust, Web Audio, and WebAssembly with SIMD"](https://cprimozic.net/blog/fm-synth-rust-wasm-simd/)
- **AudioWorklet paths in Rust:**
  - cpal's `audioworklet` backend runs audio "on a dedicated thread" but needs nightly, `-Zbuild-std` and COOP/COEP headers — [cpal README](https://github.com/RustAudio/cpal#readme)
  - firewheel-web-audio has the same requirements — [crates.io firewheel-web-audio](https://crates.io/crates/firewheel-web-audio)
  - Glicol uses a plain wasm-bindgen module inside a JS `AudioWorkletProcessor` — [glicol js/src](https://github.com/chaosprint/glicol/tree/main/js/src)
- **SIMD outside the portable path:** rubato's SIMD is x86_64/aarch64-only — [rubato README](https://github.com/HEnquist/rubato#simd-acceleration). fundsp's denormal protection is x86-only — [fundsp src/denormal.rs](https://github.com/SamiPerttu/fundsp/blob/master/src/denormal.rs)

### Inferences
- **Two browser build options:**
  - (a) A single-threaded WASM module instantiated inside a JS `AudioWorkletProcessor` (Glicol-style). This works on stable Rust with no COOP/COEP headers, with control via `port.postMessage`.
  - (b) A shared-memory multi-threaded build (cpal `audioworklet` or firewheel-web-audio) that allows lock-free SAB queues from the UI thread. It needs nightly and cross-origin isolation.

  For a collaborative web DAW that may embed or iframe content, (a) is lower-risk.
- **Performance:** Build two WASM binaries, with and without simd128, only if pre-2023 Safari matters. Otherwise ship simd128-only (Safari 16.4 shipped in early 2023; that release date is general knowledge, not taken from BCD). WASM has no FTZ/DAZ control, so feedback DSP (reverbs, IIR filters) should add explicit denormal guards — note fundsp's guard is x86-only.

### Gaps
- No rigorous, recent benchmark of fundsp (or any Rust DSP graph) in WASM versus native was found.
- The claim that WASM has no FTZ/DAZ control is from general knowledge and was not re-verified with a source in this session.

## Comparison and fit for a mini-DAW: which to use and how to combine

### Takeaway
No single Rust crate is a complete "mini-DAW engine" for both native and WASM. The best-supported combination is:
- **fundsp** for all DSP: synth voices, drums, filters, reverb, chorus, phaser, delay, distortion, limiter; plus `Net` for hot-swappable per-track chains.
- A **host/transport layer**, either Firewheel (graph + musical transport + scheduled events + sampler + WASM backend) or a thin custom engine on cpal / a JS AudioWorklet.
- A **custom bytebeat compiler/VM**.
- Optional add-ons: rustysynth for SoundFonts, rubato for SRC, signalsmith-stretch or timestretch for time-stretch, and kira's or web-audio-api's compressor ideas for dynamics.

### Cited Findings
- fundsp supplies the DSP breadth (oscillators, Moog/SVF/biquad filters, reverbs, chorus, flanger, phaser, delay, limiter, waveshapers, drum presets) plus `Net` frontend/backend with `commit()` and a `Sequencer` — [fundsp README](https://github.com/SamiPerttu/fundsp#readme); [src/prelude.rs](https://github.com/SamiPerttu/fundsp/blob/master/src/prelude.rs). It lacks a compressor, `Net` feedback loops and `Sequencer` looping — [FUTURE.md](https://github.com/SamiPerttu/fundsp/blob/master/FUTURE.md)
- Firewheel supplies the DAG engine, musical transport clock, scheduled events, sampler, silence optimization and WASM backends. It explicitly disclaims DAW scope and built-in synths — [Firewheel README](https://github.com/BillyDM/firewheel#readme); [DESIGN_DOC.md](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)
- knyst is dormant since 2024-11 — [crates.io knyst](https://crates.io/crates/knyst). glicol_synth has had no release since 2024-04 — [crates.io glicol_synth](https://crates.io/crates/glicol_synth). HexoDSP/synfx-dsp are GPL-3.0 and dormant since 2024-01 — [crates.io hexodsp](https://crates.io/crates/hexodsp)
- web-audio-api-rs is active (1.7.0, 2026-08-08), but its browser path is "experimental" — [README](https://github.com/orottier/web-audio-api-rs#readme)
- kira is very active (0.12.5, 2026-09-26) and has compressor, delay, distortion, EQ and reverb effects plus clocks, but on WASM it cannot load from files or stream — [kira README](https://github.com/tesselode/kira#readme)

### Inferences
Comparison summary (all versions and dates from crates.io, 2026-09-26):

| Crate | Latest (date) | License | Role fit | Dynamic graph | Scheduling | WASM | Maintenance |
|---|---|---|---|---|---|---|---|
| fundsp | 0.23.0 (2026-01-07) | MIT/Apache | DSP library + light graph | `Net` frontend/backend `commit()`, crossfade | `Sequencer` (seconds, no loop/tempo) | Pure Rust, no_std; SIMD via `wide` (needs +simd128); no official WASM demo | Active, 1 maintainer; last commit 2026-03 |
| firewheel | 0.14.0 (2026-09-05) | MIT/Apache | Engine/host (game-oriented) | Compiled DAG, recompiled on change | Seconds/sample/musical clocks, scheduled events | Yes: cpal web + multithreaded AudioWorklet backend (nightly, COOP/COEP) | Very active; being upstreamed into Bevy |
| knyst | 0.5.1 (2024-11-25) | MIT/Apache | Dynamic graph + synthesis | Yes, nested, feedback | Sample-accurate | Unverified | Dormant |
| glicol_synth | 0.13.5 (2024-04-23) | MIT | Small graph engine (live coding) | Yes (petgraph) | Node messages; seq nodes | Proven in AudioWorklet | Slow (last commit 2025-01) |
| web-audio-api | 1.7.0 (2026-08-08) | MIT | Web Audio clone for native | Web Audio connect/disconnect | AudioParam automation, `start(when)` | Experimental via cpal | Active |
| kira | 0.12.5 (2026-09-26) | MIT/Apache | Game audio playback + mixer | Mixer tracks, not arbitrary DSP | Clocks/ticks, tweens | Yes, with limits | Very active |
| dasp | 0.11.0 (2020-05-29) | MIT/Apache | Sample/frame utilities | `dasp_graph` (2020) | — | no_std | Frozen |
| hexodsp / synfx-dsp | 0.2.2 / 0.5.6 (2024-01-04) | **GPL-3.0+** | Modular synth engine + JIT | Yes | TSeq tracker | Unverified | Dormant |
| rustysynth | 1.3.6 (2025-08-10) | MIT | SoundFont voice | — | MIDI events | Pure Rust (unverified) | Maintained |
| rubato | 5.0.0 (2026-08-10) | MIT/Apache | Resampling | — | — | Scalar on WASM | Active; major-version churn |

- **Recommended architecture (an inference, not a documented pattern):**
  1. **Engine core** is a Rust crate compiled both natively and to WASM. It owns a transport (tempo, PPQ, loop points), a step sequencer that emits note events at *sample offsets* inside each block, and a mixer (tracks → sends → master). It holds one fundsp `Net` backend per track or instrument chain, with UI edits applied via `Net::commit()` or `crossfade`, and parameters via `shared()`/`var()` or `Setting`s.
  2. **Instruments** are fundsp graphs: drum voices from `sound::bassdrum`/`snaredrum`/`cymbal` or custom ones; subtractive synths with `poly_saw >> moog_hz`; sample playback via `playwave_at` or a custom sampler over preloaded `Wave`s.
  3. **The bytebeat instrument** is a custom `AudioUnit` wrapping a bytecode VM (optionally a Cranelift JIT natively), fed by the transport's `t`.
  4. **Effects:** fundsp reverb/delay/chorus/phaser/moog/shape/limiter. The compressor is custom-written (a feed-forward RMS/peak compressor is a small amount of code), or ported from kira's or web-audio-api's implementation (both permissive).
  5. **I/O:**
     - Native: cpal directly, or Firewheel with fundsp inside custom nodes if its transport, sampler and device fault tolerance are wanted.
     - Browser: a wasm-bindgen module run inside a JS `AudioWorkletProcessor` (Glicol pattern, stable Rust, no COOP/COEP needed), or cpal `audioworklet` / firewheel-web-audio if SharedArrayBuffer threading is acceptable.
  6. **Assets:** Decode on the control side (Symphonia natively; `decodeAudioData` in the browser), resample with rubato at import, and time-stretch offline with signalsmith-stretch (native) or a pure-Rust stretcher (web).
- **fundsp-only or Firewheel+fundsp?** fundsp-only is simpler and has a smaller WASM footprint. Firewheel+fundsp gives a battle-tested scheduler and backend, at the cost of monthly breaking API churn and a game-oriented event model that may conflict with DAW transport semantics (as Firewheel's own design doc says).

### Gaps
- No real-world open-source "mini-DAW on fundsp + WASM" project was found to validate the combined architecture.
- No head-to-head CPU benchmarks between fundsp, Firewheel and knyst were found.
- Community opinion threads (Reddit, users.rust-lang.org) comparing these engines were not found by search in this session.
