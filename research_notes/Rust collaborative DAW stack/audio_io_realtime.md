# Rust Audio I/O Backends and Real-Time-Safety Tooling (native + WASM), as of Sept 2026

Research date: 2026-09-26. Crate versions and dates come from the crates.io API (`https://crates.io/api/v1/crates/<name>`), queried on that date. Several hosts were blocked by this session's egress proxy: doc.rust-lang.org, rustwasm.github.io, blog.paul.cx, jefftk.com, mail-archive.com, groups.google.com and emscripten.org. For those sources, claims come from search-result snippets and are marked "(snippet)". Primary sources that could be read (GitHub raw files, crates.io, MDN source on GitHub) are cited directly.

---

## Q1. cpal: version, maintenance, hosts, buffer control, duplex, web backends, build requirements

### Takeaway
cpal is actively maintained. The latest release is 0.18.2 (2026-08-16), and master is already at 0.19.0-dev with a new duplex API. It is the de-facto Rust audio I/O layer and covers WASAPI/ASIO/JACK on Windows; CoreAudio (+JACK) on macOS; ALSA/PipeWire/PulseAudio/JACK on Linux; AAudio on Android; and two web hosts. The default `WebAudio` host is a main-thread design built on AudioBufferSourceNode scheduling for output and the deprecated ScriptProcessorNode for input. A separate, official `AudioWorklet` host exists behind the `audioworklet` feature. It needs nightly Rust, `-Zbuild-std`, `+atomics` with shared-memory linker args, and COOP/COEP headers.

### Cited Findings
**Version / maintenance**
- Latest release 0.18.2, published 2026-08-16. Earlier releases: 0.18.1 (2026-06-07), 0.18.0 (2026-06-06), 0.17.3 (2026-02-18), 0.17.2 (2026-02-08, **YANKED**), 0.17.1 (2026-01-04), 0.17.0 (2025-12-20), 0.16.0 (2025-06-07). About 21.8M total downloads and 6.6M recent — [crates.io cpal](https://crates.io/crates/cpal); [cpal CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- On master, `Cargo.toml` is `version = "0.19.0"`, `edition = "2024"`, `rust-version = "1.85"`, and `maintenance = { status = "actively-developed" }` — [cpal Cargo.toml](https://github.com/RustAudio/cpal/blob/master/Cargo.toml)
- The "Unreleased" section of the changelog has several breaking changes:
  - Migration to Rust 2024.
  - `DeviceTrait`/`StreamTrait` now require `Send + Sync`.
  - `StreamTrait::play` is renamed to `start`.
  - Input and output callback infos are merged into `CallbackInfo`.
  - `ErrorKind::Xrun` is removed in favour of `CallbackInfo::xrun()`.
  - New `StreamTrait::stop` drains buffered audio with a timeout.
  - The minimum Windows version is now 10.
  — [cpal CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- MSRV is 1.85 on most platforms and 1.88 for PulseAudio. tvOS and the `audioworklet` feature need nightly — [cpal README](https://github.com/RustAudio/cpal/blob/master/README.md)

**Supported hosts** (README table: platform → default host → optional hosts) — [cpal README](https://github.com/RustAudio/cpal/blob/master/README.md)
- Windows: WASAPI (default); ASIO and JACK optional. ASIO needs the Steinberg SDK (downloaded by build.rs), LLVM/Clang and `CPAL_ASIO_DIR`.
- macOS: CoreAudio; JACK optional. iOS, tvOS and visionOS: CoreAudio.
- Linux/BSD: ALSA; JACK, PipeWire and PulseAudio optional. The native PipeWire and PulseAudio hosts are new in 0.18.0 — [CHANGELOG 0.18.0](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- Linux/BSD default host priority is PipeWire > PulseAudio > ALSA (0.18.0) — [CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- ALSA headers are needed even when using JACK, PipeWire or PulseAudio — [README](https://github.com/RustAudio/cpal/blob/master/README.md)
- Android: AAudio. cpal "migrated from `oboe` to `ndk::audio`" in 0.16.0, which raised the minimum to API 26. In 0.18.0, AAudio requests `PERFORMANCE_MODE_LOW_LATENCY` when the `realtime` feature is on — [CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- WebAssembly: the "Web Audio API" host is the default and "Audio Worklet" is optional. Targets are `wasm32-unknown-unknown`, `wasm32-unknown-emscripten` (needs Emscripten 6.0.3 and wasm-bindgen 0.2.127) and `wasm32-wasip1` — [README](https://github.com/RustAudio/cpal/blob/master/README.md)
- Unreleased: Emscripten targets now go through wasm-bindgen's Emscripten integration. The old Emscripten host was removed as broken in 0.18.0 — [CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- wasm-bindgen added `--target emscripten` support in 0.2.115 (PR #4443, March 2026) — [wasm-bindgen issue #5237](https://github.com/wasm-bindgen/wasm-bindgen/issues/5237)
- Real-time thread promotion uses the `realtime` feature (Android, Linux, Windows) or `realtime-dbus` (rtkit via D-Bus on Linux). This feature was called `audio_thread_priority` before 0.18.0 — [cpal Cargo.toml](https://github.com/RustAudio/cpal/blob/master/Cargo.toml); [CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)

**Buffer-size control**
- `StreamConfig.buffer_size` is `BufferSize::Default` or `BufferSize::Fixed(n)`. Supported ranges are reported as `SupportedBufferSize::Range{min,max}` or `Unknown`. 0.18.0 added `StreamTrait::buffer_size()` (current frames per callback) and `StreamTrait::now()` — [CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- Known pitfall: on ALSA, `BufferSize::Default` can range "from a PipeWire quantum (typically 1024 frames) to `u32::MAX`" on misconfigured hardware. The README recommends `BufferSize::Fixed(1024)` or configuring the system — [README](https://github.com/RustAudio/cpal/blob/master/README.md)
- 0.17.0 made CoreAudio and AAudio configure the device buffer so callback sizes are predictable. JACK rejects `Fixed` sizes that don't match the server size. iOS gained AVAudioSession buffer control — [CHANGELOG 0.17.0](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- WebAudio host constants:
  - `DEFAULT_BUFFER_SIZE: usize = 2048`.
  - ScriptProcessor sizes are limited to the power-of-two set `[256 … 16384]`.
  — [cpal src/host/webaudio/mod.rs](https://github.com/RustAudio/cpal/blob/master/src/host/webaudio/mod.rs)
- AudioWorklet host:
  - `DEFAULT_RENDER_SIZE = 128`.
  - If the browser exposes `AudioContext.prototype.renderQuantumSize`, the supported range becomes `1 ..= sample_rate*6`; otherwise it is fixed at 128.
  - `BufferSize::Fixed` sets `renderSizeHint` on the AudioContext (0.18.0).
  — [cpal src/host/audioworklet/mod.rs](https://github.com/RustAudio/cpal/blob/master/src/host/audioworklet/mod.rs); [CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)

**Duplex (input + output)**
- Up to 0.18.2, input and output are separate streams. The idiomatic bridge is a ring buffer, which is what cpal's examples do with `ringbuf`.
- Unreleased (0.19) adds `DeviceTrait::build_duplex_stream()`, `build_duplex_stream_raw()`, `default_duplex_config()` and `supports_duplex()`, with "capture and playback from one device-level callback". It is implemented for ALSA, JACK, AudioWorklet and WebAudio.
- ASIO already had duplex fixes in 0.18.2 ("Fix duplex streams silently dropping audio when built from separate Device handles").
  — [CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- README: "cpal doesn't compose devices itself; it needs a device that already claims both capture and playback." On ALSA, separate cards must be combined with the `asym` plugin. JACK needs no extra setup — [README "Duplex Devices"](https://github.com/RustAudio/cpal/blob/master/README.md)
- The cpal AudioWorklet example page describes both modes. In two-stream mode, independent input and output streams are bridged by a ring buffer. Duplex mode "runs both directions from one callback on one clock, so it needs no ring buffer and the round trip is audibly shorter" — [examples/audioworklet/index.html](https://github.com/RustAudio/cpal/tree/master/examples/audioworklet)

**Known issues / quirks (from the changelog and README)**
- When PipeWire or PulseAudio is running, it holds ALSA `default` exclusively, so a second ALSA stream fails with `DeviceBusy`. Use the `pipewire` or `pulse` ALSA devices, or the native features — [README](https://github.com/RustAudio/cpal/blob/master/README.md)
- If the stream rate differs from PipeWire's clock, PipeWire's ALSA, JACK and Pulse compatibility layers resample, which "can cause audible glitches or underruns… without an xrun being reported." The native `pipewire` feature is "less susceptible" to this — [README](https://github.com/RustAudio/cpal/blob/master/README.md)
- 0.18.x has many xrun and timestamp fixes. Xruns are now reported for AAudio, CoreAudio, PipeWire and WASAPI capture. Timestamps now include hardware latency on WASAPI, CoreAudio, ASIO, JACK and iOS, and stay monotonic across device changes. WASAPI default streams auto-reroute when the system default device changes — [CHANGELOG 0.18.0/0.18.2](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- 0.18.2 fixed an "unsound `Send + Sync` on `Stream` when compiled with `+atomics`" in the WebAudio host. It also fixed AudioWorklet `Stream` operations so they work when called from any thread — [CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)

**Which Web API the web backends use**
- The default `WebAudio` host ("Default backend on WebAssembly") schedules output in two parts:
  - It creates `AudioBuffer`s and plays them via `AudioBufferSourceNode.start(when)` from the main thread, driven by timeouts (`schedule_initial_timeouts`). The default buffer is 2048 frames.
  - Input and duplex use a single `ScriptProcessorNode` (`onaudioprocess`).
  — [cpal src/host/webaudio/mod.rs](https://github.com/RustAudio/cpal/blob/master/src/host/webaudio/mod.rs); [Cargo.toml `wasm-bindgen` feature lists `web-sys/ScriptProcessorNode`, `AudioProcessingEvent`](https://github.com/RustAudio/cpal/blob/master/Cargo.toml)
- ScriptProcessorNode has `status: deprecated` on MDN — [MDN ScriptProcessorNode](https://developer.mozilla.org/en-US/docs/Web/API/ScriptProcessorNode)
- The `AudioWorklet` host is official, in-tree and feature-gated: "Audio Worklet backend for lower-latency web audio than the default Web Audio API, running audio on a dedicated thread." It requires `RUSTFLAGS="-C target-feature=+atomics,+bulk-memory,+mutable-globals"` and Cross-Origin headers for SharedArrayBuffer — [README](https://github.com/RustAudio/cpal/blob/master/README.md). The feature pulls in `web-sys/AudioWorklet`, `AudioWorkletNode`, `AudioWorkletNodeOptions`, `Blob` and `Url` — [Cargo.toml](https://github.com/RustAudio/cpal/blob/master/Cargo.toml)
- How the AudioWorklet host works:
  - The JS processor receives `[module, memory, handle]` via `processorOptions` and calls `bindgen.initSync({ module, memory })` inside the worklet, so it shares the main module's WebAssembly memory.
  - Interleaving and deinterleaving are done in JS "because it avoids an extra copy."
  - The worklet re-takes its Float32Array view when Wasm memory grows.
  - Three processors are registered: `CpalProcessor`, `CpalCaptureProcessor` and `CpalDuplexProcessor`.
  - The main thread polls `outputLatency` every 500 ms and publishes it to the worklet.
  — [cpal src/host/audioworklet/worklet.js](https://github.com/RustAudio/cpal/blob/master/src/host/audioworklet/worklet.js); [mod.rs](https://github.com/RustAudio/cpal/blob/master/src/host/audioworklet/mod.rs)
- The AudioWorklet host supports only `F32`, 1–32 channels and 3–768 kHz. Input and duplex are in Unreleased — [mod.rs](https://github.com/RustAudio/cpal/blob/master/src/host/audioworklet/mod.rs); [CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)

**Build requirements for `audioworklet`**
- README: "The `audioworklet` backend additionally requires `-Zbuild-std` with atomics support enabled" (nightly) — [README](https://github.com/RustAudio/cpal/blob/master/README.md)
- The example's `.cargo/config.toml` for `wasm32-unknown-unknown` sets:
  - rustflags `-C target-feature=+atomics`
  - link args `--shared-memory`, `--max-memory=1073741824`, `--import-memory`
  - exports `__wasm_init_tls`, `__tls_size`, `__tls_align`, `__tls_base`, `__heap_base`
  - `[unstable] build-std = ["std","panic_abort"]`
  - `rust-toolchain.toml` with `channel = "nightly"`
  — [cpal examples/audioworklet/.cargo/config.toml](https://github.com/RustAudio/cpal/tree/master/examples/audioworklet)
- The example's `Trunk.toml` serves `cross-origin-embedder-policy: require-corp`, `cross-origin-opener-policy: same-origin` and `cross-origin-resource-policy: same-site` — [examples/audioworklet/Trunk.toml](https://github.com/RustAudio/cpal/tree/master/examples/audioworklet)
- The explicit shared-memory link args are needed because rust-lang/rust PR #147225 ("Don't enable shared memory by default with Wasm atomics", merged 2025-10-02) stopped `+atomics` from implying shared memory. Atomics remain unstable on `wasm32-unknown-unknown` and require nightly with `-Zbuild-std` — [rust-lang/rust#147225](https://github.com/rust-lang/rust/pull/147225)

### Inferences
- For a DAW, cpal is the right native I/O layer: it is maintained, covers pro backends (ASIO, JACK, PipeWire) and now reports xruns and latency-aware timestamps. Pin 0.18.x now and plan a migration to 0.19, which renames `play`→`start`, merges `CallbackInfo` and adds duplex.
- On the web, only the `audioworklet` host is acceptable for a DAW. The default WebAudio host schedules from the main thread with a 2048-frame default (~43 ms at 48 kHz) and uses the deprecated ScriptProcessorNode for input, so UI jank directly causes dropouts.
- Using the `audioworklet` host commits the project to nightly Rust, build-std and cross-origin isolation for the web build. The native build can stay on stable.

### Gaps
- No release date or crates.io version for 0.19 could be found; it is "Unreleased" on master.
- The changelog does not say which release first added the `audioworklet` host (grep found no "Added" entry). It exists in 0.18.0, because 0.18.0 lists AudioWorklet fixes.
- No first-hand GitHub issue triage was possible, because the GitHub API was blocked for this repo in this session. Open-issue counts and CPU/latency regressions are unknown.

---

## Q2. Alternatives to cpal, and which suits a DAW engine with sample-accurate scheduling

### Takeaway
No higher-level crate is a ready-made DAW engine. rodio, kira, oddio and tinyaudio are playback- or game-oriented, and Firewheel explicitly says it is "not a DAW engine". The best-supported path is:
- **Native:** cpal for I/O (plus the `jack`/`pipewire`/`asio` features for pros) driving your own block-based engine with sample-offset event queues.
- **Web:** the same engine run through cpal's `audioworklet` host or a hand-rolled wasm-bindgen AudioWorklet.

web-audio-api-rs is worth considering if you want Web-Audio-style AudioParam automation on native. Firewheel is the most advanced cross-platform Rust graph engine to borrow design from.

### Cited Findings
**Version / maintenance table** (crates.io, 2026-09-26)

| Crate | Latest | Released | Notes |
|---|---|---|---|
| cpal | 0.18.2 | 2026-08-16 | active; 0.19 in dev — [crates.io](https://crates.io/crates/cpal) |
| rodio | 0.22.2 | 2026-03-05 | "Audio playback and recording library"; features `playback`, `recording`, symphonia decoders — [crates.io](https://crates.io/crates/rodio) |
| kira | 0.12.5 | 2026-09-26 | active (0.12.3/4/5 in Aug–Sep 2026); features `cpal-realtime`, `cpal-realtime-dbus` — [crates.io](https://crates.io/crates/kira) |
| oddio | 0.7.4 | 2023-10-15 | **dormant ~3 yrs** — [crates.io](https://crates.io/crates/oddio) |
| tinyaudio | 2.0.0 | 2025-11-15 | output-only — [crates.io](https://crates.io/crates/tinyaudio) |
| miniaudio (bindings) | 0.10.0 | 2020-07-23 | **unmaintained** — [crates.io](https://crates.io/crates/miniaudio) |
| rtaudio (RtAudio bindings) | 0.8.0 | 2026-01-21 | Meadowlark/Codeberg; features alsa, asio, coreaudio, ds, jack_linux, oss, pulse, wasapi (no web) — [crates.io](https://crates.io/crates/rtaudio) |
| jack | 0.13.5 | 2026-02-01 | RustAudio/rust-jack — [crates.io](https://crates.io/crates/jack) |
| pipewire (pipewire-rs) | 0.10.1 | 2026-08-19 | freedesktop GitLab — [crates.io](https://crates.io/crates/pipewire) |
| audio (udoprog) | 0.2.1 | 2025-09-18 | prior release 0.2.0 was 2023-12-04; `audio-device` 0.1.0-alpha.6 last released 2021-04-16 → **low activity** — [crates.io](https://crates.io/crates/audio) |
| web-audio-api | 1.7.0 | 2026-08-08 | active (1.5 May, 1.6 Jun, 1.7 Aug 2026) — [crates.io](https://crates.io/crates/web-audio-api) |
| cubeb (Mozilla) | 0.38.0 | 2026-08-18 | Firefox's audio lib bindings — [crates.io](https://crates.io/crates/cubeb) |
| firewheel / firewheel-cpal | 0.14.0 | 2026-09-05 | [crates.io](https://crates.io/crates/firewheel) |
| firewheel-web-audio | 0.5.0 | 2026-08-09 | multi-threaded wasm backend — [crates.io](https://crates.io/crates/firewheel-web-audio) |
| wasapi (HEnquist) | 0.24.0 | 2026-08-12 | [crates.io](https://crates.io/crates/wasapi) |
| coreaudio-rs | 0.14.2 | 2026-04-29 | [crates.io](https://crates.io/crates/coreaudio-rs) |
| oboe | 0.6.1 | 2024-03-03 | cpal no longer uses it — [crates.io](https://crates.io/crates/oboe) |
| portaudio | 0.8.0 | 2024-10-13 | low activity — [crates.io](https://crates.io/crates/portaudio) |
| awsm-audio-worklet | 2.5.0 | 2026-06-30 | ~362 downloads total — [crates.io](https://crates.io/crates/awsm-audio-worklet) |

**Per-crate notes**
- **web-audio-api-rs** ("A pure Rust implementation of the Web Audio API, for use in non-browser contexts"):
  - Backends: `cpal` (default), `cpal-jack`, `cpal-pipewire`, `cpal-asio`, and experimental `cubeb` (needs cmake).
  - On ALSA, the default 128-frame render size can crackle. The workaround is `latency_hint: Playback`, which raises the render size to 1024. For low latency "rely on the JACK backend."
  - Output can be piped back into the browser through cpal's wasm-bindgen backend ("experimental").
  - NodeJS bindings are used to run the official WPT harness.
  — [web-audio-api README (crates.io)](https://crates.io/crates/web-audio-api); [GitHub](https://github.com/orottier/web-audio-api-rs)
- **Firewheel** (BillyDM):
  - Graph engine "for games and other applications". It is being upstreamed as Bevy's default audio engine and "does *NOT* aim to be a complete DAW engine … the needs of game audio engines and DAW audio engines are in conflict."
  - "Properly respects realtime constraints (no mutexes!)", `no_std` compatible, with backends for Windows, Mac, Linux, Android, iOS and WebAssembly.
  — [Firewheel README](https://github.com/BillyDM/firewheel)
- The Firewheel design doc explains the DAW conflict: in a DAW, state is tied to the transport and the host may discard conflicting parameter events. In a game, user state changes at any time and events must never be discarded.
- Firewheel defines three clocks: a seconds clock (read from the OS API, accounts for underflows), a sample clock (very accurate but doesn't account for underflows) and a musical clock (beats since the transport started). Events are scheduled with `EventDelay::DelayUntilSeconds(clock_now() + delay)`.
- WASM rules in the design doc: "Don't Spawn Threads" (the backend owns the audio thread), "Don't Block Threads", no C deps (so no CLAP hosting on wasm).
  — [Firewheel DESIGN_DOC.md](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)
- **firewheel-web-audio**: "A multi-threaded wasm32-unknown-unknown Web Audio backend," stereo only. It needs nightly with `rust-src`, `+atomics,+bulk-memory,+mutable-globals` and `build-std = ["std","core","alloc","panic_abort"]`, served over HTTPS with COOP `same-origin` and COEP `require-corp` or `credentialless`. It warns that "`credentialless` may not work on Safari: the browser may throw an error in the audio worklet upon receiving shared Wasm memory" — [firewheel-web-audio README (crates.io)](https://crates.io/crates/firewheel-web-audio)
- **kira**: "backend-agnostic library to create expressive audio for games" with tweens, a mixer and "a clock system for precisely timing audio events" (e.g. a 120-tick-per-minute clock for beat-synced playback) — [kira README (crates.io)](https://crates.io/crates/kira)
- **oddio**: game-oriented, "Sans I/O", "wait-free" output, 3D spatialization — [oddio README (crates.io)](https://crates.io/crates/oddio). No release since 2023-10-15 — [crates.io](https://crates.io/crates/oddio)
- **tinyaudio**:
  - "does not support device enumeration, device selection, querying of supported formats, input capturing."
  - Android uses AAudio (API 26+).
  - The web backend uses `AudioBufferSourceNode.start(when)` driven by `setTimeout`, i.e. main-thread scheduling.
  — [tinyaudio README](https://crates.io/crates/tinyaudio); [tinyaudio src/web.rs](https://github.com/mrDIMAS/tinyaudio)
- **interflow** (SolarLiner): "a unified, opinionated interface for audio applications" over WASAPI, ASIO, CoreAudio, ALSA, PulseAudio, PipeWire and JACK, with duplex across "separate input and output devices" plus sample-rate and format conversion. It has about 67 stars and 301 commits. No web backend is mentioned, and the name `interflow` was not found on crates.io — [interflow GitHub](https://github.com/SolarLiner/interflow)
- **waw-rs**: Rust crate for writing AudioWorklet processors via a `Processor` trait and a `register!` macro. It needs `+atomics,+bulk-memory` for its `web-thread` dependency and says "This is all very experimental." It has about 18 stars and 20 commits and is not on crates.io — [waw-rs GitHub](https://github.com/Marcel-G/waw-rs)
- **wasm-bindgen official `wasm-audio-worklet` example**:
  - `WasmAudioProcessor(Box<dyn FnMut(&mut [f32]) -> bool>)` is `pack`ed into a `usize` pointer and passed to the worklet.
  - An `inline_js` function builds a Blob-URL module that `import`s the bindgen JS, calls `bindgen.initSync({ module, memory })` and `unpack`s the handle.
  - `process(inputs, outputs)` returns `this.processor.process(outputs[0][0])`, i.e. mono only.
  - It uses the nightly toolchain with `rust-src`.
  — [wasm-audio-worklet src/wasm_audio.rs](https://github.com/wasm-bindgen/wasm-bindgen/tree/main/examples/wasm-audio-worklet); [guide page](https://github.com/wasm-bindgen/wasm-bindgen/blob/main/guide/src/examples/wasm-audio-worklet.md)

### Inferences
- For sample-accurate scheduling in a DAW, none of the playback crates (rodio, kira, oddio, tinyaudio) give you a transport-owned event model. You will write your own block processor:
  - an event list per block with a sample offset for each event;
  - split processing at each event's offset;
  - transport state owned by the audio thread.
- Firewheel's clock and event design and its "no mutex" discipline are good references, but its game-oriented "never discard events" semantics differ from DAW transport semantics, as its own design doc says.
- **web-audio-api-rs** is an interesting option for "write once": you target Web Audio semantics (AudioParam automation, scheduled sources) and get the browser's native implementation on the web and a Rust implementation natively. For a DAW with custom DSP, though, you would still write AudioWorklet-style processors, so the portability gain is mainly scheduling and automation semantics.
- **rtaudio** and **interflow** are credible native-only alternatives to cpal (RtAudio's mature C++ core; interflow's duplex focus), but neither covers the web. Pairing either with a separate web path adds maintenance cost.
- Avoid miniaudio (stale since 2020), oddio (dormant) and `audio`/`audio-device` (low activity) for a new project.

### Gaps
- No benchmarks comparing callback jitter or latency across cpal, rtaudio and interflow on the same hardware were found.
- The interflow crate name and release status on crates.io could not be verified (it may be published under a different name).
- The kira and rodio changelogs were not examined for wasm-specific behaviour. Both use cpal, so they presumably inherit cpal's default WebAudio host limits, but this was not verified.

---

## Q3. WASM audio specifics: render quantum, running Rust in the worklet, SAB/COOP/COEP, wasm threads on stable, UI↔worklet messaging, latency

### Takeaway
- **Render quantum:** fixed at 128 frames today. Chrome is shipping a configurable render quantum (`renderSizeHint`; Intent to Ship July 2026), while Firefox and Safari have shown no signal.
- **Shared-memory Rust in the worklet** (one wasm module, same memory on the UI and audio threads) still requires nightly Rust (`+atomics`, `-Zbuild-std`, explicit shared-memory link args) and cross-origin isolation (COOP/COEP). GitHub Pages needs a service-worker shim.
- **Stable-Rust alternative:** instantiate a separate, non-threaded wasm instance inside the AudioWorkletGlobalScope. Feed it via MessagePort, or via SharedArrayBuffer ring buffers (ringbuf.js), which still need COOP/COEP.
- **Browser limits:** blocking (`Atomics.wait`) is disallowed in AudioWorklets, and TextDecoder is still reported missing in Chromium's worklet scope.

### Cited Findings
**Render quantum**
- MDN: "Currently, audio data blocks are always 128 frames long" — [MDN AudioWorkletProcessor.process()](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletProcessor/process). `AudioWorkletGlobalScope.currentFrame` "is incremented by 128 (the size of a render quantum)." The scope also exposes `currentTime`, `sampleRate` and `port` (MessagePort) — [MDN currentFrame](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletGlobalScope/currentFrame); [MDN AudioWorkletGlobalScope](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletGlobalScope)
- Configurable render quantum:
  - AudioContext and OfflineAudioContext take `renderSizeHint`: an integer, `"default"` (128) or `"hardware"`.
  - Chrome's Intent to Ship was posted July 23, 2026, behind the Finch feature `WebAudioConfigurableRenderQuantum`, for all six Blink platforms (snippet).
  — [blink-dev Intent to Ship (snippet)](http://www.mail-archive.com/blink-dev@chromium.org/msg17051.html); [Intent to Prototype (snippet)](https://groups.google.com/a/chromium.org/g/blink-dev/c/n4PifuLlrwc)
- The spec explainer says "Powers of two between 64 and 2048, inclusive MUST be supported." Gecko and WebKit show "no signal" (snippet) — [WebAudio explainer user-selectable-render-size](https://github.com/WebAudio/web-audio-api/blob/main/explainer/user-selectable-render-size.md); [WebKit standards-positions #662](https://github.com/WebKit/standards-positions/issues/662)
- AudioContext `latencyHint`: `"interactive"` is the default; the other values are `"balanced"`, `"playback"` or a number of seconds — [MDN AudioContext()](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/AudioContext)

**Running Rust in the AudioWorkletGlobalScope**
- TextEncoder/TextDecoder:
  - The historical blocker was that "TextEncoder and TextDecoder are not available within AudioWorklets" — [wasm-bindgen #2367](https://github.com/rustwasm/wasm-bindgen/issues/2367)
  - wasm-bindgen PR #3329 (merged 2023-02-27) made the JS shim use them only conditionally, so a polyfill is no longer required unless string conversion is used — [wasm-bindgen PR #3329](https://github.com/wasm-bindgen/wasm-bindgen/pull/3329)
- A September 2026 bug report from a small project says browser audio "failed silently in Chromium 151" for two reasons:
  - "No `TextDecoder` in `AudioWorkletGlobalScope`" (wasm-bindgen glue threw before `registerProcessor`).
  - "A posted `WebAssembly.Module` arrives as `messageerror`." The fix was to transfer the raw bytes and let `initSync` compile them inside the worklet.
  This is a single low-profile source and should be treated as indicative only — [AnthonE/Gates PR #152](https://github.com/AnthonE/Gates/pull/152)
- Workarounds discussed on the wasm-bindgen issue include passing a TextDecoder via MessagePort, or "not using wasm-bindgen on the AudioWorklet thread" and keeping the worklet side minimal (snippet) — [wasm-bindgen #2367](https://github.com/rustwasm/wasm-bindgen/issues/2367)
- The shared-memory pattern is the one both cpal's AudioWorklet host and the wasm-bindgen example use: pass `[module, memory, handle]` in `processorOptions`, then call `initSync({module, memory})` in the worklet — [cpal worklet.js](https://github.com/RustAudio/cpal/blob/master/src/host/audioworklet/worklet.js); [wasm-bindgen example](https://github.com/wasm-bindgen/wasm-bindgen/tree/main/examples/wasm-audio-worklet)
- Emscripten route: `-sAUDIO_WORKLET -sWASM_WORKERS` builds on Wasm Workers and SharedArrayBuffer. Its AudioParams "can affect the audio computation at sample precise accuracy," and `emscripten_audio_worklet_post_function_*()` provides message passing (snippet) — [Emscripten Wasm Audio Worklets API (snippet)](https://emscripten.org/docs/api_reference/wasm_audio_worklets.html)

**Cross-origin isolation / hosting**
- MDN: "To use shared memory your document must be in a secure context and cross-origin isolated." Check `crossOriginIsolated`. The relevant headers are COOP, COEP and CORP — [MDN SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)
- Required headers are `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp` (or `credentialless`) — [ringbuf.js README](https://github.com/padenot/ringbuf.js); [firewheel-web-audio README](https://crates.io/crates/firewheel-web-audio)
- GitHub Pages cannot set headers. coi-serviceworker "reloads the page on the user's first load" to inject COOP/COEP via a service worker. It must be a separate file served from your own origin (not a CDN, not bundled) over HTTPS or localhost — [gzuidhof/coi-serviceworker](https://github.com/gzuidhof/coi-serviceworker); see also [tomayac blog (2025-03-08)](https://blog.tomayac.com/2025/03/08/setting-coop-coep-headers-on-static-hosting-like-github-pages/)
- cpal's example serves the headers through Trunk's `[serve.headers]` and mentions an `assets/_headers` file (Netlify/Cloudflare-Pages style) — [cpal examples/audioworklet/Trunk.toml](https://github.com/RustAudio/cpal/tree/master/examples/audioworklet)
- Safari and `credentialless` — sources conflict:
  - One snippet says Safari added credentialless in 16.4; others say there is "no support on Safari" as of Dec 2024 — [andrewlock.net (snippet)](https://andrewlock.net/understanding-security-headers-part-3-cross-origin-embedder-policy/)
  - firewheel-web-audio warns credentialless "may not work on Safari" for shared Wasm memory in the worklet — [firewheel-web-audio](https://crates.io/crates/firewheel-web-audio)
- An aggregator lists SharedArrayBuffer (when isolated) in Chrome 68+, Firefox 79+ and Safari 15.2+ (snippet) — [testmuai (snippet, aggregator)](https://www.testmuai.com/learning-hub/sharedarraybuffer-browser-support/)

**Wasm threads on stable Rust (2026)**
- The wasm-bindgen guide says: "Rust does not ship a precompiled target (e.g. standard library) which has threading support enabled… you'll need to recompile the standard library with… `-C target-feature=+atomics`. Note that this requires a nightly Rust toolchain." `--target web` or `no-modules` is required because the module imports memory — [wasm-bindgen raytrace guide](https://github.com/wasm-bindgen/wasm-bindgen/blob/main/guide/src/examples/raytrace.md)
- wasm-bindgen caveats:
  - "The main thread in a browser cannot block… you can't do so much as acquire a mutex."
  - "There is no standard notion of a 'thread'… TLS destructors will never run" (`__wbindgen_thread_destroy` exists).
  - `--target bundler` is unsupported with threads.
  — [raytrace guide](https://github.com/wasm-bindgen/wasm-bindgen/blob/main/guide/src/examples/raytrace.md)
- Default-enabled wasm features on `wasm32-unknown-unknown` are `multivalue`, `mutable-globals`, `reference-types`, `sign-ext`, `nontrapping-fptoint` and `bulk-memory` (the last two since Rust 1.87/LLVM 20). `atomics` is not among them. Unwinding (`-Cpanic=unwind`) also still needs `-Zbuild-std` — [rustc platform-support wasm32-unknown-unknown.md](https://github.com/rust-lang/rust/blob/main/src/doc/rustc/src/platform-support/wasm32-unknown-unknown.md)
- PR #147225 (merged 2025-10-02) decoupled shared memory from `+atomics`. Threaded builds must now pass `--shared-memory --max-memory=… --import-memory --export=__wasm_init_tls …` explicitly. Atomics "remain an unstable feature requiring -Zbuild-std" — [rust-lang/rust#147225](https://github.com/rust-lang/rust/pull/147225)
- `wasm32-wasip1-threads` exists but is not a browser target and "is not a stable target" (snippet) — [rustc book wasm32-wasip1-threads (snippet)](https://doc.rust-lang.org/rustc/platform-support/wasm32-wasip1-threads.html)
- Helper crates:
  - `wasm_thread` 0.3.3 (2024-10-29, 1.35M downloads).
  - `web-workers` 0.3.7 (2026-09-23), which says it "Needs nightly compiler for atomics, and COOP/COEP headers".
  - `wasm-bindgen-rayon` 1.3.0 (2024-12-21).
  — [crates.io wasm_thread](https://crates.io/crates/wasm_thread); [mlm-games/web-workers](https://github.com/mlm-games/web-workers); [crates.io wasm-bindgen-rayon](https://crates.io/crates/wasm-bindgen-rayon)
- `wasm-bindgen` 0.2.129 and `web-sys` 0.3.106 were both released 2026-09-25. wasm-bindgen now lives in the `wasm-bindgen/wasm-bindgen` org (moved from rustwasm) — [crates.io wasm-bindgen](https://crates.io/crates/wasm-bindgen)

**UI thread ↔ worklet communication**
- ringbuf.js (Paul Adenot, Mozilla) is "a thread-safe wait-free single-consumer single-producer ring buffer for the web" using SharedArrayBuffer. It includes `audioqueue.ts` ("audio data streaming, without using postMessage") and `param.ts` ("parameter changes… index and value without using postMessage"). The latest version is 0.4.0 (TypeScript port). Its README lists Firefox, Chrome and Safari as compatible (as of 2023-05-25) — [padenot/ringbuf.js](https://github.com/padenot/ringbuf.js)
- After setup, ringbuf.js reads and writes "without causing any garbage collection or making any allocations" (snippet) — [search summary of ringbuf.js/blog (snippet)](https://blog.paul.cx/post/a-wait-free-spsc-ringbuffer-for-the-web/)
- Blocking is off-limits in the worklet. The Web Audio spec discussion says: "There is no way we can ever allow Atomics.wait in an AudioWorklet (this is fundamentally against the programming model used in real-time programming)" (snippet; the fetched page did not show this comment). `Atomics.waitAsync` is the non-blocking alternative — [WebAudio/web-audio-api #1848 (snippet)](https://github.com/WebAudio/web-audio-api/issues/1848); [tc39 Atomics.waitAsync](https://tc39.es/proposal-atomics-wait-async/)
- MessagePort is available as `AudioWorkletGlobalScope.port` and `AudioWorkletNode.port` — [MDN AudioWorkletGlobalScope](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletGlobalScope)

**Browser latencies / RT thread**
- Chromium's AudioWorklet thread uses a real-time priority thread "only when an AudioContext is spawned from a top-level document and is a real-time one" (snippet) — [Chromium platform-architecture-dev (snippet)](https://groups.google.com/a/chromium.org/g/platform-architecture-dev/c/0wS55qzWMaw/m/B5A1ByIeAwAJ); [crbug 40563687](https://issues.chromium.org/issues/40563687)
- Measured figures (snippets only, dates unclear, likely 2020–2024):
  - `outputLatency` of ~15.4 ms in Firefox vs 24 ms in Chrome with wired headphones — [jamieonkeys (snippet)](https://www.jamieonkeys.dev/posts/web-audio-api-output-latency/)
  - An echo-test round trip of 62 ms in Firefox vs 124 ms in Chrome — [Jeff Kaufman (snippet)](https://www.jefftk.com/p/audioworklet-latency-firefox-vs-chrome)
- Safari implements `baseLatency` but not `outputLatency` (snippet) — [jamieonkeys (snippet)](https://www.jamieonkeys.dev/posts/web-audio-api-output-latency/); see [MDN outputLatency](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/outputLatency)
- A low-quality 2026 blog claims AudioWorklet reaches "5–20 ms on desktop and 30–100 ms on mobile" (unverified) — [StageAlly (snippet)](https://stageally.com/articles/best-browser-for-web-audio)

### Inferences
**Recommended dual-target architecture options**
1. **Shared-memory (nightly) path.** Use cpal `audioworklet` (or firewheel-web-audio) so the same Rust engine, `rtrb` queues and atomics run on both UI and audio threads with shared linear memory. The code structure matches native most closely.
   - Costs: nightly + build-std, COOP/COEP everywhere (every third-party embed needs CORP/CORS), and Safari credentialless issues.
   - Risk: any contended `std::sync::Mutex` would try to block, which is disallowed in the worklet and on the main thread.
2. **Stable, message-passing path.**
   - Compile the DSP engine as its own non-threaded wasm module (stable Rust, no atomics).
   - Fetch the bytes on the main thread, transfer them to the worklet and compile/instantiate them there. Transferring bytes sidesteps the reported Module-postMessage problem.
   - Keep the worklet glue free of wasm-bindgen string APIs, so no TextDecoder is needed.
   - Send parameter and edit messages over MessagePort (simple, but allocates and GCs in JS) or over a SAB ring buffer (ringbuf.js; requires COOP/COEP but not wasm threads). JS in the worklet copies from the SAB into the wasm instance's memory once per 128-frame quantum.
   - This keeps the engine crate identical across targets. Only the "host shell" differs: a cpal callback natively, and `process()` → `engine.process(block)` on the web.
3. Either way, design the engine around a **fixed internal block size that divides 128**, or a variable block with sample-offset events. Chrome's `renderSizeHint` may later allow larger or smaller quanta, but Firefox and Safari will stay at 128 for now.
- A realistic expectation is that web output latency will be several times native ASIO/JACK/CoreAudio latency, with no input monitoring comparable to native. Use web mainly for composition and collaboration, and native for tracking/recording.

### Gaps
- No authoritative, current (2025–2026) cross-browser latency benchmark was found. The key latency pages (jefftk, jamieonkeys) were blocked, so figures come from snippets.
- The exact Chrome milestone shipping `renderSizeHint` could not be confirmed (the blink-dev archive was blocked).
- Whether Firefox and Safari now expose `TextDecoder` in AudioWorkletGlobalScope could not be confirmed from primary sources.
- Safari's COEP `credentialless` support status is contradictory across sources.

---

## Q4. Real-time-safety crates, best practices for UI↔audio messaging, behaviour on wasm32

### Takeaway
The standard Rust DAW toolkit is:
- `rtrb` (wait-free SPSC) or `ringbuf` for command and event queues;
- `triple_buffer` for "latest value" state snapshots (meters, UI views);
- `basedrop` (or sending old objects back on a return queue) so nothing is freed on the audio thread;
- `arc-swap` (lock-free reads) for swapping immutable engine snapshots, but only with deferred deallocation;
- `assert_no_alloc` and/or RTSan (`rtsan-standalone`) to catch allocations and blocking in debug and CI;
- `audio_thread_priority` (via cpal's `realtime` features) for thread priority.

`crossbeam-channel` is not strictly RT-safe. On wasm32, priority and denormal control are unavailable (no-op or impossible), and blocking primitives must never be used on the audio or main thread.

### Cited Findings
**Crate status** (crates.io, 2026-09-26)
- `rtrb` 0.4.0 (2026-08-17), with a 0.3.5 backport (2026-08-18). About 12.1M downloads.
  - "A wait-free single-producer single-consumer (SPSC) ring buffer."
  - `no_std` with `alloc`, MSRV 1.38.
  - Tested under Miri and ThreadSanitizer.
  - Derived from a crossbeam PR.
  — [crates.io rtrb](https://crates.io/crates/rtrb); [rtrb README](https://github.com/mgeier/rtrb)
- `ringbuf` 0.5.2 (2026-09-13; 0.5.0 was 2026-05-03). About 19.5M downloads.
  - "Lock-free SPSC FIFO ring buffer with direct access to inner data."
  - `HeapRb`/`StaticRb`/`LocalRb`, overwriting insertion, and async and blocking variants.
  - Optional `portable-atomic` "to allow usage on smaller systems without CAS operations."
  — [ringbuf README](https://crates.io/crates/ringbuf)
- `triple_buffer` 9.0.0 (2026-02-22; MSRV 1.86): "triple buffering, useful for sharing frequently updated data between threads" — [triple-buffer CHANGELOG](https://github.com/HadrienG2/triple-buffer/blob/master/CHANGELOG.md); [crates.io](https://crates.io/crates/triple_buffer)
- `basedrop` 0.1.3 (2025-10-29; the previous release was 0.1.2 in 2021): "smart pointers analogous to Box and Arc which mark their contents for deferred collection on another thread rather than immediately freeing it, making them safe to drop on a real-time thread." `SharedCell` provides get/set/replace — [basedrop README](https://crates.io/crates/basedrop); [micahrj basedrop post (snippet)](https://micahrj.github.io/posts/basedrop/)
- `rtrb-basedrop` is an rtrb fork that uses basedrop `Shared` so the ring buffer's Vec is never freed on the RT thread (snippet) — [docs.rs rtrb-basedrop (snippet)](https://docs.rs/rtrb-basedrop/latest/rtrb_basedrop/)
- `assert_no_alloc` 1.1.2 (2021-08-03). **No release in five years**, yet about 1.46M recent downloads.
  - A custom global allocator that aborts or warns on (de)allocation inside `assert_no_alloc(|| …)`.
  - The default `disable_release` feature makes it a no-op in release builds; `warn_debug`/`warn_release` change the behaviour.
  — [assert_no_alloc README](https://crates.io/crates/assert_no_alloc); [GitHub](https://github.com/Windfisch/rust-assert-no-alloc)
- `rtsan-standalone` 0.3.0 (2025-09-27): RealtimeSanitizer (LLVM RTSan) for Rust.
  - Mark functions with `#[nonblocking]` and run with `RTSAN_ENABLE=1`.
  - It reports "Intercepted call to real-time unsafe function `calloc`…" with a stack trace.
  - Linux, macOS and iOS only; downloads prebuilt libs by default.
  — [rtsan-standalone-rs](https://github.com/realtime-sanitizer/rtsan-standalone-rs); [crates.io](https://crates.io/crates/rtsan-standalone)
- `no_denormals` 0.3.0 (2025-09-11): "Temporarily turn off floating point denormals" — [crates.io](https://crates.io/crates/no_denormals)
- `arc-swap` 1.9.2 (2026-06-28): "All the read operations are always lock-free. Most of the time, they are actually wait-free… Writers are lock-free." `load_full` is "lock-free and wait-free" — [arc-swap docs/performance.rs](https://github.com/vorner/arc-swap/blob/master/src/docs/performance.rs); [arc-swap lib.rs](https://github.com/vorner/arc-swap/blob/master/src/lib.rs)
- `atomic_float` 1.1.0 (2024-08-31; stable, low churn) — [crates.io](https://crates.io/crates/atomic_float). `portable-atomic` 1.15.0 (2026-08-09) also offers atomic floats — [crates.io](https://crates.io/crates/portable-atomic)
- `rt-history` 4.1.0 (2026-02-22): "An RT-safe history log with error checking" — [crates.io](https://crates.io/crates/rt-history)
- `audio_thread_priority` (Mozilla) 0.38.0 (2026-09-17, MPL-2.0; frequent releases, e.g. 0.36 and 0.37 in Jul–Aug 2026). Platform mechanisms:
  - macOS: Mach time-constraint policy.
  - Windows: MMCSS "Pro Audio".
  - Linux: rtkit over D-Bus by default, or `SCHED_FIFO` via `pthread_setschedparam` when the `dbus` feature is off (needs root, `CAP_SYS_NICE` or `RLIMIT_RTPRIO`).
  - Android: dedicated module.
  - **Other platforms: "a no-op that reports success"**, which includes wasm.
  — [audio_thread_priority src/lib.rs](https://github.com/mozilla/audio_thread_priority/blob/master/src/lib.rs); [crates.io](https://crates.io/crates/audio_thread_priority)
- cpal uses `audio_thread_priority` 0.36 behind the `realtime` and `realtime-dbus` features. As of 0.18.0 it reports `ErrorKind::RealtimeDenied` instead of printing to stderr — [cpal Cargo.toml](https://github.com/RustAudio/cpal/blob/master/Cargo.toml); [CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- Other queues: `heapless` 0.9.3 (2026-04-30, has `spsc`), `bbqueue` 0.7.0 (2026-03-14), `thingbuf` 0.1.6 (2024), `llq` 0.1.1 (2021) and `left-right` 0.11.8 — [crates.io heapless](https://crates.io/crates/heapless), [bbqueue](https://crates.io/crates/bbqueue), [llq](https://crates.io/crates/llq), [left-right](https://crates.io/crates/left-right)

**crossbeam-channel RT-safety** (from source; `crossbeam-channel` 0.5.17, 2026-09-05)
- `SyncWaker` wraps `inner: Mutex<Waker>`, and crossbeam's `utils::Mutex` is a thin wrapper over `std::sync::Mutex`.
- `notify()` checks an `is_empty: AtomicBool` first and takes the mutex only if some thread is registered as waiting.
- The bounded (array) flavour's `write()` calls `self.receivers.notify()`.
- The unbounded (list) flavour allocates blocks (`Global.allocate_zeroed`) while sending.
— [crossbeam-channel waker.rs](https://github.com/crossbeam-rs/crossbeam/blob/master/crossbeam-channel/src/waker.rs); [utils.rs](https://github.com/crossbeam-rs/crossbeam/blob/master/crossbeam-channel/src/utils.rs); [flavors/array.rs](https://github.com/crossbeam-rs/crossbeam/blob/master/crossbeam-channel/src/flavors/array.rs); [flavors/list.rs](https://github.com/crossbeam-rs/crossbeam/blob/master/crossbeam-channel/src/flavors/list.rs)

**Principles**
- Ross Bencina's rules: don't block the audio callback, don't allocate ("the memory allocator may have to ask the OS for more memory… the OS may… page some memory to/from disk"), keep algorithms predictable, and pre-allocate. Use lock-free FIFOs between GUI and audio (snippet) — [Ross Bencina, "Real-time audio programming 101: time waits for nothing"](http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing)
- Firewheel's design is an example: the context flushes queued events "as a group" once per frame, so events meant for the same cycle arrive together. Graph edits compile a new schedule off-thread and send it to the executor over "a realtime-safe message channel" — [Firewheel DESIGN_DOC](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)

**wasm32 specifics**
- The main thread cannot block ("can't do so much as acquire a mutex"), and `Atomics.wait` is disallowed in AudioWorklets (see Q3) — [wasm-bindgen raytrace guide](https://github.com/wasm-bindgen/wasm-bindgen/blob/main/guide/src/examples/raytrace.md); [WebAudio #1848 (snippet)](https://github.com/WebAudio/web-audio-api/issues/1848)
- Denormals:
  - Wasm scalar floating point must handle subnormals correctly; only SIMD ops are allowed to flush them (snippet) — [WebAssembly/simd #2 (snippet)](https://github.com/WebAssembly/simd/issues/2)
  - A proposal to expose FTZ/DAZ control in Relaxed SIMD, opened 2021-08-06, is still open, with no existing mechanism — [WebAssembly/design #1429](https://github.com/WebAssembly/design/issues/1429)
- `audio_thread_priority` is a no-op on wasm — [lib.rs](https://github.com/mozilla/audio_thread_priority/blob/master/src/lib.rs)
- RTSan only supports Linux, macOS and iOS — [rtsan-standalone-rs](https://github.com/realtime-sanitizer/rtsan-standalone-rs)

### Inferences
**Recommended messaging pattern for a Rust DAW (native and web shared-memory builds)**
1. **UI → audio commands:** `rtrb` SPSC of small `Copy`/POD enums (param changes with sample offsets, transport commands). Prefer this over `crossbeam-channel`: if the UI thread ever blocks in `recv()` on a crossbeam channel that the audio thread sends on, the audio thread's `try_send` will take a `std::sync::Mutex` to wake it. Unbounded crossbeam senders also allocate.
2. **Heavy state** (new graph, sample buffers, plugin instances):
   - Build it on a non-RT thread and send ownership as `Box`/`basedrop::Owned` pointers through the SPSC queue.
   - The audio thread swaps it in and sends the old object back on a return queue, or lets basedrop's collector free it.
   - With `arc-swap`, the audio thread can load snapshots lock-free, but it must never be the thread that drops the last `Arc`.
3. **Audio → UI** (meters, playhead, scopes): `triple_buffer` for latest-value state, and `rtrb`/`ringbuf` for sample streams. On the web, the same Rust structures work only with the shared-memory (`+atomics`) build. Otherwise mirror them with ringbuf.js over SAB.
4. **Scalar params:** `AtomicU32` with `f32::to_bits`, or `atomic_float`, is enough for "latest value wins." For sample-accurate automation, use queued events instead.
5. **Enforcement:**
   - Wrap the process callback in `assert_no_alloc` in debug builds (native and wasm, since it is only a global allocator).
   - Run RTSan in Linux and macOS CI.
   - Enable cpal `realtime`/`realtime-dbus`.
   - Handle denormals: FTZ/DAZ via `no_denormals` natively; on wasm, explicit flushing or a tiny DC/noise offset in feedback paths (filters, reverbs), because hardware FTZ cannot be enabled there.

On non-atomics wasm (the stable path), `std::sync::atomic` types still compile, but there is only one thread per module instance. rtrb and friends are then pointless across the UI/worklet boundary; the boundary becomes MessagePort or SAB+JS.

### Gaps
- No primary source was found for whether `no_denormals` compiles as a no-op or fails on `wasm32`.
- No current Rust-specific "RT audio best practices" long-form article (2025–2026) was retrieved. Billy Messenger's and Micah Johnston's blogs were only reachable as snippets or blocked.
- `assert_no_alloc` appears unmaintained (last release 2021). No maintained fork was confirmed.

---

## Q5. Timing and sync: sample-accurate scheduling, transport/clock design, MIDI clock, Ableton Link

### Takeaway
Sample accuracy comes from the engine, not the I/O layer:
- Keep a monotonically increasing sample counter owned by the audio thread.
- Timestamp every event in samples, or in musical time converted to samples at block start.
- Split each block at event offsets.
- Map to wall-clock and latency via cpal's `StreamTimestamp` (now latency-inclusive on most hosts) or, on the web, `currentFrame`/`currentTime`.

For Ableton Link, `rusty_link` (official abl_link wrapper, GPL-2.0+, active) is the usable option natively. A pure-Rust implementation (`ableton-link-rs`, GPL-3.0) exists but has a lagging crates.io release. Link cannot run in a browser. midir covers MIDI I/O, including Web MIDI.

### Cited Findings
- cpal timing:
  - 0.18.0 added `StreamTrait::now()` "to query the current instant on the stream's clock" and reworked `StreamInstant` to mirror `std::time::Instant`/`Duration`.
  - Callback timestamps now include hardware latency on WASAPI ("hardware pipeline latency"), CoreAudio ("device latency and safety offset"), ASIO ("driver-reported hardware latency") and JACK ("precise hardware deadline", "port latency" in 0.18.2). WebAudio includes "base and output latency."
  - 0.18.2: "Timestamps now stay monotonic across device and graph changes."
  - Unreleased: `StreamTimestamp` with `callback` and `device` fields.
  — [cpal CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)
- Web: `AudioWorkletGlobalScope.currentFrame` is "the ever-increasing current sample-frame of the audio block being processed… incremented by 128," and `currentTime` equals the context's `currentTime` — [MDN currentFrame](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletGlobalScope/currentFrame); [MDN AudioWorkletGlobalScope](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletGlobalScope)
- Clock-design reference (Firewheel):
  - The seconds clock is read from the OS audio API and "correctly accounts for any output underflows."
  - The sample clock "does not correctly account for any output underflows."
  - The musical clock is started, paused and stopped by the user and counts `f64` beats.
  - Events are scheduled with `EventDelay::DelayUntilSeconds(...)`.
  — [Firewheel DESIGN_DOC](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)
- kira provides "a clock system for precisely timing audio events" (tick-based clocks) — [kira README](https://crates.io/crates/kira)
- Emscripten Wasm Audio Worklets expose AudioParams with "sample precise accuracy" (snippet) — [Emscripten docs (snippet)](https://emscripten.org/docs/api_reference/wasm_audio_worklets.html)
- **Ableton Link**
  - `rusty_link` 0.4.9 (2026-04-02; 0.4.7 and 0.4.8 in Jan 2026), GPL-2.0-or-later. It wraps "the official C 11 wrapper extension, abl_link" and needs CMake 3.14+. Thread- and realtime-safety doc comments from `abl_link.h` are copied, so it is known which functions are safe to call from the audio callback — [rusty_link README](https://crates.io/crates/rusty_link); [GitHub](https://github.com/anzbert/rusty_link)
  - `ableton-link-rs`: "A native Rust implementation" (Tokio-based; macOS, Linux and Windows; `no_std` core types) with an optional LinkAudio feature ("Stream PCM audio between Link peers"), GPL-3.0.
    - crates.io latest is 0.1.2 (2025-08-19), while the GitHub README already references `0.3.0`, so the published crate lags the repo.
    — [ableton-link-rs README](https://github.com/anweiss/ableton-link-rs); [crates.io](https://crates.io/crates/ableton-link-rs)
  - `ableton-link` 0.1.0 (2019-04-30): **abandoned** — [crates.io](https://crates.io/crates/ableton-link)
  - Also: `esp-idf-ableton-link` 0.1.0 (ESP32) and `prat-clock` 0.1.1 ("Ableton-Link-compatible musical clock") — [crates.io search](https://crates.io/search?q=ableton)
- **MIDI**
  - `midir` 0.11.0 (2026-04-18) supports ALSA, WinMM, CoreMIDI, WinRT, JACK, "Web MIDI (Chrome, Opera, perhaps others browsers)" and Android (API 29+). There is a `coremidi_send_timestamped` feature — [midir README](https://github.com/Boddlnagg/midir); [crates.io](https://crates.io/crates/midir)
  - `wmidi` 4.0.11 (2026-04-19) handles MIDI message types — [crates.io](https://crates.io/crates/wmidi)

### Inferences
**Transport design for native and web**
- The audio thread owns `sample_pos: u64` and the transport state: tempo map, play/stop and loop points.
- The UI sends commands tagged with a target sample, or "ASAP"; these are applied at the start of the next block.
- Musical-time events are converted to sample offsets inside the block using the tempo map, so both targets behave identically.
- On web, `currentFrame` gives an equivalent monotonically increasing sample counter.

**Latency compensation:** use cpal's latency-inclusive `StreamTimestamp` (native) and `outputLatency`/`baseLatency` (web; Safari lacks `outputLatency`) to align the UI playhead with audible output.

**Collaboration sync across machines:** Link (via `rusty_link`) is ideal natively but impossible in the browser, because Link peer discovery needs UDP on the LAN. A collaborative web DAW needs its own network clock sync (e.g. an NTP-style offset estimation over WebSocket/WebRTC) mapped onto the sample clock. Link's GPL licensing (or Ableton's proprietary licence) must also be considered for distribution.

### Gaps
- No Rust crate for MIDI Clock (24 PPQN) generation or chase with jitter smoothing was identified. It would likely be hand-rolled on top of midir.
- The Link protocol version wrapped by rusty_link 0.4.9 was not confirmed (its CHANGELOG could not be fetched).
- No source was found on how well ableton-link-rs interoperates with official Link peers.
