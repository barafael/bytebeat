# Rust GUI frameworks for a mini-DAW UI (native desktop + browser from one codebase), as of September 2026

Scope: step-sequencer grid, piano roll, pattern/song timeline, mixer (knobs/faders/meters), waveform and spectrum displays, drag and drop, collaborator presence cursors. Criteria: native + web from one codebase, 60fps dense custom rendering, custom-widget ergonomics, audio widget availability, accessibility, maturity/maintenance, license, audio track record.

Method note: version numbers and dates come from the crates.io API, queried 2026-09-26 (cited as crates.io crate pages). Several primary sites (docs.rs, lib.rs, dioxuslabs.com, v2.tauri.app, codeberg.org, billydm.github.io, boringcactus.com, news.ycombinator.com, discourse.iced.rs) were blocked by the research sandbox's egress proxy. For those, the claims below rely on search-engine snippets of the pages and are marked "(via search snippet)". Treat them as slightly lower confidence.

## egui / eframe: web support, custom-painting performance, audio widgets, audio apps, current version

### Takeaway
egui is the most mature single-codebase native+web option. eframe 0.36.2 (2026-09-08) runs on wgpu by default, uses WebGPU on the web and falls back to WebGL, supports file drag-and-drop on both targets, and has AccessKit on native. It has the largest audio-widget ecosystem of the Rust GUIs (knobs, faders, meters, plots, node graphs), though most of those crates are small, single-author projects. Immediate mode costs CPU, because layout and tessellation run every frame. That matters for meters that need a repaint every frame and for plots with very many points. The escape hatch is a custom wgpu `PaintCallback`.

### Cited Findings
- **Version and license:** egui/eframe 0.36.2 was released 2026-09-08, after 0.36.1 (2026-08-07), 0.36.0 (2026-08-05) and 0.35.0 (2026-06-25). The license is MIT OR Apache-2.0. The crate has about 24.5M total downloads and about 5.6M recent ones. — [crates.io egui](https://crates.io/crates/egui), [crates.io eframe](https://crates.io/crates/eframe)
- **0.36.0 release focus:** "drastically improves the mobile keyboard experience (when using eframe web)". It also adds drag-to-open panels and window chrome theme sync, and makes IME, autocomplete and autocorrect work on iOS and Android through eframe web. — [egui releases](https://github.com/emilk/egui/releases), [newreleases 0.36.0](https://newreleases.io/project/github/emilk/egui/release/0.36.0) (via search snippet)
- **Renderer:** eframe 0.34.0 (2026-03-26) "Make `wgpu` the default renderer for `eframe` and egui.rs". Since 0.26.0 (2024-02-05), "When using `wgpu` on web, `eframe` will try to use WebGPU if available, then fall back to WebGL". 0.34.1 (2026-03-27) re-enabled the WebGL fallback for the wgpu backend. — [eframe CHANGELOG](https://github.com/emilk/egui/blob/main/crates/eframe/CHANGELOG.md)
- **Backends:** the README lists `egui_glow` for "rendering egui with glow on native and web" and `egui-wgpu` for the WebGPU API. — [egui README](https://github.com/emilk/egui)
- **Stated performance:** "For most cases you can expect `egui` to take up 1-2 ms per frame, but `egui` still has a lot of room for optimization." — [egui README](https://github.com/emilk/egui)
- **Stability caveat:** "If you want something that doesn't break when you upgrade it, egui isn't for you (yet)." The project ships breaking 0.x releases every few months (0.35 in June 2026, 0.36 in August 2026). — [egui README](https://github.com/emilk/egui), [crates.io egui](https://crates.io/crates/egui)
- **Immediate-mode layout paradox:** "to know the size of the window, we must do the layout, but the layout code also checks for interaction". A two-pass mode was investigated in issue #843. — [egui README](https://github.com/emilk/egui), [egui issue #843](https://github.com/emilk/egui/issues/843)
- **Accessibility:** AccessKit integration was "optional, but enabled by default" from 0.20.0 (2022-12-08). The README says AccessKit "currently implements the native accessibility APIs on Windows and macOS". On the web, "there is an experimental built-in screen reader". — [eframe CHANGELOG](https://github.com/emilk/egui/blob/main/crates/eframe/CHANGELOG.md), [egui README](https://github.com/emilk/egui)
- **Multiple windows:** "Multiple viewports/windows" arrived in eframe 0.24.0 (2023-11-23). — [eframe CHANGELOG](https://github.com/emilk/egui/blob/main/crates/eframe/CHANGELOG.md)
- **File drag-and-drop:**
  - "Added dragging and dropping files into egui" landed in 0.14.0 (2021-08-24).
  - 0.36.0 (2026-08-05) stores `web_sys::File` inside `DroppedFile`, so dropped audio files can be read in the browser.
  - [eframe CHANGELOG](https://github.com/emilk/egui/blob/main/crates/eframe/CHANGELOG.md)
- **Web clipboard:** copy/cut and click-to-copy fixes for Safari landed in 0.24.0/0.24.1 (Nov 2023). — [eframe CHANGELOG](https://github.com/emilk/egui/blob/main/crates/eframe/CHANGELOG.md)
- **Custom-paint performance pitfall:** an egui_plot with 500k points drove CPU to about 50% while the GPU sat idle. Profiling blamed tessellation and repeated vector reallocation. The suggested fixes were pre-reserving capacity, a custom wgpu `PaintCallback` that bypasses CPU tessellation, and downsampling. — [egui discussion #3810](https://github.com/emilk/egui/discussions/3810)
- **Complex UIs tax the CPU:** a very large UI in a scroll area with long scrollback costs CPU because content is laid out every frame. An optimization tracking issue exists. — [egui optimization tracking #1196](https://github.com/emilk/egui/issues/1196), [egui discussion #5587](https://github.com/emilk/egui/discussions/5587) (via search snippet)
- **Audio and related widget crates (all MIT or MIT/Apache):**

  | Crate | Latest release | What it provides |
  |---|---|---|
  | `egui_knob` | 0.6.3 (2026-09-01) | Knob with Wiper/Dot styles and label positions |
  | `armas-audio` | 0.2.2 (2026-05-14) | "Audio/DAW UI components for egui — faders, knobs, timelines, MIDI controllers"; docs also list piano roll and mixer strip. Only 257 downloads |
  | `egui-cha-ds` | 0.9.0 (2026-07-16) | Design system that includes a vertical fader, rotary knob and level meter |
  | `egui_fader` | 0.1.2 (2025-06-12) | Fader |
  | `egui-audio` | GitHub only, not on crates.io | Collection of audio widgets |
  | `egui_plot` | 0.37.0 (2026-08-05) | Plots |
  | `egui_dnd` | 0.17.0 (2026-08-06) | Drag and drop for reordering |
  | `egui-snarl` | 0.12.0 (2026-08-26) | Node graphs |

  Sources: [crates.io egui_knob](https://crates.io/crates/egui_knob), [crates.io armas-audio](https://crates.io/crates/armas-audio), [armas_audio docs](https://docs.rs/armas-audio/latest/armas_audio/) (via search snippet), [crates.io egui-cha-ds](https://crates.io/crates/egui-cha-ds), [egui_cha_ds docs](https://docs.rs/egui-cha-ds/latest/egui_cha_ds/) (via search snippet), [crates.io egui_fader](https://crates.io/crates/egui_fader), [egui-audio GitHub](https://github.com/Cannedfood/egui-audio), [crates.io egui_plot](https://crates.io/crates/egui_plot), [crates.io egui_dnd](https://crates.io/crates/egui_dnd), [crates.io egui-snarl](https://crates.io/crates/egui-snarl)
- **Plugin hosting:**
  - `egui-baseview` 0.7.2 (2026-09-18) now lives under Codeberg RustAudio. It depends on egui ^0.36.1, baseview ^0.3.4 and wgpu ^30, with both egui-wgpu and egui_glow.
  - `baseview` 0.3.4 was released 2026-09-12.
  - [crates.io egui-baseview](https://crates.io/crates/egui-baseview), [crates.io baseview](https://crates.io/crates/baseview)
- **nih-plug's view of its own egui adapter:** the `nih_plug_egui` README says "Consider using `nih_plug_iced` or `nih_plug_vizia` instead." It gives no reason. — [nih_plug_egui README](https://github.com/robbert-vdh/nih-plug/blob/master/nih_plug_egui/README.md)
- **nice-plug adapter:** `nice-plug-egui` 0.5.1 (2026-09-14) is the egui adapter for nice-plug, the maintained community fork of nih-plug. It is the most-downloaded nice-plug GUI adapter, with about 2.7k downloads against 637 for the iced adapter. — [crates.io nice-plug-egui](https://crates.io/crates/nice-plug-egui), [crates.io nice-plug-iced](https://crates.io/crates/nice-plug-iced)
- **Audio apps on egui:**
  - "Yadaw", a lightweight cross-platform DAW built in Rust with egui (via an AlternativeTo listing snippet).
  - sowbug/groove, a DAW engine whose egui GUI is described as "read-only and very incomplete".
  - An experimental Glicol front end in egui (via search snippet).
  - An HN commenter: "in terms of audio effect plugins, I think EGUI is usable".
  - The egui README names only Rerun Viewer as a flagship app; no audio apps.
  - Sources: [groove](https://github.com/sowbug/groove), [HN comment](https://news.ycombinator.com/item?id=35958029) (via search snippet), [Yadaw on AlternativeTo](https://alternativeto.net/software/yadaw/?p=4) (via search snippet), [egui README](https://github.com/emilk/egui)
- **Games/visualizers:** `bevy_egui` 0.42.0 (2026-08-16, MIT) keeps egui usable inside Bevy. — [crates.io bevy_egui](https://crates.io/crates/bevy_egui)

### Inferences
- egui is the lowest-risk option for "one Rust codebase, native plus browser, with custom widgets". eframe web is first-party and is exercised by the egui.rs demo. Rendering goes through the same wgpu path on both targets, and file drop works on both.
- For a DAW, widget density is less of a problem than steady-state repaint. Meters and playheads force a repaint every frame, so the full UI layout runs at 60fps. Mitigations:
  - keep grid, piano roll and waveform as single custom-painted regions (one `Painter`/mesh), not thousands of `ui.button`s;
  - pre-compute waveform peak mipmaps;
  - use `PaintCallback` with wgpu for waveform and spectrum when point counts get large.
- nih-plug's "consider iced/vizia" note probably reflects immediate-mode costs in plugin hosts and egui's frequent breaking changes. The source gives no reason, so this is inference only.

### Gaps
- No authoritative list of production audio apps built on egui was found. The Yadaw evidence comes only from an AlternativeTo listing.
- Whether egui's multi-viewport works on web (most likely it falls back to in-canvas windows) was not confirmed from docs.
- AccessKit's web adapter status for egui in 2026 is unclear beyond the README's "experimental built-in screen reader" on web.

## iced: web support status, custom widgets (Canvas/Shader), audio usage (iced_audio, nih_plug_iced), current version

### Takeaway
iced 0.14.0 (2025-12-07, MIT) is the current release, with no newer release in the ~10 months since. The old DOM-based `iced_web` is a separate legacy repo. Today iced runs in the browser through wgpu, and 0.14 made WebGPU the default instead of WebGL. `Canvas` with `Cache` covers retained custom drawing, and a `shader` widget exposes raw wgpu. iced_audio has been revived: 0.17.1 (2026-09-24) targets iced 0.14 and nice-plug. Upstream iced has no accessibility (AccessKit) support.

### Cited Findings
- **Version history:** iced 0.14.0 was published 2025-12-07. Before that came 0.13.1 (2024-09-19), which gives a cadence of roughly one release per year. The license is MIT. — [crates.io iced](https://crates.io/crates/iced)
- **0.14.0 release notes:**
  - Reactive rendering (#2662) and time-travel debugging (#2910).
  - An animation API (#2757), headless-mode testing (#2698) and end-to-end testing (#3059).
  - Input method support (#2777), hot reloading (#3000) and comet debugger/devtools foundations.
  - New table, grid, sensor, float and pin widgets.
  - Concurrent image decoding (#3092), primitive culling in column/row (#2611) and wgpu 27.0 (#3097).
  - [iced 0.14.0 release](https://github.com/iced-rs/iced/releases/tag/0.14.0). The fetch tool reported a date of "December 7, 2024", but crates.io shows 2025-12-07. The crates.io date is authoritative.
- **0.14 web changes (via search snippet of the release/Phoronix):**
  - "enables the WebGPU backend in wgpu by default instead of WebGL".
  - Fixed "WebGPU failing to boot in Chromium" and a WebGL crash.
  - "Iced now targets the `#iced` container by default on Wasm".
  - [iced releases](https://github.com/iced-rs/iced/releases), [Phoronix forum](https://www.phoronix.com/forums/forum/software/programming-compilers/1597623-iced-0-14-released-for-popular-rust-cross-platform-gui-library)
- **Legacy DOM runtime:** `iced_web` is a separate WebAssembly runtime that produced VDOM nodes from iced widgets. It is legacy, and web support now goes through wgpu/canvas. — [iced_web repo](https://github.com/iced-rs/iced_web), [discourse on wasm/webgl flag](https://discourse.iced.rs/t/allowing-users-to-compile-wgpu-without-the-webgl-flag-in-wasm-builds/59) (via search snippet)
- **Canvas `Cache`:** it "stores generated Geometry to avoid recomputation, and will not redraw its geometry unless the dimensions of its layer change or it is explicitly cleared". — [iced canvas Cache docs](https://docs.rs/iced/latest/iced/widget/canvas/type.Cache.html) (via search snippet)
- **Known 0.14 Canvas regression:** issue #3173 reports flicker when a cached Canvas redraws large images. — [iced issue #3173](https://github.com/iced-rs/iced/issues/3173)
- **Many-rectangle Canvas performance:** a forum thread exists ("Rendering way too many rectangles on a canvas"), but its content was not retrievable. — [iced discourse](https://discourse.iced.rs/t/rendering-way-too-many-rectangles-on-a-canvas/113)
- **Accessibility:**
  - "As of version 0.14, iced does not depend on accesskit". Accessibility issue #552 has been open since October 2020.
  - The fork `plushie-iced` (0.8.4, 2026-05-08) adds a full AccessKit tree (AT-SPI2, UIA, NSAccessibility).
  - [iced issue #552](https://github.com/iced-rs/iced/issues/552), [plushie-iced](https://github.com/plushie-ui/plushie-iced), [crates.io plushie-iced](https://crates.io/crates/plushie-iced)
- **File drag-and-drop:** requested in issue #158 to expose winit's `DroppedFile`. No evidence was found that file drop works on the iced web target. — [iced issue #158](https://github.com/iced-rs/iced/issues/158)
- **Multi-window helper:** a helper crate, `iced-multi-window`, exists for managing multiple windows. — [crates.io iced-multi-window](https://crates.io/crates/iced-multi-window)
- **iced_audio:**
  - 0.17.1 was released 2026-09-24, after 0.17.0 (2026-09-09), 0.16.0 (2026-08-17) and 0.15.0 (2026-07-29). It depends on iced ^0.14.0 and nice-plug-core ^0.4. The license is MIT.
  - Widgets are `HSlider`, `VSlider`, `Knob`, `Ramp`, `XYPad` and `ModRangeInput`, with modulation-range styles. No meter widget is listed.
  - It has an optional `nice-plug` feature.
  - [crates.io iced_audio](https://crates.io/crates/iced_audio), [iced_audio README](https://github.com/iced-rs/iced_audio)
- **Plugin adapters:** `iced_baseview` 0.5.2 (2026-09-13) lives on Codeberg RustAudio. `nice-plug-iced` 0.4.1 was released 2026-09-14. — [crates.io iced_baseview](https://crates.io/crates/iced_baseview), [crates.io nice-plug-iced](https://crates.io/crates/nice-plug-iced)
- **nih-plug's recommendation:** nih-plug recommends iced or vizia over egui for plugin GUIs. — [nih_plug_egui README](https://github.com/robbert-vdh/nih-plug/blob/master/nih_plug_egui/README.md)
- **Positioning in surveys:** iced is "functional, declarative, inspired by Elm" and "has a more native feel than egui but requires more setup". — [Wren Learns Rust, 2026 landscape](https://wrenlearnsrust.com/posts/2026-03-11-rust-gui-landscape-2026.html) (via search snippet)

### Inferences
- iced is a credible native+web choice for a DAW. The Elm-style update/view fits a CRDT/collaboration state model well, and Canvas caches let a static grid or waveform avoid re-tessellation while an uncached overlay layer carries the playhead, meters and presence cursors.
- The main risks for iced:
  - no accessibility upstream;
  - roughly yearly releases;
  - WebGPU-by-default on web, so Safari/Firefox coverage depends on the WebGL fallback feature;
  - unverified file drop in the browser.
- iced_audio's revival under nice-plug (four releases in July–September 2026) makes iced plus iced_audio the best-supplied option for knobs and sliders among retained-mode Rust GUIs. Meters would still need custom drawing.
- From prior knowledge, unverified here: iced has built-in multi-window via `iced::daemon` since 0.13.

### Gaps
- No iced version after 0.14.0 was found. Whether 0.15 is imminent is unknown.
- Web clipboard, file drop and IME behaviour of iced in the browser could not be confirmed.
- The iced discourse thread about many-rectangle performance could not be retrieved (DNS failure).

## vizia: audio-plugin-oriented GUI, web support, status

### Takeaway
vizia (0.4.0, 2026-04-23, MIT) is a declarative, retained, CSS-styled Rust GUI that ships a baseview backend for audio plugins and includes AccessKit accessibility. It is the GUI behind nih-plug's `nih_plug_vizia`. Web/WASM support exists only as "some examples run" and is not a first-class target. Its crates.io footprint is small (about 6.7k downloads), so treat it as a niche, plugin-first toolkit.

### Cited Findings
- **Versions:** vizia 0.4.0 was released 2026-04-23. Earlier releases were 0.3.0 (2025-04-16), 0.2.0 (2024-11-28) and 0.1.0 (2021-09-17). The license is MIT, and the crate has 6,651 total downloads. — [crates.io vizia](https://crates.io/crates/vizia)
- **README features:**
  - "Make your applications accessible to assistive technologies such as screen readers, powered by accesskit".
  - Over 25 ready-made views and 4,250+ Tabler SVG icons.
  - Reactive bindings ("Views derive from application state").
  - Stylesheets with hot-reloading, and animations.
  - "Vizia provides an alternative baseview windowing backend for audio plugin development".
  - [vizia README](https://github.com/vizia/vizia)
- **Web:** the library supports WebAssembly via `cargo run-wasm --release --example name`, "though some examples are not compatible with the web target". The README fetched from GitHub did not mention web. — [lib.rs vizia / docs.rs vizia](https://docs.rs/crate/vizia/latest) (via search snippet), [vizia README](https://github.com/vizia/vizia)
- **Plugin example:** a vizia + nih-plug plugin example repo exists. — [vizia-plug](https://github.com/vizia/vizia-plug)
- **nice-plug adapter:** `nice-plug-vizia` is not published on crates.io; the API query returned "not found". nice-plug publishes only egui and iced adapters there. — [crates.io nice-plug-egui](https://crates.io/crates/nice-plug-egui), [crates.io nice-plug-iced](https://crates.io/crates/nice-plug-iced)

### Inferences
- vizia is well suited to plugin-style UIs (knobs, parameter panels) and has accessibility. It is a poor fit for the "same codebase in the browser" requirement, because web is not a first-class target and its web examples are partial.
- If the project later wants a CLAP/VST3 plugin form of the synth, vizia remains relevant only as a plugin GUI, not as the DAW shell.

### Gaps
- No confirmation was found of which specific nih-plug plugins use vizia (from memory: Diopser and Spectral Compressor), or of any 2026 audio apps using vizia.
- Vizia's knob/meter widget set (built-in or companion crates) was not verified.
- Whether nice-plug hosts a vizia adapter in its repo (unpublished) could not be checked, because Codeberg was blocked.

## Slint: web (WASM) support, licensing, custom drawing, audio usage

### Takeaway
Slint 1.18.1 (2026-09-21) is the most "product-like" toolkit: frequent releases, a DSL with tooling, and paid support. Its web target renders to a WebGL `<canvas>` with no DOM, and Slint itself does not recommend it for general web apps. Screen readers are "not available" on web. It has a triple license: GPLv3, a royalty-free license for desktop/mobile/web that requires attribution, or commercial. Custom dense drawing works through the `Path` element or wgpu interop (1.12+). No audio-app track record was found.

### Cited Findings
- **Version and license:** Slint 1.18.1 was released 2026-09-21, 1.18.0 on 2026-09-16 and 1.17.0 on 2026-06-24. The license is `GPL-3.0-only OR LicenseRef-Slint-Royalty-free-2.0 OR LicenseRef-Slint-Software-3.0`. — [crates.io slint](https://crates.io/crates/slint)
- **Royalty-free license terms:** it grants a "royalty-free, non-exclusive license to use ... distribute the Software as part of a Desktop, Mobile, or Web Application". It does not permit use in embedded systems, or distribution of an application "that exposes the APIs" of Slint. — [Slint Royalty-free License 2.0 PDF](https://slint.dev/agreements/slint-royalty-free-license.pdf), [LicenseRef-Slint-Royalty-free-2.0.md](https://github.com/slint-ui/slint/blob/master/LICENSES/LicenseRef-Slint-Royalty-free-2.0.md)
- **Attribution requirement:** under the royalty-free license you must either include the `AboutSlint` widget "in an 'About' screen or dialog that is accessible from the top level menu" or display "the Slint attribution badge on a public webpage". — [Slint FAQ](https://github.com/slint-ui/slint/blob/master/FAQ.md)
- **Web platform (official docs, via search snippet):**
  - Slint "renders your UI into a HTML canvas element using WebGL, without using the DOM or CSS".
  - "Accessibility features (such as screen readers) are not available".
  - Text is rendered by Slint, not the browser.
  - Running in the browser "is currently not recommended for building general-purpose web applications". The suggested uses are demos, apps "where the web is not the primary platform", and tools/dashboards.
  - [Slint Web docs](https://docs.slint.dev/latest/docs/slint/guide/platforms/web/)
- **Custom rendering:** Slint 1.12 added wgpu integration "to embed WGPU based rendering libraries such as Bevy". You can render an underlay/overlay via `Window::set_rendering_notifier()` or import a `wgpu::Texture` via `slint::Image::try_from`. The wgpu dependency later moved to 29 behind an `unstable-wgpu-29` feature. — [Slint 1.12 blog](https://slint.dev/blog/slint-1.12-released), [slint::wgpu_28 docs](https://docs.slint.dev/latest/docs/rust/slint/wgpu_28/), [Slint CHANGELOG](https://github.com/slint-ui/slint/blob/master/CHANGELOG.md) (via search snippet)
- **Real-time plotting:** a community repo does real-time plotting by rendering into a wgpu texture and compositing it into the Slint scene. A GitHub discussion asks whether Slint is good for real-time data visualization in the browser. — [slint_realtime_plotting_experiments](https://github.com/okhsunrog/slint_realtime_plotting_experiments), [Slint discussion #2801](https://github.com/slint-ui/slint/discussions/2801)
- **Accessibility on Windows:** Slint (with Dioxus, egui and WinSafe) "fully support[s] Windows Narrator", and Slint handles IME correctly. — [boringcactus 2025 survey](https://www.boringcactus.com/2025/04/13/2025-survey-of-rust-gui-libraries.html) (via search snippet)

### Inferences
- For a DAW, Slint's DSL suits standard panels (mixer strip layout, dialogs). Dense, custom, per-frame surfaces (grid, piano roll, waveform, spectrum) would likely need the wgpu-texture route. Getting that same custom-render path working in the WebGL browser build adds risk.
- The web target is explicitly second-class, and the royalty-free license's attribution clause is acceptable but non-trivial. An open-source GPL project could use GPLv3.

### Gaps
- No audio apps or plugins built with Slint were found.
- Whether Slint's wgpu interop works on the wasm/WebGL target was not confirmed.
- No data was found on Slint web performance with thousands of elements.

## Web-tech UIs: Dioxus (webview / web / Blitz native), Leptos/Yew + Tauri; audio-engine integration over IPC and AudioWorklet

### Takeaway
The "web UI everywhere" approach (Dioxus, or Leptos/Yew inside Tauri) gives the best browser story, with real DOM, accessibility and CSS. On desktop, though, the UI runs in a webview separate from the native Rust audio engine. Every meter or waveform update must cross IPC, and Tauri's docs say events are "not designed for low latency or high throughput". Use Tauri Channels or raw-binary responses and batch at display rate. Dioxus desktop's `eval` bridge chokes on large payloads. Dioxus Native (Blitz, wgpu) removes the webview but is still labelled experimental. In the browser, the engine belongs in an AudioWorklet running WASM, with SharedArrayBuffer ring buffers for meters. That requires COOP/COEP cross-origin isolation.

### Cited Findings
- **Dioxus versions:** stable 0.7.10 was released 2026-07-30 and 0.8.0-alpha.1 on 2026-07-31. The license is MIT OR Apache-2.0. — [crates.io dioxus](https://crates.io/crates/dioxus)
- **Dioxus 0.7.0 (2025-10-31):** introduced "the first-ever version of Dioxus Native: a new renderer that paints Dioxus apps entirely on the GPU with WGPU". The 0.7.0-alpha.0 came out 2025-05-14. — [Dioxus 0.7 release](https://github.com/DioxusLabs/dioxus/releases/tag/v0.7.0), [Dioxus 0.7 blog](https://dioxuslabs.com/blog/release-070/) (via search snippet)
- **Dioxus Native status:** described as "an experimental renderer that runs on desktop and mobile platforms". `dioxus-native` 0.8.0-alpha.1 was released 2026-07-31. — [crates.io dioxus-native](https://crates.io/crates/dioxus-native), [Dioxus native docs mirror](https://mintlify.wiki/DioxusLabs/dioxus/platforms/native) (via search snippet)
- **Blitz:**
  - Blitz is "a radically modular HTML/CSS rendering engine" built from stylo, html5ever, taffy, parley, accesskit, vello and wgpu.
  - The stable release is 0.2.1; 0.3.0-beta.2 came out 2026-08-24.
  - A `<canvas>` with an `onpaint`/`CustomPaintCtx` custom-paint hook is described in docs (via search snippet). An issue asks how to get a wgpu surface from a canvas.
  - Sources: [Blitz repo](https://github.com/DioxusLabs/blitz), [crates.io blitz](https://crates.io/crates/blitz), [Dioxus issue #3725](https://github.com/DioxusLabs/dioxus/issues/3725)
- **Dioxus desktop:**
  - It uses the system WebView via wry, and "your Rust code runs natively, which means that browser APIs are not available, so rendering WebGL, Canvas, etc is not as easy as the Web".
  - The `eval` path "cannot handle large payloads" and "freezes both the WebView and the backend when the payload exceeds 100k elements in debug mode, and 1M elements in release mode".
  - [Dioxus desktop guide](https://dioxuslabs.com/learn/0.7/guides/platforms/desktop/) (via search snippet), [Dioxus issue #1915](https://github.com/DioxusLabs/dioxus/issues/1915)
- **Tauri versions:** Tauri 2.12.0 was released 2026-09-26, alongside 3.0.0-alpha.3 the same day. The license is Apache-2.0 OR MIT. wry is at 0.57.0 (2026-09-08). — [crates.io tauri](https://crates.io/crates/tauri), [crates.io wry](https://crates.io/crates/wry)
- **Tauri events vs channels:**
  - "The event system is not designed for low latency or high throughput situations". Event payloads are always JSON strings, "unsuitable for larger messages".
  - "Channels are designed to be fast and deliver ordered data". Tauri itself uses them for download progress, child-process output and WebSocket messages.
  - [Tauri docs source: calling-frontend.mdx](https://github.com/tauri-apps/tauri-docs/blob/v2/src/content/docs/develop/calling-frontend.mdx)
- **Tauri binary IPC:**
  - Tauri v2 offers `ipc::Response` for raw `Vec<u8>` and `Channel` for push streaming, but no codec/framing layer.
  - The third-party `tauri-wire` claims a 65,536-byte blob round-trips in 600µs against 6.7ms for Tauri's JSON path. These are the author's own benchmarks.
  - A "non-scientific benchmark" of binary IPC reported about 5ms on macOS but about 200ms on Windows for 10MB.
  - [tauri-wire](https://github.com/userFRM/tauri-wire), [tauri-conduit benchmarks](https://github.com/userFRM/tauri-conduit/blob/master/BENCHMARKS.md), [Tauri issue #7127](https://github.com/tauri-apps/tauri/issues/7127), [Tauri discussion #5690](https://github.com/tauri-apps/tauri/discussions/5690) (via search snippets; the exact thread for the 5ms/200ms figure was not pinned down)
- **Leptos and Yew:** Leptos 0.8.21 was released 2026-09-26 (0.9.0-beta2 the same day). Yew 0.23.0 was released 2026-03-10. — [crates.io leptos](https://crates.io/crates/leptos), [crates.io yew](https://crates.io/crates/yew)
- **AudioWorklet constraints:**
  - AudioWorklets can run WASM DSP, but `fetch()` and dynamic `import()` are forbidden in `AudioWorkletGlobalScope`.
  - A WASM ring buffer bridges the 128-frame render quantum and larger DSP block sizes.
  - SharedArrayBuffer with Atomics avoids MessagePort serialization for metering.
  - SharedArrayBuffer requires cross-origin isolation (COOP/COEP headers, secure context).
  - [Chrome audio worklet design pattern](https://developer.chrome.com/blog/audio-worklet-design-pattern), [Chrome Labs WASM ring buffer sample](https://googlechromelabs.github.io/web-audio-samples/audio-worklet/design-pattern/wasm-ring-buffer/), [Chrome Labs SAB + Worker sample](https://googlechromelabs.github.io/web-audio-samples/audio-worklet/design-pattern/shared-buffer/), [WorkAdventure worklet write-up](https://workadventu.re/tech/building-an-easy-to-use-browser-noise-suppression-library-in-an-audio-worklet/), [videocall.rs AudioWorklet article](https://engineering.videocall.rs/posts/how-to-make-javascript-audio-not-suck/) (via search snippets)
- **Survey framing:** "Diet Electron is definitely better than regular Electron". The same survey says Dioxus supports Windows Narrator and IME. — [boringcactus 2025 survey](https://www.boringcactus.com/2025/04/13/2025-survey-of-rust-gui-libraries.html) (via search snippet)

### Inferences
- The web-UI route gives the best accessibility and familiar DnD/clipboard/shortcut semantics, and the browser build is "native" there. It creates two engine hosts: native cpal behind Tauri IPC on desktop, and WASM-in-AudioWorklet in the browser.
- To keep meter and waveform traffic cheap on desktop:
  - send decimated peak/RMS frames at 30–60Hz over a Tauri Channel as binary, not JSON events;
  - send waveform overviews once per clip, then draw on `<canvas>`/WebGL in the webview.
  - A simpler and more symmetric alternative is to run the engine as WASM in an AudioWorklet in both the browser and the Tauri webview, which makes the webview "just a browser". The cost is WebView audio-latency and platform variance, especially WebKitGTK on Linux; this is an inference, not verified here.
- Dioxus desktop's eval bottleneck makes it a weak fit for 60fps meters unless drawing moves fully into web-side JS/WASM. Dioxus Native/Blitz is promising but experimental and has no web target of its own, since on web Dioxus renders to the DOM. The custom-paint canvas API would need a separate web path.

### Gaps
- No measured latency numbers for Tauri Channel throughput at 60Hz meter rates were found.
- No first-hand evidence of an audio app shipped on Dioxus or Tauri was found.
- The WebKitGTK/WKWebView AudioWorklet latency comparison was not researched.

## Other frameworks with traction: Freya, Xilem/Masonry, gpui, Makepad, Bevy UI / bevy_egui, nannou (plus stale ones)

### Takeaway
Only Makepad combines native and web from one codebase with a proven music demo, the Ironfish synth. But Makepad has not published to crates.io since 1.0.0 (2025-05-13) and has repositioned as an "AI-accelerated" environment. gpui gained experimental web support (merged 2026-02-26) but its crates.io release is stale (0.2.2, Oct 2025). Freya is native-only (Skia). Xilem is explicitly experimental, and its web backend is DOM-based and separate. Bevy plus bevy_egui runs on web but is a game engine. nannou 0.20 (2026-06-22) is a creative-coding framework rebuilt on Bevy. Floem, Cushy and Yarrow are stale.

### Cited Findings
- **Makepad (status):**
  - `makepad-widgets`, `makepad-platform`, `makepad-audio-graph` and `makepad-example-ironfish` are all at 1.0.0 (2025-05-13), with no later crates.io release.
  - The README describes Makepad as "An AI-accelerated application and game development environment for Rust" that compiles to "wasm/webGL, osx/metal, windows/dx11 linux/opengl". Web builds use `cargo makepad wasm run -p <example> --release`.
  - The license is MIT per the README; the crates are MIT OR Apache-2.0.
  - [Makepad README](https://github.com/makepad/makepad), [crates.io makepad-widgets](https://crates.io/crates/makepad-widgets), [crates.io makepad-audio-graph](https://crates.io/crates/makepad-audio-graph)
- **Makepad (Ironfish):**
  - Ironfish is "an electronic synthesizer written entirely in Makepad Framework".
  - A third-party-hosted web build exists at shades-makepad.apps.loskutoff.com.
  - The repo was reported (via search snippet) as updated 2026-09-01, and `makepad-studio` on crates.io is a "Placeholder for the Studio2 desktop client crate".
  - [makepad-example-ironfish on lib.rs](https://lib.rs/crates/makepad-example-ironfish) (via search snippet), [Ironfish web build](https://shades-makepad.apps.loskutoff.com/), [makepad-studio on lib.rs](https://lib.rs/crates/makepad-studio) (via search snippet)
- **gpui (release and web):**
  - crates.io `gpui` 0.2.2 was released 2025-10-22 under Apache-2.0. Development happens inside the Zed repo.
  - PR #50228, "GPUI on the web", implements "a basic web platform ... targeting wasm32-unknown-unknown" and was merged 2026-02-26.
  - Noted limitations: blocking sync primitives cannot use `Atomics.wait` on the main thread. Follow-up TODOs mention app menus and input events.
  - A community discussion shows an experimental browser build of Zed.
  - [crates.io gpui](https://crates.io/crates/gpui), [Zed PR #50228](https://github.com/zed-industries/zed/pull/50228), [Zed discussion #60629](https://github.com/zed-industries/zed/discussions/60629)
- **gpui (components):** `gpui-component` 0.6.6 (2026-09-21) offers "60+ desktop UI components for GPUI" and has about 80k recent downloads. — [crates.io gpui-component](https://crates.io/crates/gpui-component)
- **Freya:**
  - 0.4.3 (stable) was released 2026-08-30, and 0.5.0-rc.7 on 2026-09-21. The license is MIT.
  - 0.4 "no longer depends on Dioxus" and has its own reactive model. Its core can be embedded in other backends.
  - Repo descriptions say "Cross-platform native GUI library" and, in older forks, "non-web GUI library ... powered by Skia".
  - [crates.io freya](https://crates.io/crates/freya), [Freya 0.4 post](https://freyaui.dev/posts/0.4) (via search snippet), [Freya repo](https://github.com/marc2332/freya)
- **Xilem/Masonry:**
  - Both are at 0.4.0 (2025-10-29) under Apache-2.0, with no newer crates.io release.
  - Xilem is "An experimental Rust native UI framework". Masonry is a fork of the discontinued Druid.
  - `xilem_web` targets the browser DOM, a separate backend from Masonry.
  - [crates.io xilem](https://crates.io/crates/xilem), [xilem repo](https://github.com/linebender/xilem), [Linebender October 2025](https://linebender.org/blog/tmil-22/) (via search snippet)
- **Bevy:** 0.19.1 was released 2026-08-13 and 0.20.0-rc.1 on 2026-09-15. `bevy_egui` 0.42.0 was released 2026-08-16. File drag-and-drop on the WASM target has a known issue. — [crates.io bevy](https://crates.io/crates/bevy), [crates.io bevy_egui](https://crates.io/crates/bevy_egui), [Bevy issue #6822](https://github.com/bevyengine/bevy/issues/6822)
- **nannou:** 0.20.0 (2026-06-22) is the first release since 0.19.0 (2024-01-17) and is rebuilt on Bevy 0.19. Render-to-texture work was still under discussion in September 2026. — [crates.io nannou](https://crates.io/crates/nannou), [nannou issue #1048](https://github.com/nannou-org/nannou/issues/1048), [nannou issue #1097](https://github.com/nannou-org/nannou/issues/1097)
- **Stale projects (flag as unmaintained or slow):**
  - Floem 0.2.0 (2024-11-14).
  - Cushy 0.4.0 (2024-08-20).
  - Yarrow 0.0.1 (2024-06-06), described as "A non-declarative GUI library in Rust with extreme performance and control, geared towards audio software".
  - [crates.io floem](https://crates.io/crates/floem), [crates.io cushy](https://crates.io/crates/cushy), [crates.io yarrow](https://crates.io/crates/yarrow)
- **Meadowlark:** Meadowlark (a Rust DAW) is on hiatus. Its author, BillyDM, wrote "DAW Frontend Development Struggles". He found that DAW GUIs "might just have one of the most complicated GUIs out of any piece of software". Waveforms need peak search and pixel-by-pixel rendering, automation Béziers are slow, and piano-roll clips contain lots of rectangles. He considered Flutter, Dioxus, GTK4 and JUCE, then created Yarrow. His audio engine `firewheel` is active (0.14.0, 2026-09-05). — [DAW Frontend Development Struggles](https://billydm.github.io/blog/daw-frontend-development-struggles/) (via search snippet), [Why I'm Taking a Break from Meadowlark](https://billydm.github.io/blog/why-im-taking-a-break-from-meadowlark/) (via search snippet), [crates.io firewheel](https://crates.io/crates/firewheel)

### Inferences
- Makepad is the closest fit on paper: GPU shader-based widgets, native plus WebGL, and a synth UI demo. Its crates.io release has been frozen for 16 months, its focus has shifted, and it uses a bespoke DSL and build tool (`cargo makepad`), which makes it a maintenance and hiring risk.
- gpui's web support is too new and experimental (Feb 2026) to bet on for the browser half.
- Freya, Floem, Cushy and Yarrow either lack web support or are stale.
- Bevy is only sensible if the project wants a game-engine-like visualizer; for DAW panels you would still use egui via bevy_egui.

### Gaps
- No 2026 official Makepad web demo URL or statement of Makepad's release plans was found.
- The Ironfish example page on GitHub returned 404, so the example may have moved.
- The gpui web backend's GPU API (WebGPU vs WebGL) is not stated in the PR.

## Cross-target platform features: multi-window, file dialogs (rfd on web), drag-and-drop of audio files, clipboard, shortcuts, high-DPI

### Takeaway
On the web, "multi-window" and native file dialogs don't exist in the usual sense. rfd works on WASM only through `AsyncFileDialog`, and there saving happens when `FileHandle::write` triggers the browser's download prompt. egui/eframe is the only canvas-based Rust toolkit found with documented file drag-and-drop on both native and web (`web_sys::File` in `DroppedFile` since 0.36). The DOM approaches (Dioxus/Leptos/Tauri) get browser-native DnD, clipboard and shortcuts for free.

### Cited Findings
- **rfd on WASM:**
  - rfd 0.17.2 was released 2026-01-12 under MIT.
  - `AsyncFileDialog` is supported on WASM32 ("async only").
  - On WASM, "`save_file` returns immediately without a dialog prompt. Instead the user is prompted by their browser on where to save the file when `FileHandle::write` is used". WASM `save_file` arrived in 0.12.0.
  - [crates.io rfd](https://crates.io/crates/rfd), [rfd AsyncFileDialog docs](https://docs.rs/rfd/latest/rfd/struct.AsyncFileDialog.html) (via search snippet), [rfd repo](https://github.com/PolyMeilex/rfd)
- **egui/eframe:** file DnD has been supported since 0.14.0, and on web the `web_sys::File` handle is stored since 0.36.0. Multi-viewport arrived in 0.24.0. Safari clipboard fixes landed in 0.24.x. — [eframe CHANGELOG](https://github.com/emilk/egui/blob/main/crates/eframe/CHANGELOG.md)
- **iced:** file DnD relies on winit's `DroppedFile` (issue #158). A multi-window helper crate exists. — [iced issue #158](https://github.com/iced-rs/iced/issues/158), [crates.io iced-multi-window](https://crates.io/crates/iced-multi-window)
- **Bevy on WASM:** file drag-and-drop has a known issue. — [Bevy issue #6822](https://github.com/bevyengine/bevy/issues/6822)
- **WASM DnD generally:** when compiled for WASM, "browsers often take control of drag and drop behavior, and drag and drop events may not be triggered in the application" (via search snippet of related issues). — [Uno WASM DnD issue](https://github.com/unoplatform/uno/issues/13559)
- **winit:** `winit` 0.31.0-beta.3 was released 2026-09-04, with 0.31 still in beta. egui, iced and Bevy all sit on winit for native windowing. — [crates.io winit](https://crates.io/crates/winit)
- **Slint web:** it renders to canvas with no DOM, and accessibility is unavailable. — [Slint Web docs](https://docs.slint.dev/latest/docs/slint/guide/platforms/web/) (via search snippet)
- **Dioxus desktop:** browser APIs are not directly available from Rust, so canvas and WebGL need JS interop. — [Dioxus desktop guide](https://dioxuslabs.com/learn/0.7/guides/platforms/desktop/) (via search snippet)

### Inferences
- For a DAW with a browser build, plan for these web-specific adaptations:
  - replace multi-window mixers/editors with docked panels;
  - replace native open/save dialogs with rfd async or `<input type=file>` plus download;
  - feed audio-file import through DnD of `web_sys::File` bytes, never paths;
  - avoid shortcuts the browser reserves (Ctrl+W, Ctrl+T, Ctrl+N).
- High-DPI is handled by winit on native and by `devicePixelRatio` on web in egui/iced. Framework-specific confirmation was not gathered.

### Gaps
- Per-framework verification of keyboard-shortcut handling and high-DPI behaviour on web was not done.
- Whether iced's wasm target forwards file drops, and whether iced's clipboard works in the browser, remains unverified.

## Rendering and hit-testing thousands of cells (16×64×patterns) and real-time meters: performance pitfalls

### Takeaway
A 16×64 step grid (1,024 cells) is trivial for any GPU-backed toolkit if drawn as one batched custom-painted region and hit-tested arithmetically. The real costs are:
- immediate-mode full layout and tessellation every frame in egui when meters force continuous repaint;
- per-widget overhead if each cell is a widget or DOM node;
- large point sets (waveforms, spectra, automation curves), which should be pre-decimated or GPU-rendered.

Retained toolkits (iced Canvas `Cache`, Slint/wgpu textures) can cache static layers and redraw only the overlay (playhead, meters, cursors).

### Cited Findings
- **egui:** 1-2ms/frame typical. Tessellation dominates for huge shape counts: 500k plot points gave about 50% CPU. Recommended fixes are a wgpu `PaintCallback`, pre-reserved buffers and downsampling. — [egui README](https://github.com/emilk/egui), [egui discussion #3810](https://github.com/emilk/egui/discussions/3810)
- **egui:** "Since an immediate mode GUI does a full layout each frame ... if you have a very complex GUI this can tax the CPU". — [egui optimization tracking #1196](https://github.com/emilk/egui/issues/1196) (via search snippet)
- **iced:** `Cache` avoids regenerating geometry unless it is cleared or resized. 0.14 added reactive rendering and primitive culling in rows/columns. A cached-canvas flicker regression with large images is open (#3173). — [iced Cache docs](https://docs.rs/iced/latest/iced/widget/canvas/type.Cache.html) (via search snippet), [iced 0.14.0 release](https://github.com/iced-rs/iced/releases/tag/0.14.0), [iced issue #3173](https://github.com/iced-rs/iced/issues/3173)
- **DAW-specific costs:** waveforms need peak searching plus pixel-by-pixel rendering, automation Béziers are slow, and piano rolls contain many rectangles. — [BillyDM, DAW Frontend Development Struggles](https://billydm.github.io/blog/daw-frontend-development-struggles/) (via search snippet)
- **Webview bridges:** Dioxus desktop `eval` freezes above about 100k elements (debug) or 1M (release). Tauri events are JSON and "not designed for low latency or high throughput"; channels are "designed to be fast". — [Dioxus issue #1915](https://github.com/DioxusLabs/dioxus/issues/1915), [Tauri calling-frontend docs](https://github.com/tauri-apps/tauri-docs/blob/v2/src/content/docs/develop/calling-frontend.mdx)
- **Browser metering:** SharedArrayBuffer plus Atomics avoids MessagePort serialization but requires COOP/COEP. — [Chrome Labs SAB sample](https://googlechromelabs.github.io/web-audio-samples/audio-worklet/design-pattern/shared-buffer/), [Chrome audio worklet design pattern](https://developer.chrome.com/blog/audio-worklet-design-pattern)

### Inferences
- **Grid and piano roll:** draw the grid as one custom widget. Compute `cell = floor((pos - origin) / cell_size)` for hover, click and drag-paint instead of creating 1,024+ interactive widgets. Emit a single mesh (egui `Mesh`/`Painter::rect_filled` batch, iced `Canvas` frame). For the piano roll, cull notes outside the viewport before building geometry.
- **Waveforms:** store multi-resolution min/max peak pyramids per clip (computed off the UI thread). Draw one vertical line or quad per pixel column. For large or zoomed-out views, upload the peaks to a GPU texture or buffer through wgpu (egui `PaintCallback`, iced `shader`, Slint wgpu texture).
- **Meters and spectrum:** the audio thread writes peak/RMS and FFT bins into lock-free triple buffers or atomics (native) or a SharedArrayBuffer (web). The UI reads them once per frame. In egui this means continuous `request_repaint()`, which makes the whole UI re-layout at 60fps. Keep the rest of the UI cheap, or throttle when idle.
- **Presence cursors:** draw them as a top overlay layer (egui foreground layer, iced uncached Canvas layer, DOM absolutely-positioned elements) driven by the collaboration layer's awareness state. They cost a handful of shapes per frame in every framework.

### Gaps
- No published head-to-head 60fps benchmark of egui vs iced vs Slint vs Makepad for grid/meter workloads was found.
- The iced forum thread on "way too many rectangles" was unreachable.

## Overall comparison matrix and recommendation (synthesis for the report writer)

### Takeaway
For "native desktop plus browser from one Rust codebase" with dense custom 60fps widgets:
1. **egui/eframe** is the pragmatic first choice. It is mature and active (0.36.2, Sept 2026), has first-party web (WebGPU with WebGL fallback), file DnD on both targets and the richest audio-widget and plugin ecosystem, under MIT/Apache.
2. **iced** (0.14, Dec 2025) is the retained/Elm alternative. It has revived audio widgets (iced_audio 0.17.1) but no accessibility and a slower release cadence.
3. **Web-tech plus Tauri** (Leptos or Dioxus) is best for browser fidelity and accessibility, but needs an IPC design for meters and waveforms on desktop.

Slint (web "not recommended", no web a11y, attribution license), vizia (web partial), Makepad (stale crates, bespoke tooling), gpui (web experimental), Freya (no web) and Xilem (experimental) are weaker fits.

### Cited Findings
- All versions, dates, licenses and web-support facts in the matrix below are cited in the sections above. Key anchors: [crates.io egui](https://crates.io/crates/egui), [eframe CHANGELOG](https://github.com/emilk/egui/blob/main/crates/eframe/CHANGELOG.md), [crates.io iced](https://crates.io/crates/iced), [iced 0.14.0 release](https://github.com/iced-rs/iced/releases/tag/0.14.0), [crates.io iced_audio](https://crates.io/crates/iced_audio), [crates.io vizia](https://crates.io/crates/vizia), [crates.io slint](https://crates.io/crates/slint), [Slint Web docs](https://docs.slint.dev/latest/docs/slint/guide/platforms/web/), [crates.io dioxus](https://crates.io/crates/dioxus), [crates.io tauri](https://crates.io/crates/tauri), [Makepad README](https://github.com/makepad/makepad), [Zed PR #50228](https://github.com/zed-industries/zed/pull/50228)
- nih-plug, the dominant Rust plugin framework, "is currently in maintenance mode". Users are pointed to the community fork nice-plug on Codeberg RustAudio: nice-plug 0.4.2 (2026-09-14), ISC license, with egui and iced adapters published. — [nih-plug README](https://github.com/robbert-vdh/nih-plug), [crates.io nice-plug](https://crates.io/crates/nice-plug), [nice-plug search snippet](https://codeberg.org/RustAudio/nice-plug)

### Inferences
Comparison matrix. "a11y" means accessibility. Dates are latest stable releases on crates.io as of 2026-09-26.

| Framework | Latest stable (date) | License | Native + web, one codebase | Dense custom drawing at 60fps | Model | Audio widgets | a11y | Audio track record | Maintenance |
|---|---|---|---|---|---|---|---|---|---|
| egui/eframe | 0.36.2 (2026-09-08) | MIT/Apache-2.0 | Yes, first-party; wgpu, WebGPU with WebGL fallback | Good via Painter/Mesh; wgpu `PaintCallback` for heavy plots; watch CPU with continuous repaint | Immediate | egui_knob, armas-audio, egui-cha-ds, egui_fader, egui_plot, egui-snarl, egui_dnd | AccessKit native; experimental on web | nih-plug/nice-plug adapter, egui-baseview; hobby DAWs | Very active; frequent breaking 0.x |
| iced | 0.14.0 (2025-12-07) | MIT | Yes; wgpu with WebGPU default (WebGL fallback) | Good with Canvas `Cache` plus `shader` widget | Retained, Elm-style | iced_audio 0.17.1 (knob, sliders, XY pad, mod range; no meters) | None upstream (fork plushie-iced adds it) | nice-plug-iced, iced_baseview | Active; roughly yearly releases |
| vizia | 0.4.0 (2026-04-23) | MIT | Partial (some examples run on wasm) | Custom views; not assessed | Retained, CSS-styled | Plugin-oriented | AccessKit | nih-plug adapter | Active but small |
| Slint | 1.18.1 (2026-09-21) | GPLv3 / royalty-free with attribution / commercial | WebGL canvas; "not recommended" for general web | `Path` element or wgpu texture interop (1.12+) | Retained DSL | None found | Native yes; web none | None found | Very active, commercial backing |
| Dioxus (+ Blitz) | 0.7.10 (2026-07-30) | MIT/Apache-2.0 | Web = DOM; desktop = webview (native Blitz experimental) | Via canvas/WebGL in JS; `eval` bottleneck on desktop | React-like | Web/JS ecosystem | Good (DOM) | None found | Very active |
| Leptos/Yew + Tauri | Leptos 0.8.21 / Tauri 2.12.0 (2026-09-26) | MIT or MIT/Apache | Web = DOM; desktop = Tauri webview | Canvas/WebGL; IPC via Channels/binary | Fine-grained reactive | Web/JS ecosystem | Good (DOM) | None found | Very active |
| Makepad | 1.0.0 (2025-05-13) | MIT/Apache-2.0 | Yes (WebGL) | Excellent (shader-based) | Retained, live DSL | Ironfish synth demo | Unknown | Ironfish synth | Repo active, crates stale |
| gpui | 0.2.2 (2025-10-22) | Apache-2.0 | Web experimental (Feb 2026) | Excellent (Zed) | Hybrid | gpui-component (general) | Unknown | None | Active in the Zed repo |
| Freya | 0.4.3 (2026-08-30) | MIT | No web | Skia | Reactive | None | Unknown | None | Active, small |
| Xilem/Masonry | 0.4.0 (2025-10-29) | Apache-2.0 | Web via separate DOM backend | Vello | Reactive | None | AccessKit (Masonry; not verified here) | None | Experimental |

- Recommended pattern (inference): egui/eframe for the whole DAW UI on both targets, with an audio engine crate shared by native (cpal) and web (WASM in AudioWorklet), and meters/scopes fed through lock-free buffers or a SharedArrayBuffer.
- Structure the dense surfaces (step grid, piano roll, arrangement, waveforms) as custom-painted widgets with arithmetic hit-testing, and upgrade to wgpu `PaintCallback` if profiling shows tessellation hotspots.
- Keep the UI state model framework-agnostic (a command/CRDT layer), so a later move to iced, or to a Tauri/Leptos UI, stays possible.

### Gaps
- No independent 2026 benchmark comparing these frameworks on DAW-like workloads was found.
- No 2026 survey article was retrievable in full: boringcactus and Wren Learns Rust were blocked, and only search snippets were used.
- areweguiyet.com was not consulted directly.
