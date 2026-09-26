# Rust crates for plugins, sample/audio file I/O, and MIDI (for a small collaborative DAW, native and WASM)

Research date: 2026-09-26. Version numbers and release dates come from the crates.io API (`https://crates.io/api/v1/crates/<name>`), queried on that date; each crate's crates.io page is cited below. "WASM OK" means the crate is pure Rust with no C/C++ build step or OS-only I/O. For most crates that is an inference from their dependencies and READMEs, not a compile test; the few confirmed cases are marked.

Tooling note: GitHub's API, codeberg.org, lib.rs, caniuse.com, MDN, truce.audio, webaudiomodules.com, cdm.link, librearts.org and forums.steinberg.net were blocked by the egress proxy during this session. Some claims about those sites therefore rest on search-engine summaries. Each such claim is marked "(search summary)".

---

## Q1. Plugin authoring: nih-plug and forks, clack, vst3 / Coupler, truce, baseview, LV2, clap-sys, and licensing (VST3 and CLAP)

### Takeaway
nih-plug is in maintenance mode. Its community successor is **nice-plug** (RustAudio, on Codeberg and crates.io, v0.4.2, 2026-09-14, ISC). nice-plug uses the MIT-licensed `vst3` crate, which removes the GPLv3 requirement that VST3 builds of nih-plug carried. That became possible because Steinberg relicensed the VST 3 SDK under MIT (VST 3.8.0, October 2025). For low-level CLAP work, **clack** (prokopyl) now publishes to crates.io (v0.2.0, 2026-09-12) and describes itself as feature-complete but still subject to API changes. **truce** covers the most formats (CLAP/VST3/VST2/LV2/AU/AAX), but its custom license is not OSI-approved. The Rust LV2 crates are abandoned.

### Cited Findings

**Licensing of the plugin standards**
- Steinberg released the VST 3.8 SDK in October 2025 under the MIT license. It replaced the earlier dual license (proprietary plus GPLv3). The search summary gives 3.8.0 as released 2025-10-20, and Steinberg's press release is dated 2025-10-29. — [Sonicstate](https://sonicstate.com/news/2025/10/30/vst-3-now-available-under-mit-license/); [Steinberg press release PDF](https://ocl-steinberg-live.steinberg.net/_storage/asset/819253/storage/master/Press%20Release%20-%202025-10-29%20-%20VST%203.8%20-%20EN.pdf); [Steinberg forum: VST 3.8.0 SDK released](https://forums.steinberg.net/t/vst-3-8-0-sdk-released/1011988) (search summary; the page itself could not be fetched)
- Under the new license, developers no longer sign a proprietary Steinberg VST3 agreement, and closed-source or commercial projects can use VST3 without releasing their source. — [Sonicstate](https://sonicstate.com/news/2025/10/30/vst-3-now-available-under-mit-license/) (search summary)
- Libre Arts reported that ASIO also became GPL-compatible in the same move. — [Libre Arts headline: "VST3 becomes open-source, ASIO goes GPL-compatible"](https://librearts.org/2025/11/steinberg-relicenses-vst3-and-asio/) (headline only; article blocked)
- CLAP is MIT-licensed. clap-wrapper is the official project for wrapping CLAP plugins into other formats. — [free-audio/clap on GitHub](https://github.com/free-audio/clap)
- clap-wrapper-rs 0.2.0 started embedding the VST3 and AUv2 SDKs directly in the crate, "possible thanks to VST3 SDK's new MIT license". — [clap-wrapper crate README/changelog](https://crates.io/crates/clap-wrapper)

**nih-plug and its forks**
- The nih-plug README says the framework is "in maintenance mode" and points active users to a community fork on Codeberg. It supports VST3 and CLAP, plus standalone builds with JACK. GUI adapters exist for egui, iced and VIZIA. — [robbert-vdh/nih-plug](https://github.com/robbert-vdh/nih-plug)
- nih-plug licensing: the framework and its example plugins are ISC. Its VST3 bindings (`vst3-sys`) are GPLv3, so any VST3 plugin built with nih-plug must comply with GPLv3. — [robbert-vdh/nih-plug README](https://github.com/robbert-vdh/nih-plug)
- nih-plug is not published on crates.io; it is used as a git dependency. The crates.io lookup for `nih_plug` returns not-found. — [crates.io API lookup](https://crates.io/crates/nih_plug)
- On 2026-03-29, BillyDM announced a "new maintained hard fork of NIH-plug" at github.com/BillyDM/nih-plug. It contains the framework only, not the original plugins. — [nih-plug issue #265](https://github.com/robbert-vdh/nih-plug/issues/265)
- **nice-plug** (codeberg.org/RustAudio/nice-plug) "started out as a fork of the awesome NIH-plug framework … has since become its own separate community-led project, and is now the recommended toolkit for Rust audio plugin developers". It describes itself as "currently experimental, and has recently undergone some large changes". — [nice-plug crate README](https://crates.io/crates/nice-plug)
- nice-plug 0.4.2 was released 2026-09-14. It was first published on crates.io 2026-06-04 and is licensed ISC. Sibling crates include nice-plug-egui 0.5.1, nice-plug-iced 0.4.1, cargo-nice-plug 0.1.1 and audioadapter-compat-nice-plug 2.0.0. — [crates.io: nice-plug](https://crates.io/crates/nice-plug); [crates.io: nice-plug-egui](https://crates.io/crates/nice-plug-egui)
- nice-plug 0.4.2 depends on `clap-sys ^0.5.0` and, optionally, `vst3 ^0.3.0` (the MIT/Apache coupler bindings rather than GPLv3 `vst3-sys`). Its other optional dependencies are `cpal ^0.18.2`, `midir ^0.11.0` and `jack`, used for standalone builds. Its licensing section lists only ISC, plus CC BY-SA 4.0 for the logos, with no GPLv3 caveat. — [crates.io dependency API for nice-plug 0.4.2](https://crates.io/crates/nice-plug/0.4.2/dependencies); [nice-plug README](https://crates.io/crates/nice-plug)
- nice-plug features include VST3 and CLAP export macros (`nice_export_<api>!`) and standalones with JACK audio/MIDI/transport. It has a declarative `#[derive(Params)]` system with smoothers, serde-persisted state and state migrations. Editors are any GUI that runs on baseview (OpenGL, wgpu or softbuffer rendering). It supports polyphonic note expressions, MIDI CC, pitch bend and SysEx for CLAP and VST3, plus CLAP mono/poly modulation and remote-control pages. It is "tested on Linux and Windows, with limited testing on macOS." — [nice-plug README](https://crates.io/crates/nice-plug)
- Other nih-plug forks exist on GitHub (MikesRuthless12, azusayn, seith-miller). The librestrings project filed an issue titled "Nih-plug in maintenance mode". — [search results](https://github.com/petterthowsen/librestrings/issues/1); [MikesRuthless12/nih-plug](https://github.com/MikesRuthless12/nih-plug)

**clack (CLAP)**
- clack consists of `clack-plugin`, `clack-host` and `clack-extensions`, built on `clap-sys`. It has "reached feature-complete status" but may still make breaking API changes. It is MIT OR Apache-2.0 and includes a CPAL-based host example. — [prokopyl/clack](https://github.com/prokopyl/clack)
- Release history for clack-plugin and clack-host: 0.1.0 (2026-05-03), 0.1.1 (2026-07-29), 0.2.0 (2026-09-12). Earlier versions were git-only. — [crates.io: clack-plugin](https://crates.io/crates/clack-plugin); [crates.io: clack-host](https://crates.io/crates/clack-host)
- clack's stated goal is "the full functionality of the CLAP plugin APIs through fully memory-safe and thread-safe wrappers"; it is low-level and "as close as possible to the underlying CLAP C API". — [clack-plugin README](https://crates.io/crates/clack-plugin)
- clap-sys 0.5.0 was released 2025-01-03 under MIT/Apache-2.0 (repo micahrj/clap-sys). — [crates.io: clap-sys](https://crates.io/crates/clap-sys)
- clap-wrapper-rs 0.3.1 was released 2026-05-18 under MIT OR Apache-2.0. It exports Rust CLAP plugins as VST3 and AUv2, and its README names clack as an intended use case. It builds with the `cc` crate rather than cmake. Standalone builds are not supported yet, and AUv2 is limited to 4 plugins per binary. — [crates.io: clap-wrapper](https://crates.io/crates/clap-wrapper)

**vst3 crate / Coupler**
- `vst3` 0.3.0 was released 2025-12-07 under MIT OR Apache-2.0 (repo coupler-rs/vst3-rs). It contains bindings "generated from the original C++ headers". It offers abstractions for COM objects, but "these bindings are unsafe, and no attempt is made to abstract over the VST 3 API itself." — [crates.io: vst3](https://crates.io/crates/vst3)
- `vst3-sys` (the older RustAudio GPLv3 bindings) is not on crates.io. — [crates.io lookup](https://crates.io/crates/vst3-sys); GPLv3 status per [nih-plug README](https://github.com/robbert-vdh/nih-plug)
- Coupler (coupler-rs/coupler) is a framework targeting VST3 and CLAP, with AUv2 and AAX planned. It is MIT/Apache-2.0 and "early in development and should not be considered production-ready". On crates.io it is only a placeholder, `coupler` 0.0.0 from 2022-03-31. — [coupler-rs/coupler](https://github.com/coupler-rs/coupler); [crates.io: coupler](https://crates.io/crates/coupler)
- The legacy VST2 crate `vst` 0.4.0 was last released 2023-03-15. — [crates.io: vst](https://crates.io/crates/vst)

**truce**
- truce 6.3.0 was released 2026-07-18. `truce-core` was first published 2026-03-23. The license is `LicenseRef-TruceLicense-1.0`. — [crates.io: truce](https://crates.io/crates/truce); [crates.io: truce-core](https://crates.io/crates/truce-core)
- truce exports CLAP, VST3, VST2, LV2, AU v2, AU v3 (macOS and iOS), AAX and standalone from one Rust codebase, on macOS, Windows, Linux and iOS. Its README does not mention WASM or web. GUI options are built-in widgets, egui, iced, Slint, Vizia or a raw window handle. It has about 214 stars. — [truce-audio/truce on GitHub](https://github.com/truce-audio/truce)
- The Truce License v1.0 is described as dual Apache-2.0/MIT with restrictions and is **not OSI-approved**. A commercial license is required to redistribute truce as a paid framework or commercial service. Plugin authors are otherwise unrestricted: no per-plugin fees or revenue caps, and free OSI-licensed projects are exempt. — [truce-audio/truce README](https://github.com/truce-audio/truce)
- truce offers hot reload through a `shell` feature that dlopens a hot-reloadable logic dylib, plus a `cargo-truce` CLI for signing, notarization, installers and validation. `truce-clap` uses `clap-sys ^0.5`. `truce-vst3` depends on `cc`, which suggests it compiles C/C++ glue. — [truce README on crates.io](https://crates.io/crates/truce); [crates.io deps: truce-vst3 / truce-clap](https://crates.io/crates/truce-vst3)
- A fork of truce called MOOSE (Matari-Audio) exists. — [Matari-Audio/moose](https://github.com/Matari-Audio/moose)

**baseview and LV2**
- baseview 0.3.4 was released 2026-09-12 under MIT OR Apache-2.0 (RustAudio). It is "a low-level windowing system geared towards making audio plugin UIs", abstracting winapi, cocoa and xcb. Companion adapters: egui-baseview 0.7.2 (2026-09-18) and iced_baseview 0.5.2 (2026-09-13). — [crates.io: baseview](https://crates.io/crates/baseview); [crates.io search results](https://crates.io/crates/egui-baseview)
- `lv2` 0.6.0 and `lv2-sys` 2.0.0 (RustAudio/rust-lv2) were last released 2020-10-23. — [crates.io: lv2](https://crates.io/crates/lv2); [crates.io: lv2-sys](https://crates.io/crates/lv2-sys)

### Inferences
- **Recommendation for authoring:** make nice-plug the primary export wrapper for our own instruments. It carries on nih-plug's API, is published on crates.io, is ISC-licensed with MIT VST3 bindings, and is endorsed by RustAudio. If we want minimal wrappers and full control over CLAP, use clack-plugin, with clap-wrapper-rs for VST3/AU export. Consider truce only if AU/AAX/iOS are needed and the non-OSI license is acceptable. Its license mostly restricts reselling the framework itself, but it would complicate an open-source DAW's licensing story.
- The GPLv3 problem of shipping VST3 from Rust is effectively solved by the MIT SDK plus the MIT/Apache `vst3` crate. Projects still pinned to original nih-plug and `vst3-sys` remain formally under GPLv3 for their VST3 builds unless they migrate.
- None of these frameworks targets wasm32 or the web. On the web, plugin export means writing our own glue (see Q2 and Q3).
- nice-plug describes itself as experimental and recently restructured, and clack 0.x warns of breaking changes. Expect churn, and isolate plugin-export code behind our own internal instrument trait.

### Gaps
- Whether BillyDM's GitHub fork and Codeberg nice-plug are the same effort under different names (both come from RustAudio-adjacent maintainers) could not be verified, because Codeberg was blocked.
- The exact terms of the VST3 MIT relicense could not be read from primary text: whether VSTGUI is included, and whether a separate agreement is still required to use the VST logo/trademark.
- clack 0.2.0 changelog and CLAP spec version supported: not retrieved.
- No independent maturity data for truce (production users, issue backlog); it was only published in 2026.

---

## Q2. Hosting third-party plugins from a Rust app (CLAP/VST3/LV2), sandboxing, and the web alternative (WAM 2.0)

### Takeaway
For native hosting, **clack-host** is the only mature, safe Rust option. It hosts CLAP only, ships a cpal host example, and is MIT/Apache, v0.2.0. Hosting VST3 means writing a host on the raw unsafe `vst3` COM bindings, or hosting everything through CLAP. LV2 hosting exists via `livi`, which is work-in-progress and last released 2024-11. No ready-made, production Rust out-of-process sandbox was found. `plugin_host` (0.1.0) advertises one, but its format bridges are stubs. On the web, native plugins cannot run. The standard for interoperable browser plugins is **Web Audio Modules (WAM) 2.0**, which has no Rust SDK; Rust DSP can be packaged as a WAM through custom JS/AudioWorklet glue.

### Cited Findings
- clack-host is for building CLAP hosts, and the repo provides a CPAL-based host example described as "more featured and functional". — [prokopyl/clack](https://github.com/prokopyl/clack)
- A search summary describes clack as effectively "the available functional alternative" for CLAP hosting in Rust. — [docs.rs clack_host](https://docs.rs/clack-host/latest/clack_host/) (search summary)
- `plugin_host` 0.1.0 (2026-02-25, MIT, 189 downloads) advertises a VST3/CLAP scanner and "configurable in-process or out-of-process per plugin" sandboxing with auto-restart on crash. However, its README says the ClapBridge and Vst3Bridge "are fully scaffolded with all required method stubs … Connect clap-sys or vst3-sys to activate". Actual plugin loading is therefore not implemented. — [crates.io: plugin_host](https://crates.io/crates/plugin_host)
- The `plugin_host` README states the sandbox trade-off: in-process gives "Maximum" performance but "Plugin crash = DAW crash"; out-of-process has "IPC overhead" but "Plugin crash = auto restart". — [crates.io: plugin_host](https://crates.io/crates/plugin_host)
- `livi` 0.7.5 (2024-11-12, MIT) is "a library for hosting LV2 plugins", with the caveat "This is a work in progress and has not yet been full tested". It supports urid map/unmap, options, buf-size and worker, and has a JACK example. — [crates.io: livi](https://crates.io/crates/livi)
- The `vst3` crate provides unsafe COM bindings and helpers for implementing COM interfaces, with no higher-level abstraction. — [crates.io: vst3](https://crates.io/crates/vst3)
- WAM 2.0 defines `WebAudioModule` (entry point: metadata, lifecycle, GUI creation), `WamNode` (extends AudioNode), `WamProcessor` (extends AudioWorkletProcessor, runs on the audio thread) and `WamDescriptor`. It supports scheduling parameter automation, MIDI, transport and OSC events, plus event connections between WAM instances. The legacy API lives in branch v10. — [webaudiomodules/api](https://github.com/webaudiomodules/api)
- WAM 2.0 was released in 2021. It is open source, distributed as GitHub repositories and npm modules, and described as "VST for the web", with SDKs, dozens of plugins and several hosts. Code "written in low-level languages (C/C++, Rust, etc.)" can run in browsers through WebAssembly and AudioWorklets. — [WAM docs intro](https://www.webaudiomodules.com/docs/intro/) (search summary); [WAM 2.0 paper, ACM WWW '22 companion](https://dl.acm.org/doi/fullHtml/10.1145/3487553.3524225)
- The wasm-bindgen guide has a "Wasm Audio Worklet" example. Threads inside worklets need a rebuilt std with `-C target-feature=+atomics`, which requires nightly. There are known friction points when running wasm-bindgen output inside AudioWorkletProcessor. — [wasm-bindgen guide: Wasm Audio Worklet](https://wasm-bindgen.github.io/wasm-bindgen/examples/wasm-audio-worklet.html); [wasm-bindgen discussion #3928](https://github.com/wasm-bindgen/wasm-bindgen/discussions/3928)
- `waw-rs` ("Rust Web Audio Worklets without crying", Marcel-G) is a GitHub project for writing AudioWorklets in Rust. It is not on crates.io under that name. — [Marcel-G/waw-rs](https://github.com/Marcel-G/waw-rs); crates.io lookup returned not-found
- A related experiment in the WASM *component-model* style exists: "wasm component as audio plugin". — [wasm-audio/wasm-audio-examples](https://github.com/wasm-audio/wasm-audio-examples)

### Inferences
- **Recommendation for native hosting:** host CLAP via clack-host as the primary path. For VST3 there are three options:
  1. Write a VST3 host on the `vst3` crate. This is significant unsafe COM work.
  2. Defer VST3 support.
  3. Rely on the growing set of plugins that ship CLAP builds.

  LV2 hosting (livi) is low priority and Linux-centric.
- **Sandboxing:** nothing reusable exists. We would build our own out-of-process host: a helper process per plugin (or per group), shared-memory audio buffers, and an IPC control channel. This design follows the pattern `plugin_host` describes, not its code. It costs a buffer of latency or tight synchronization per block. It fits a collaborative DAW, where one user's crashing plugin should not take down the session.
- **Collaboration implication:** third-party native plugins are not portable across collaborators (not everyone owns them, and the web cannot run them). Our internal instruments must be the portable baseline. Native-plugin tracks should be freezable/bounceable to audio, so web and other collaborators can hear them.
- **Web:** Rust cannot target WAM directly, since no Rust WAM SDK was found. A WAM wrapper would be a thin JS `WebAudioModule` subclass plus a `WamProcessor` that forwards `process()` and WAM events into a Rust-compiled wasm module. Hosting third-party WAMs in our web build is feasible, but only in the browser build.

### Gaps
- No Rust WAM SDK or crate found. The WAM license and current SDK version/activity could not be verified, because webaudiomodules.com was blocked.
- I found no well-known open-source Rust DAW that hosts CLAP in production to cite as a reference implementation (Meadowlark's status was not verified).
- Real-world latency and overhead numbers for out-of-process plugin hosting (e.g., Bitwig's sandbox modes) were not researched.

---

## Q3. Designing our own instrument format that runs in-engine on web and native, and optionally exports as CLAP/VST3

### Takeaway
Keep instrument DSP in plain Rust crates with no platform dependencies. Define a small internal trait (process block, sample-accurate events, parameter descriptors, serde state) and write thin adapters for:
- the native engine (cpal thread);
- the web engine (AudioWorklet plus wasm);
- plugin export via nice-plug or clack, with clap-wrapper for VST3/AU;
- optionally, a JS WAM wrapper for other web hosts.

The research supports this layering. Every plugin framework is native-only, while the DSP-side crates (fundsp, rubato, rustysynth, oxisynth) are pure Rust.

### Cited Findings
- nice-plug "does not make any assumptions on how you want to process audio" and uses a declarative parameter system with stable string IDs (`#[id = "..."]`), serde-persisted state and migrations. — [nice-plug README](https://crates.io/crates/nice-plug)
- truce separates `truce-core` traits and `truce-params` from per-format crates (`truce-clap`, `truce-vst3`, `truce-lv2`…). `truce::plugin!` generates the format glue from one declaration. — [truce README](https://crates.io/crates/truce)
- The CLAP event model (note expressions, poly modulation) is exposed by both nice-plug and clack. — [nice-plug README](https://crates.io/crates/nice-plug); [clack-plugin README](https://crates.io/crates/clack-plugin)
- The WAM event model covers parameter automation, MIDI, transport and OSC. — [webaudiomodules/api](https://github.com/webaudiomodules/api)
- `fundsp` 0.23.0 (2026-01-07, MIT OR Apache-2.0) is available as a pure-Rust DSP graph library. — [crates.io: fundsp](https://crates.io/crates/fundsp)
- `cpal` 0.18.2 (2026-08-16, Apache-2.0) is the audio I/O layer that nice-plug's standalone builds use. — [crates.io: cpal](https://crates.io/crates/cpal); [nice-plug deps](https://crates.io/crates/nice-plug/0.4.2/dependencies)
- `web-audio-api` 1.7.0 (2026-08-08, MIT) is a pure Rust implementation of the Web Audio API for non-browser contexts. — [crates.io: web-audio-api](https://crates.io/crates/web-audio-api)

### Inferences
- A suggested internal contract, which should map one-to-one onto CLAP concepts for a cheap export later:

  ```
  trait Instrument {
      fn params() -> &'static [ParamDesc];
      fn activate(sr, max_block);
      fn process(&mut self, out: &mut [&mut [f32]], events: &[TimedEvent]);
      fn save_state() -> Vec<u8>;
      fn load_state(&[u8]);
  }
  ```

  - `ParamDesc` should carry a stable string ID, range, default, and unit/automation flags.
  - `TimedEvent` should carry a sample offset plus a payload: note on/off with note ID, per-note expression, param change, MIDI bytes.
  - Keep `process` allocation-free and `no_std`-friendly where practical, so the same code runs in an AudioWorklet (wasm32, single-threaded) and on a native realtime thread.
- Parameter IDs and state blobs should be designed so a project file created on web opens on native, and vice versa. That is the core requirement of a collaborative DAW. Serde-serialized state with version migrations follows the nice-plug model.
- For export: a `nice-plug` adapter crate can wrap each `Instrument`, giving CLAP, VST3 and standalone. Alternatively, a `clack-plugin` adapter plus `clap-wrapper` gives CLAP, VST3 and AUv2. Neither adapter is compiled for wasm32.
- A GPU/native GUI (baseview + egui) will not port to the web as-is. If the UI is a web UI (DOM or canvas), native plugin export needs a separate UI or a webview-in-plugin. None of the reviewed frameworks advertised webview editors, so this was not verified.

### Gaps
- No existing Rust project was found that already ships the same instrument code as a web AudioWorklet, a WAM *and* a CLAP/VST3 plugin; no reference architecture to cite.
- Whether egui-in-baseview editors could be reused on the web (egui does support wasm via eframe) was not investigated.

---

## Q4. Sample/audio file I/O: decoding, encoding, streaming, resampling, WASM

### Takeaway
**Symphonia 0.6.1** (2026-08-13, MPL-2.0, pure Rust) is the decoder of choice: WAV, AIFF, CAF, FLAC, MP3, OGG/Vorbis, ALAC, AAC-LC, ADPCM, MKV/WebM and MP4. It decodes only (no encoding) and has **no Opus decoder** ("not started"). For export:
- WAV: **hound** (stable but last released 2023).
- FLAC: **flacenc** (pure Rust).
- Opus: the pure-Rust **opus-rs**, which is new and makes WASM claims.
- MP3 (via LAME) and Vorbis (via libvorbis): C-library bindings, harder to build for wasm32, and LAME's crate is LGPL.

Use **rubato 5.0.0** for sample-rate conversion. Use **creek** for realtime disk streaming on native only.

### Cited Findings

**Decoding**
- symphonia 0.6.1 was released 2026-08-13 under MPL-2.0, with 6.9M recent downloads. MSRV for 0.6.x is 1.85. — [crates.io: symphonia](https://crates.io/crates/symphonia); [pdeljanov/Symphonia README](https://github.com/pdeljanov/Symphonia)
- Symphonia format and codec status:
  - Containers: WAV Excellent, FLAC Excellent, OGG Great, MP4/ISO Great, AIFF Great, MKV/WebM Good, CAF Good.
  - Codecs: MP3, Vorbis, PCM and FLAC Excellent; ALAC Great; AAC-LC Great; ADPCM Good; **Opus and HE-AAC "not started"**.
  - Symphonia is decode/demux only (no encoding). It targets roughly ±15% of FFmpeg's performance.
  - By default it enables only royalty-free codecs; others (AAC, MP4, etc.) need feature flags.
  - "Providing a WASM API for web usage" is listed as a *planned* feature.

  — [Symphonia README](https://github.com/pdeljanov/Symphonia); [symphonia crate README](https://crates.io/crates/symphonia)
- Symphonia describes itself as "a pure Rust audio decoding and media demuxing library". — [symphonia crate README](https://crates.io/crates/symphonia)
- `symphonia-adapter-libopus` 0.3.0 (2026-05-16, MIT OR Apache-2.0) exists as an add-on Opus decoder for Symphonia. Its name indicates it binds C libopus; this was not verified in its README. — [crates.io: symphonia-adapter-libopus](https://crates.io/crates/symphonia-adapter-libopus)
- hound 3.5.1 (last release 2023-09-25, Apache-2.0) reads and writes WAV. It is heavily used (19M downloads). — [crates.io: hound](https://crates.io/crates/hound)
- claxon 0.4.3 (FLAC decoder) was last released 2020-08-09 under Apache-2.0. — [crates.io: claxon](https://crates.io/crates/claxon)
- lewton 0.10.2 (Vorbis decoder) was last released 2021-01-20. — [crates.io: lewton](https://crates.io/crates/lewton)
- audrey 0.3.0 (multi-format wrapper) was last released 2021-01-14. — [crates.io: audrey](https://crates.io/crates/audrey)
- dasp 0.11.0 (sample/frame conversion) was last released 2020-05-29. — [crates.io: dasp](https://crates.io/crates/dasp)
- wavers 1.5.1 (2024-12-29, MIT) is a WAV reader/writer supporting i16/i32/f32/f64 with experimental i24, built around `Wav::from_path`. — [crates.io: wavers](https://crates.io/crates/wavers)

**Streaming**
- creek 1.2.3 (2025-09-22, MIT OR Apache-2.0, now hosted at codeberg.org/Meadowlark/creek) provides "Realtime-safe streaming to/from audio files on disk". It decodes via Symphonia, except that AAC and isomp4 do not work with it yet, and encodes only to WAV. It uses cache buffers plus a look-ahead buffer, and "automatically spawns an 'IO server' thread". — [crates.io: creek](https://crates.io/crates/creek)

**Encoding**
- flacenc 0.5.1 (2025-12-18, Apache-2.0) is a pure-Rust FLAC encoder with experimental decoding. It uses a "fake" portable_simd on stable, with nightly SIMD available as an option. — [crates.io: flacenc](https://crates.io/crates/flacenc)
- vorbis_rs 0.5.6 (2026-07-30, BSD-3-Clause) binds the C libvorbis, libvorbisenc and vorbisfile libraries, with aoTuV and Lancer patches. MSRV is 1.87. It says the older `vorbis`/`vorbis-sys` crates depend on a libvorbis with known vulnerabilities. — [crates.io: vorbis_rs](https://crates.io/crates/vorbis_rs)
- mp3lame-encoder 0.2.5 (2026-08-20) is licensed **LGPL-3.0**: "LAME library is under LGPL License. Hence this crate is licensed under the same … license". — [crates.io: mp3lame-encoder](https://crates.io/crates/mp3lame-encoder)
- `opus` 0.4.0 (2026-08-23, MIT/Apache-2.0) is a set of libopus bindings. It needs cmake and a C compiler by default, via opusic-sys. — [crates.io: opus](https://crates.io/crates/opus)
- `audiopus` 0.2.0 was last released 2021-04-22 (ISC). `ogg-opus` 0.1.2 was last released 2021-05-18. — [crates.io: audiopus](https://crates.io/crates/audiopus); [crates.io: ogg-opus](https://crates.io/crates/ogg-opus)
- `opus-rs` 0.1.34 (2026-09-22, BSD-3-Clause, repo restsend/opus-rs) is "a pure-Rust implementation of the Opus audio codec (RFC 6716), ported from … libopus 1.6". It encodes and decodes, and claims to be "production-ready". It supports `#![no_std]` without alloc and says it "runs on bare metal, RTOS, and WebAssembly". It handles raw Opus packets; Ogg encapsulation would come separately, e.g. from the `ogg` crate. — [crates.io: opus-rs](https://crates.io/crates/opus-rs)
- `ogg` 0.9.2 (2025-01-12, BSD-3-Clause) is the RustAudio Ogg container crate. — [crates.io: ogg](https://crates.io/crates/ogg)

**Resampling**
- rubato 5.0.0 (2026-08-10, MIT OR Apache-2.0) covers "real-time audio streams to offline batch processing". It includes "high-quality asynchronous sinc resamplers and fast synchronous FFT-based resamplers" and is "designed with real-time safety in mind". Buffers go through the companion `audioadapter` 5.0.0 (2026-07-31). — [crates.io: rubato](https://crates.io/crates/rubato); [crates.io: audioadapter](https://crates.io/crates/audioadapter)

### Inferences
- **WASM compatibility, by dependency analysis rather than build testing:**
  - Pure Rust and very likely fine on wasm32-unknown-unknown when fed from memory (`Cursor<Vec<u8>>`): symphonia, hound (with in-memory `Read`/`Write`), flacenc, rubato, opus-rs (self-declared), ogg, midly/wmidi.
  - C bindings, painful for wasm32-unknown-unknown (would need an emscripten target or wasm-compiled C via clang/wasi-sdk): vorbis_rs, mp3lame-encoder, opus (libopus), and likely symphonia-adapter-libopus.
  - Native-only by design: creek (spawns an OS thread and does file-system I/O).
- **Browser sample loading path:** File API or drag-drop, then `file.arrayBuffer()`, then `Uint8Array`, then a wasm `Vec<u8>`, then `symphonia::core::io::MediaSourceStream` over a `Cursor`, then decode to f32, then rubato to the project sample rate. Browser `decodeAudioData` is an alternative, but it resamples to the AudioContext rate and codec support differs across browsers. Symphonia gives identical behaviour on web and native, which matters for collaboration determinism.
- **Recommended stack:**
  - Decode: symphonia.
  - Export: WAV via hound (or wavers); FLAC via flacenc; Opus via opus-rs, good for compressed sharing and sync of collaborative assets in both web and native (evaluate maturity, since it is a young crate).
  - MP3/Vorbis export: native-only, optional; note LGPL obligations for LAME.
  - Resampling: rubato.
  - Disk streaming: creek on native; on the web, load samples fully into memory, or stream chunks from IndexedDB/OPFS with custom code.
- **Licensing:** Symphonia's MPL-2.0 is file-level copyleft. Modifications to Symphonia's own files must be shared, but linking into a proprietary or differently licensed app is allowed. hound, flacenc, rubato and opus-rs are permissive.

### Gaps
- No primary source found confirming that Symphonia 0.6 builds for wasm32-unknown-unknown (it is pure Rust, and the README lists a WASM *API* as planned). A quick `cargo build --target wasm32-unknown-unknown` test is recommended.
- Symphonia 0.6 release notes (what changed from 0.5) were not retrieved.
- Quality and compliance of opus-rs relative to libopus has not been independently verified; the "production-ready" label is the author's own claim.
- Whether `wavers` supports reading from byte slices (needed on the web) was not verified.

---

## Q5. SoundFont / SFZ playback: rustysynth, oxisynth, sfizz

### Takeaway
Two pure-Rust SF2 synths exist:
- **rustysynth** 1.3.6 (2025-08-10, MIT): no dependencies beyond std, includes reverb/chorus and a MIDI-file sequencer.
- **OxiSynth** 0.1.0 (2025-05-25, LGPL-2.1): a FluidSynth-inspired port with a live browser demo. WASM is a first-class goal.

For SFZ, no Rust crate or sfizz binding was found on crates.io.

### Cited Findings
- RustySynth is "a SoundFont MIDI synthesizer written in pure Rust, ported from MeltySynth". It suits real-time and offline synthesis, supports standard MIDI files with dynamic tempo, and has "no dependencies other than the standard library". Features: envelopes, LPF, vibrato/mod LFOs, bank select, modulation, pitch bend, tuning, reverb and chorus. License MIT. — [crates.io: rustysynth](https://crates.io/crates/rustysynth)
- rustysynth 1.3.6 was released 2025-08-10. — [crates.io: rustysynth](https://crates.io/crates/rustysynth)
- OxiSynth is "a pure safe Rust SoundFont™ synthesizer, inspired by … FluidSynth", "built with WASM in mind from the get go", with a browser demo at oxisynth.netlify.app. Reverb and chorus live in separate crates, and it has per-channel tuning. It is used by Neothesia and microwave. — [PolyMeilex/OxiSynth](https://github.com/PolyMeilex/OxiSynth)
- oxisynth 0.1.0 was released 2025-05-25 under LGPL-2.1. — [crates.io: oxisynth](https://crates.io/crates/oxisynth)
- Crates named `sfizz` or `sfizz-sys` do not exist on crates.io. — crates.io lookups ([sfizz](https://crates.io/crates/sfizz))
- FluidSynth C bindings (`rust-fluidsynth`) and `fluidlite` exist as C-based alternatives. — [scholtzan/rust-fluidsynth](https://github.com/scholtzan/rust-fluidsynth); [docs.rs fluidlite](https://docs.rs/fluidlite/latest/fluidlite/struct.Synth.html)

### Inferences
- **Recommendation:** rustysynth, for its MIT license, zero dependencies, easy wasm and a built-in MIDI-file sequencer that suits a preview/GM-fallback instrument. OxiSynth is an option if FluidSynth-compatible behaviour is needed, but LGPL-2.1 imposes relinking obligations. Those obligations are awkward for a statically linked wasm or native binary, so check them against the project license.
- SFZ support would mean writing our own SFZ parser and sampler on top of symphonia and rubato, or binding sfizz (C++) on native only.

### Gaps
- SF3 (compressed SoundFont) support in either crate was not verified.
- No maintained Rust SFZ player crate was found; a search for smaller community SFZ crates was not exhaustive.

---

## Q6. MIDI: device I/O (native and Web MIDI), message types, SMF files, MIDI 2.0, clock sync, MPE, browser support

### Takeaway
Use **midir 0.11.0** (2026-04-18, MIT) for device I/O on all native platforms and in the browser (Web MIDI on wasm32). For parsing and writing Standard MIDI Files, use **midly** (fast and complete; last release 2023, but stable). Use **wmidi** or **midi-msg** for realtime message types; midi-msg covers MPE and MTC. MIDI 2.0 (`midi2`) is early-stage. **Safari (macOS and iOS) has no Web MIDI, and WebKit has declined to ship it.** Browser MIDI therefore works in Chromium browsers and Firefox 108+ only.

### Cited Findings

**Device I/O**
- midir 0.11.0 was released 2026-04-18 under MIT (Boddlnagg/midir, about 827 stars). Backends: ALSA and JACK (Linux), CoreMIDI (macOS/iOS), WinMM and WinRT (Windows), "Web MIDI (Chrome, Opera, perhaps others browsers)" and Android (API 29+). It supports virtual ports "except on Windows" and full SysEx. — [crates.io: midir](https://crates.io/crates/midir); [Boddlnagg/midir](https://github.com/Boddlnagg/midir)
- midir 0.11.0 pulls in `js-sys`, `wasm-bindgen` and `web-sys` on `cfg(target_arch = "wasm32")`, so the Web MIDI backend is built in on wasm32 targets. — [crates.io dependency API: midir 0.11.0](https://crates.io/crates/midir/0.11.0/dependencies)
- `web-midi` 0.1.0 (a web-sys wrapper) has not been updated since 2020-11-14. — [crates.io: web-midi](https://crates.io/crates/web-midi)

**Browser support for Web MIDI**
- Web MIDI works in Chrome 43+, Edge 79+, Opera 30+, Samsung Internet 4+ and Firefox 108+. Safari does not support it on macOS in any version. WebKit has said it will not ship Web MIDI, citing fingerprinting, and has no public roadmap as of 2026. — [Super Simple Piano, "Web MIDI in 2026"](https://www.supersimplepiano.com/blog/web-midi-browser-compatibility-2026) (search summary; secondary source); [TestMu: midi on Safari](https://www.testmuai.com/web-technologies/midi-safari/) (search summary)
- A third-party Safari web extension, "Safari-WebMIDI", adds `navigator.requestMIDIAccess()` backed by CoreMIDI behind a per-site permission prompt. — [triglav-modular/Safari-WebMIDI](https://github.com/triglav-modular/Safari-WebMIDI)

**Messages and files**
- wmidi 4.0.11 (2026-04-19, MIT, RustAudio) encodes and decodes MIDI messages. It is no_std, with "no memory allocations (therefore realtime safe) for parsing and encoding". — [crates.io: wmidi](https://crates.io/crates/wmidi)
- midly 0.5.3 was last released 2023-01-01 under the Unlicense. It is "a feature-complete MIDI decoder and encoder" for both .mid files and live packets, zero-copy, with optional no_std/no-alloc. Its benchmarks parse a 24 MB file in 60 ms, against 214–20,575 ms for other libraries. — [crates.io: midly](https://crates.io/crates/midly)
- midi-msg 0.9.0 (2026-05-08, MIT) aims to be "a complete representation of the MIDI 1.0 Detailed Specification and its many extensions". It covers MIDI Time Code (MTC) and **MIDI Polyphonic Expression 1.0 (RP-053)**, has optional `sysex` and `file` (SMF) features and supports no_std. "MIDI 2.0 may be supported at a later date." — [crates.io: midi-msg](https://crates.io/crates/midi-msg)
- nodi 1.0.3 (2025-01-01, MIT) provides MIDI-file playback on top of midly and midir: time-mapped events, track merge, bar splitting and transposition. — [crates.io: nodi](https://crates.io/crates/nodi)
- midi2 0.11.1 (2026-04-09, MIT OR Apache-2.0, midi2-dev/bl-midi2-rs) provides strongly typed MIDI 2.0 UMP messages based on revision 1.1 of the spec. It warns: "still in early development. Expect breaking changes and bugs". — [crates.io: midi2](https://crates.io/crates/midi2)
- Plugin-side MPE and expressions: nice-plug supports "polyphonic note expression events as well as MIDI CCs, channel pressure, and pitch bend for CLAP and VST3". — [nice-plug README](https://crates.io/crates/nice-plug)

### Inferences
- **Recommendation:**
  - Device I/O: midir everywhere. Web MIDI access is async and permission-gated, so design the UI to request it on user action. Treat MIDI as unavailable on Safari/iOS and fall back to an on-screen keyboard or QWERTY input.
  - Internal representation: wmidi (allocation-free) for realtime messages, or midi-msg if MPE/MTC typing is wanted.
  - Import/export of .mid: midly.
  - Our own event model: use CLAP-style note IDs and per-note expressions internally rather than raw MPE channels. Translate MPE (via midi-msg) at the device boundary.
- **MIDI clock sync:** wmidi and midi-msg can parse clock and SPP messages, and midi-msg can parse MTC, but no crate for tempo estimation, PLL or jitter smoothing was found. Expect to implement clock-in smoothing yourself. In a collaborative, networked setting, session tempo sync should run over our own network protocol, not MIDI clock.
- MIDI 2.0: defer. midi2 is early, and midir, Web MIDI and the plugin frameworks all expose MIDI 1.0 bytes.

### Gaps
- MDN and caniuse primary compatibility tables could not be fetched (both blocked). The browser-version numbers come from secondary pages via search summary.
- midir's Web MIDI details were not confirmed in docs: whether SysEx permission is requested, whether port enumeration needs async init, and whether it works on wasm32 in Firefox.
- No dedicated Rust MIDI-clock-sync crate was evaluated. A `midi_clock_sync` crate is mentioned in `plugin_host`'s README, but its quality is unknown.

---

## Q7. OSC for live-performance integration (brief)

### Takeaway
**rosc** 0.11.4 (2025-03-23, MIT/Apache-2.0) is a pure-Rust OSC 1.0 encoder/decoder and the obvious choice. It handles serialization only, so the transport is ours. That is UDP on native; browsers cannot open UDP sockets, so the web build needs a WebSocket/WebRTC bridge.

### Cited Findings
- "rosc is an implementation of the OSC 1.0 protocol in pure Rust", dual MIT/Apache-2.0. — [crates.io: rosc](https://crates.io/crates/rosc)
- rosc 0.11.4 was released 2025-03-23 and has about 547k total downloads. — [crates.io: rosc](https://crates.io/crates/rosc)
- The WAM 2.0 event model includes OSC messages, alongside MIDI, automation and transport. — [webaudiomodules/api](https://github.com/webaudiomodules/api)

### Inferences
- Use rosc for packet encoding and decoding on both targets. On the web, carry OSC packets over the same WebSocket/WebRTC channel as collaboration traffic, or through a small native bridge that relays UDP.

### Gaps
- Browser UDP limits were not re-verified with a primary source here; they are standard web platform behaviour.
- OSC 1.1 or bundle-timing specifics in rosc were not reviewed.

---

## Summary table (for the report writer)

| Area | Crate | Latest (date) | License | WASM | Status / note |
|---|---|---|---|---|---|
| Plugin authoring | nice-plug | 0.4.2 (2026-09-14) | ISC | No | Recommended nih-plug successor (RustAudio); experimental |
| Plugin authoring | nih-plug | git only | ISC (+GPLv3 VST3 via vst3-sys) | No | Maintenance mode |
| CLAP plugin/host | clack-plugin / clack-host | 0.2.0 (2026-09-12) | MIT/Apache | No | Feature-complete, API may change |
| CLAP bindings | clap-sys | 0.5.0 (2025-01-03) | MIT/Apache | n/a | Raw FFI |
| CLAP→VST3/AU | clap-wrapper (rs) | 0.3.1 (2026-05-18) | MIT/Apache | No | Embeds MIT VST3 SDK |
| VST3 bindings | vst3 (coupler) | 0.3.0 (2025-12-07) | MIT/Apache | No | Unsafe raw COM bindings |
| Framework | coupler | 0.0.0 placeholder (2022) | MIT/Apache | No | Early dev, not production-ready |
| Framework | truce | 6.3.0 (2026-07-18) | Truce License 1.0 (not OSI) | No | CLAP/VST3/VST2/LV2/AU/AAX |
| Windowing | baseview | 0.3.4 (2026-09-12) | MIT/Apache | No | Plugin GUI windows |
| LV2 authoring | lv2 / lv2-sys | 0.6.0 / 2.0.0 (2020-10-23) | MIT/Apache | No | **Abandoned** |
| LV2 host | livi | 0.7.5 (2024-11-12) | MIT | No | WIP |
| Plugin host | plugin_host | 0.1.0 (2026-02-25) | MIT | No | **Bridges are stubs** |
| Decode | symphonia | 0.6.1 (2026-08-13) | MPL-2.0 | Likely (pure Rust) | No Opus, no encode |
| WAV | hound | 3.5.1 (2023-09-25) | Apache-2.0 | Likely | Stable, dormant |
| WAV | wavers | 1.5.1 (2024-12-29) | MIT | Unverified | |
| FLAC dec | claxon | 0.4.3 (2020-08-09) | Apache-2.0 | Likely | Dormant (use symphonia) |
| Vorbis dec | lewton | 0.10.2 (2021-01-20) | MIT/Apache | Likely | Dormant (use symphonia) |
| Multi-decode | audrey | 0.3.0 (2021-01-14) | MIT/Apache | – | **Abandoned** |
| Disk streaming | creek | 1.2.3 (2025-09-22) | MIT/Apache | No (threads/fs) | Meadowlark, Codeberg |
| FLAC enc | flacenc | 0.5.1 (2025-12-18) | Apache-2.0 | Likely (pure Rust) | |
| Vorbis enc | vorbis_rs | 0.5.6 (2026-07-30) | BSD-3 | Hard (C) | aoTuV+Lancer |
| MP3 enc | mp3lame-encoder | 0.2.5 (2026-08-20) | LGPL-3.0 | Hard (C) | LGPL |
| Opus | opus | 0.4.0 (2026-08-23) | MIT/Apache | Hard (C, cmake) | libopus bindings |
| Opus | opus-rs | 0.1.34 (2026-09-22) | BSD-3 | Yes (claimed) | Pure-Rust port of libopus 1.6; young |
| Opus | audiopus | 0.2.0 (2021-04-22) | ISC | – | Stale |
| Resample | rubato | 5.0.0 (2026-08-10) | MIT/Apache | Likely | Realtime-safe |
| SF2 | rustysynth | 1.3.6 (2025-08-10) | MIT | Likely (no deps) | |
| SF2 | oxisynth | 0.1.0 (2025-05-25) | LGPL-2.1 | Yes (demo) | FluidSynth-inspired |
| SFZ | (sfizz bindings) | none on crates.io | – | – | Gap |
| MIDI I/O | midir | 0.11.0 (2026-04-18) | MIT | Yes (Web MIDI) | No Safari |
| MIDI msgs | wmidi | 4.0.11 (2026-04-19) | MIT | Yes | no_std, alloc-free |
| MIDI msgs+SMF | midi-msg | 0.9.0 (2026-05-08) | MIT | Yes | MPE, MTC |
| SMF | midly | 0.5.3 (2023-01-01) | Unlicense | Yes | Stable, fast |
| SMF playback | nodi | 1.0.3 (2025-01-01) | MIT | Partial | midly+midir |
| MIDI 2.0 | midi2 | 0.11.1 (2026-04-09) | MIT/Apache | Likely | Early dev |
| OSC | rosc | 0.11.4 (2025-03-23) | MIT/Apache | Codec yes; no UDP in browser | |
| Audio I/O | cpal | 0.18.2 (2026-08-16) | Apache-2.0 | Yes (wasm-bindgen backend) | |
| DSP | fundsp | 0.23.0 (2026-01-07) | MIT/Apache | Likely | |

Sources for the table: see the per-crate citations above (crates.io pages, queried 2026-09-26). cpal's wasm-bindgen backend is referenced in the [web-audio-api README](https://crates.io/crates/web-audio-api).
