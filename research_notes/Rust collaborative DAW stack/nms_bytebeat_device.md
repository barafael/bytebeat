# No Man's Sky ByteBeat Device and classic "bytebeat": feature baseline for a miniature collaborative DAW

> Method note (read first): the research sandbox's egress proxy blocked direct page fetches of nomanssky.com, nomanssky.fandom.com, nomanssky.miraheze.org, steamcommunity.com, gamerguides.com, gearnews.com, nomansskyresources.com, thesixthaxis.com, arxiv.org/ar5iv and countercomplex.blogspot.com. GitHub was reachable. As a result, **most NMS facts below come from search-engine snippets of those pages, not from full-page reads.** Where a snippet could have come from any of several pages in the result set, I name the most likely page and flag the attribution as "snippet". Official Hello Games wording (quoted by several outlets identically) is the highest-confidence material. Exact numeric parameters (BPM range, operator list, etc.) could not be verified and appear under Gaps.

## 1. When was the ByteBeat Device introduced, and what is its UI and workflow?

### Takeaway
ByteBeat arrived in **Update 2.24 ("Beyond Development Update 5 – ByteBeat"), released 16 December 2019**, which was a post-launch patch in the Beyond cycle, not part of the Beyond 2.0 launch. It is a buildable, powered base part. When you place it, it starts playing procedurally generated music straight away. You can then open a Sequencer UI (melody, drums, arpeggiator, octave, key, tempo) and an "Advanced Waveform UI" where you edit the maths operators that make the sound. The only big change since then came in **Prisms (3.5, June 2021)**, which added a portable ByteBeat Library/player, track sharing and an improved drum synth.

### Cited Findings
- Hello Games' dev blog post is titled "Beyond Development Update 5 – ByteBeat" and dated December 2019. — [Hello Games](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/)
- The patch version is 2.24, reported live on all platforms on 16 Dec 2019. — [TheSixthAxis](https://www.thesixthaxis.com/2019/12/16/no-mans-sky-beyond-update-2-24-5-beatbyte-patch-notes/); [PlayStation LifeStyle](https://www.playstationlifestyle.net/2019/12/16/no-mans-sky-music-update-bytebeat/); [NMS Wiki (Miraheze) – Update 2.24](https://nomanssky.miraheze.org/wiki/Update_2.24)
- Official patch note: ByteBeat was "added as a new base prop that allows players to generate and compose their own procedural music. Switches and cabling have been added to allow this music to control other base features." — [TheSixthAxis (patch notes)](https://www.thesixthaxis.com/2019/12/16/no-mans-sky-beyond-update-2-24-5-beatbyte-patch-notes/)
- Official description: "Once placed in your base and powered, the ByteBeat will immediately begin to produce sound… ByteBeat formulas are made out of simple waveforms that are manipulated through maths – but by default, the device handles all of the mathematical heavy lifting, procedurally generating random presets for you to play with. Dedicated audiophiles have the option to explore deeper, manually sketching out note sequences, rhythms, and even manipulating the raw sounds." — [Hello Games dev update](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/); [NMS Resources](https://www.nomansskyresources.com/bytebeat)
- Official description: "The Sequencer UI lets you customise your track, allowing you to modify the melody and drums. If enabled, the arpeggiator will fill in notes automatically, and other panels allow you to adjust the octave, key and tempo." — [Hello Games dev update](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/)
- Official description: "Players who want to dive deep into the maths behind the waveform can use the Advanced Waveform UI to fine tune the mathematical operators at the heart of their sound." — [NMS Resources](https://www.nomansskyresources.com/bytebeat); [wccftech](https://wccftech.com/bytebeat-audio-creation-app-added-to-no-mans-sky-in-game/)
- GamesRadar describes the reveal trailer as showing "a sequencer, synchroniser, and waveform tree, not to mention nested arrangements and synchronized devices." — [GamesRadar+](https://www.gamesradar.com/no-mans-skys-new-bytebeat-synthesizer-lets-you-make-custom-tracks-to-play-in-your-base/)
- The wiki describes it as "an advanced audio generator, allowing the user to synthesise complex musical arrangements. Multiple devices may be placed in sequence or connected to lights to generate spectacular displays." — [NMS Wiki (Fandom) – ByteBeat Device](https://nomanssky.fandom.com/wiki/ByteBeat_Device)
- Crafting: Metal Plating ×3 + Gold ×50 + Antimatter ×1. The blueprint comes from the Construction Research Station on the Space Anomaly and costs Salvaged Data. The device must be powered. — [NMS Wiki (Fandom) – ByteBeat Device](https://nomanssky.fandom.com/wiki/ByteBeat_Device)
- UI layout (snippet): "the Melody Panel on top and the Rhythm Panel on the bottom"; you "select notes on the Melody Panel and set the beat on the Rhythm panel"; you can set "the Beats Per Minute, the Key, the Time Signature, and the Pitch." — [Gamer Guides](https://www.gamerguides.com/no-mans-sky/guide/walkthrough/tips-and-tricks/how-to-build-and-use-a-bytebeat-device) (snippet)
- The Prisms update (3.5, early June 2021) added "a ByteBeat library and music player… to the Quick Menu, under Utilities" and "Significant improvements… to the ByteBeat's drum synthesizer, allowing for the creation of meatier drum loops and driving rhythms." — [HITC Prisms patch notes](https://www.hitc.com/en-gb/2021/06/03/no-mans-sky-prisms-update-patch-notes/); [Hello Games – Prisms](https://www.nomanssky.com/prisms-update/)
- Patch 3.51 "Fixed some mistranslated text in the ByteBeat Library." — [NMS Wiki (Fandom) – Update 3.51](https://nomanssky.fandom.com/wiki/Update_3.51)

### Inferences
- The workflow is **preset-first and then refine**. A new device plays a random procedural preset straight away, and editing is optional. For a DAW baseline, this means "new track = playable randomized pattern" is part of what NMS offers.
- The device has about three editing layers: (a) sequencer (melody + rhythm grid), (b) global/musical settings (BPM, key, octave, time signature, pitch), and (c) sound design (envelope + advanced waveform operator tree). A fourth, multi-device layer (Synchroniser) sits above them.

### Gaps
- I found no evidence of ByteBeat changes after Prisms/3.51 (2021) up to Sept 2026. Searches for 2024–2025 ByteBeat changes only turned up the 2019 feature list and community requests for an upgrade. The device looks functionally unchanged since 2021, but I could not confirm this against every patch note.
- I could not read the full dev-blog text or any screenshots, so the exact on-screen panel names and tab order are not verified beyond the quotes above.

## 2. Exact feature inventory: voices, tracks, steps, keys, tempo, effects, randomization, parameters

### Takeaway
Each ByteBeat Device is **one "voice"** (the Steam guide compares it to one singer in a choir). It has a melodic line with a **7-note pitch range**, a selectable **4, 8 or 16-step** grid, an **arpeggiator** (on/off dot rows, note speed 1–4), a **three-element drum machine** (hi-hat, snare, bass/kick) with **13 selectable sounds per element**, an **envelope editor**, and an **Advanced Waveform UI** (a "waveform tree" of maths operators). Global settings are BPM, key, octave, time signature, pitch, volume and attenuation. Up to **8 devices** can be linked. I found no evidence of dedicated reverb, delay, filter or swing controls.

### Cited Findings
- Feature list reported for the device: "Melody Sequencer, Envelope editor, Waveform editor, BPM/Key/Volume/Attenuation controls, Drum machine, Arpeggiator, Synchroniser, the ability to link ByteBeats, and the ability to use audio to control objects." — search snippet from the result set containing [XainesWorld](https://www.xainesworld.com/make-actual-music-in-no-man-s-sky-with-bytebeat/) and the [Fandom ByteBeat Device page](https://nomanssky.fandom.com/wiki/ByteBeat_Device). The search engine labelled this "recent updates", but it matches the Dec 2019 launch feature set. (snippet)
- **Pitch range:** "The 7-note range per device is insufficient… An octave would be better, with 2 or 3 octaves per device being much better still." — [Steam: "Bytebeat upgrades – a detailed petition"](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/) (snippet)
- **Step count:** the game has step-sequencer options "4" for a "slow singer", "8" for a "medium singer" or "16" for an "upbeat singer". The petition says that "having options for 4, 8 and 16-step sequencer confuses many players" and that this "confines users to the same value note per device". — [Steam petition](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/) (snippet)
- **Note length:** the petition asks for notes that "can be inputted and then dragged horizontally… over multiple steps to create longer notes" and "dragged in half to the left to create notes of half their value". This implies the current grid has no ties or variable note lengths. — [Steam petition](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/) (snippet)
- **Arpeggiator:** "the section above the step sequencer where you can see rows of dots that can be turned on or off – each dot representing a note stab. You can alter the number of notes the Arp plays…". **Note speed:** "'1' is slow and less embellished, '4' is fast and very embellished." — [Steam guide "My First ByteBeat -- in Spaaace!"](https://steamcommunity.com/sharedfiles/filedetails/?id=1941411630) (snippet)
- **Drums:** "you are able to select thirteen different sounds for each of the percussive elements (high hat, snare, bass) in the ByteBeat Device's Rhythm Panel." — snippet from a result set containing [Gamer Guides](https://www.gamerguides.com/no-mans-sky/guide/walkthrough/tips-and-tricks/how-to-build-and-use-a-bytebeat-device) and [wccftech](https://wccftech.com/bytebeat-audio-creation-app-added-to-no-mans-sky-in-game/). I could not confirm which page it came from.
- **Drum synth was upgraded in Prisms (2021)** ("meatier drum loops and driving rhythms"). — [HITC Prisms patch notes](https://www.hitc.com/en-gb/2021/06/03/no-mans-sky-prisms-update-patch-notes/)
- **Envelope editor:** "if you start the note steeply, the note starts percussively on the beat; if you start flatly, the note fades in and sounds off-beat or even jazzy; if you end steeply, the note is played fully to the end; if you end flatly, the note fades out softly." This implies attack and release shape control. — likely the [Steam guide](https://steamcommunity.com/sharedfiles/filedetails/?id=1941411630) (snippet; attribution probable, not certain)
- **Advanced Waveform UI / waveform tree:** "ByteBeat formulas are made out of simple waveforms that are manipulated through maths", and the Advanced Waveform UI is used "to fine tune the mathematical operators at the heart of their sound". The trailer shows a "waveform tree". — [Hello Games](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/); [GamesRadar+](https://www.gamesradar.com/no-mans-skys-new-bytebeat-synthesizer-lets-you-make-custom-tracks-to-play-in-your-base/)
- **Randomization:** "by default, the device handles all of the mathematical heavy lifting, procedurally generating random presets for you to play with." — [Hello Games](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/)
- **Globals:** BPM, Key, Time Signature, Pitch — [Gamer Guides](https://www.gamerguides.com/no-mans-sky/guide/walkthrough/tips-and-tricks/how-to-build-and-use-a-bytebeat-device) (snippet); octave, key, tempo — [Hello Games](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/); Volume/Attenuation — XainesWorld/Fandom snippet (above).
- **Song state size:** "Each song from any specific Bytebeat Device is translated into a 60-character string of code". The alphabet is described as 64 characters: A–Z, a–z, 0–9, "+" and "/". — [NMS Wiki (Fandom) – ByteBeat Catalogue](https://nomanssky.fandom.com/wiki/ByteBeat_Catalogue) (snippet)
- gearnews (a music-tech site) calls it "a sequencer, synthesis and sound generation engine in which you can create your own music that plays within your base". — [gearnews](https://www.gearnews.com/sequencing-in-an-infinite-universe-bytebeat-for-no-mans-sky/) (snippet)

### Inferences
- The 64-character alphabet is standard Base64. A 60-character Base64 string holds about 45 bytes (60 × 6 bits = 360 bits). So a whole device state (notes, drums, envelope, operator tree, globals) fits in roughly 45 bytes. That points to a small, quantized parameter set (a few bits per step or parameter), not free-form code or audio. A DAW that "matches" NMS needs only a tiny patch format. A DAW that "exceeds" it can offer a shareable text code that is still short, such as Base64 of a compressed binary state.
- "Same value note per device" plus a fixed 4/8/16 step choice suggests one step resolution per device and no per-note length. A polyphonic, multi-length piano roll would already exceed NMS.
- "Time Signature" and the 4/8/16 choice may be the same control seen from two sides (steps per bar versus note rate). The sources do not settle this.

### Gaps
- **BPM range, key/scale list (major/minor/modes?), octave range, time-signature options, pitch range in semitones: no numbers found.** I could not open the wiki, Steam guide or Gamer Guides pages.
- **Swing/shuffle:** no source mentions it. Treat as absent (unconfirmed).
- **Effects (reverb, delay, distortion, filter):** no source mentions dedicated FX. The Advanced Waveform operators may give distortion or waveshaping, but that is unconfirmed. Hello Games has not published the operator list (for example which bitwise or arithmetic ops, or which base waveforms) in anything I could read.
- **Polyphony per device:** the arpeggiator implies chord-to-arp behaviour, but I found no statement on simultaneous notes per device.
- Pattern chaining (A/B patterns, song mode) within one device is not documented. "Nested arrangements" (GamesRadar) may refer to Synchroniser-driven multi-device arrangements; unconfirmed.

## 3. Multi-device interaction, switches/cables, multiplayer and sharing

### Takeaway
Devices sync by **snapping them together or joining them with a ByteBeat Cable**. Linked devices share **tempo and key** but keep their own notes, drums and sounds. **Up to 8 devices** can be linked, and a **Synchroniser tab** controls when each one is active. The **ByteBeat Switch** sends rhythmic power pulses (for example to lights). Since Prisms (2021), players can **record a base's ByteBeat output into a personal Library of 8 slots**, play it anywhere from the Quick Menu, and **send tracks to nearby players**. Songs also circulate as **~60-character Base64 codes**, edited in via save editors, and through community hubs.

### Cited Findings
- "The ByteBeat Device can be synchronised by snapping it to other ByteBeats, or connecting the devices with a new type of cable, allowing you to layer multiple tracks and create more complex arrangements." — [Hello Games](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/); [NMS Resources](https://www.nomansskyresources.com/bytebeat)
- "A new ByteBeat Switch allows players to power other devices, such as lights, with bursts of power that sync up to the rhythm of your track." — [NMS Resources](https://www.nomansskyresources.com/bytebeat); related wiki pages: [Bytebeat Switch](https://nomanssky.fandom.com/wiki/Bytebeat_Switch), [ByteBeat Cable](https://nomanssky.fandom.com/wiki/ByteBeat_Cable), [NMS Resources – Bytebeat Cable](https://www.nomansskyresources.com/constructed-technology/bytebeat-cable)
- Community tutorials use Switches for light shows, e.g. "ByteBeat Light Sequencer – ByteBeat Switches Guide". — [YouTube (NMS Scottish Rod)](https://www.youtube.com/watch?v=HRl-Oxy7nHs)
- "When machines are joined together the tempo and key will match across them all, but the note sound, drums, melody etc remain exclusive for each machine." — snippet from a result set containing the [Steam guide](https://steamcommunity.com/sharedfiles/filedetails/?id=1941411630) and [Miraheze Update 2.24](https://nomanssky.miraheze.org/wiki/Update_2.24)
- "You can also connect a total of eight ByteBeat devices together and control when they are active via the Synchroniser tab." — snippet from a result set containing [GamesRadar+](https://www.gamesradar.com/no-mans-skys-new-bytebeat-synthesizer-lets-you-make-custom-tracks-to-play-in-your-base/), [wccftech](https://wccftech.com/bytebeat-audio-creation-app-added-to-no-mans-sky-in-game/) and [Gamer Guides](https://www.gamerguides.com/no-mans-sky/guide/walkthrough/tips-and-tricks/how-to-build-and-use-a-bytebeat-device)
- The "choir" model: "Imagine each ByteBeat device as a singer in a choir. Several devices adjacent to each other will connect. If you have 8 devices connected, you have 8 voices in your choir." — [Steam guide](https://steamcommunity.com/sharedfiles/filedetails/?id=1941411630) (snippet)
- Prisms: "When within a base (your own or another player's) with active ByteBeat Devices, you can save the devices' output as a track to your Library… The ByteBeat Library can be used to send tracks to nearby players. Sent tracks are saved to their Library." — [HITC Prisms patch notes](https://www.hitc.com/en-gb/2021/06/03/no-mans-sky-prisms-update-patch-notes/); [Hello Games – Prisms](https://www.nomanssky.com/prisms-update/)
- "The library has room for 8 custom songs… so any more than that must be stored at each base." The Crescendo Project is described as "a creative hub in Euclid where ByteBeat composers can share their creations, and visitors can collect the tracks for their Library." — snippet from a result set containing [Hello Games – Prisms Development Update (July 2021)](https://www.nomanssky.com/2021/07/prisms-development-update/) and the [ByteBeat Catalogue](https://nomanssky.fandom.com/wiki/ByteBeat_Catalogue)
- Save-file level: songs live in a "ByteBeatLibrary" package with "MySongs" slots numbered "0" to "7", and can be extracted or injected as .txt via a save editor. — [NMS Wiki (Fandom) – ByteBeat Catalogue](https://nomanssky.fandom.com/wiki/ByteBeat_Catalogue) (snippet)
- The community keeps a list of published player songs (the ByteBeat Catalogue) and a subreddit, r/NMSByteBeatFans. — [ByteBeat Catalogue](https://nomanssky.fandom.com/wiki/ByteBeat_Catalogue)
- Steam thread titles show common sharing and copying needs: "Share your byte beat music", "Bytebeat copy all sequencers?", "ByteBeat Creation Tool?". — [Steam: Share your byte beat music](https://steamcommunity.com/app/275850/discussions/0/5940851794735101641/); [Steam: Bytebeat copy all sequencers?](https://steamcommunity.com/app/275850/discussions/0/592890004253002649/); [Steam: ByteBeat Creation Tool?](https://steamcommunity.com/app/275850/discussions/0/3802778829239173751/)

### Inferences
- The NMS "multitrack" model is **N independent single-voice devices (N ≤ 8) sharing a transport (tempo) and a key**. The Synchroniser acts as a simple arranger that decides when each device is active. A DAW baseline therefore needs at least: 8 tracks, a shared transport and key, per-track mute and activity scheduling (song/arrangement), and per-track instrument state.
- Collaboration in NMS is **asynchronous and object-based**. You visit someone's base, save the output to your library, receive a track from a nearby player, or paste a code with a save editor. I found no evidence of **real-time co-editing of one device's pattern by several players**. That is the most obvious "and then some" axis for a collaborative DAW.
- The ByteBeat Switch shows that **music-as-control-signal** (rhythmic triggers driving other things) is part of the NMS feature set. A DAW equivalent would be step or trigger outputs such as MIDI/OSC out, automation lanes, or visualizer hooks.
- The 8-slot library, the "0–7" save slots and the 8-device link limit all line up on the number 8. This looks like a deliberate engine constant.

### Gaps
- Whether visitors to an **uploaded base** in a different session hear or can copy the devices is only implied by the Prisms text ("your own or another player's" base). I did not verify how base upload interacts with this.
- I found no documentation of **how multiplayer sessions handle simultaneous edits** to one device (lock, last-writer-wins, host authority).
- I could not read the exact Synchroniser semantics (per-bar on/off patterns? sequence order?) or the Switch parameters (which step or instrument triggers it).
- I found no evidence of any official in-game "song code" copy/paste UI. The 60-character code appears to be a save-file representation used with save editors (inference from the Catalogue snippet).

## 4. Known limitations, community complaints and community usage

### Takeaway
The main complaints are the **7-note pitch range per device**, **confusing 4/8/16-step modes with one note value per device**, **no variable note length**, **limited library storage (8 slots)**, and a UI that is **opaque to non-musicians and still constraining for musicians**. The community still makes covers and soundtracks, light shows and dedicated music bases, and writes tutorials.

### Cited Findings
- The petition says a 7-note range "actually forces technically adept players to use creative but unnecessarily complex workarounds for simple tasks. An octave would be better, with 2 or 3 octaves per device being much better still, and this single change would be transformative." — [Steam petition](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/) (snippet)
- The petition says the 4/8/16 options "confuse many players" and "confine users to the same value note per device, significantly curtailing musical options". — [Steam petition](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/) (snippet)
- Players want "QoL improvements, more storage, and perhaps radio stations selectable in spacecraft cockpits to hear featured player songs". ByteBeat "can feel obscure or frustrating for users without music backgrounds", and even musicians find "the bytebeat device dynamics beyond their abilities". The petition notes ByteBeat has been in the game about 5.5 years and is "ripe for a serious upgrade in 2025". — snippets from a result set containing the [Steam petition](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/) and [Steam: ByteBeat? Seriously?](https://steamcommunity.com/app/275850/discussions/0/2865910147607071740/)
- Other complaint threads by title: "new bytebeat thing .. why no note edit", "Is bytebeat broken", "ByteBeat: Very disappointing." — [Steam](https://steamcommunity.com/app/275850/discussions/0/1747891643717382973/); [Steam](https://steamcommunity.com/app/275850/discussions/0/601897727418203200/); [Steam](https://steamcommunity.com/app/275850/discussions/0/2865910147607395035/?ctp=2)
- Tutorials include "No Man's Sky ByteBeat Device Tutorial [A Detailed Guide]", "A (Beginners) Guide to Bytebeat in No Man's Sky", and the Steam guide "My First ByteBeat -- in Spaaace!". — [YouTube](https://www.youtube.com/watch?v=ptUKrg6v1Os); [YouTube](https://www.youtube.com/watch?v=SMXF73df1FI); [Steam guide](https://steamcommunity.com/sharedfiles/filedetails/?id=1941411630)
- Content made with it: a "No Man's Sky ByteBeat Soundtracks/Covers" YouTube playlist, a "Prisms Byte Beat Collection Playlist" build guide (2021), the Crescendo Project hub in Euclid, the ByteBeat Catalogue and r/NMSByteBeatFans. — [YouTube playlist](https://www.youtube.com/playlist?list=PLSAz05xjco2Yu_f52QvNC3RW-wDRp8W4t); [YouTube](https://www.youtube.com/watch?v=lK-IvohdZsk); [ByteBeat Catalogue](https://nomanssky.fandom.com/wiki/ByteBeat_Catalogue)
- A ByteBeat Poster decor part also exists. — [NMS Resources – Bytebeat Poster](https://www.nomansskyresources.com/construction-parts/bytebeat-poster)

### Inferences
- "Covers" of real songs point to user demand for **more pitch range, chromatic notes and longer patterns** than the device allows. Players work around the 7-note limit by stacking several devices.
- The "8 slots, store the rest in bases" limit plus the use of save editors to move songs point to demand for **unlimited cloud or project storage and a first-class share code or link**.

### Gaps
- I could not read full thread bodies, so the petition's complete list (it is described as "detailed") is only partly captured.
- I did not find Hello Games responses to the petition, or any announcement of a ByteBeat overhaul, up to Sept 2026.
- The task mentions a "ByteBeat Amplifier". **I found no source confirming a part by that name.** Searching the exact phrase returned only Device, Switch, Cable and Poster pages. It is probably not a real part; treat it as unverified.

## 5. Classic bytebeat, and whether NMS's device is formula bytebeat or a step sequencer/synth

### Takeaway
Classic bytebeat (viznut, Sept–Oct 2011) is a **single integer expression of the sample counter `t`**, evaluated about **8000 times per second**, with the **low 8 bits used as unsigned 8-bit mono PCM**. Examples are `t*((t>>12|t>>8)&63&t>>4)` (viznut) and `t*((t>>5|t>>8)>>(t>>16))` (tejeez). Modern players add signed bytebeat, floatbeat and funcbeat modes, several sample rates, stereo, RPN syntax and URL sharing. **NMS's ByteBeat is not a text-formula bytebeat engine.** It is a **step sequencer + arpeggiator + drum machine** whose melodic *timbre* comes from a bytebeat-inspired tree of maths operators on simple waveforms, with the tree editable in the "Advanced Waveform UI". The name borrows the technique, but musical structure comes from the sequencer, not from `t`-driven formula structure. That last point is inferred; see Inferences.

### Cited Findings
- viznut's post "Algorithmic symphonies from one line of code -- how and why?" appeared on 2 Oct 2011 on countercomplex. Viznut is Ville-Matias Heikkilä, a Finnish demoscener. A YouTube video of seven programs on 26 Sept 2011 came first, and Bemmu's online JS tool widened participation. — [countercomplex](http://countercomplex.blogspot.com/2011/10/algorithmic-symphonies-from-one-line-of.html); mirror [viznut.fi](http://viznut.fi/texts-en/bytebeat_algorithmic_symphonies.html); follow-up [Some deep analysis of one-line music programs](http://countercomplex.blogspot.com/2011/10/some-deep-analysis-of-one-line-music.html)
- The canonical programs are collected in a repo (source posted 2011-10-02). `1-viznut.c` is `main(t){for(t=0;;t++)putchar(t*((t>>12|t>>8)&63&t>>4));}` and `2-tejeez.c` is `main(t){for(t=0;;t++)putchar(t*((t>>5|t>>8)>>(t>>16)));}`. The playback script pipes "8kHz 8-bit" data through `sox -r 8000 -c 1 … -t u8` to aplay. — [kragen/viznut-music (GitHub)](https://github.com/kragen/viznut-music) (README, `1-viznut.c`, `2-tejeez.c` and `alsaplay` read directly)
- viznut's arXiv paper "Discovering novel computer music techniques by exploring the space of short computer programs" (arXiv:1112.1368, Dec 2011) defines the form `main(){int t=0;for(;;t++)putchar(EXPRESSION);}`. The expression is "evaluated with 32 or more bits of integer accuracy", and "only the eight lowest bits of each result show up in the output, interpreted as unsigned 8-bit PCM sample values with the rate of 8000 samples per second." — [ar5iv 1112.1368](https://ar5iv.labs.arxiv.org/html/1112.1368) (snippet; page not directly fetchable)
- The task's example `t*(t>>5|t>>8)` is the core of tejeez's formula, which adds `>>(t>>16)` (repo above). viznut's own first formula is the `(t>>12|t>>8)&63&t>>4` one.
- Community definition: "a single-line formula that defines a waveform as a function of time, processed usually 8000 times per second, resulting in an audible waveform with a 256-step resolution"; output is a "headerless unsigned 8 bit mono 8kHz audio stream". — [EnBeat wiki (GitHub)](https://github.com/Chasyxx/EnBeat_NEW/wiki) (snippet)
- **Format variants** (dollchan Bytebeat Composer):
  - Bytebeat: unsigned 8-bit 0–255, wrapped and floored.
  - Signed Bytebeat: signed 8-bit, wrapped.
  - Floatbeat: −1.0…1.0 ("higher quality audio since it is not limited to 255 values").
  - Funcbeat: code runs once and returns a function that is called with time in seconds and outputs −1.0…1.0.

  — [dollchan Bytebeat composer](https://dollchan.net/bytebeat/) (snippet). The snippet gives the signed range as "−127 to 128", which conflicts with greggman's "−128 to 127"; the latter is standard two's-complement.
- dollchan is a "Live editing algorithmic music generator" with "a collection of many formulas from around the internet". Its library is stored as gzip-compressed JSON plus separate JS files for large songs, with optional PHP/MySQL database tooling and a discussion forum (dollchan.net/btb/). — [SthephanShinkufag/bytebeat-composer (GitHub)](https://github.com/SthephanShinkufag/bytebeat-composer)
- **greggman HTML5 Bytebeat:**
  - Output types: Bytebeat (0–255), Floatbeat (−1…+1), Signed Bytebeat (−128…127).
  - Syntax: infix JS (with Math auto-prefixing), postfix/RPN (with `dup`, `swap`, `pick`), Glitch Machine `glitch://` format, and a function mode where `t` is in seconds.
  - Sample rates: 8000, 11000, 22050, 44100 and 48000 Hz.
  - Stereo via `[left, right]` arrays.
  - `mouseX`/`mouseY` inputs.
  - A reusable WebAudio `ByteBeatNode`.
  - URL-encoded sharing and visualizers.

  — [greggman/html5bytebeat (GitHub)](https://github.com/greggman/html5bytebeat)
- NMS's own wording, "ByteBeat formulas are made out of simple waveforms that are manipulated through maths", edited through an "Advanced Waveform UI" of "mathematical operators" shown as a "waveform tree". — [Hello Games](https://www.nomanssky.com/2019/12/beyond-development-update-5-bytebeat/); [GamesRadar+](https://www.gamesradar.com/no-mans-skys-new-bytebeat-synthesizer-lets-you-make-custom-tracks-to-play-in-your-base/)

### Inferences
- **NMS = hybrid; the pitch and rhythm structure comes from the sequencer, not from a formula.** Evidence:
  1. Notes are entered on a melody grid with a 7-note range and 4/8/16 steps.
  2. Drums are chosen from 13 samples or synth sounds per element on a rhythm grid.
  3. Key, octave, BPM and time signature are musical parameters that classic bytebeat does not have (a classic formula has no notion of key or BPM except implicitly).
  4. The whole state fits in about 45 bytes (the 60-character Base64 code), too small for arbitrary expressions plus sequences, but consistent with a quantized operator tree and parameters.

  The "formula" part is best read as **a small expression tree (operators over oscillators, possibly with bitwise ops) that defines the waveform/timbre of each note**. No source I found says players can type free-form `t`-expressions, and none says the operator tree runs at 8 kHz/8-bit. **Not confirmed:** whether the NMS operator tree includes bitwise ops (`&`, `|`, `>>`) like classic bytebeat.
- **Requirements implication:** a DAW that "matches NMS" needs a step sequencer, arpeggiator, drum machine, envelope and an operator-tree synth. A DAW that also covers "real" bytebeat needs a separate **formula instrument**: an expression parser (infix, and optionally RPN), modes (bytebeat, signed, floatbeat, funcbeat), a selectable sample rate (8 kHz default, resampled to the device rate), stereo returns, and ideally tempo- or key-aware variables so formulas can sync to the shared transport. NMS lacks all of these.

### Gaps
- I could not read the full arXiv paper or viznut's blog, so the quotes above come from snippets. The repo contents were read directly.
- dollchan's live-app features (sample-rate selector, oscilloscope/spectrogram, recording/export, URL sharing) could not be confirmed from the GitHub README fetch, which gave limited detail. greggman's feature list is confirmed.

## 6. What an "and then some" DAW would plausibly add beyond NMS (grounded in user asks)

### Takeaway
User requests centre on **range and expressiveness** (1–3+ octaves, chromatic notes, variable note lengths), **storage and sharing** (more than 8 slots, easy copy/share, radio-style playback), **copy and paste of sequencers**, and **approachability**. The bytebeat tooling ecosystem adds **text-formula instruments, several output formats and sample rates, stereo, URL sharing and visualizers**. Real-time collaboration is missing from NMS altogether.

### Cited Findings
- Wider pitch range per voice (octave; ideally 2–3 octaves). — [Steam petition](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/) (snippet)
- Variable note lengths (drag to extend across steps or halve), instead of the global 4/8/16 step-value choice. — [Steam petition](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/) (snippet)
- More storage than the 8-slot library, plus "radio stations selectable in spacecraft cockpits to hear featured player songs". — [Steam petition / discussions](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/) (snippet); 8-slot limit: [ByteBeat Catalogue](https://nomanssky.fandom.com/wiki/ByteBeat_Catalogue) (snippet)
- Copying whole multi-sequencer setups ("Bytebeat copy all sequencers?"), and an external "ByteBeat Creation Tool?". — [Steam](https://steamcommunity.com/app/275850/discussions/0/592890004253002649/); [Steam](https://steamcommunity.com/app/275850/discussions/0/3802778829239173751/)
- Approachability: a UI that feels "obscure or frustrating" for non-musicians. — [Steam petition](https://steamcommunity.com/app/275850/discussions/3/601908674062998667/) (snippet)
- Formula-instrument feature bar set by web bytebeat tools: several output modes, infix and RPN syntax, sample rates from 8 to 48 kHz, stereo, URL-shareable songs, visualizers and a large formula library. — [greggman/html5bytebeat](https://github.com/greggman/html5bytebeat); [dollchan composer](https://github.com/SthephanShinkufag/bytebeat-composer)

### Inferences
Candidate requirements baseline. "Match" means NMS parity; "Exceed" means grounded extensions.

| Area | Match NMS (baseline) | Exceed (and then some) |
|---|---|---|
| Voices/tracks | ≥8 single-voice tracks sharing tempo and key | Unlimited or large track count; per-track key override; polyphony |
| Sequencer | 4/8/16-step grid; 7-note melody range; drum grid (hi-hat/snare/kick, 13 sounds each) | Chromatic multi-octave piano roll; per-note length/ties; velocity; per-step probability; swing (absent in NMS as far as found); pattern chaining and song mode |
| Arpeggiator | On/off dot rows; note-count; speed 1–4 | Arp modes (up/down/random), rate sync, gate, octave span |
| Sound design | Envelope (attack/release shape); operator-tree waveform editor; random presets | Full ADSR; filters; FX sends (reverb/delay/distortion, none documented in NMS); a **text bytebeat/floatbeat/funcbeat formula instrument** synced to the transport |
| Globals | BPM, key, octave, time signature, pitch, volume/attenuation | Tempo automation; scale/mode picker; master bus FX; metering |
| Arrangement | Synchroniser: schedule when each of up to 8 devices is active | Timeline or clip launcher; automation lanes |
| Control out | ByteBeat Switch → rhythmic triggers to lights | MIDI/OSC out, trigger lanes, visualizer hooks |
| Storage/sharing | ~60-character Base64 song code; 8-slot personal library; send to nearby player | Unlimited project library; share links and codes; WAV/stem export; import of classic bytebeat URLs and formulas |
| Collaboration | Asynchronous: visit a base, save output, send to a nearby player | **Real-time multi-user co-editing** with presence and conflict resolution (no NMS equivalent found) |
| Onboarding | Instantly playable random preset | "Generate/randomize" per track and per parameter; guided presets; undo/redo history |

### Gaps
- There is no direct user-survey evidence on real-time collaborative editing. That item is an inference from the gap in NMS, not a documented user request.
- The swing, FX and filter entries in the "Exceed" column rest on their absence from NMS sources. They are not backed by specific NMS user requests I could read.
