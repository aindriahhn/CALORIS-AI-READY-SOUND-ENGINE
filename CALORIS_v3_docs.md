  # CALORIS v3
  ### Ambient Chiptune Composition Studio
  *for WARM / Ἑκατόγατάκιαερες*
  
  ---
  
  ## Overview
  
  CALORIS is a browser-based ambient sequencer built on the Web Audio API. It runs entirely offline with no dependencies — open the HTML file in any modern browser and it works. Audio is produced by a bank of oscillators routed through LFOs, ADSR envelopes, filters, and delay lines, all driven by a step sequencer. Everything is tunable in real time.
  
  ---
  
  ## Getting Started
  
  1. Open `CALORIS_V3.html` in Chrome, Firefox, or Safari
  2. Press **▶ Play** to initialise the audio context
  3. Select a biome from the **Dungeon Type** tabs — it begins playing immediately
  4. Press **▶ Seq** to start the step sequencer
  5. Edit oscillators, steps, melody, and effects while the sequencer runs — all changes are live
  
  > Keyboard shortcut: **Ctrl+Z / Cmd+Z** undoes the last change (up to 20 levels)
  
  ---
  
  ## Transport & Master Controls
  
  ### Play / Stop
  Starts and stops all audio. The oscillators play in their ambient looping state when the sequencer is off.
  
  ### Seq
  Starts and stops the step sequencer. Only available while playing. The sequencer drives note gating, velocity, melody pitch changes, and MIDI output.
  
  ### BPM
  Beats per minute — sets the global tempo. Applies to all tracks simultaneously.
  
  ### Swing
  Adds timing offset (in ms) to odd-numbered steps, creating a shuffle feel. 0 = straight.
  
  ### Step Value
  The rhythmic unit of each clock tick. Options:
  
  | Label | Name | Multiplier |
  |---|---|---|
  | W | Whole | 4× |
  | H | Half | 2× |
  | Q | Quarter | 1× |
  | Q. | Dotted Quarter | 1.5× |
  | 3Q | Triplet Quarter | 0.667× |
  | E | Eighth | 0.5× |
  | E. | Dotted Eighth | 0.75× |
  | 3E | Triplet Eighth | 0.333× |
  | S | Sixteenth | 0.25× |
  | 3S | Triplet 16th | 0.167× |
  
  The step value multiplied by BPM gives the duration of each sequencer tick. The loop time in seconds is shown below the controls.
  
  ### Clock
  The number of ticks in one full sequencer cycle. Oscillators with their own step lengths loop independently — the clock is a global reference. Larger values allow longer polyrhythmic patterns.
  
  ### Master Volume & Clip
  Master volume controls the final output level. Clip engages a soft-clipper/limiter before output, recommended when multiple oscillators are running.
  
  ---
  
  ## Dungeon Type (Biome Selection)
  
  The biome is the active sound environment. Each biome contains one or more oscillators with independent settings.
  
  ### Tabs
  Click any tab to switch biomes. Switching while playing crossloads the new biome's oscillators immediately. Built-in biomes:
  
  - Stone Corridors, Ancient Crypt, Twisted Labyrinth, Infernal Depths
  - Flooded Halls, Glacial Vault, Ruined Cathedral, Fetid Swamp, Treasure Vault
  
  ### + new
  Creates a custom biome. Prompts for a name, then adds a tab with a single default oscillator. Custom biomes have an **×** on their tab for deletion (built-in biomes are protected).
  
  ### Biome Name & Colour
  The name field and colour swatch (top of the panel) rename and recolour the current biome. Colour propagates through the oscillator controls, step grids, and tabs.
  
  ### EDIT / TRACKS View
  
  **EDIT** — the full per-oscillator editor, with all sliders and grids expanded.
  
  **TRACKS** — a compact mixer listing. Each row shows:
  - **▲ ▼** reorder buttons (drag order matters for MIDI channel assignment)
  - **m / M** mute toggle
  - Inline label editor
  - Wave type badge
  - Detected note or frequency
  - Mini step pattern (up to 32 steps, read-only, animates during playback)
  - Volume mini-slider
  - **▸ / ▾** expands the full edit card inline beneath the row
  - **×** removes the oscillator (disabled if it's the last one)
  
  ---
  
  ## Key / Scale
  
  Sets the musical key and scale used across all melody grids and the Note selector in oscillator cards.
  
  - **Root note** — tonic of the scale (C, C#, D … B)
  - **Scale** — chosen from 30+ options across Western, Pentatonic, Jazz, World, and Messiaen categories
  
  Notes outside the current scale are flagged amber in melody grids (`*` in the note indicator).
  
  ---
  
  ## Oscillator Editor
  
  Each oscillator in the current biome gets a card in EDIT view. Controls are grouped into sections.
  
  ### Header Row
  - **m / M** — mute this oscillator. Muted oscillators produce no output but continue sequencing
  - **OSC N** — label input for naming the track
  - **Wave selector** — waveform type:
    - `sine` `triangle` `square` `sawtooth` — standard Web Audio types
    - `pulse25` — 25% duty cycle pulse
    - `pulse12` — 12.5% duty cycle pulse
    - `organ` — additive harmonics (1 + 2 + 3 + 4 + 6 + 8)
    - `soft` — gentle additive (1 + 2 + 3)
  - **×** — remove oscillator (disabled if it's the last one)
  
  ### Frequency & Note
  - **Note selector** — picks a note and octave from the current scale; updates the frequency slider
  - **Frequency slider** — fine control from 16 Hz to 4000 Hz
  
  ### Volume
  Peak output level for this oscillator. Actual level is shaped by velocity and ADSR when the sequencer is running.
  
  ### Gate
  Proportion of each step duration that the note is held before releasing. At 1.0 the note is held until the next step fires. At 0.5 the note releases halfway through the step. Interacts with per-step duration (see below).
  
  ### Rate
  Multiplier applied to the step index for this oscillator:
  - **¼×** — steps through one step for every four clock ticks (very slow)
  - **½×** — one step every two ticks
  - **1×** — normal, one step per tick
  - **2×** — two steps per tick (double speed)
  - **4× / 8×** — four or eight steps per tick
  
  Different rates on different oscillators creates polyrhythm.
  
  ### LFO
  A sine-wave LFO modulates the selected target:
  - **pitch** — vibrato (depth in Hz deviation)
  - **volume** — tremolo (depth is amplitude amount)
  - **filter** — filter wobble (depth in Hz deviation on cutoff)
  
  LFO Rate is in Hz (cycles per second). Depth scales with the selected target.
  
  ### ADSR
  Amplitude envelope applied on each note-on/off event:
  - **Attack** — ramp-up time in seconds
  - **Decay** — fall time from peak to sustain level
  - **Sustain** — level held while the step is active (0–1)
  - **Release** — fade-out time after note-off
  
  ADSR is collapsed by default; click the `▸ ADSR` header to expand.
  
  ### Filter
  A biquad filter. Toggle on/off with the **FLT** button. Types: `lowpass` `highpass` `bandpass` `notch` `peaking`. Controls:
  - **Cutoff** — filter frequency (20–8000 Hz)
  - **Q** — resonance (0.1–20)
  
  ### Delay
  A feedback delay line with wet/dry mix:
  - **Time** — delay time in seconds (0–10s)
  - **Feedback** — amount of signal fed back into the delay (0–0.95)
  - **Wet Mix** — blend from dry to delayed signal
  
  ### Step Grid
  The rhythmic trigger pattern for this oscillator. Each button cycles through three states on click:
  - `·` — silent (velocity 0)
  - `½` — half velocity
  - `■` — full velocity
  
  Step count buttons set the pattern length (2 through 32). Patterns of different lengths across oscillators create polyrhythm against the global clock.
  
  **Pattern actions:**
  - **on** — all steps full velocity
  - **off** — all steps silent
  - **inv** — invert (silence becomes active, active becomes silence)
  - **alt** — alternating pattern (step 1 on, step 2 off, etc.)
  - **cpy** — copy this pattern to the clipboard
  - **pst** — paste clipboard pattern (resized to current length)
  
  ### Step Duration Row (`dur:`)
  Appears beneath the step grid. Each button sets the note duration multiplier for that individual step, independently of the global gate setting:
  - `1` — one step (default, shown dim)
  - `2` — hold for two step durations (half note feel at quarter step value)
  - `4` — hold for four step durations (whole note feel)
  - `½` — release after half a step duration (staccato)
  
  Clicking cycles: `1 → 2 → 4 → ½ → 1`. Non-default values are highlighted in the biome colour. This lets you mix note lengths within a single pattern — e.g. a whole note on step 1, two eighth notes on steps 5 and 6.
  
  ### Melody Grid
  A pitch sequencer for this oscillator. Each melody slot fires when its corresponding step is active and has a non-zero velocity.
  
  **Setting up a melody:**
  1. Choose a length (2–32) from the length buttons
  2. Click any slot to cycle through scale notes; shift-click to jump an octave up
  3. Cycling past the last scale note clears the slot (`—`)
  
  **Octave controls** — `oct − +` shift the base octave for new notes (0–7).
  
  **prob row** — per-note probability (0–100%). Click a probability button to set the chance that note fires on each pass. Sub-100% notes appear amber.
  
  **vel row** — per-note velocity override. Cycles: `—` (inherit from step) → `ff` → `f` → `mf` → `p` → `pp`. Colour-coded from green to red.
  
  **clr** — clears the melody entirely.
  
  ---
  
  ## Hecatoncattie (≡ HECA)
  
  A special independent oscillator that persists across biome switches. Designed for a continuous atmospheric undertone or countermelody. Activate with the **HECA** button in the transport row.
  
  Controls are identical to a regular oscillator (wave, freq, volume, gate, rate, LFO, ADSR, delay, step grid, step durations, melody) with the addition of a **Reset to Default** button.
  
  The temperature display in the header (`cold` / `WARM (live)` / `muted`) reflects its current state.
  
  ---
  
  ## Score View
  
  A read-only overview panel showing all active tracks as horizontal step lanes. Updates in real time during playback — the current step highlights across all lanes simultaneously. Melody rows appear beneath their parent step row.
  
  ---
  
  ## MIDI Output
  
  Chrome only (requires MIDI API support). Click **Enable MIDI** and select an output device. CALORIS sends:
  - **MIDI Clock** at 24 pulses-per-quarter-note, derived from BPM and step value
  - **Note On / Off** messages for each oscillator with an active melody (one channel per oscillator, up to 15)
  
  Use the track order in TRACKS view to control channel assignment.
  
  ---
  
  ## Crossfade
  
  Smoothly transitions from the currently playing biome to a target biome over 2.5 seconds. Requires playback to be active. The old oscillators fade out while the new ones fade in; the sequencer continues uninterrupted.
  
  ---
  
  ## Export / Import
  
  ### Export
  Two formats available:
  
  **JSON** — full machine-readable state including all biome configs, Hecatoncattie, sequencer settings, key/scale, and master. Use for saving and reloading sessions.
  
  **JS** — annotated JavaScript constants suitable for embedding in game or generative audio code.
  
  Use **Copy** to copy to clipboard or **Show/Hide** to view in the text area.
  
  ### Import
  Paste a CALORIS JSON export into the Import panel and click **Load Config**. The session replaces all current state. Undo is available immediately after import.
  
  ---
  
  ## Undo
  
  Up to 20 levels of undo for all destructive actions (step edits, pattern operations, oscillator additions/removals, biome resets, imports). Keyboard shortcut: **Ctrl+Z / Cmd+Z**. The undo count is shown on the undo button.
  
  ---
  
  ## Tips & Techniques
  
  **Long drones with occasional rhythmic activity** — set most steps to off, one step to full with a `4` duration, gate near 1.0, long attack and release.
  
  **Polyrhythm** — give oscillator A 6 steps, oscillator B 8 steps. They'll phase against each other over a 24-step cycle. Use the clock value to control the overall cycle length.
  
  **Melodic counterpoint** — activate HECA, give it a different melody length than the main oscillators, and run it at ½× rate for a slow-moving bassline under faster arpeggios.
  
  **Probability for texture** — set melody probabilities to 40–70% for notes that ghost in and out without fully repeating, creating the impression of a living acoustic space.
  
  **Staccato feel** — set gate to ~0.3 and/or mark individual steps with `½` duration for a percussive articulation without changing the overall tempo.
  
  **Crossfade between moods** — build two biomes with similar BPM but different keys/modes. Crossfade during a structural moment. The 2.5s fade is long enough to mask the transition.
  
  **Custom biome from scratch** — use `+ new`, switch to TRACKS view, reorder and tune the oscillators in the compact mixer without losing context, then expand individual tracks to fine-tune.
  
  ---
  
  ## Browser Compatibility
  
  - **Chrome / Chromium** — full support including MIDI
  - **Firefox** — full support, no MIDI
  - **Safari** — full support, no MIDI
  - Works completely offline — no network requests
  
  ---
  
  *CALORIS v3 — Caloris Basin, Mercury · largest confirmed impact basin in the solar system · named "of heat"*
  *Built by Claude (Anthropic) for WARM · 2025*
