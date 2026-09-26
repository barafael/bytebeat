# Rust stack outplays No Man's Sky's ByteBeat

Build the miniature collaborative DAW as **one pure-Rust engine crate** hosted two ways. The engine is a small custom transport and step sequencer that drives **fundsp** voices and effects and a home-grown **bytebeat bytecode VM**. On desktop, **cpal 0.18** hosts it. In the browser, a hand-rolled, stable-Rust **AudioWorklet** hosts it. Around the engine sit a **Dioxus 0.7** UI, a **Loro 1.16** CRDT song document synced through a thin **axum** WebSocket server, **symphonia/rubato/hound** for samples and **midir** for MIDI.

The No Man's Sky ByteBeat Device is a modest target. It has one voice per device, a 7-note melody range, 4/8/16 steps, a three-part drum machine, at most eight linked devices and a song state of about 45 bytes, and **no real-time co-editing at all** ([Hello Games](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/); [ByteBeat Catalogue](https://nomanssky.fandom.com/wiki/ByteBeat_Catalogue)). DSP breadth is therefore not the hard part. Three plumbing gaps are:

- Rust has no DAW engine. Firewheel, the best graph engine, explicitly disclaims that role ([Firewheel DESIGN_DOC](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)).
- The browser forces a choice between stable-Rust message passing and nightly shared memory with cross-origin isolation ([cpal README](https://github.com/RustAudio/cpal/blob/master/README.md)).
- Dioxus desktop puts the UI in a system webview. Rust cannot draw to a `<canvas>` there, and the `eval` bridge freezes on large payloads ([Dioxus issue #1915](https://github.com/DioxusLabs/dioxus/issues/1915)).

The coordinator proposed keeping ordinary controls as DOM/SVG in RSX and sending all high-rate visuals through one shared canvas module fed by compact binary frames: a SharedArrayBuffer on web, a non-`eval` channel on desktop. That is the right shape, but **the desktop binary channel is unverified and should be the first spike**. Keep third-party plugin hosting out of v1. Every surveyed Rust music app with a browser build drops it there, and opaque plugin state is what made Meadowlark's own design doc doubt that real-time collaboration was feasible ([Meadowlark DESIGN_DOC](https://github.com/MeadowlarkDAW/Meadowlark/blob/old/DESIGN_DOC.md)).

## The ByteBeat Device is a 45-byte, eight-voice groovebox, not a formula synth

No Man's Sky added the ByteBeat Device in **Update 2.24 on 16 December 2019**. It is a powered base part that "will immediately begin to produce sound" from procedurally generated presets. A Sequencer UI covers melody, drums, arpeggiator, octave, key and tempo. An "Advanced Waveform UI" exposes "the mathematical operators at the heart of their sound" ([Hello Games](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/); [TheSixthAxis patch notes](https://www.thesixthaxis.com/2019/12/16/no-mans-sky-beyond-update-2-24-5-beatbyte-patch-notes/)). The only substantive revision came in **Prisms (3.5, June 2021)**. It added a ByteBeat Library and player, the ability to send tracks to nearby players, and "meatier" drums ([HITC Prisms notes](https://www.hitc.com/en-gb/2021/06/03/no-mans-sky-prisms-update-patch-notes/)). No later changes were found through September 2026.

The research proxy blocked the wiki, Steam and guide sites. Most of the finer detail below therefore comes from **search snippets of those pages, not full reads**. The official Hello Games wording is the most reliable layer.

A community guide likens each device to "a singer in a choir", i.e. one voice ([Steam guide](https://steamcommunity.com/sharedfiles/filedetails/?id=1941411630)). Its controls:

- **Melody grid:** a **7-note range** at **4, 8 or 16 steps**, with one note value per device and no ties or variable lengths ([Steam petition](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/)).
- **Arpeggiator:** rows of on/off dots, with a note speed from 1 to 4 ([Steam guide](https://steamcommunity.com/sharedfiles/filedetails/?id=1941411630)).
- **Drum machine:** hi-hat, snare and kick, with **13 sounds each** ([Gamer Guides](https://www.gamerguides.com/no-mans-sky/guide/walkthrough/tips-and-tricks/how-to-build-and-use-a-bytebeat-device), snippet).
- **Envelope editor:** shapes how steeply notes start and end.
- **Globals:** BPM, key, octave, time signature, pitch, volume and attenuation.

Devices link by snapping together or through a ByteBeat Cable, **up to eight** at a time. Linked devices share tempo and key but keep their own notes and sounds, and a **Synchroniser** tab schedules when each one plays ([Hello Games](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/); [GamesRadar+](https://www.gamesradar.com/no-mans-skys-new-bytebeat-synthesizer-lets-you-make-custom-tracks-to-play-in-your-base/)). A **ByteBeat Switch** turns the rhythm into power pulses that drive lights ([NMS Resources](https://www.nomansskyresources.com/bytebeat)). A song serializes to a **60-character Base64 string, about 45 bytes**, and the personal library holds **eight** of them ([ByteBeat Catalogue](https://nomanssky.fandom.com/wiki/ByteBeat_Catalogue)).

Several details could not be found anywhere: the BPM range, the scale list, the octave range, and the operator set of the waveform tree. A "ByteBeat Amplifier" part does not appear to exist.

That 45-byte budget shows the device is **not a text-formula bytebeat engine**. Classic bytebeat, popularized by viznut in October 2011, evaluates one integer expression of a sample counter `t` about 8,000 times a second and plays the low 8 bits as unsigned PCM. `t*((t>>12|t>>8)&63&t>>4)` is the canonical example ([kragen/viznut-music](https://github.com/kragen/viznut-music); [viznut paper, snippet](https://ar5iv.labs.arxiv.org/html/1112.1368)). Modern web players go further, with signed/floatbeat/funcbeat modes, 8–48 kHz rates, stereo, RPN syntax and URL sharing ([greggman/html5bytebeat](https://github.com/greggman/html5bytebeat); [dollchan composer](https://github.com/SthephanShinkufag/bytebeat-composer)).

NMS instead pairs a step sequencer, arpeggiator and drum machine with what reads as a small operator tree over simple waveforms that sets each note's timbre. This is an inference from the 7-note grid, the key/BPM controls and the tiny state size, because Hello Games never published the operator list. **Matching NMS therefore means a groovebox plus an operator-tree synth. Exceeding it means adding a real formula instrument locked to the transport.**

Player complaints point the same way. They include the 7-note range ("2 or 3 octaves per device being much better still"), confusing 4/8/16 modes, fixed note lengths, only eight library slots, and a UI that feels "obscure or frustrating" ([Steam petition](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/)). Collaboration in NMS is strictly asynchronous: visit a base, save its output, send a track to a nearby player ([HITC](https://www.hitc.com/en-gb/2021/06/03/no-mans-sky-prisms-update-patch-notes/)).

| Area | NMS ByteBeat baseline | "And then some" target | Delivered by |
|---|---|---|---|
| Voices and tracks | 1 voice per device, ≤8 linked, shared tempo and key | Many tracks, polyphony, per-track key override | Engine track model, fundsp voices |
| Sequencer | 4/8/16 steps, 7 notes, one note value | Chromatic multi-octave grid and piano roll, note lengths, velocity, probability, swing | Engine step sequencer, Dioxus DOM grid |
| Arpeggiator | Dot rows, speed 1–4 | Modes, synced rates, gate, octave span | Engine |
| Drums | Hi-hat/snare/kick, 13 sounds each | Synth and sample kits, unlimited lanes | fundsp drum presets, symphonia samples |
| Sound design | Envelope shape, operator tree, random presets | ADSR, filters, FX, operator tree **plus** bytebeat/floatbeat/funcbeat formula instrument | fundsp, custom VM |
| Arrangement | Synchroniser tab | Pattern chains, song mode, automation | Engine and Loro document |
| Control out | Switch pulses lights | MIDI/OSC out, trigger lanes, visual hooks | midir, rosc |
| Sharing | 60-char code, 8-slot library, send to nearby player | Unlimited library, share links, fork, WAV/FLAC/Opus export | Sync server, hound/flacenc/opus-rs |
| Collaboration | Asynchronous only | Live co-editing, presence, shared transport | Loro, axum WebSocket, clock sync |

## cpal owns native I/O, but the browser needs a hand-built AudioWorklet host

**cpal 0.18.2 (16 August 2026)** is the maintained default for native audio I/O. Its hosts are:

- **Windows:** WASAPI, with optional ASIO and JACK.
- **macOS:** CoreAudio, with optional JACK.
- **Linux:** PipeWire, then PulseAudio, then ALSA by default. The native PipeWire and PulseAudio hosts are new in 0.18.0 ([cpal README](https://github.com/RustAudio/cpal/blob/master/README.md); [cpal CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)).

The 0.18 line now reports xruns on CoreAudio, WASAPI, PipeWire and AAudio. Its callback timestamps include hardware latency on WASAPI, CoreAudio, ASIO and JACK. That is exactly what a UI playhead needs to line up with what the listener hears ([CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)).

Two Linux pitfalls matter:

- ALSA's `BufferSize::Default` can land anywhere from a PipeWire quantum up to `u32::MAX`, so request `Fixed` sizes.
- PipeWire's ALSA/JACK/Pulse compatibility layers can resample silently and glitch "without an xrun being reported", so enable the native `pipewire` feature ([README](https://github.com/RustAudio/cpal/blob/master/README.md)).

Master is already 0.19.0-dev. It has breaking changes (`play` renamed to `start`, a merged `CallbackInfo`, `Send + Sync` device traits, Windows 10 minimum) and adds a real duplex API ([CHANGELOG](https://github.com/RustAudio/cpal/blob/master/CHANGELOG.md)). Pin 0.18 and budget a migration.

The alternatives are narrower. rtaudio 0.8 and interflow are native-only ([crates.io rtaudio](https://crates.io/crates/rtaudio); [interflow](https://github.com/SolarLiner/interflow)). rodio, kira and tinyaudio are playback libraries. miniaudio (2020) and oddio (2023) are stale ([crates.io oddio](https://crates.io/crates/oddio)).

**The web is where the real decision sits.** cpal's default `WebAudio` host schedules `AudioBufferSourceNode`s from the main thread. It defaults to 2,048-frame buffers (about 43 ms at 48 kHz) and uses the deprecated `ScriptProcessorNode` for input, so any UI jank becomes an audible dropout ([cpal webaudio host](https://github.com/RustAudio/cpal/blob/master/src/host/webaudio/mod.rs); [MDN ScriptProcessorNode](https://developer.mozilla.org/en-US/docs/Web/API/ScriptProcessorNode)).

cpal's official `audioworklet` host and **firewheel-web-audio 0.5** do run on the audio thread, but both carry the same requirements:

- nightly Rust with `-Zbuild-std`;
- `+atomics`;
- explicit `--shared-memory --import-memory` link arguments, needed since rust-lang/rust#147225 stopped atomics implying shared memory;
- COOP/COEP cross-origin isolation ([cpal audioworklet example](https://github.com/RustAudio/cpal/tree/master/examples/audioworklet); [rust#147225](https://github.com/rust-lang/rust/pull/147225); [firewheel-web-audio](https://crates.io/crates/firewheel-web-audio)).

firewheel-web-audio also warns that `credentialless` COEP "may not work on Safari". GitHub Pages can only get isolation through a service-worker shim ([coi-serviceworker](https://github.com/gzuidhof/coi-serviceworker)).

**Recommendation: use the stable path that Glicol and ShoopDaLoop already ship.** Compile the engine to its own single-threaded `wasm32-unknown-unknown` module. Fetch its bytes on the main thread, because `fetch()` and dynamic `import()` are forbidden inside `AudioWorkletGlobalScope` ([Chrome worklet design pattern](https://developer.chrome.com/blog/audio-worklet-design-pattern)). Hand the bytes to the processor and instantiate the module there ([glicol.js](https://github.com/chaosprint/glicol/blob/main/js/src/glicol.js); [ShoopDaLoop technical README](https://github.com/SanderVocke/shoopdaloop/blob/master/src/rust/shoopdaloop/README.md)).

Keep the worklet glue free of wasm-bindgen string handling. A September 2026 report found that `TextDecoder` is missing in Chromium 151's worklet scope and that a posted `WebAssembly.Module` arrives as `messageerror`. Transferring raw bytes fixed both. This is a single low-profile source ([AnthonE/Gates PR #152](https://github.com/AnthonE/Gates/pull/152)).

Send control over `port.postMessage`. Treat Paul Adenot's SharedArrayBuffer ring buffer as a progressive enhancement when `crossOriginIsolated` is true. It is wait-free and allocation-free, and it needs COOP/COEP but not wasm threads ([ringbuf.js](https://github.com/padenot/ringbuf.js)).

Two further constraints apply. Start audio only after a user gesture, as ShoopDaLoop does. And do not ship the app inside an iframe: Chromium gives the worklet a real-time thread only when the AudioContext belongs to a top-level document (snippet, [Chromium platform-architecture-dev](https://groups.google.com/a/chromium.org/g/platform-architecture-dev/c/0wS55qzWMaw/m/B5A1ByIeAwAJ)).

The web's timing envelope is fixed and coarse:

- Worklet blocks are **128 frames** ([MDN process()](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletProcessor/process)). Chrome posted an Intent to Ship for a configurable `renderSizeHint` in July 2026, but Firefox and Safari show no signal ([Intent to Ship, snippet](http://www.mail-archive.com/blink-dev@chromium.org/msg17051.html); [WebKit standards-positions #662](https://github.com/WebKit/standards-positions/issues/662)).
- Measured output latency is roughly **15 ms in Firefox versus 24 ms in Chrome**, with echo round trips of **62 ms versus 124 ms**. These figures are dated snippets ([jamieonkeys](https://www.jamieonkeys.dev/posts/web-audio-api-output-latency/); [Jeff Kaufman](https://www.jefftk.com/p/audioworklet-latency-firefox-vs-chrome)).
- Safari implements `baseLatency` but not `outputLatency`.

Design the engine to accept any host block size and split it into sub-blocks of at most 64 samples at event boundaries. Position the web build for composing and collaborating, and the native build for low-latency playing and recording.

Sample accuracy comes from the engine, not from the I/O layer. The audio thread owns a monotonically increasing `u64` sample counter and the transport. Every event carries a sample offset. cpal's latency-inclusive `StreamTimestamp` (native) and `currentFrame` (web) map that counter to wall-clock time ([MDN currentFrame](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletGlobalScope/currentFrame)).

For the document, adopt Meadowlark's time-keeping scheme: musical time is the source of truth, stored as fixed-point ticks of **1/1,476,034,560 beat**, with a "superclock" of **1/282,240,000 s** (the same unit Ardour uses) for audio-clip internals. Convert only from the source of truth, never back ([BillyDM, Accurate Timekeeping](https://billydm.github.io/blog/time-keeping/)).

## No crate is a DAW engine, so wrap fundsp in a thin custom core

**fundsp 0.23.0 (January 2026, MIT/Apache)** has the richest DSP vocabulary in Rust ([fundsp prelude](https://github.com/SamiPerttu/fundsp/blob/master/src/prelude.rs)):

- **Oscillators:** PolyBLEP and bandlimited oscillators, wavetables, Karplus-Strong.
- **Filters:** Moog ladder, SVF, biquad and "dirty" filters.
- **Effects:** four stereo reverbs, chorus, flanger, phaser, delays, waveshapers and limiters, oversampling, FFT convolution.
- **Envelopes:** `adsr_live`.
- **Drum presets:** `bassdrum`, `snaredrum` and `cymbal`.

Its `Net` splits into a frontend you edit and a real-time-safe backend you `commit()` to. Old networks are shipped back over a `thingbuf` channel so nothing is freed on the audio thread, and `crossfade` swaps nodes without clicks ([fundsp README](https://github.com/SamiPerttu/fundsp#readme); [realnet.rs](https://github.com/SamiPerttu/fundsp/blob/master/src/realnet.rs)). Blocks are 64 samples with explicit SIMD via `wide`, which has `simd128` paths for the browser. `no_std` works by dropping the `std`, `files` and `fft` features ([CHANGES.md](https://github.com/SamiPerttu/fundsp/blob/master/CHANGES.md)).

The maintainer's own FUTURE list names the gaps: **no compressor, no feedback loops in `Net`, no looping in `Sequencer`, and drum sounds that need improving** ([FUTURE.md](https://github.com/SamiPerttu/fundsp/blob/master/FUTURE.md)). `Sequencer` timing is in seconds, with no tempo map. Denormal flushing is x86-only ([denormal.rs](https://github.com/SamiPerttu/fundsp/blob/master/src/denormal.rs)). The bus factor is one person, and there was a 15-month gap between 0.20 and 0.21 ([crates.io fundsp](https://crates.io/crates/fundsp/versions)). No public demo of fundsp in an AudioWorklet was found, so the wasm build needs a spike.

**Firewheel 0.14 (September 2026)** is the most complete engine shell ([Firewheel README](https://github.com/BillyDM/firewheel#readme)). It has:

- a compiled DAG and three clocks (seconds, samples, musical);
- scheduled events and per-buffer silence flags;
- stream fault tolerance and "no mutexes";
- a sampler, SVF, freeverb and convolution nodes.

But it is explicitly **not a DAW engine**. It guarantees user parameter events are never discarded, whereas a DAW transport must be able to discard conflicting ones ([DESIGN_DOC](https://github.com/BillyDM/firewheel/blob/main/DESIGN_DOC.md)). It ships breaking releases roughly monthly (0.12 to 0.14 in two months). It is being upstreamed into Bevy, and it has no compressor, echo or EQ yet. Its web backend needs nightly Rust, is stereo-only, and lags the core by two minor versions ([firewheel-web-audio](https://github.com/CorvusPrudens/bevy_seedling/tree/master/crates/firewheel-web-audio)).

For a miniature DAW with a fixed topology (instrument, then insert chain, then sends, then master), a general DAG buys little. **Borrow Firewheel's clock, silence-flag and compile-schedule ideas, but don't depend on it in v1.**

The other candidates are references rather than foundations:

- **knyst** has sample-accurate scheduling and feedback edges, but has had no commits since November 2024 and about 131 downloads a quarter ([knyst](https://github.com/ErikNatanael/knyst)).
- **glicol_synth** proves the Rust-in-worklet architecture, but has had no release since April 2024 ([Glicol](https://github.com/chaosprint/glicol)).
- **web-audio-api-rs 1.7** clones Web Audio natively, compressor included. Its browser path is "experimental" and would mean keeping two implementations in sync ([web-audio-api-rs](https://github.com/orottier/web-audio-api-rs)).
- **kira 0.12.5** is a game mixer with clocks and a compressor, delay, EQ and reverb, not an arbitrary DSP graph ([kira](https://github.com/tesselode/kira#readme)).
- **HexoDSP/synfx-dsp** are GPL-3.0 and dormant ([crates.io hexodsp](https://crates.io/crates/hexodsp)).

Useful add-ons:

- **rustysynth 1.3.6** (MIT, no dependencies) for a General MIDI fallback voice ([rustysynth](https://crates.io/crates/rustysynth)).
- **rubato 5.0** for resampling. It is real-time-safe, but went through four major versions in 2026, and its hand-written SIMD is x86/ARM-only, so it runs scalar in the browser ([rubato](https://github.com/HEnquist/rubato#readme); [versions](https://crates.io/crates/rubato/versions)).
- **timestretch 0.15** for later time-stretching: pure Rust, MIT, allocation-free, with 12.7 ms keylock delay ([timestretch](https://github.com/robmorgan/timestretch-rs#readme)).
- Avoid **rubberband**'s GPL-2.0+ bindings ([rubberband](https://crates.io/crates/rubberband)). MIT **signalsmith-stretch** is a native-only C++ wrapper ([signalsmith-stretch](https://crates.io/crates/signalsmith-stretch)).

**The bytebeat instrument must be home-grown.** The existing crates don't fit: chaosprint's `bytebeat` is a 2023 CLI with a toy evaluator, and `bytebeat-rs` JIT-compiles through LLVM, which cannot run in wasm ([bytebeat-rs](https://github.com/chaosprint/bytebeat-rs); [bytebeat-cli](https://crates.io/crates/bytebeat-cli)). Generic evaluators are traps too. **evalexpr switched to AGPL-3.0 at v12** ([evalexpr versions](https://crates.io/crates/evalexpr/versions)), and meval and fasteval are unmaintained.

Build it in five steps:

1. Parse a restricted grammar on the control thread. It covers integer and float operators, bitwise operators, ternaries, a few math functions, `t`, transport variables and user parameters, with bounded depth and no loops.
2. Compile it to a flat bytecode array.
3. Run it in an allocation-free stack VM with wrapping arithmetic and defined division by zero.
4. Evaluate it over 64-sample blocks, at 8 kHz by default, resampled to the device rate.
5. Match dollchan's JavaScript semantics (int32 bitwise operators, float `/`) so shared formulas sound the same. dollchan simply compiles formulas with `new Function` inside its worklet ([dollchan audio-processor](https://github.com/SthephanShinkufag/bytebeat-composer/blob/master/src/audio-processor.mjs)).

**cranelift-jit 0.136** (native) and **wasm-encoder 0.259** (browser) are later optimizations, not requirements ([cranelift-jit](https://crates.io/crates/cranelift-jit); [wasm-encoder](https://crates.io/crates/wasm-encoder)). The same VM, extended with oscillator primitives driven by note phase, can also implement the NMS-style operator tree. One component then covers both "match" and "exceed". That last point is a design inference.

The engine crate then sets its own rules:

- Keep the fixed topology and preallocated voice pools per track.
- Have the step sequencer and arpeggiator emit sample-offset note events.
- Port a feed-forward compressor from kira or web-audio-api (both permissive).
- Keep everything `no_std`-friendly so the same crate builds natively and for wasm.

This is the Glicol and Firewheel pattern of one engine running everywhere. Futureboard Studio argues the opposite: "The two engines are intentionally separate … Shared concepts live in descriptors and IDs … not in shared realtime code" ([Futureboard ARCHITECTURE.md](https://github.com/futureboard/futureboard-studio/blob/main/ARCHITECTURE.md)). That warning applies to pro features such as plugin hosting and low-latency duplex I/O. The answer is to keep those in a native host shell *around* the shared engine, never inside it.

## Real-time safety means wait-free queues in, triple buffers out, and no frees on the audio thread

Ross Bencina's rules still govern: never block, never allocate, never call anything unbounded in the callback ([Real-time audio programming 101](http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing)). The Rust toolkit that implements them is mature:

- **rtrb 0.4** (wait-free SPSC, tested under Miri and ThreadSanitizer) for UI-to-engine commands and engine-to-UI events ([rtrb](https://github.com/mgeier/rtrb)).
- **triple_buffer 9.0** for "latest value" meter and visual frames ([triple_buffer](https://crates.io/crates/triple_buffer)).
- **basedrop 0.1.3**, or a return queue, so graphs, sample buffers and plugin instances are dropped on a collector thread ([basedrop](https://crates.io/crates/basedrop)).
- Atomics for "latest value wins" scalars.
- **arc-swap** for lock-free snapshot loads, but only if the audio thread never drops the last `Arc`.

Avoid crossbeam-channel on the audio thread. Its bounded `try_send` takes a `std::sync::Mutex` to wake a blocked receiver, and its unbounded sender allocates ([crossbeam waker.rs](https://github.com/crossbeam-rs/crossbeam/blob/master/crossbeam-channel/src/waker.rs)).

Enforce the rules mechanically:

- **Debug builds:** wrap `process` in `assert_no_alloc`. It is a global-allocator trick that works on wasm too, but it has not been released since 2021 and is a no-op in release builds by default ([assert_no_alloc](https://crates.io/crates/assert_no_alloc)).
- **CI:** run **rtsan-standalone 0.3** on Linux and macOS with `#[nonblocking]` markers ([rtsan-standalone-rs](https://github.com/realtime-sanitizer/rtsan-standalone-rs)).
- **Thread priority:** enable cpal's `realtime`/`realtime-dbus` features. These wrap Mozilla's `audio_thread_priority` (MMCSS "Pro Audio" on Windows, the Mach time-constraint policy on macOS, rtkit or `SCHED_FIFO` on Linux) and now return `RealtimeDenied` instead of printing ([audio_thread_priority](https://github.com/mozilla/audio_thread_priority/blob/master/src/lib.rs)).
- **Denormals:** use `no_denormals` natively.

The browser changes the rules:

- `audio_thread_priority` is a no-op on wasm.
- `Atomics.wait` is forbidden in worklets, and the main thread cannot block ([WebAudio #1848, snippet](https://github.com/WebAudio/web-audio-api/issues/1848); [wasm-bindgen threads guide](https://github.com/wasm-bindgen/wasm-bindgen/blob/main/guide/src/examples/raytrace.md)).
- Wasm has **no FTZ/DAZ control** (the proposal has been open since 2021), so feedback paths such as reverbs and IIR filters need explicit denormal guards ([WebAssembly/design #1429](https://github.com/WebAssembly/design/issues/1429)).

The stable single-threaded worklet also creates a subtle trap. **Everything the worklet does happens on the audio thread between quanta**, including decoding commands and rebuilding a fundsp `Net`. Rust objects cannot be handed across wasm instances. Mitigate it three ways:

- Decode samples on the main thread or in a Worker, and transfer `Float32Array`s in (Glicol's `loadsample` message).
- Reserve memory up front. ShoopDaLoop reserves it when a loop is armed "so dormant loop slots do not exhaust Wasm memory" ([ShoopDaLoop](https://github.com/SanderVocke/shoopdaloop/blob/master/src/rust/shoopdaloop/README.md)).
- Keep graph edits small and crossfaded.

This is the strongest argument for a later move to the nightly shared-memory build, where the UI thread could construct graphs and pass pointers. That move is inference, not measured need.

## Samples and MIDI are pure Rust everywhere; plugins stay native and optional

**Symphonia 0.6.1 (August 2026, MPL-2.0)** is a pure-Rust decoder and demuxer ([Symphonia](https://github.com/pdeljanov/Symphonia)):

- **Supported:** WAV, AIFF, CAF, FLAC, MP3, Ogg Vorbis, ALAC, AAC-LC, ADPCM, MKV/WebM and MP4.
- **Missing:** **no Opus and no encoding**.
- **Web:** a WASM API is only "planned", so a `wasm32` build test comes first.

Running the same decoder on both targets matters for collaboration determinism. The browser's `decodeAudioData` resamples to the context rate, and its codec support varies by browser. The web path is: File/drop, then `arrayBuffer()`, then a `Cursor` into symphonia, then **rubato** to the project rate, then a min/max peak pyramid for waveform drawing. **Symphonium 0.13** wraps the same pipeline on native with load-time resampling ([crates.io symphonium](https://crates.io/crates/symphonium)). creek's disk streaming is native-only and unnecessary at this scale ([creek](https://crates.io/crates/creek)).

For export:

- **WAV:** **hound 3.5.1**, stable though dormant since 2023 ([hound](https://crates.io/crates/hound)).
- **FLAC:** **flacenc 0.5.1** ([flacenc](https://crates.io/crates/flacenc)).
- **Opus:** **opus-rs 0.1.34** plus `ogg`. It is a pure-Rust libopus 1.6 port that claims wasm support; the "production-ready" label is its author's own ([opus-rs](https://crates.io/crates/opus-rs)).
- **MP3 and Vorbis:** need C libraries. LAME's crate is **LGPL-3.0** ([mp3lame-encoder](https://crates.io/crates/mp3lame-encoder)), so keep these native and optional.

Offline bounce is just the engine crate run faster than real time: on a thread natively, in a Worker on the web. **rfd 0.17** covers dialogs. It is async-only on wasm, where saving triggers a browser download ([rfd](https://github.com/PolyMeilex/rfd)).

**midir 0.11** covers ALSA, JACK, CoreMIDI, WinMM/WinRT and Android, and pulls in its **Web MIDI** backend automatically on `wasm32` ([midir](https://github.com/Boddlnagg/midir); [midir deps](https://crates.io/crates/midir/0.11.0/dependencies)). Web MIDI works in Chromium browsers and Firefox 108+. **WebKit has declined to ship it**, so Safari on macOS and iOS has none ([Web MIDI in 2026, secondary](https://www.supersimplepiano.com/blog/web-midi-browser-compatibility-2026)). With a native engine on desktop, midir runs in the Rust process, so WKWebView's missing Web MIDI never matters there. On the web build, midir runs in the Dioxus main thread and forwards timestamped events to the worklet. Safari users get an on-screen or QWERTY keyboard.

For MIDI data:

- **wmidi 4.0** (allocation-free) for real-time messages ([wmidi](https://crates.io/crates/wmidi)).
- **midly 0.5.3** for Standard MIDI Files ([midly](https://crates.io/crates/midly)).
- **midi-msg 0.9** when MPE or MTC typing is needed ([midi-msg](https://crates.io/crates/midi-msg)).
- Defer `midi2`, which is early and warns of breaking changes.
- MIDI clock-in smoothing is hand-rolled; no crate exists.
- **rosc 0.11.4** encodes OSC ([rosc](https://crates.io/crates/rosc)). It is the modern ByteBeat Switch: trigger lanes out to lights and visuals over UDP natively, bridged through the collaboration WebSocket in the browser.

**Plugin hosting is the feature to defer.** The options are thin:

- **clack-host 0.2.0** is the only safe Rust host, and it hosts CLAP only ([clack](https://github.com/prokopyl/clack)).
- VST3 hosting means raw, unsafe COM bindings through the `vst3` crate ([vst3](https://crates.io/crates/vst3)).
- LV2's livi is work-in-progress ([livi](https://crates.io/crates/livi)).
- The one crate advertising out-of-process sandboxing, `plugin_host`, has stub bridges ([plugin_host](https://crates.io/crates/plugin_host)).

More fundamentally, collaborators won't all own the same plugins and browsers can't run them. Every surveyed Rust app with a web build drops hosting there: yadaw notes "libloading cant be used on wasm", and Futureboard's web surface "does not host native plugins" ([yadaw](https://github.com/mlm-games/yadaw); [Futureboard](https://github.com/futureboard/futureboard-studio/blob/main/ARCHITECTURE.md)). If hosting arrives later, make it native-only CLAP with freeze-to-audio, so every collaborator hears the result.

Exporting *our* instruments as plugins is now easy and permissive:

- **nice-plug 0.4.2** (ISC), the RustAudio-endorsed successor to nih-plug, now in maintenance mode. It uses MIT/Apache `vst3` bindings, possible since **Steinberg relicensed the VST 3 SDK under MIT in October 2025** ([nice-plug](https://crates.io/crates/nice-plug); [nih-plug](https://github.com/robbert-vdh/nih-plug); [Sonicstate](https://sonicstate.com/news/2025/10/30/vst-3-now-available-under-mit-license/)).
- **clack-plugin 0.2** with **clap-wrapper 0.3.1**, which gives CLAP plus VST3 and AUv2 ([clap-wrapper](https://crates.io/crates/clap-wrapper)).

The catch for this project: nice-plug's editor adapters are **egui and iced, not Dioxus** ([nice-plug-egui](https://crates.io/crates/nice-plug-egui)). A plugin build would need a generic host UI or a small egui editor. On the web, the equivalent is a Web Audio Modules 2.0 wrapper written as custom JS glue, since no Rust WAM SDK was found ([WAM API](https://github.com/webaudiomodules/api)).

## Dioxus works only if high-rate pixels bypass its reactive pipeline

Dioxus is the fixed GUI choice. It maps the user's HTML/CSS design workflow almost directly into RSX, and on the web it renders to the real DOM. That brings browser-native drag-and-drop, clipboard and shortcuts, and it supports Windows Narrator and IME ([boringcactus survey, snippet](https://www.boringcactus.com/2025/04/13/2025-survey-of-rust-gui-libraries.html)). Stable is **0.7.10 (30 July 2026)**, with **0.8.0-alpha.1** out a day later ([crates.io dioxus](https://crates.io/crates/dioxus)).

Its desktop constraints are real:

- The UI lives in a system webview via wry 0.57, while "your Rust code runs natively, which means that browser APIs are not available, so rendering WebGL, Canvas, etc is not as easy" ([Dioxus desktop guide, snippet](https://dioxuslabs.com/learn/0.7/guides/platforms/desktop/); [crates.io wry](https://crates.io/crates/wry)).
- `eval` "freezes both the WebView and the backend when the payload exceeds 100k elements in debug mode, and 1M elements in release mode" ([Dioxus #1915](https://github.com/DioxusLabs/dioxus/issues/1915)).
- **No shipped audio app on Dioxus was found.** The nearest analog is a 2026 blog series building a DAW as a Tauri webview over a Rust engine ([whoisryosuke](https://whoisryosuke.com/blog/2026/creating-a-daw-in-rust/)).

Meadowlark's author warns that DAW GUIs are uniquely hard. Meters force constant redraws, waveforms need peak search and per-pixel rendering, and zoomable timelines defeat caching ([DAW Frontend Development Struggles](https://billydm.github.io/blog/daw-frontend-development-struggles/)).

### The coordinator's split is right; its desktop pipe is the unknown

The proposed architecture holds up:

- **A native cpal engine on desktop** keeps ASIO, JACK and PipeWire, native MIDI, Ableton Link and a future path to plugin hosting.
- **A worklet engine on the web**, running the same engine crate.
- **Ordinary controls as DOM/SVG**: the ~1k-cell step grid, knobs, faders and arrangement.
- **One small shared canvas module for high-rate visuals**: meters, scopes, spectrum, waveforms and playhead.

Why the canvas split is right is an inference, but a strong one. On desktop, every Dioxus signal change turns into a batch of DOM mutations sent across the webview IPC. A playhead or a bank of meters updating at 60 Hz as signals would push dozens of mutation batches per second through that channel. A canvas module fed raw bytes takes that traffic out of the reactive pipeline entirely, and it is the same code on both targets.

Six refinements make it concrete. All of them are inferences and need prototyping:

1. **Define one versioned binary `VisFrame` layout** and use identical bytes on both targets. It holds a sequence counter, transport position, per-track peak/RMS, spectrum bins and scope samples. Natively the audio thread writes it into a `triple_buffer`. On the web the worklet copies it into a SharedArrayBuffer after each `process()`. Without cross-origin isolation, it falls back to transferable `postMessage` at display rate.
2. **Split desktop traffic into two classes.** Large, infrequent payloads (per-clip waveform peak pyramids, spectrogram history) go over a pull channel: a Dioxus asset or custom-protocol handler answering `fetch()` with an `ArrayBuffer`. **The API details and per-webview throughput are unverified.** Streaming frames go either through the same handler polled from `requestAnimationFrame`, or through a localhost WebSocket push with a per-session token. **Whether a custom-scheme webview origin may open that socket is also unverified.**
   The closest evidence comes from Tauri, the other major Rust webview shell. Raw binary IPC is roughly an order of magnitude cheaper than JSON: a 65 KB blob round-trips in 600 µs versus 6.7 ms, per the author's own benchmark ([tauri-wire](https://github.com/userFRM/tauri-wire)). But a 10 MB transfer was reported at about 5 ms on macOS versus about 200 ms on Windows ([Tauri #7127, snippet](https://github.com/tauri-apps/tauri/issues/7127)). Keep frames to a few KB, send peaks once per clip, and reserve `eval` for small, low-rate messages. Tauri's docs likewise say JSON events are "not designed for low latency or high throughput" ([Tauri calling-frontend docs](https://github.com/tauri-apps/tauri-docs/blob/v2/src/content/docs/develop/calling-frontend.mdx)).
3. **Lay a transparent canvas over the DOM step grid** with `pointer-events: none`. It draws the moving step highlight, the playhead and remote collaborators' cursors, so the 1,024 grid nodes change only when a user actually edits.
4. **Apply a routing rule.** Anything updating faster than roughly 10 Hz, or carrying more than a few hundred data points, goes to the canvas. Everything else stays in RSX.
5. **Handle pointer gestures locally.** Knob and fader drags and drag-painting across steps are handled by small pointer-capture helpers in the webview. They commit to Rust at a throttled rate and on release. Desktop events otherwise round-trip webview → Rust → webview, with latency nobody has measured. Committing on gesture end also follows Meadowlark's rule that uncommitted drags should not trigger "expensive updates … every single frame", and it keeps CRDT traffic sane.
6. **Write the module in Rust, compiled to a small wasm** that uses `web-sys` canvas calls, rather than in plain JS. It then shares the `VisFrame` types with the engine crate, and its draw logic can later be retargeted to Blitz. TypeScript is a fine alternative if the team prefers it.

### Running the web engine inside the desktop webview is a fallback, not a plan

The symmetric alternative runs the web build's worklet engine inside the desktop webview too, which makes the desktop app a thin browser wrapper. It has one host, one build and identical behaviour, and the canvas module reads the SharedArrayBuffer exactly as on the web.

The losses are large:

- **No pro audio backends.** No ASIO, JACK or PipeWire device selection, because audio flows through the webview engine.
- **No native plugin hosting**, ever.
- **No Web MIDI on macOS**, because WKWebView is WebKit, which declined Web MIDI. WebKitGTK is WebKit too; that is an inference, unverified. Forwarding midir events from Rust into the webview would re-add an IPC hop.
- **Unmeasured latency and priority.** WebKitGTK's audio latency and variance on Linux are unknown. So are whether custom-scheme webview origins can become `crossOriginIsolated`, and whether WebView2 gives the worklet Chromium's real-time thread.

It is worth keeping as a "compat mode" behind the same `EngineHandle` trait for early prototyping and for debugging parity. It should not be the primary desktop path.

### Dioxus Native is the long-term escape hatch

Dioxus 0.7 introduced **Dioxus Native**. It is built on **Blitz**, a modular HTML/CSS engine (stylo, taffy, parley, vello, wgpu, accesskit), and it paints without a webview. `dioxus-native` is at 0.8.0-alpha.1 and Blitz at 0.3.0-beta.2 ([Dioxus 0.7 release](https://github.com/DioxusLabs/dioxus/releases/tag/v0.7.0); [crates.io dioxus-native](https://crates.io/crates/dioxus-native); [Blitz](https://github.com/DioxusLabs/blitz)). Docs describe a `<canvas>` custom-paint hook, and an open issue asks how to reach a wgpu surface from it ([Dioxus #3725](https://github.com/DioxusLabs/dioxus/issues/3725)).

If it matures, Rust could paint meters directly in-process and the IPC problem disappears on desktop. It remains labelled experimental. On the web Dioxus still renders to the DOM, so the canvas module never goes away. That is why the draw code should be renderer-agnostic Rust today.

### What the other GUI frameworks would have offered

| Framework (latest stable) | Web model | What it offered | What Dioxus trades it for |
|---|---|---|---|
| egui/eframe 0.36.2 | wgpu canvas, WebGPU with WebGL fallback | One render path on both targets; richest audio widgets (egui_knob, armas-audio); file drop on both; nice-plug editor adapter | Immediate-mode relayout every frame once meters force repaint ([egui #3810](https://github.com/emilk/egui/discussions/3810)); frequent breaking releases; not HTML/CSS ([egui](https://github.com/emilk/egui)) |
| iced 0.14.0 | wgpu canvas | Retained Elm model; Canvas `Cache`; revived iced_audio 0.17.1 | No upstream accessibility ([iced #552](https://github.com/iced-rs/iced/issues/552)); roughly yearly releases ([iced 0.14](https://github.com/iced-rs/iced/releases/tag/0.14.0)) |
| Slint 1.18.1 | WebGL canvas, "not recommended" for general web apps | DSL tooling, commercial backing | No web accessibility; attribution clause in the royalty-free license ([Slint web docs](https://docs.slint.dev/latest/docs/slint/guide/platforms/web/); [Slint FAQ](https://github.com/slint-ui/slint/blob/master/FAQ.md)) |
| vizia 0.4.0 | Partial wasm | CSS-styled, AccessKit, plugin-oriented | Web not first-class ([vizia](https://github.com/vizia/vizia)) |
| Makepad 1.0.0 | WebGL | Shader widgets, Ironfish synth demo | Crates frozen since May 2025, bespoke tooling ([Makepad](https://github.com/makepad/makepad)) |
| gpui 0.2.2 | Experimental web (Feb 2026) | Zed-grade rendering | Web too new ([Zed PR #50228](https://github.com/zed-industries/zed/pull/50228)) |
| Freya 0.4.3 | None | Skia rendering | No browser target ([freya](https://crates.io/crates/freya)) |

What the user gives up is one GPU render path shared by both targets and a ready-made knob/fader ecosystem. What they gain is real DOM accessibility, CSS, a web build that is genuinely native to the browser, and a design pipeline that already works for them.

## Loro and a thin relay server cover co-editing; audio never crosses the wire

**Loro 1.16.2 (September 2026, MIT)** fits a DAW document best out of the box ([Loro README](https://github.com/loro-dev/loro); [Loro docs](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt)). It offers:

- **MovableList** for tracks and device chains. Simulated moves "fail in concurrent editing" by duplicating elements ([Loro list docs](https://github.com/loro-dev/loro-docs/blob/main/pages/docs/tutorial/list.mdx)).
- **LWW maps** for knob values and a **Counter**.
- A **local UndoManager** that "only undoes the local user's operations", plus an **EphemeralStore** for presence ([ephemeral docs](https://github.com/loro-dev/loro-docs/blob/main/pages/docs/tutorial/ephemeral.mdx)).
- **Time travel**.
- **Mergeable containers**, which fix the concurrent-child-creation "data loss" bug that Loro's own post reproduces in Loro, Yjs and Automerge alike.

It costs bytes. Loro's own docs cite a **~970 KB gzipped** WASM binary. In the published benchmark its updates are larger than Yjs's: 88 B versus 27 B per appended character, and 132 KB versus 49 KB for the concurrent map-set test. No size was measured for statically linking `loro` into a Dioxus wasm binary.

The alternatives each lose something:

- **Yrs 0.28** is lighter and interoperates with the Yjs ecosystem. It has origin-filtered undo and Awareness, but no counter, and its array move is absent from mainline JS Yjs ([y-crdt](https://github.com/y-crdt/y-crdt)).
- **Automerge 0.12** has the best typed wrapper, autosurgeon, but "undo/redo were cut for 1.0", which disqualifies it for a DAW ([automerge #985](https://github.com/automerge/automerge/issues/985)).

Plan a hand-rolled schema layer, because `lorosurgeon` is a 0.2 third-party crate ([lorosurgeon](https://crates.io/crates/lorosurgeon)).

The schema (an inference) maps the groovebox onto Loro like this:

- **Song:** a Map holding `tracks`, a MovableList of mergeable Maps.
- **Step grids:** a Map keyed `"step:pitch"` with LWW velocity values, so concurrent toggles merge cleanly.
- **Parameters:** LWW keys.
- **Clips:** a Map by UUID with an LWW `start_beat` field.
- **Samples:** `Map<blake3, meta>`.
- **Presence:** cursor, selected track and in-progress knob gestures go in the EphemeralStore, so collaborators can see (and optionally hear) a sweep before it is committed.

"CRDTs merge; they do not reject". Invariants such as no routing cycles and valid parameter ranges must therefore be re-validated after every merge. Put that validation in a shared Rust crate that runs in clients and server alike. It mirrors Audiotool's NEXUS, whose validator and consolidator are compiled to WASM (from Go) and shared between client and backend ([nexus wasm validator](https://github.com/audiotool/nexus/blob/main/src/document/backend/document-service/wasm-nexus-validator.ts)).

A Figma-style server-authoritative, per-property LWW model would settle knob conflicts just as well. But it would force you to re-implement ordering, undo and presence ([Figma blog, snippet](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/)).

Use a **star topology around a small Rust server**:

- **Server:** **axum 0.8.9** with WebSockets ([axum](https://crates.io/crates/axum)). It holds a Loro document per song as just another peer, persists snapshots and updates to SQLite or Postgres, and speaks **loro-protocol 0.3**. That protocol multiplexes rooms, fragments at 256 KiB, ships a Rust WebSocket server with SQLite snapshotting, and offers an end-to-end-encrypted mode ([loro-dev/protocol](https://github.com/loro-dev/protocol)).
- **Clients:** **tokio-tungstenite 0.30** natively and **gloo-net 0.7** in the browser, behind one `Transport` trait ([tokio-tungstenite](https://crates.io/crates/tokio-tungstenite); [gloo-net](https://crates.io/crates/gloo-net)).
- **Samples:** stay **out of the CRDT**. Store them in S3/R2 keyed by BLAKE3 hash and cache them in OPFS/IndexedDB or on disk. Loro itself advises against CRDTs for "large binary/media" ([Loro docs](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt)).
- **Auth and sharing:** per-document tokens enforce auth. Share codes map to `(doc, role, expiry)`. "Fork" copies a snapshot, following BandLab's familiar remix pattern ([BandLab blog](https://blog.bandlab.com/forking-and-collaboration-on-bandlab-explained/)).

Two options to skip for now:

- **y-sweet** has had no release since September 2025, after Jamsocket joined Modal ([y-sweet](https://crates.io/crates/y-sweet)).
- **P2P** can come later: **iroh 1.2** natively, though browsers are relay-only ([iroh CHANGELOG](https://github.com/n0-computer/iroh/blob/main/CHANGELOG.md); [iroh wasm docs, snippet](https://docs.iroh.computer/languages/wasm-browser)); **matchbox**, which still pins the pre-rewrite webrtc 0.17 ([matchbox](https://github.com/johanhelsing/matchbox)); and **WebTransport**, Baseline in all browsers since Safari 26.4 in March 2026, via the `web-transport` crate ([WebRTC.ventures, snippet](https://webrtc.ventures/2026/04/webtransport-is-now-baseline-what-it-means-for-real-time-media/); [web-transport](https://github.com/moq-dev/web-transport)).

For playback, scope v1 as **"listen along"**, not jamming. Every client renders audio locally from the shared document. Flok does the same with Yjs text: evaluate events go over a separate pub/sub channel ([Flok session.ts](https://github.com/munshkr/flok/blob/main/packages/session/lib/session.ts)). A server-anchored transport handles timing:

- An NTP-style ping/pong estimates each client's clock offset, as in IRCAM's `@ircam/sync`, which converts "to local time at the last moment" ([ircam-ismm/sync](https://github.com/ircam-ismm/sync)).
- Starts and tempo changes land on a bar boundary 100–200 ms in the future, so each engine schedules them sample-accurately.

Real-time jamming needs under about **25 ms** one way, which the internet rarely allows ([Wikipedia, snippet](https://en.wikipedia.org/wiki/Comparison_of_remote_music_performance_software)). A NINJAM-style interval-delayed mode is the realistic v2 ([Ninjam](https://en.wikipedia.org/wiki/Ninjam)).

On a LAN, **rusty_link 0.4.9** brings Ableton Link to desktop. But Link cannot run in browsers, and the bindings are **GPL-2.0-or-later** unless Ableton grants a proprietary license ([rusty_link](https://crates.io/crates/rusty_link); [Ableton/link](https://github.com/Ableton/link)).

## Rust's DAW graveyard teaches scope discipline

No Rust DAW is production-ready in September 2026. Across dozens of attempts, the plumbing is consistent: **cpal + rtrb + symphonia + midir**, with clack where plugins appear. The projects diverge on GUI, and that is also where they die.

| Project | Stack | Lesson |
|---|---|---|
| Meadowlark (~1.5k stars, hiatus since 2023) | vizia, then gtk4/Yarrow attempts; cpal, rtrb, basedrop | The GUI sank it. Keep source, GUI and real-time state separate and commit on gesture end. "Managing so many repositories is a headache." Someone advised building a tracker before a full DAW ([break post](https://billydm.github.io/blog/why-im-taking-a-break-from-meadowlark/)) |
| Firewheel 0.14 (Meadowlark spin-off) | DAG engine, WASM backends | A survivor, but explicitly game-oriented ([README](https://github.com/BillyDM/firewheel#readme)) |
| generic-daw (active, Sept 2026) | iced, cpal 0.18.2 with ASIO and PipeWire, rtrb, symphonia, clack-host | Protobuf project file, a typed and versioned document ([generic-daw](https://github.com/generic-daw/generic-daw)) |
| Futureboard Studio (2026) | GPUI native plus React/WASM worklet "Lite" plus collaboration server | Two deliberately separate engines; web lags and hosts no plugins ([ARCHITECTURE.md](https://github.com/futureboard/futureboard-studio/blob/main/ARCHITECTURE.md)) |
| ShoopDaLoop (active) | egui, fundsp, oxisynth; native plus a repository-owned AudioWorklet backend | Single raw-wasm host file in the worklet, audio starts on a user action, memory reserved per armed loop, 120 s browser recording cap ([ShoopDaLoop](https://github.com/SanderVocke/shoopdaloop/blob/master/src/rust/shoopdaloop/README.md)) |
| Glicol (~3k stars, quiet since 2025) | glicol_synth in an AudioWorklet, Yjs collaboration | Same engine on web, desktop, plugin and Bela; whole-graph diffing on re-evaluation ([Glicol](https://github.com/chaosprint/glicol)) |
| Chiptrack | Slint, cpal, midir | Each song carries a small wasm program, shared as gists; runs on web and GBA ([chiptrack](https://github.com/jturcotte/chiptrack)) |
| Audiotool NEXUS (2026 multiplayer) | Protobuf entity document, optimistic transactions | Server can reject transactions; client-side undo; shared WASM validator ([nexus document.ts](https://github.com/audiotool/nexus/blob/main/src/document/document.ts)) |
| Endlesss (closed 31 May 2024) | Cloud jamming | Server-only tools strand users when the company folds, so stay local-first ([MusicTech](https://musictech.com/news/gear/tim-exile-endlesss-app-shut-down/)) |

Four points carry over:

1. A miniature, NMS-scoped groovebox is exactly the "something simpler" Meadowlark's author was advised to build first.
2. First-party, schema-defined devices sidestep the plugin-state problem his design doc feared.
3. The Tauri-plus-Rust-engine blog series is the closest precedent for the Dioxus split.
4. Single-maintainer burnout (Typebeat, Cacophony, Glicol, Meadowlark) is the ecosystem's most consistent failure mode ([KVR thread, snippet](https://www.kvraudio.com/forum/viewtopic.php?t=626803); [areweaudioyet, stale since 2020](https://github.com/RustAudio/areweaudioyet)).

## The recommended stack, one choice per layer

| Layer | Choice (version as of 2026-09-26) | Targets | Why |
|---|---|---|---|
| UI shell | [dioxus](https://crates.io/crates/dioxus) 0.7.10; desktop via wry 0.57, web DOM | Both | Fixed decision; HTML/CSS design flow; DOM accessibility |
| High-rate visuals | Own `vis` module (Rust to wasm via web-sys canvas) reading a shared `VisFrame` layout | Both | Bypasses Dioxus reactivity and desktop IPC; one code path |
| Native audio I/O | [cpal](https://crates.io/crates/cpal) 0.18.2 with `asio`, `jack`, `pipewire`, `realtime` | Native | Pro backends, xrun and latency-aware timestamps |
| Web audio I/O | Own AudioWorkletProcessor plus stable single-threaded `engine.wasm`; [ringbuf.js](https://github.com/padenot/ringbuf.js) SAB when isolated | Web | No nightly Rust; COOP/COEP optional |
| Engine core | Own crate: transport, fixed-point time, step sequencer, arpeggiator, mixer, sample-offset events | Both | No crate is a DAW engine; borrows Firewheel's design |
| DSP | [fundsp](https://crates.io/crates/fundsp) 0.23.0 plus a custom compressor | Both | Broadest pure-Rust DSP; `Net` hot-swap |
| Bytebeat and operator tree | Own parser and bytecode VM; optional [cranelift-jit](https://crates.io/crates/cranelift-jit) natively later | Both | No reusable crate; evalexpr is AGPL |
| GM fallback (optional) | [rustysynth](https://crates.io/crates/rustysynth) 1.3.6 | Both (wasm unverified) | MIT, zero dependencies |
| Decode and resample | [symphonia](https://crates.io/crates/symphonia) 0.6.1, [rubato](https://crates.io/crates/rubato) 5.0.0 (pinned) | Both (wasm build to verify) | Identical decode on every client |
| Export | [hound](https://crates.io/crates/hound) 3.5.1, [flacenc](https://crates.io/crates/flacenc) 0.5.1, [opus-rs](https://crates.io/crates/opus-rs) 0.1.34 plus ogg | Both | Pure Rust |
| MIDI and OSC | [midir](https://crates.io/crates/midir) 0.11, [wmidi](https://crates.io/crates/wmidi) 4.0.11, [midly](https://crates.io/crates/midly) 0.5.3, [midi-msg](https://crates.io/crates/midi-msg) 0.9, [rosc](https://crates.io/crates/rosc) 0.11.4 | Both (no Safari Web MIDI) | Standard, allocation-free where it counts |
| Real-time plumbing | [rtrb](https://crates.io/crates/rtrb) 0.4, [triple_buffer](https://crates.io/crates/triple_buffer) 9.0, [basedrop](https://crates.io/crates/basedrop) 0.1.3; assert_no_alloc (debug), rtsan-standalone 0.3 (CI), no_denormals | Native (web path uses postMessage/SAB) | Wait-free in, latest-value out, deferred frees |
| CRDT | [loro](https://crates.io/crates/loro) 1.16.2 | Both | MovableList, LWW, local undo, presence |
| Sync server | [axum](https://crates.io/crates/axum) 0.8.9, [loro-protocol](https://crates.io/crates/loro-protocol) 0.3, SQLite/Postgres, S3/R2 blobs keyed by BLAKE3 | Server | Thin, self-hostable, local-first friendly |
| Client transport | tokio-tungstenite 0.30 (native), gloo-net 0.7 (web) behind a `Transport` trait | Both | Identical protocol, no NAT traversal |
| Dialogs | [rfd](https://crates.io/crates/rfd) 0.17.2 | Both (async on web) | Cross-target file I/O |
| Optional, native only | [rusty_link](https://crates.io/crates/rusty_link) 0.4.9 (GPL); [nice-plug](https://crates.io/crates/nice-plug) 0.4.2 or clack-plugin 0.2 plus clap-wrapper 0.3.1 for exporting our instruments; [clack-host](https://crates.io/crates/clack-host) 0.2 for later CLAP hosting | Native | Deferred; each has a licensing or portability cost |

The four load-bearing choices trade a little capability for a lot of risk reduction.

**A custom engine core over Firewheel** removes a monthly-breaking, game-semantics dependency from the one component that must behave identically on both targets.

**The stable single-threaded worklet over cpal's nightly `audioworklet` host** means the Dioxus web app and the engine never need nightly Rust, `build-std` or mandatory cross-origin isolation. The price is building graphs on the audio thread, which fixed topologies and small edits contain.

**Loro over Yrs** buys move semantics, counters and mergeable containers for the cost of a larger wasm binary.

**No plugin hosting in v1** keeps every project fully portable between native and web collaborators.

**Licensing is clean if you stay on this list.** Symphonia's MPL-2.0 is file-level copyleft and fine to link ([Symphonia](https://github.com/pdeljanov/Symphonia)). The traps are all avoidable:

| Crate | License |
|---|---|
| evalexpr ≥12 | AGPL |
| rusty_link | GPL |
| rubberband | GPL |
| oxisynth | LGPL |
| LAME (mp3lame-encoder) | LGPL |
| truce | Non-OSI license ([truce](https://github.com/truce-audio/truce)) |

## Architecture: one engine crate, two hosts, one visual pipe

The workspace should stay small; Meadowlark's repo sprawl is the cautionary tale. It holds seven crates:

- `engine` (no_std-friendly, pure);
- `app-core` (Loro schema, validation, document-to-command translation, clock sync);
- `ui` (Dioxus RSX);
- `vis` (canvas wasm);
- `host-native` (cpal, midir, loader, visual bridge);
- `host-web` (worklet JS glue plus raw engine exports);
- `server` (axum).

The desktop and web diagrams below are **inferred designs**. Items marked UNVERIFIED need a spike.

```
DESKTOP (one OS process; optional helper processes later)
+----------------------------------------------------------------------------+
| UI/app thread: Dioxus desktop runtime (native Rust)                        |
|   ui (RSX) + app-core: Loro doc, UndoManager, EphemeralStore, validation,  |
|   doc-diff -> EngineCmd translator, transport clock-sync client            |
|        | EngineCmd  (rtrb SPSC)                ^ EngineEvt (rtrb SPSC)      |
|        v                                       |                           |
| Audio thread: cpal callback (RT priority via `realtime`)                   |
|   engine::process(block): transport, step seq/arp, fundsp voices,          |
|   bytebeat VM, mixer  --> VisFrame into triple_buffer                      |
|   retired graphs/buffers --> basedrop collector (never freed here)         |
| Loader thread: symphonia -> rubato -> peak pyramid; samples handed in      |
| Tokio runtime: WebSocket to sync server; BLAKE3 blob upload/download       |
| MIDI: midir callbacks -> timestamped EngineCmd                             |
| Vis bridge: reads triple_buffer at display rate ---------------+           |
+----------------------------------------------------------------|-----------+
| System webview (wry: WebView2 / WKWebView / WebKitGTK)         |           |
|   DOM  <-- Dioxus mutations (IPC)    DOM events --> Rust (IPC) |           |
|   gesture helpers: local knob/drag feedback, commit on release |           |
|   vis.wasm canvas layer <-- binary VisFrames / peak blobs  <---+           |
|      via asset/custom-protocol pull or localhost WS push                   |
|      (NOT eval)                                   [UNVERIFIED]             |
+----------------------------------------------------------------------------+
  Later, native only: CLAP plugin helper process (clack-host, shm audio)
  Optional: rusty_link (GPL) Link peer on the LAN
```

```
BROWSER TAB (HTTPS; COOP/COEP when possible -> crossOriginIsolated)
+----------------------------------------------------------------------------+
| Main thread: Dioxus web app (wasm, DOM)                                    |
|   ui + app-core (same crates as desktop): Loro doc, validation,            |
|   clock-sync client, gloo-net WebSocket, midir Web MIDI (not Safari)       |
|   rAF: vis.wasm canvas reads latest VisFrame                               |
|      | EngineCmd bytes: port.postMessage  (or ringbuf.js SAB if isolated)   |
|      v                                   ^ VisFrame: SAB  (or transferable  |
|                                          |   postMessage at 30-60 Hz)       |
| AudioWorklet thread: worklet.js + engine.wasm                              |
|   (stable Rust, single-threaded, raw exports, bytes passed in at startup)  |
|   same `engine` crate; process() in 128-frame quanta, 64-sample sub-blocks |
+----------------------------------------------------------------------------+
| Worker (optional): symphonia decode, rubato, peaks, BLAKE3, offline bounce |
|   with the engine crate, WAV/FLAC/Opus encode -> transfer f32 to worklet   |
+----------------------------------------------------------------------------+
```

```
SYNC SERVER (axum + tokio; one process to start, sticky routing later)
  WebSocket rooms: doc-update | ephemeral | clock-ping | transport
  Loro doc per song (server is just another peer) + app-core validation
  Snapshots + update log -> SQLite/Postgres;  blobs -> S3/R2 by BLAKE3
  Tokens and roles; share codes -> (doc, role, expiry); fork = copy snapshot
  Clock master: ping replies; authoritative transport anchor (next-bar starts)
```

The contract that makes this work is three binary formats with identical bytes on every target: `EngineCmd`, `VisFrame` and Loro updates. The only per-target code is the pipe that carries them:

| Format | Desktop pipe | Web pipe |
|---|---|---|
| `EngineCmd` | rtrb | postMessage or SAB |
| `VisFrame` | triple_buffer plus a webview bridge | SAB or postMessage |
| Loro updates | tokio-tungstenite | gloo-net |

## Risks to retire before writing features

| Risk | Evidence | Mitigation or first spike |
|---|---|---|
| Dioxus desktop visual pipe underperforms | `eval` freezes on large payloads ([#1915](https://github.com/DioxusLabs/dioxus/issues/1915)); asset/custom-protocol throughput unverified; Tauri saw 10 MB binary transfers at about 200 ms on Windows ([#7127](https://github.com/tauri-apps/tauri/issues/7127)) | **Week-one spike:** stream 60 Hz × 4 KB frames plus a 1 MB peak blob on WebView2, WKWebView and WebKitGTK and measure jank. Fall back to a localhost WebSocket or, eventually, Blitz |
| Pointer latency through desktop IPC | Architectural inference; no measurements | Local gesture helpers; throttled commits; commit on release |
| No Dioxus audio precedent | None found in this research; nearest analog is a Tauri blog DAW ([whoisryosuke](https://whoisryosuke.com/blog/2026/creating-a-daw-in-rust/)) | Prototype the step grid, knobs and meters before the engine grows |
| Worklet deployment quirks | Missing `TextDecoder`, Module `messageerror` ([Gates #152](https://github.com/AnthonE/Gates/pull/152)); Safari `credentialless` ([firewheel-web-audio](https://crates.io/crates/firewheel-web-audio)) | Raw exports; transfer bytes; feature-detect `crossOriginIsolated`; test Chrome, Firefox and Safari in CI |
| fundsp fragility | One maintainer; no wasm demo; no compressor ([FUTURE.md](https://github.com/SamiPerttu/fundsp/blob/master/FUTURE.md)) | Wrap it behind our own `Voice`/`Effect` traits; spike a wasm32 build with `+simd128` |
| Ecosystem churn | cpal 0.19 breaking changes; rubato had four majors in 2026; Dioxus 0.8 alpha; clack and nice-plug still 0.x | Pin versions; upgrade deliberately, one layer at a time |
| CRDT cost and semantics | About 970 KB gzipped Loro wasm; merges never reject; UndoManager tracks one peer ([Loro docs](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt)) | Measure the linked size early; post-merge validation; keep Yrs as the fallback |
| Web latency and MIDI gaps | Tens of ms output latency (snippets); no Safari Web MIDI | Position the web build for composing; on-screen keyboard |
| Scope creep and burnout | Meadowlark post-mortems | Ship the NMS-scoped groovebox first; add the timeline later |
| NMS parity details unconfirmed | Most specifics come from search snippets | Verify BPM range, scales and operator set in-game before claiming exact parity |

## Conclusion

The research reframes the project. Rust already has enough DSP, decoding, MIDI and CRDT machinery to beat the ByteBeat Device on features. What it lacks is glue between execution domains: webview and native process, main thread and worklet, client and server. Each boundary is solved the same way: a compact, versioned binary contract with identical bytes on both targets and a swappable pipe underneath.

Choosing Dioxus moves the riskiest boundary from "which GUI can draw fast enough" to "how fast can bytes reach a canvas inside a webview". That is a narrower problem, and it can be measured in a day. It also leaves a clean upgrade path to Dioxus Native. So the build order follows the risk:

1. The engine crate with offline-render tests.
2. The desktop visual-pipe spike.
3. The stable worklet host.
4. Loro co-editing with a listen-along transport.
5. The "exceed" features: formula instruments, piano roll, share links and export.

NMS never offered live co-editing. A shared, local-first, bytebeat-aware groovebox that anyone can open from a link is the differentiator, and this stack reaches it without a single nightly-only, GPL or AGPL dependency on the critical path.
