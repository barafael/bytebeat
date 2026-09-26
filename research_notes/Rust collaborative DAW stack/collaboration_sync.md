# Real-time collaboration for a Rust mini-DAW (native + WASM): CRDTs, sync, networking, playback sync

Research date: 2026-09-26. Version numbers and release dates come from the crates.io and npm registry APIs, queried live on that date, unless another source is given. Several primary sites (iroh.computer, automerge.org, loro.dev, figma.com, webrtc.rs, webkit.org, docs.rs) were blocked by the research environment's egress proxy. Where possible I read the same content from the projects' GitHub sources (for example the loro-docs repo, the automerge.github.io repo and the iroh CHANGELOG). Where I could not, I relied on search-engine snippets, and I mark those "(search snippet)".

---

## 1. CRDT libraries: status, WASM support, data-model fit, undo, presence, size, performance, typed wrappers, license

### Takeaway
**Loro (1.16.x, MIT)** fits a DAW data model best out of the box. It has a Map with last-writer-wins (LWW) semantics for knob values, a native **MovableList** for clips, tracks and device chains, a Counter, a MovableTree, a local-only UndoManager, an EphemeralStore for presence, and time travel/checkout. Its costs are the largest WASM binary (about 0.9–0.97 MB gzipped) and larger per-update encodings. **Yrs (0.28, MIT)** is the right pick when you want interoperability with the JS Yjs ecosystem: y-sweet, Liveblocks Yjs hosting, y-websocket, Flok-style apps. It is mature, has an UndoManager filtered by origin, and has Awareness built in. It has no counter type, and its array-move is not in mainline JS Yjs. **Automerge (Rust crate 0.12 = JS 3.5, MIT)** has the best typed Rust wrapper (autosurgeon) and an automerge-repo-compatible Rust repo layer (samod 0.14). It has had no built-in undo since 1.0 and no move-list. diamond-types, cola and the `crdts` crate are niche primitives (text only, or low-level) and not suitable as the song store.

### Cited Findings

**Versions and dates (registry data, 2026-09-26)**
- `loro` 1.16.2 released 2026-09-21 (1.16.0 on 2026-09-06, 1.13.0 on 2026-06-05, 1.10.0 on 2025-11-27), MIT license, about 440k recent downloads — [crates.io/crates/loro](https://crates.io/crates/loro). JS package `loro-crdt` latest 1.16.3 (2026-09-21) — [npm loro-crdt](https://www.npmjs.com/package/loro-crdt).
- The Loro 1.0 announcement is dated 2024-10-23 and promises "a stable encoding schema, 10–100x faster document import, advanced version control". Loro is usable "in Rust, JS (via WASM), and Swift" — [Loro 1.0 blog (source in loro-docs repo)](https://github.com/loro-dev/loro-docs/blob/main/pages/blog/v1.0.mdx). (Oddity: npm's registry records a `loro-crdt@1.0.0` publish timestamp of 2023-03-23, which conflicts with the 2024 announcement. It may be an early placeholder publish, so treat the blog date as authoritative.)
- `yrs` 0.28.0 released 2026-09-17 (0.27.0 on 2026-06-03, 0.26.0 on 2026-05-04), MIT license, about 1.7M recent downloads — [crates.io/crates/yrs](https://crates.io/crates/yrs). JS `yjs` latest stable is 13.6.33 (2026-09-23). A **Yjs v14 is in beta** (`beta: 14.0.0-16`, `next: 14.0.0-8` dist-tags) — [npm yjs](https://www.npmjs.com/package/yjs).
- `automerge` (Rust) 0.12.0 released 2026-09-16. The cadence has been roughly monthly: 0.8.0 on 2026-03-25, 0.9.0 on 2026-04-22, 0.10.0 on 2026-06-05, 0.11.0 on 2026-08-12. Some `1.0.0-alpha/beta` versions published in April–June 2025 were **yanked**, so the Rust crate stays on 0.x while JS is on 3.x — [crates.io/crates/automerge](https://crates.io/crates/automerge). JS `@automerge/automerge` latest is 3.5.0 (2026-09-16). 3.0.0 was published 2025-07-14 — [npm @automerge/automerge](https://www.npmjs.com/package/@automerge/automerge).
- `autosurgeon` 0.14.0 (2026-09-17), `samod` 0.14.0 (2026-09-17), `samod-core` 0.14.0 — [crates.io/crates/autosurgeon](https://crates.io/crates/autosurgeon), [crates.io/crates/samod](https://crates.io/crates/samod).
- `diamond-types` 1.0.0 was last published 2022-08-25 (ISC license). The README warns: "the package published to cargo is quite out of date, both in terms of API and performance", and "This version of diamond types only supports plain text editing. Work is underway to add support for other JSON-style data types" — [diamond-types README](https://github.com/josephg/diamond-types).
- `cola` 0.5.1 (2025-07-06) is "a CRDT specialized for real-time collaborative editing of plain text documents" — [cola README](https://github.com/nomad/cola), [crates.io/crates/cola](https://crates.io/crates/cola).
- `crdts` 7.3.2 was last published 2023-08-08 (Apache-2.0). It is "a family of CRDT's supporting both State and Op based replication" (MVReg, Map, and others) and needs manual causal-context handling — [rust-crdt README](https://github.com/rust-crdt/rust-crdt), [crates.io/crates/crdts](https://crates.io/crates/crdts).
- `y-octo` 0.1.1 (2026-09-22) is from toeverything/AFFiNE: a "High-performance and thread-safe CRDT implementation compatible with Yjs". It is an alternative Rust Yjs implementation, at 0.1 so immature — [crates.io/crates/y-octo](https://crates.io/crates/y-octo).

**Data types relevant to a DAW**
- Loro containers: Text (Fugue), Rich Text, **Movable Tree**, **Movable List**, **Last-Write-Wins Map**, plus "Mergeable map-key children via `ensure_mergeable_*`", time travel, "Version Control with Real-Time Collaboration", and shallow snapshots ("like Git shallow clone") — [Loro README](https://github.com/loro-dev/loro).
- Loro `MovableList` "supports additional Set and Move operations". Simulating move with delete+insert "fails in concurrent editing scenarios" because it duplicates elements. MovableList is "approximately 80% slower in encode/decode and consumes about 50% more memory compared to the List" for insert/delete-only workloads. It uses Fugue plus Kleppmann's "Moving Elements in List CRDTs" algorithm — [Loro list docs](https://github.com/loro-dev/loro-docs/blob/main/pages/docs/tutorial/list.mdx).
- Loro Counter: "Counters are special CRDTs that sum concurrent increments". There is also `LoroMap.ensureMergeableCounter(key)` — [Loro llms-full docs](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).
- Loro **Mergeable Containers** (blog dated 2026-06-09) fix a classic JSON-CRDT problem. When two peers concurrently create a child container under the same map key, only one stays visible after merge, which "looks like data loss". The blog says the bug is reproducible "in all three" of Loro, Yjs and Automerge. The fix is `ensureMergeableList/Map/Text/MovableList(key)`, which derives the child's identity from its parent, key and type — [Loro blog: Mergeable Containers](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt) (file `pages/blog/mergeable-containers.mdx`).
- Loro's own comparison table: Movable Tree ✅ Loro / ❌ Yjs / ❌ Automerge ("Inventor"); Movable List ✅ Loro / ❌ Yjs / ❌ Automerge ("Inventor"); Version Control ✅ Loro, ✅ Automerge, ❌ Yjs; Byzantine-fault-tolerance ✅ only Automerge. Yjs time travel needs the user to store a version vector plus a delete set — [Loro docs comparison table (llms-full)](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).
- Yrs feature parity table: YMap, YArray, YText, XML, sub-documents, sticky indexes, **Undo Manager ✅**, **Awareness ✅**, snapshots ✅, and **"YArray: move" ✅ in yrs, but in JS Yjs only on a separate "move branch"**. Network providers: yrs-warp (WebSocket) and yrs-webrtc — [y-crdt README](https://github.com/y-crdt/y-crdt).
- Yrs `Awareness` is in `yrs::sync::awareness`: "a simple shared state protocol that can be used for non-persistent data like awareness information (cursor, username, status, ..)" — [yrs awareness.rs](https://github.com/y-crdt/y-crdt/blob/main/yrs/src/sync/awareness.rs). The separate `y-sync` crate (0.4.0, last published 2023-11-17) implemented the Yjs sync and awareness protocols — [y-sync README](https://github.com/y-crdt/y-sync), [crates.io/crates/y-sync](https://crates.io/crates/y-sync).
- Automerge supports maps, lists, text (collaborative strings by default in 3.0, `ImmutableString` for non-merging strings) and counters. Its changelog references counter increments and "conflicted values" exposed to the API — [Automerge 3.0 blog (source)](https://github.com/automerge/automerge.github.io/blob/main/content/blog/automerge-3.md), [automerge Rust CHANGELOG](https://github.com/automerge/automerge/blob/main/rust/CHANGELOG.md).
- Automerge 0.12.0 (Rust) / 3.5.0 (JS) added **Author IDs**, which are "opaque bytes" that identify authors independently of actor IDs and are stored in change metadata — [automerge Rust CHANGELOG](https://github.com/automerge/automerge/blob/main/rust/CHANGELOG.md), [automerge JS CHANGELOG](https://github.com/automerge/automerge/blob/main/javascript/CHANGELOG.md).

**Undo/redo**
- Loro `UndoManager` is **local**: it "only undoes the local user's operations, not remote operations". Options include `maxUndoSteps` (default 100), `mergeInterval` (default 1000 ms) and `excludeOriginPrefixes`, and it transforms cursors. Limitation: "It can only track a single peer. When the peer ID of the document changes, it will clear the undo stack" — [Loro Undo docs (llms-full)](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt), [docs.rs UndoManager](https://docs.rs/loro/latest/loro/struct.UndoManager.html).
- Yrs `UndoManager` tracks changes by **transaction origin** (`tracked_origins`, `include_origin`, `exclude_origin`), which gives per-user undo when each user's local transactions carry their own origin — [yrs undo.rs](https://github.com/y-crdt/y-crdt/blob/main/yrs/src/undo.rs).
- Automerge: "undo/redo were cut for 1.0" (issue #985). A community wrapper, `automerge-repo-undo-redo` (onsetsoftware), is "experimental" and "uses [official APIs] to alter the history of your document, which may have unexpected results" — [automerge issue #985](https://github.com/automerge/automerge/issues/985), [automerge-repo-undo-redo](https://github.com/onsetsoftware/automerge-repo-undo-redo/). Academic work exists: "Extending Automerge: Undo, Redo, and Move" (PLF 2023) — [SPLASH 2023](https://2023.splashcon.org/details/plf-2023-papers/2/Extending-Automerge-Undo-Redo-and-Move).

**Presence/awareness**
- Loro `EphemeralStore` is "a timestamp-based, last-write-wins key-value store" that "doesn't persist in the CRDT Document". It sends only updated entries, has a default timeout of 30000 ms, and emits events for local, remote and timeout changes — [Loro Ephemeral Store docs](https://github.com/loro-dev/loro-docs/blob/main/pages/docs/tutorial/ephemeral.mdx).
- Yrs has Awareness (see above). samod exposes `DocHandle::ephemeral` streams (automerge-repo "ephemeral messages") — [samod CHANGELOG](https://github.com/alexjg/samod/blob/main/CHANGELOG.md).

**Typed/schema wrappers for Rust**
- `autosurgeon`: `#[derive(Reconcile, Hydrate)]` on Rust structs; `reconcile()` writes a struct into an automerge doc as a minimal diff and `hydrate()` reads it back — [autosurgeon README](https://github.com/automerge/autosurgeon).
- `lorosurgeon` 0.2.1 (created 2026-03-10, about 8k downloads, third-party, rlch/lorosurgeon): "Derive macros for bidirectional serialization between Rust types and Loro CRDT containers". It is **immature** — [crates.io/crates/lorosurgeon](https://crates.io/crates/lorosurgeon).
- Loro Mirror (JS/React, 2025-09-22) keeps "a typed, immutable app-state view in sync with a Loro CRDT document", turning `setState` diffs into container-level ops — [Loro Mirror blog (llms-full)](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).
- `yrs-kvstore` 0.3.0 (2024-07-09) is a "Generic persistence layer over Yrs documents" — [crates.io/crates/yrs-kvstore](https://crates.io/crates/yrs-kvstore).

**WASM binary size (JS bundles built from the Rust cores)**
- In Loro's copy of the crdt-benchmarks table: yjs 13.6.15 is 84,017 B (25,105 B gzipped); **ywasm 0.17.4 is 938,991 B (284,616 B gzipped)**; **loro 1.0.0-beta.2 is 2,919,363 B (894,460 B gzipped)**; loro-old 0.15.2 is 592,039 B gzipped; **automerge 2.1.10 is 1,696,176 B (591,049 B gzipped)** — [Loro performance docs (llms-full)](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).
- Loro's own docs say to consider alternatives if "Your application is sensitive to bundle size (Loro WASM binary ~970KB gzipped)" — [Loro docs (llms-full)](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).
- The older dmonad table has yjs 13.6.11 at 20,100 B gzipped, ywasm 0.9.3 at 213,833 B gzipped, loro 0.10.1 at 399,276 B gzipped and automerge 2.1.10 at 604,118 B gzipped — [dmonad/crdt-benchmarks](https://github.com/dmonad/crdt-benchmarks).

**Performance and encoding size (crdt-benchmarks, N=6000; versions as listed, not the 2026 releases)**
- B4, the real-world text trace (259,778 ops). Time: yjs 2,616 ms, ywasm 17,556 ms, loro 2,271 ms, loro-old 768 ms, automerge 7,109 ms, automerge-wasm 2,775 ms. docSize: yjs 226,981 B, loro 230,556 B, automerge 129,116 B. parseTime: loro 6 ms, yjs 27 ms, automerge 1,185 ms — [Loro performance docs (llms-full)](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).
- B3.1, where 20√N clients concurrently set a number in a Map (closest to "many users twisting knobs"). Time: yjs 54 ms, loro 27 ms, automerge 1,058 ms (automerge-wasm 21 ms). updateSize: yjs 49,167 B, loro 132,376 B, automerge 283,296 B — [same source](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).
- Per-update overhead (B1.1, appending characters): avgUpdateSize is 27 B for yjs/ywasm, 88 B for loro 1.0-beta and 121 B for automerge — [same source](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).
- Automerge 3.0 (2025-07-14) "cut memory usage by over 10x". Pasting Moby Dick used 700 MB in Automerge 2 and 1.3 MB in Automerge 3. A document that "hadn't loaded after 17 hours" now loads "in 9 seconds". The file format is unchanged from Automerge 2 — [Automerge 3.0 blog (source)](https://github.com/automerge/automerge.github.io/blob/main/content/blog/automerge-3.md).
- Loro 1.0 claims a 10–100x import-speed improvement, trading about 2x snapshot size for about 10x import speed, and has an internal LSM-like block store of roughly 4 KB blocks — [Loro 1.0 blog](https://github.com/loro-dev/loro-docs/blob/main/pages/blog/v1.0.mdx).

**Persistence**
- Loro recommends periodic snapshots plus frequent delta "updates" exports stored in fast KV storage. It validates checksums on import, so "corrupted binary payloads… are rejected" — [Loro persistence docs](https://github.com/loro-dev/loro-docs/blob/main/pages/docs/tutorial/persistence.mdx).
- Loro sync: "Two documents with concurrent edits can be synchronized by just two message exchanges" using `export({mode:"update", from: versionVector})` — [Loro sync docs](https://github.com/loro-dev/loro-docs/blob/main/pages/docs/tutorial/sync.mdx).

**Maturity notes**
- The Automerge README says the Rust API "is low level and not well documented… you may want to look into autosurgeon" — [Automerge README](https://github.com/automerge/automerge).
- samod README: "experimental… very much a work in progress… don't use this anywhere serious yet". It is still released monthly (0.8 in March 2026 through 0.14 in September 2026) — [samod README](https://github.com/alexjg/samod), [samod CHANGELOG](https://github.com/alexjg/samod/blob/main/CHANGELOG.md).
- The older `automerge_repo` crate (0.3.0, 2025-10-03) is "not compatible with the JavaScript automerge-repo" on disk or over the wire — [lib.rs automerge_repo](https://lib.rs/crates/automerge_repo) (search snippet), [crates.io](https://crates.io/crates/automerge_repo).

### Inferences
- **Mapping a DAW song onto Loro:** `song` (Map) → `tracks` (MovableList of mergeable Maps) → each track has `devices` (MovableList), `params` (Map of knob→f64, which is LWW per key), `patterns` (Map id→Map with a `steps` List, or a Map keyed by step index), and `arrangement.clips` (MovableList, or a Map id→{start, len, pattern_id} where time position is a plain LWW field). Counters are only for truly additive quantities; "likes" or play-counts fit, but knob values should be LWW, not counters. Presence (cursor, selected track, playhead) goes in the EphemeralStore.
- **Arrangement clips:** for timeline clips, position is usually a numeric `start_beat` field rather than a list index, so clips can be a Map keyed by clip UUID with LWW fields. MovableList is most valuable for *ordered* collections the user reorders: tracks, mixer channel order, device/effect chains, pattern order in a song-mode chain. The same Map-by-UUID with LWW fields trick lets Yrs/Automerge approximate a DAW without a native move op. You can also use fractional-index ordering (Loro ships `loro_fractional_index`).
- **Step grids:** a Map keyed by `"step:pitch"` holding a bool/velocity (LWW) merges concurrent toggles cleanly and avoids list-index conflicts.
- For a Rust-first codebase where both native and WASM builds share one core, Loro and Yrs compile into your own wasm32 binary. You don't pay for the JS package's bundle, but you still pay for the CRDT code (hundreds of KB gzipped). Automerge's Rust API is lower-level (autosurgeon helps). Loro's typed wrapper (lorosurgeon) is young, so plan to write a thin hand-rolled schema layer.
- Automerge's lack of built-in undo is a real cost for a DAW, where undo is essential. Loro and Yrs both offer local, per-user undo, which is the behavior users expect in multiplayer.

### Gaps
- No 2026-dated head-to-head benchmark of loro 1.16 vs yrs 0.28 vs automerge 0.12 (Automerge 3) on map-heavy, non-text workloads. The published tables use 2024-era versions (loro 1.0-beta, automerge 2.1.10).
- I found no measured WASM size for a Rust app that statically links `loro` or `yrs` (as opposed to the JS bundles).
- I could not confirm whether `samod` builds for `wasm32-unknown-unknown` (its features are tokio/axum/tungstenite/gio, with no browser transport listed).
- The status of Yjs v14 and its binary compatibility with yrs 0.28 is unverified. v14 is still in beta on npm.

---

## 2. CRDT vs. a simpler authoritative server (Figma-style); what collaborative music tools actually do

### Takeaway
Figma deliberately uses a **server-authoritative, per-property last-writer-wins** model inspired by CRDTs but not a true CRDT. For a DAW whose state is mostly "objects with properties" (knob values, clip positions), that model is sufficient and simpler. A real CRDT (Loro/Yrs) mainly buys you offline editing, P2P, history/time travel and ready-made undo and presence. Commercial music tools (Audiotool, BandLab) are cloud/server-centred. Hobby and research tools (Flok, sequencer.party/WAM Jam Party) use Yjs, often P2P over WebRTC.

### Cited Findings
- Figma: "multiplayer servers keep track of the latest value that any client has sent for a given property on a given object… similar to a last-writer-wins register in CRDT literature except the server can define the order of events". Unrelated properties on the same object don't conflict. Figma's structure "isn't a single CRDT" but is "inspired by multiple separate CRDTs". The server defining the order removes the need for vector clocks and tombstone garbage collection — [Figma blog: How Figma's multiplayer technology works](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/) (search snippet).
- Plane (Jamsocket) is "heavily inspired by Figma's multiplayer infrastructure, which dynamically spawns a process for each active document" — [Plane README](https://github.com/jamsocket/plane).
- Loro's own guidance: "CRDTs merge; they do not reject". They are a poor fit for "Hard invariants", "Exclusive ownership", "Authorization decisions that must be enforced at write time". Suggested hybrid: "CRDT for UI drafts, authoritative booking via server" — [Loro docs: When Not to Use CRDTs (llms-full)](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).
- Audiotool 3.0 is "a ground-up rebuild of its browser-based DAW that brings real-time multiplayer music creation" — [ProSoundWeb](https://www.prosoundweb.com/new-audiotool-3-0-multiplayer-digital-audio-workstation-now-available/). Its **NEXUS** SDK connects "to a multiplayer session on beta.audiotool.com and modify projects in real time". NEXUS "is built on Protocol Buffers, and defines every entity and action in the DAW" — (search snippets from [npm @audiotool/nexus](https://www.npmjs.com/package/@audiotool/nexus) and [gearnews](https://www.gearnews.com/audiotool-studio-nexus-daw-tech/)). The `@audiotool/nexus` package (0.0.19, 2026-09-26, "pre 1.0", MIT) depends on `@bufbuild/protobuf` and `@connectrpc/connect(-web)` — [npm registry @audiotool/nexus](https://www.npmjs.com/package/@audiotool/nexus), [audiotool/nexus repo](https://github.com/audiotool/nexus).
- BandLab: "Live Session on the web Mix Editor allows you to work in real-time with your collaborators", with cloud sync and automatic version snapshots. Secondary sources note "simultaneous real-time editing has limitations", with much collaboration being async (forking) — [BandLab blog: Forking and collaboration](https://blog.bandlab.com/forking-and-collaboration-on-bandlab-explained/), [Audeobox BandLab guide](https://www.audeobox.com/learn/bandlab/bandlab-collaboration-features/) (secondary).
- Flok ("Web-based P2P collaborative editor for live coding music and graphics", GPL-3+, now developed on Codeberg) depends on `yjs ^13.6.21`, `y-webrtc`, `y-websocket`, `y-indexeddb` and `y-codemirror.next` — [Flok README](https://github.com/munshkr/flok), [Flok packages/web/package.json](https://github.com/munshkr/flok/blob/main/packages/web/package.json). It has shipped Strudel support since early on — [Strudel blog](https://strudel.cc/blog/) (search snippet). Its changelog shows 1.3.0 on 2025-01-05 — [Flok CHANGELOG](https://github.com/munshkr/flok/blob/main/CHANGELOG.md).
- "WAM Jam Party uses a P2P design with Yjs". The sequencer.party designer wrote `y-pojo` to sync Yjs docs to and from plain objects — [INRIA HAL hal-04695228](https://inria.hal.science/hal-04695228/document) (search snippet; full text blocked).

### Inferences
- Commercial DAWs (Audiotool, BandLab) behave like **server-authoritative session backends**. Audiotool's protobuf/ConnectRPC SDK suggests a server-mediated entity/action model rather than a client-side CRDT, but I found no public statement of their conflict-resolution algorithm.
- For a *miniature* DAW, the conflict profile is benign. Most concurrent edits touch different objects or properties, and same-property conflicts (two people twisting one knob) are fine as LWW. The hard parts are ordered collections (track/device reorder), concurrent creation of nested objects, undo, and presence. Loro solves those directly. A Figma-style server would need you to implement fractional indexing, undo and presence yourself.
- **Recommended hybrid:** a CRDT document (Loro) for the song state, plus a thin authoritative server for identity, permissions, share codes, asset upload and the transport/clock master. The server also acts as always-on peer and persistence (see section 4). This keeps the offline and P2P options open without making the server the conflict resolver.

### Gaps
- There are no public engineering write-ups on Audiotool's, Soundtrap's or BandLab's actual sync algorithms (OT, CRDT or LWW), so the characterization above is inferred.
- I could not read the WAM Jam Party paper's full text (blocked), so its transport/clock-sync details are unknown.

---

## 3. Networking/transport: which work on native AND wasm32 with the same code; NAT traversal and relays

### Takeaway
**WebSockets** are the only transport that is trivially identical on native and in the browser. On native, use tokio-tungstenite 0.30 or an axum 0.8 server; in the browser, use gloo-net 0.7 or ws_stream_wasm 0.7. They need a server but no NAT traversal. **iroh 1.x** (stable since 2026-06-15) gives native↔native QUIC with hole-punching and relay fallback, and compiles to wasm, but **browser nodes are relay-only**. **Matchbox** is the easiest Rust native+wasm WebRTC P2P (data channels) but needs a signaling server and TURN for hard NATs. **WebTransport** is now Baseline in all major browsers (Safari 26.4, March 2026), and `web-transport` provides one API over quinn (native) and the browser API (wasm). rust-libp2p 0.57 has websys transports but is heavy. webrtc-rs and str0m are native-only (in the browser you use the built-in RTCPeerConnection).

### Cited Findings

**WebSocket crates**
- `tokio-tungstenite` 0.30.0 (2026-07-11), `axum` 0.8.9 (2026-04-14), `gloo-net` 0.7.0 (2026-03-25), `ws_stream_wasm` 0.7.5 (2025-06-08, Unlicense) — crates.io registry data ([tokio-tungstenite](https://crates.io/crates/tokio-tungstenite), [axum](https://crates.io/crates/axum), [gloo-net](https://crates.io/crates/gloo-net), [ws_stream_wasm](https://crates.io/crates/ws_stream_wasm)).
- samod ships `axum` and `tungstenite` features for WebSocket sync with JS automerge-repo — [samod Cargo.toml](https://github.com/alexjg/samod/blob/main/samod/Cargo.toml).

**iroh (n0)**
- The iroh 1.0.0 release is dated **2026-06-15** (rc.0 on 2026-05-07, rc.1 on 2026-05-27). Since then: 1.0.1 (06-29), 1.0.2 (07-06), 1.0.3 (07-20), 1.1.0 (08-25) and **1.2.0 (2026-09-09)**. 1.0 added "Bearer token access control without an external service" to iroh-relay — [iroh CHANGELOG](https://github.com/n0-computer/iroh/blob/main/CHANGELOG.md).
- iroh is "an API for dialing by public key". It hole-punches when needed and falls back to "an open ecosystem of public relay servers". It is built on QUIC via **noq** (n0's Quinn fork), with streams, datagrams and stream priorities. Licensed MIT/Apache-2.0 — [iroh README](https://github.com/n0-computer/iroh).
- The 1.0 announcement mentions 65 versions leading to 1.0, public relays seeing "more than 200 million endpoints created in the last 30 days", and custom transports (BLE, LoRa, WiFi Aware, Tor) — [iroh blog: Iroh 1.0](https://www.iroh.computer/blog/v1) (search snippet); see also [TechTimes on iroh 1.0, 2026-06-16](https://www.techtimes.com/articles/318490/20260616/peer-peer-library-iroh-10-ships-dial-devices-key-not-ip-address.htm).
- In the browser, "All connections from browsers need to flow via a relay server because browsers don't support sending UDP packets". There is "no hole punching" and no local discovery or mDNS. Connections stay end-to-end encrypted through the relay. n0 notes that WebTransport with `serverCertificateHashes` or WebRTC might enable direct browser connections in the future — [iroh docs: WebAssembly and Browsers](https://docs.iroh.computer/languages/wasm-browser) (search snippet). iroh has compiled to `wasm32-unknown-unknown` since roughly v0.33 (early 2025) — [iroh CHANGELOG](https://github.com/n0-computer/iroh/blob/main/CHANGELOG.md), [iroh 0.33 blog](https://www.iroh.computer/blog/iroh-0-33-0-browsers-and-discovery-and-0-RTT-oh-my) (search snippet).
- iroh protocol crates are **still 0.x** and on a separate cadence: `iroh-gossip` 0.101.0, `iroh-docs` 0.101.0 and `iroh-blobs` 0.103.0, all released 2026-06-15 — crates.io ([iroh-gossip](https://crates.io/crates/iroh-gossip), [iroh-docs](https://crates.io/crates/iroh-docs), [iroh-blobs](https://crates.io/crates/iroh-blobs)).
- iroh-gossip uses HyParView/PlumTree epidemic broadcast trees over topics, with a sans-IO `proto` module — [iroh-gossip README](https://github.com/n0-computer/iroh-gossip). A gossip chat example runs "both in the browser and on the command line" — [iroh-examples](https://github.com/n0-computer/iroh-examples) (search snippet).

**WebRTC**
- `matchbox_socket` 0.14.0 (2026-02-13; previous releases 0.13 in 2025-10 and 0.12 in 2025-05): "Painless peer-to-peer WebRTC networking for rust's native and wasm applications". It offers "unreliable and reliable data channels, with configurable ordering". It needs a signaling server (`matchbox_server`/`matchbox_signaling`), after which "data will flow directly between peers" — [matchbox README](https://github.com/johanhelsing/matchbox). On native it depends on `webrtc = "0.17"` (the pre-rewrite webrtc-rs line), with `async-tungstenite` for signaling — [matchbox_socket Cargo.toml](https://github.com/johanhelsing/matchbox/blob/main/matchbox_socket/Cargo.toml). Its showcase includes "Lavagna – collaborative blackboard" — [matchbox README](https://github.com/johanhelsing/matchbox).
- `webrtc` (webrtc-rs) 0.21.0 was released 2026-09-19. The v0.20 line (announced 2026-07-31) is "a complete rewrite" as "a thin async layer on top of the … Sans-I/O rtc crate", runtime-agnostic with Tokio and smol backends. 0.21 is "the run-up to 1.0" and "the recommended choice today". It includes a data-channel back-pressure fix ("a slow consumer used to lose messages on a reliable channel") — [webrtc-rs README](https://github.com/webrtc-rs/webrtc), [webrtc.rs blog v0.20.0](https://webrtc.rs/blog/2026/07/31/announcing-webrtc-v0.20.0.html) (search snippet). The sans-IO core lists `wasm32-wasip2` (not browser) as a build target — [webrtc.rs blog](https://webrtc.rs/blog/2026/07/31/announcing-webrtc-v0.20.0.html) (search snippet).
- `str0m` 0.24.0 (2026-09-25) is "A Sans I/O WebRTC implementation in Rust", deliberately "not a standard RTCPeerConnection API". Its examples are browser↔server SFU — [str0m README](https://github.com/algesten/str0m), [crates.io/crates/str0m](https://crates.io/crates/str0m).

**libp2p**
- `libp2p` 0.57.0 was released 2026-09-11 (0.56.0 was 2025-06-27) — [crates.io/crates/libp2p](https://crates.io/crates/libp2p). WASM transports: `libp2p-webrtc-websys` (wraps the browser RTCPeerConnection, requires `Swarm::with_wasm_executor`), `libp2p-webtransport-websys` and `libp2p-websocket-websys` — [docs.rs libp2p-webrtc-websys](https://docs.rs/libp2p-webrtc-websys), [crates.io libp2p-webtransport-websys](https://crates.io/crates/libp2p-webtransport-websys), [docs.rs libp2p-websocket-websys](https://docs.rs/libp2p-websocket-websys/latest/libp2p_websocket_websys/).

**WebTransport**
- "Safari 26.4 ships WebTransport out of the box". Baseline across Chrome, Edge, Firefox and Safari was reached in March 2026 — [WebRTC.ventures, April 2026](https://webrtc.ventures/2026/04/webtransport-is-now-baseline-what-it-means-for-real-time-media/), [WebKit Safari 26.4 features](https://webkit.org/blog/17862/webkit-features-for-safari-26-4/) (search snippets).
- `web-transport` 0.12.0 (moq-dev, 2026-08-20) is a "Generic WebTransport API with native (web-transport-quinn) and WASM (web-transport-wasm) support". There is also a `web-transport-noq` backend (the noq Quinn fork, the same QUIC stack iroh uses) and `qmux` for WebTransport-over-WebSocket fallback — [web-transport README](https://github.com/moq-dev/web-transport), [crates.io/crates/web-transport](https://crates.io/crates/web-transport).
- `wtransport` 0.7.2 (2026-08-11) is a pure-Rust WebTransport server/client. The README says it is "not considered completely production-ready" — [wtransport README](https://github.com/BiagioFesta/wtransport).

### Inferences
- **Same code native+wasm:** (a) WebSocket, behind a small trait with two impls (tokio-tungstenite / gloo-net), or `ewebsock`-style wrappers; (b) matchbox_socket for WebRTC P2P; (c) `web-transport` for client↔server QUIC; (d) iroh, where the browser is relay-only. The CRDT sync payloads (Loro updates, a few bytes to KB) are transport-agnostic, so start with WebSocket and keep a `Transport` trait.
- **NAT traversal:** client→server WebSocket/WebTransport needs none. WebRTC (matchbox) needs STUN and, for symmetric NATs or corporate networks, TURN. Expect to run coturn or pay for TURN. iroh handles traversal on native but forces browser traffic through relays (n0's public relays or a self-hosted `iroh-relay`). If browsers are first-class clients, a relay or server is unavoidable anyway, which argues for a server-centric topology.
- matchbox's native side pins the old webrtc 0.17 line, while webrtc-rs has since moved to a rewritten 0.20/0.21 stack. That is a maintenance risk to watch.

### Gaps
- I found no published hole-punching success-rate statistics for iroh 1.x (the perf dashboard at perf.iroh.computer was not reachable).
- I could not verify whether iroh 1.x browser builds can use WebTransport or WebRTC for direct paths yet (the docs describe it as a future possibility).
- I found no benchmarks comparing WebSocket vs. WebTransport latency for small CRDT updates.

---

## 4. Sync servers/backends: y-sweet, automerge-repo sync server, Loro sync, Jamsocket/Plane, PartyKit, Liveblocks, ElectricSQL

### Takeaway
Rust-native, self-hostable options exist for each CRDT family. **y-sweet** (Yjs, Rust server, S3 persistence, client tokens) is the most complete but has had little activity since Jamsocket joined Modal (July 2025). **loro-protocol** has Rust WS client and server crates with optional SQLite snapshots and an E2EE mode. **samod** can act as a Rust automerge-repo sync server compatible with JS clients. **Plane** is an open-source Rust orchestrator for one process per document. PartyKit/PartyServer and Liveblocks are JS/Cloudflare-centred (Liveblocks' open-sourced server is AGPL and not production-self-hostable). For this project, a small axum server embedding the CRDT is the lowest-risk path.

### Cited Findings
- **y-sweet**: "an open-source document store and realtime sync backend, built on top of the Yjs CRDT library". It "Persists document data to S3-compatible storage", "Scales horizontally with a session backend model", "Deploys as a native Linux process", "Provides document-level access control via client tokens", and is "Written in Rust". MIT license — [y-sweet README](https://github.com/jamsocket/y-sweet). Last releases: crate `y-sweet` 0.9.1 and npm `@y-sweet/client` 0.9.1, both **2025-09-16** — [crates.io/crates/y-sweet](https://crates.io/crates/y-sweet), [npm @y-sweet/client](https://www.npmjs.com/package/@y-sweet/client).
- **Jamsocket → Modal** (announced 2025-07-10): the co-founders joined Modal, and existing Jamsocket and Y-Sweet services "will continue to operate as normal" — [Jamsocket blog](https://jamsocket.com/blog/jamsocket-is-joining-modal), [Modal blog](https://modal.com/blog/jamsocket-is-joining-modal), [HN thread](https://news.ycombinator.com/item?id=44522952) (search snippets). There is an HN thread titled "Anyone else seeing data disappear on y-sweet from Jamsocket / Modal" — [HN 45317376](https://news.ycombinator.com/item?id=45317376) (title only; content not verified).
- **Plane**: "a distributed system for running stateful WebSocket backends at scale", inspired by Figma's process-per-document model — [Plane README](https://github.com/jamsocket/plane).
- **Loro Protocol** (blog 2025-10-30): a "small, transport-agnostic syncing protocol" that multiplexes rooms over one connection. It has a 256 KiB max message size with fragmentation, and supports Loro documents, the Loro ephemeral store and Yjs via adaptors. Transports include "WebSocket or any integrity-preserving transport (e.g., WebRTC)". Rust crates: `rust/loro-protocol`, `rust/loro-websocket-client`, and `rust/loro-websocket-server` ("Minimal async WS server with optional SQLite snapshotting"). **E2EE (`%ELO`)**: "The server never decrypts", using AES-GCM — [loro-dev/protocol README](https://github.com/loro-dev/protocol). The `loro-protocol` crate is at 0.3.0 (2026-06-12) — [crates.io/crates/loro-protocol](https://crates.io/crates/loro-protocol).
- **automerge-repo-sync-server**: "A very simple automerge-repo synchronization server… an unsecured Express app", configured with PORT and DATA_DIR and available as a Docker image — [automerge-repo-sync-server README](https://github.com/automerge/automerge-repo-sync-server). automerge-repo has pluggable network adapters (websocket, MessageChannel, BroadcastChannel) and storage adapters (IndexedDB, nodefs) — [automerge-repo README](https://github.com/automerge/automerge-repo). `@automerge/automerge-repo` latest is 2.6.0-alpha.3 (2026-08-07) — [npm](https://www.npmjs.com/package/@automerge/automerge-repo).
- **samod**: "A Rust peer running samod can synchronize directly with a JavaScript peer running automerge-repo" — [samod (search snippet)](https://crates.io/crates/samod). Its README says it "aims to replace automerge-repo-rs" — [samod README](https://github.com/alexjg/samod).
- **yrs-warp** 0.9.0 (2025-07-21) is the WebSocket provider for yrs — [crates.io/crates/yrs-warp](https://crates.io/crates/yrs-warp), [y-crdt README](https://github.com/y-crdt/y-crdt).
- **Liveblocks**: the "core WebSocket server is open source under the AGPL v3 license, while client libraries are… Apache 2.0". However, "full self-hosting for production is not yet available" — [Liveblocks blog](https://liveblocks.io/blog/open-sourcing-the-liveblocks-sync-engine-and-dev-server) (search snippet). `@liveblocks/client` is at 3.24.2 (2026-09-21) — [npm](https://www.npmjs.com/package/@liveblocks/client). Liveblocks also offers Yjs hosting — [Liveblocks Yjs hosting](https://liveblocks.io/technology/hosting-platform-for-yjs).
- **PartyKit/PartyServer**: npm `partykit` was last published 2025-05-21 (0.0.115). `y-partyserver` 2.2.0 (2026-04-24) is the "Yjs backend for PartyServer" — [npm partykit](https://www.npmjs.com/package/partykit), [npm y-partyserver](https://www.npmjs.com/package/y-partyserver).
- **iroh-docs**: "Multi-dimensional key-value documents". Each entry's value is a BLAKE3 hash plus size plus timestamp. Sync uses range-based set reconciliation, persistence uses `redb`, and it builds on iroh-blobs and iroh-gossip. The namespace key serves "as a token of write capability" — [iroh-docs README](https://github.com/n0-computer/iroh-docs).

### Inferences
- iroh-docs is an LWW key→blob-hash store, not a structured CRDT. It is useful for *asset manifests* but not for the song document itself. Loro/Yrs updates can instead be broadcast over iroh-gossip if you want a P2P path.
- **Recommended backend:** your own axum server (Rust) holding Loro docs in memory per room. It persists snapshots and updates to SQLite or Postgres locally and to S3/R2 in the cloud, speaks loro-protocol (or a simple custom framing) over WebSocket, enforces auth per room, and issues share codes. Put Plane or a simple sticky-routing layer in front only when you need to scale past one box. If you prefer the Yjs route, self-host y-sweet, but accept the maintenance risk given ~12 months without a release.
- ElectricSQL is Postgres-centric read-path sync. I did not research it in depth, and it is not a natural fit for fine-grained concurrent song editing.

### Gaps
- I could not confirm y-sweet's maintenance status after September 2025 beyond registry dates. The repo reportedly had activity in December 2025 (search snippet), but there have been no new releases.
- I did not verify PartyKit's ownership or roadmap (believed to be Cloudflare-owned since 2024, but unverified here).
- I did not research ElectricSQL.

---

## 5. Playback sync across collaborators (shared transport, clock sync, Ableton Link, latency; editing vs. jamming)

### Takeaway
Treat **collaborative editing plus a shared transport** ("everyone hears the same song position, started on the next bar") as the realistic scope. That needs only an NTP-style clock-offset estimate (±5–20 ms is fine) and quantized start/tempo changes on a shared timeline, which is the same idea Ableton Link uses on a LAN. **Real-time jamming over the internet** is a separate problem: it needs under ~25 ms one-way, which is often physically impossible. If you do it, use NINJAM-style interval-delayed jamming rather than trying to beat latency.

### Cited Findings
- **Ableton Link** "synchronizes musical beat, tempo, and phase across multiple applications running on one or more devices… on a local network… anyone can start or stop while still staying in time. Anyone can change the tempo, the others will follow." It is header-only C++. It is **dual-licensed GPLv2+ / proprietary** (contact Ableton for proprietary use). The codebase now also includes `LinkAudio.hpp` ("Link Audio") — [Ableton/link README](https://github.com/Ableton/link).
- Link keeps "independent timelines" per app and maintains a temporal relationship between them. Phase is relative to a quantum, and peers with the same quantum align barlines. Start/stop "follow[s] the global launch quantization" — [Ableton Link documentation](https://ableton.github.io/link/), [SuperCollider LinkClock help](https://doc.sccode.org/Classes/LinkClock.html) (search snippets).
- Link's README advises deriving beat time from the **audio device's sample clock/host time** passed to the audio callback (for example CoreAudio's `mHostTime`) rather than a naive system-time read — [Ableton/link README](https://github.com/Ableton/link).
- `rusty_link` 0.4.9 (2026-04-02) wraps Ableton's `abl_link` C API and **must be GPL-2.0-or-later** because of Link's licensing — [crates.io/crates/rusty_link](https://crates.io/crates/rusty_link), [rusty_link GitHub](https://github.com/anzbert/rusty_link) (search snippet). A pure-Rust `ableton-link-rs` implementation (GPL-3.0) also exists — [crates.io ableton-link-rs](https://crates.io/crates/ableton-link-rs) (search snippet).
- `@ircam/sync` "synchronises all clients to a server master clock… Everybody can use the common master clock to schedule synchronized events. A good practice is to convert to local time at the last moment". It uses a ping/pong protocol over any transport (WebSocket example). Caveat: don't block the sync process, and keep it in the same thread/clock domain. Paper: Lambert, Robaszkiewicz, Schnell, "Synchronisation for Distributed Audio Rendering over Heterogeneous Devices, in HTML5", Web Audio Conference 2016 — [ircam-ismm/sync README](https://github.com/ircam-ismm/sync).
- Over WebSocket, developers "must implement application-level ping/pong heartbeats to estimate RTT and clock offset, essentially reinventing a simplified NTP". Clock drift of even 0.1% accumulates — [GetStream blog](https://getstream.io/blog/webrtc-websocket-av-sync/) (search snippet).
- Network music thresholds: the accepted "ensemble performance threshold" is about **<25 ms**. Above ~100 ms "it becomes pretty hard to play and especially sing" — [Comparison of remote music performance software (Wikipedia)](https://en.wikipedia.org/wiki/Comparison_of_remote_music_performance_software), [Baltimore Recorders NMP](https://www.baltimorerecorders.org/html/general/networked_musical_performance.html) (search snippets).
- **NINJAM** "delay[s] all received audio until it can be synchronized with other players based on the musical form", with latency "measured in measures". JackTrip and Jamulus aim for true low latency on good wired networks — [Ninjam (Wikipedia)](https://en.wikipedia.org/wiki/Ninjam), [Anna Xambó NMP review](https://annaxambo.me/blog/research/2020/06/02/network-music-performance/) (search snippets).

### Inferences
- **Shared transport design:**
  - Store `transport = {state: playing|stopped, tempo_bpm, anchor_beat, anchor_server_time, quantum}` in a server-authoritative channel, or in an LWW CRDT key with the server as tiebreaker.
  - Each client estimates `offset = server_time − local_time` with NTP-style ping/pong. Take the minimum-RTT sample of N and keep smoothing continuously.
  - Each client maps the shared anchor to its own `AudioContext.currentTime` (web) or audio-callback host time (native).
  - "Play" is scheduled for the next bar boundary at least ~100–200 ms in the future, so every client can schedule it sample-accurately.
  - Tempo changes are scheduled at a future beat as well.
  - Because song data is shared via the CRDT, each client renders audio locally, so no audio streaming is needed for "listen together".
- **Offset accuracy:** accuracy of a few ms to ~20 ms is inaudible as "sync" for listen-along. There is no need for sub-ms precision unless people are co-performing in one room (use Link on the LAN there).
- **Ableton Link** is valuable for native desktop users on a LAN (sync with Live, Bitwig and so on), but the GPL obligation means the app must be GPL or get Ableton's proprietary license. Link does not run in browsers (UDP multicast).
- **Scope recommendation:** v1 = async-safe collaborative editing plus presence plus shared transport ("listen along"). v2 (optional) = NINJAM-style interval jamming (record one bar or phrase locally, share it as an audio blob, and have others hear it one interval later), which fits the CRDT/asset architecture. Avoid JackTrip-style real-time audio.

### Gaps
- I found no Rust crate that implements an NTP-style clock-sync client/server for both native and wasm. It is simple to hand-roll, but I could not cite one.
- I could not access Ableton's Link docs site directly for protocol internals (the ping-based "GhostXForm" measurement) beyond search snippets.

---

## 6. Asset sync: audio samples as large binary blobs (content addressing, iroh-blobs, S3)

### Takeaway
Keep samples **out of the CRDT**. Store a content hash (BLAKE3) plus metadata in the song doc and move the bytes separately. Use S3/R2 (presigned upload and download via your server) for the browser-first path, and optionally **iroh-blobs** (BLAKE3 verified streaming, compiles for wasm-in-browser but relay-only there) for native P2P. Note that the current iroh-blobs line is flagged "not yet production quality".

### Cited Findings
- iroh-blobs is a "blob and blob sequence transfer" protocol "based on BLAKE3 verified streaming" that can request "blobs or ranges of blobs". A **Link** is "a 32 byte BLAKE3 hash of a blob", and `BlobTicket` bundles the endpoint address and hash for sharing. **"NOTE: this version of iroh-blobs is not yet considered production quality. For now, if you need production quality, use iroh-blobs 0.35"** — [iroh-blobs README](https://github.com/n0-computer/iroh-blobs). Latest is 0.103.0 (2026-06-15) — [crates.io/crates/iroh-blobs](https://crates.io/crates/iroh-blobs).
- iroh-blobs' Cargo.toml has explicit `wasm-in-browser` target dependencies (`cfg(all(target_family = "wasm", target_os = "unknown"))`) — [iroh-blobs Cargo.toml](https://github.com/n0-computer/iroh-blobs/blob/main/Cargo.toml). A browser iroh-blobs example exists in iroh-examples — [iroh-examples](https://github.com/n0-computer/iroh-examples) (search snippet).
- iroh's README describes iroh-blobs as "content-addressed blob transfer scaling from kilobytes to terabytes" — [iroh README](https://github.com/n0-computer/iroh).
- y-sweet persists docs to S3-compatible storage, "like Figma" — [y-sweet README](https://github.com/jamsocket/y-sweet).
- Loro's docs advise against CRDTs when "Your data isn't JSON-like (e.g., large binary/media streaming)" — [Loro docs (llms-full)](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).

### Inferences
- **Schema:** `samples: Map<hash, {name, bytes, sr, channels, duration}>` in the CRDT, with instruments referencing samples by hash. Clients check a local cache (OPFS/IndexedDB on the web, a disk cache on native) by hash, then fetch from the server/S3 (`GET /blobs/{blake3}`). The server verifies the hash on upload, so storage is naturally deduplicated.
- BLAKE3 hashing in wasm is fast (the `blake3` crate has a portable implementation). You can hash before upload so duplicates are skipped.
- For small bytebeat-style or sampled one-shots, a simple server blob store is enough. Add iroh-blobs only if LAN or P2P sharing of large packs matters.

### Gaps
- I did not verify S3 presigned-URL crate options (for example `aws-sdk-s3`, `rusty-s3`) or R2 specifics in this pass.
- There is no measured iroh-blobs throughput in browser relay-only mode.

---

## 7. Auth, permissions and "share a song via link/code"

### Takeaway
CRDTs don't enforce permissions, so the server (or a capability-key scheme) must. The proven pattern is **per-document client tokens** issued by your backend (y-sweet's model), with roles (viewer/editor) checked when a client joins a room and, for write-protection, enforced by rejecting updates from read-only connections. Share codes map to a doc ID plus a capability. P2P alternatives use cryptographic capabilities (iroh-docs namespace keys, iroh tickets) and optional E2EE (Loro `%ELO`).

### Cited Findings
- y-sweet: "document-level access control via client tokens", which developers generate "through backend endpoints before client connections"; the client calls `createYjsProvider(doc, docId, '/api/my-auth-endpoint')` — [y-sweet README](https://github.com/jamsocket/y-sweet).
- iroh-docs: the namespace key is "a token of write capability". Its public key (NamespaceId) identifies the replica, and author keys prove authorship — [iroh-docs README](https://github.com/n0-computer/iroh-docs).
- iroh-blobs `BlobTicket::new(addr, hash, format)` is a shareable ticket string — [iroh-blobs README](https://github.com/n0-computer/iroh-blobs). iroh-relay 1.0 added "Bearer token access control without an external service" — [iroh CHANGELOG](https://github.com/n0-computer/iroh/blob/main/CHANGELOG.md).
- Loro Protocol `%ELO` E2EE: the server "indexes plaintext headers only to support backfill and routing" and never decrypts — [loro-dev/protocol README](https://github.com/loro-dev/protocol).
- automerge-repo-sync-server is an "unsecured Express app" (no auth built in) — [automerge-repo-sync-server README](https://github.com/automerge/automerge-repo-sync-server).
- Automerge 0.12 / 3.5 records Author IDs in change metadata, useful for attribution ("who changed this knob") — [automerge Rust CHANGELOG](https://github.com/automerge/automerge/blob/main/rust/CHANGELOG.md).
- Loro: "CRDTs merge; they do not reject". Authorization "must be enforced at write time" by an authority — [Loro docs (llms-full)](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt).
- BandLab's "fork" model (copy a project to remix) is a popular sharing pattern in music apps — [BandLab blog](https://blog.bandlab.com/forking-and-collaboration-on-bandlab-explained/).

### Inferences
- **Share code design:** the server generates a short, human-friendly code (for example 6–8 Crockford base32 chars, like No Man's Sky-style base codes). It maps to `(doc_id, role, expiry)`, and redeeming it yields a signed room token (JWT/PASETO). Separate "view/listen" and "edit" codes. "Fork" = server copies the snapshot to a new doc ID, which is trivial with CRDT snapshots and shallow snapshots.
- **Read-only enforcement with a CRDT:** the server simply doesn't apply or relay updates from viewer connections. Because Loro/Yjs updates are opaque binary, the server must decode enough to validate (or trust role checks at the connection level).
- **Attribution:** map CRDT peer IDs to user IDs server-side (Loro peer ID / Yjs client ID / Automerge author) for "edited by" UI and per-user undo scoping.

### Gaps
- I found no off-the-shelf Rust library for share-code issuance or room tokens specific to CRDT servers. It is standard web-auth plumbing.

---

## 8. Recommended architecture (synthesis)

### Takeaway
Build a **star topology around a small Rust relay/persistence server** (axum + WebSocket, Loro documents per song, S3/R2 blobs, token auth). Every client, native or browser, runs the same Rust core with **Loro** for song state (MovableList, LWW maps, Counter, UndoManager, EphemeralStore presence) and a transport trait over WebSocket. Add a server-clock "shared transport" for listen-along playback. Keep P2P as an optional later layer: iroh 1.x for native LAN/internet, matchbox/WebRTC if browser-to-browser without a server ever becomes a requirement.

### Cited Findings
- Loro provides the needed containers, local undo, ephemeral presence and a Rust WS server/client protocol — [Loro README](https://github.com/loro-dev/loro), [Loro Undo/Ephemeral docs](https://github.com/loro-dev/loro-docs/blob/main/public/llms-full.txt), [loro-dev/protocol](https://github.com/loro-dev/protocol).
- Browsers cannot hole-punch with iroh (relay-only), so a relay is needed for browser participants regardless — [iroh docs: WebAssembly and Browsers](https://docs.iroh.computer/languages/wasm-browser) (search snippet).
- Server-defined ordering removes CRDT metadata overhead in Figma's design — [Figma blog](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/) (search snippet).
- WebTransport is now available in all major browsers (March 2026), and `web-transport` offers one native+wasm API — [WebRTC.ventures](https://webrtc.ventures/2026/04/webtransport-is-now-baseline-what-it-means-for-real-time-media/) (search snippet), [web-transport README](https://github.com/moq-dev/web-transport).

### Inferences
- **Topology choice:**
  - *P2P mesh:* fragile with browsers (TURN, signaling) and poor for persistence.
  - *Authoritative server with an OT/LWW model:* simplest, but you re-implement ordering, undo and presence, and lose offline and history.
  - *Relay server + CRDT (recommended):* the server is just another Loro peer that persists and authorizes. Clients can edit offline and merge. History and time travel give free "project versions".
- **Component picks (Sept 2026):**
  - `loro` 1.16 as the CRDT.
  - `axum` 0.8 + `tokio-tungstenite` 0.30 on the server.
  - `tokio-tungstenite` (native client) and `gloo-net` 0.7 / `web-sys` WebSocket (wasm client).
  - loro-protocol (0.3), or a minimal custom framing: `{room, kind: doc-update | ephemeral | clock-ping | transport}`.
  - SQLite/Postgres + S3/R2 for snapshots and blobs.
  - BLAKE3 content hashes for samples.
  - A custom NTP-style ping for the clock.
  - Optional `rusty_link` (GPL) on desktop.
- **Fallbacks:**
  - If JS-ecosystem interop matters (for example embedding Yjs editors or using y-sweet or Liveblocks hosting), choose `yrs` 0.28 instead. Model ordered lists as Maps with fractional indices, because the Yrs move op isn't in mainline Yjs, and implement counters as maps of per-client sums.
  - If typed Rust structs and the automerge-repo ecosystem matter more than undo and move, choose automerge 0.12 + autosurgeon + samod.
- **Flagged as immature or risky:** samod (self-described experimental), lorosurgeon (0.2), iroh-blobs current line (not production quality per its README), y-sweet (no release since 2025-09), wtransport (not fully production-ready), matchbox (pins old webrtc 0.17), Yjs v14 (beta), and y-octo (0.1).

### Gaps
- There is no public reference implementation of a Rust + Loro collaborative *music* app to validate the schema against. The DAW schema mapping above is inference.
- End-to-end latency budgets (edit→remote apply) over WebSocket vs. WebTransport vs. iroh relay were not measured.
