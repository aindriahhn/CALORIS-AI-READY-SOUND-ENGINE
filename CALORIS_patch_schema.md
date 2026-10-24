# CALORIS Patch JSON Schema (v14)

Import path: **"◀ Paste" / "◀ File"** in the JSON section ("Paste CALORIS JSON here…").
Contract extracted from `doImport()` in `CALORIS_v14_0_adsa.html`. Feed this file —
not the 295KB app — to a composer (human or agent).

## Top level

`{version, patches, sequencer, key, master, chain, weightedPool, weightedPoolLayer, score, scoreIntent}`

- `version`: number. **Ignored on import** (exporter writes `5`). Informational only.
- `patches`: **REQUIRED.** Object mapping patch-key → patch object. Legacy alias
  `dungeonAudio` is accepted. Missing → import fails loudly (`"Missing patches"`).
  Must contain the currently selected patch key — on a fresh boot that is `init`,
  otherwise the import errors.
- `sequencer`, `key`, `master`: optional; an omitted section leaves current values alone.
- `score`: optional object — the v17 arrangement: tracks of clips layered in
  time (see **The score** below). Handed to `SCORE.importScore` on import;
  an absent `score` leaves the current arrangement untouched.
- `scoreIntent`: optional object — the composer's statement of intent. **Never
  read by the importer** (verified: `doImport` only touches `patches`,
  `sequencer`, `key`, `master`; every other top-level field is silently
  ignored). Required by composer convention — see its own section below. A
  submission without a coherent `scoreIntent` is a chain, not a score.
- Unknown top-level fields are silently ignored.

## sequencer — all fields optional, applied only when present

- `bpm`: number (UI offers 20–300; import does not validate).
- `stepCount`: steps per bar (UI offers 2–14, 16, 24, 32; import accepts any).
- `swing`: ms added to odd steps, 0–80. Applied when `!== undefined`.
- `stepValueKey`: one of `w h q qd tq e ed te s ts` (whole → triplet-16th).
  Unknown keys are silently ignored.
- `stepValueMult`: **IGNORED on import** (re-derived from `stepValueKey`). Do not write it.

## key

- `rootNote`: default `'C'`. Must be one of `C C# D D# E F F# G G# A A# B`
  (sharps only); anything else corrupts the derived scale notes.
- `scaleKey`: default `'chromatic'`; unknown keys fall back to chromatic. Valid:
  `chromatic major minor dorian phrygian lydian mixolydian locrian blues
  diminished augmented persian arabic hirajoshi insen kumoi`
- `scaleName`, `notes`: **IGNORED** (re-derived). Do not write them.

## master

- `volume`: default `0.8`. **Trap:** explicit `0` coerces to `0.8` (`||` fallback).
- `clipEnabled`: default `true`; only explicit `false` disables.
- `reverbSize`, `reverbDecay`: applied when truthy.
- `reverbWetMaster`: applied when `!== undefined`.

## chain — ordered arrangement (v15.8+)

- `chain`: optional array of `{key, bars, trans}`. Imported into the ordered
  arrangement chain (`S.chainList`); `S.chainPos` resets to 0.
- `key`: must match a patch key in `patches`; entries with unknown keys are
  silently dropped.
- `bars`: bars to play before advancing (>=1; coerced to at least 1).
- `trans`: `'fade'` or `'cut'`; anything else coerces to `'fade'`.
- Omitted or empty → existing in-memory chain is left unchanged.

## weightedPool — generative pool (v15.8+)

- `weightedPool`: optional array of `{key, weight}`. Imported into the weighted
  pool; entries with unknown keys are silently dropped.
- `weight`: selection weight (>=1; coerced to at least 1). Higher = more likely.
- `weightedPoolLayer`: optional 1–4; which pool layer to load into.
  Out-of-range values are ignored.
- Omitted → existing in-memory pool is left unchanged.

## scoreIntent — the composer's statement of intent

Structure an AI (or human) can grok at a glance. Write it **before** you
write a single patch.

- Optional object. **The importer never reads it** — it is invisible to
  playback and cannot break an import. It exists for the composer and
  the reviewer.
- By composer convention, every submission MUST include it, and every
  field MUST be answered in concrete musical terms. A `scoreIntent`
  that restates the chain ("arc: intro, verse, chorus, outro") is a
  failed `scoreIntent`.
- Fields (all strings, except `arc` which is an array of strings):
  - `arc`: 3–7 stage words naming the energy curve, e.g.
    `["low", "build", "peak", "break", "resolve"]`. Each stage should
    map to a patch or chain entry.
  - `motive`: the recurring musical idea, and how it changes across the
    piece — stated where, developed where, changed how.
  - `tension`: where expectation builds, and by what means (density,
    repetition, delay, register).
  - `release`: where it breaks, and how (`cut`, strip-back, return).
  - `silence`: what the quiet moments mean — which rests are
    compositional, not accidental.
  - `loop`: how the end composes back into the beginning — or why the
    hard restart is deliberate.
- Worked example:

```json
"scoreIntent": {
  "arc": ["low", "build", "peak", "return", "dissolve"],
  "motive": "4-note E-E-G-A cell at E2 in the verse; stretched to an 8-note answer an octave up in the chorus",
  "tension": "verse holds on the obsessive 4-note cell with driving hats; the chorus is delayed a full cycle",
  "release": "hard cut into the chorus — the snare backbeat arrives for the first time",
  "silence": "outro notes at prob 60-90 so the motive dissolves instead of ending; the resonator lets each note evaporate",
  "loop": "the thinning texture fades straight back into the sparse verse — the piece breathes in a circle"
}
```

- Review rule: if you cannot tell the arc, the motive, and the release
  apart from the chain order alone, the piece is a chain. Send it back.

## Patch object

- Key: arbitrary string. Include an `init` patch (fresh boot selects it).
- `type`: `'synth'` | `'kit'`. Missing → synth routing (a kit's drums lose the drum bus).
- `name`: tab label. `color`: CSS color for the tab.
- `oscillators`: **REQUIRED** array. Missing → patch is dropped. Empty array
  or an array with no plain-object voices → patch is dropped too (counted in
  the "N invalid patches skipped" status; if every patch drops, the import
  errors with "No valid patches").

## Oscillator — required fields

The import normalizes every patch through `_normalizePatch`, which filters the
`oscillators` array to plain objects and runs each survivor through `makeOsc` —
so most fields are optional and fall back to sane defaults. The only
import-level failure is a patch whose `oscillators` array is missing, or
contains zero valid voice objects after filtering.

Because `makeOsc` supplies defaults, no individual voice field can fail the
import. Defaults applied to a voice whose field is missing or invalid:

- `wave`: `'square'`. Unknown wave strings fall back to a sine at playback
  (see wave table below). Missing `wave` is fine.
- `freq`: `220`. Applied if non-finite or `<= 0`. Noise/drum voices ignore it.
- `volume`: `.04`. Applied if non-finite or `< 0`.
- `pan`: `0` if non-finite.
- `steps`, `stepProbs`, `stepAccents`, `stepDurs`, `melody`, `harmonics`,
  `harmonicRatios`, `harmonicPhases`: coerced to `[]` if not arrays.
  `harmonics` defaults to a single-fundamental `[1, 0, …, 0]`;
  `harmonicRatios` to `[1, 2, …, 16]`; `harmonicPhases` to all zeros.
  `steps` is auto-resized to `stepCount` at render/playback.
- `midiNote`, `midiChannel`: `null` if non-finite.
- `sampleBuffer`: stripped (can't arrive via JSON anyway).

## Wave table

- `sine square sawtooth triangle` — native oscillator types.
- `pulse25 pulse12 pulse10 organ organ2 strings brass clarinet flute soft` —
  periodic-wave tables.
- `custom` — built from `harmonics[16]`, `harmonicRatios[16]`, `harmonicPhases[16]`;
  non-integer ratios produce an inharmonic partial bank.
- `noise-white noise-pink` — looped noise buffer (`freq` ignored).
- `drum-kick drum-snare drum-hat drum-cymbal drum-tom drum-clap drum-rim` —
  one-shot drum synth. Drum-shaping fields (`drumTone`, `drumToneEnd`,
  `drumToneTime`, `drumNoise`, `drumNoiseFreq`, `drumNoiseQ`) fall back to
  built-in drum presets when omitted.
- `sample` — needs a decoded `sampleBuffer`, which cannot arrive via JSON. Do not use.
- Unknown wave string: silently falls back to a default sine. No crash.

## Oscillator — sequencer fields (all optional)

- `steps`: gate array (`0` rest, nonzero hit). Auto-resized to `stepCount`.
- `stepProbs`: per-step 0–100 (default 100). `stepAccents`: per-step velocity
  multiplier (default 1). `stepDurs`: per-step duration multiplier (default 1).
- `stepLocks`: `{stepIndex: {filterFreq?, filterQ?, pan?, pitch?}}`, pitch in
  semitones. `null`/`{}` = off.
- `melody`: array of `{note, octave, vel?, prob?}` or `null`s, read at
  `step % length`. `note` ∈ sharps list. `vel` 0–1, `prob` 0–100.
- `melodyOctave`: default 3 (editor hint; playback uses each slot's `octave`).
- `gateTime`: note length as fraction of step (default 1).
- `rateMult`: polymeter multiplier (default 1).
- `muted`: bool. `label`: string.
- Kit voices (`type: 'kit'` patches only): `stepData` = per-step
  `{on:0/1, vel:1, pitch:0, decay:1}`. Auto-created when missing — omit it
  entirely. A `score` is not a kit (its virtual patch is `type: 'synth'`),
  so drums in a score never use `stepData` — see **The score**.

## Oscillator — sound fields (all optional, safe `||` fallbacks)

- ADSR: `attack` .005, `decay` .1, `sustain` .8, `release` .15.
- `pan` 0 (−1..1). `gainTrim` dB (−24..12).
- Filter (`filterEnabled`): `filterType` lowpass|highpass|bandpass|notch|peaking,
  `filterFreq` 800, `filterQ` 1.
- Delay: `delayTime`, `delayFeedback`, `delayWet` (all 0 = off). `reverbSend` 0.
- Chorus (`chorusEnabled`): `chorusType` chorus|flanger, `chorusRate` .8,
  `chorusDepth` 15, `chorusFeedback` 0, `chorusWet` .5.
- Distortion (`distEnabled`): `distType` soft|hard|foldback, `distDrive` 50, `distWet` 1.
- Unison: `unisonCount` 1 (1–8), `unisonDetune` 20.
- LFO: `lfoRate` .1, `lfoDepth` 2, `lfoTarget` pitch|volume|filter;
  `lfo2Rate` .3, `lfo2Depth` 0, `lfo2Target` pitch|volume|filter|pan.
- FM (`fmEnabled` and `fmDepth` > 0): `fmRatio` 2, `fmDepth` in Hz.
- `glide`: portamento seconds (0 = off).

## Import failure modes

**Loud** (status shows `Error: …`; prior patches kept, undo available):
invalid JSON; missing `patches`; `patches` lacking the selected key; a patch
lacking an `oscillators` array; a file where every patch is dropped by
normalization ("No valid patches in file").
**Silent**: everything else — unknown fields, keys, waves, scales, and any
individual voice field coerce to defaults. A patch is quietly skipped if its
`oscillators` array is empty or contains no plain-object voices. Write minimal
patches; the engine fills the rest.

## Worked example (valid — imports and renders as-is)

```json
{
  "version": 5,
  "patches": {
    "init": {
      "type": "synth",
      "name": "Schema Demo",
      "color": "#00ff00",
      "oscillators": [
        {
          "wave": "sine", "freq": 110, "volume": 0.05,
          "steps": [1,0,0,0,1,0,0,0],
          "attack": 0.01, "decay": 0.2, "sustain": 0.7, "release": 0.3
        },
        {
          "wave": "triangle", "freq": 220, "volume": 0.03,
          "steps": [1,1,1,1,1,1,1,1],
          "melody": [
            {"note": "A", "octave": 3},
            {"note": "C", "octave": 4},
            {"note": "E", "octave": 4},
            {"note": "G", "octave": 3}
          ],
          "reverbSend": 0.5
        }
      ]
    }
  },
  "sequencer": {"bpm": 60, "stepCount": 8, "swing": 0, "stepValueKey": "e"},
  "key": {"rootNote": "A", "scaleKey": "dorian"},
  "master": {"volume": 0.8, "clipEnabled": true, "reverbSize": 2.5, "reverbDecay": 3.5, "reverbWetMaster": 0.7}
}
```

Two voices: a gated A1 sine pulse and a triangle lead cycling Am7. Paste it,
press Play.

- `arpPool`: `'all'` (default) or `'bar'`. With `'bar'` the arp pool is only the gated melody notes in the
current bar (stepCount steps), so the arpeggio follows chord changes written into `melody`.
- `sustain`: 0 is honoured (pluck/decay-only voices); previously it fell back to .8.

## Resonator (per oscillator, optional)

`"resonance": {"enabled": true, "type": "string"|"bar"|"membrane", "damping": 0.5, "feedback": 0.97, "mix": 0.5, "trackPitch": true}`

A Karplus-Strong waveguide loop after the envelope, so it rings after note-off. `damping` 0..1 (higher = darker, shorter highs), `feedback` 0..0.995 (ring time, defined as per-period gain at 220 Hz and scaled with pitch so every note rings equally long), `mix` 0..1 (`dryWet` is accepted as an alias). Works on pitched waves and noise (noise + melody = plucks). Not applied to drum or sample voices. An AudioWorklet runs a true single-period loop at any pitch; if it can't load, a delay-loop fallback (n periods above ~345 Hz) is used.

## The score (arrangement)

The v17 arrangement layer. Where a `chain` plays patches one after another,
a score layers **tracks** that play simultaneously (verified against the
SCORE system and `renderScoreToWAV` in `CALORIS_v17_11_5_bench_1.html`).

```json
"score": {
  "loop": false,
  "endBar": null,
  "tracks": [
    {
      "name": "Bass", "color": "#0f0", "muted": false, "solo": false,
      "timbre": {"patchKey": "init", "oscIdx": 0},
      "clips": [
        {"id": "c1", "loop": {"patchKey": "init", "oscIdx": 1},
         "startBar": 0, "bars": 4, "fadeIn": 0, "fadeOut": 1}
      ]
    }
  ]
}
```

- **References are by `patchKey` + `oscIdx`, never by value.** A track's
  `timbre` names the oscillator whose sound it plays; a clip's `loop` names
  the oscillator whose pattern it plays. Both resolve as
  `patches[patchKey].oscillators[oscIdx]` in the same file. On import the
  score is sanitized: tracks whose `timbre` does not resolve, and clips
  whose `loop` does not resolve, are dropped. At most 24 tracks survive.
- **Timbre vs pattern.** The track voice takes its sound (wave, ADSR,
  filter, effects, base `volume`) from the `timbre` oscillator. While a
  clip is active, the **pattern keys** are copied from the clip's
  oscillator: `steps`, `stepData`, `stepDurs`, `stepProbs`, `stepAccents`,
  `stepLocks`, `melody`, `melodyOctave`, `gateTime`, `rateMult`, `freq`,
  and the `arp*` fields. Note `freq` is a pattern key: a clip can carry
  its own pitch base. Everything not in that list stays the timbre's.
- **Clips.** `startBar` (integer ≥ 0) and `bars` (integer ≥ 1) place the
  clip on the bar grid; `fadeIn` / `fadeOut` are in bars and scale the
  track volume linearly (0 = hard edge). Where clips overlap on a track,
  the one with the latest `startBar` wins. Where no clip covers a
  position, the track is **silent**: gaps are drop-outs, not holds.
- **Phase follows the global step counter, not the clip start.** A
  pattern is read at `floor(absStep × rateMult) % patternLength`, where
  `absStep` is the monotonic global step (`bar × stepCount + step`)
  running from the score's start. A clip boundary swaps *which* pattern
  is read, never the counter, so a clip that starts mid-pattern enters
  the pattern mid-cycle. Corollary: a pattern of length L sits at its
  own step 0 under a clip starting at bar B only when
  `(B × stepCount) % L == 0`. At the usual 16 steps per bar, **64-step
  loops must start on multiples of 4 bars** (bars 0, 4, 8, …) or they
  enter rotated.
- **End and loop.** `endBar` > 0 cuts everything past it; null = the
  score ends after its last clip. With `loop: true` the global counter
  wraps at `endBars × stepCount`, which is the only phase reset there is.
- **Drums in a score use `steps` and `stepAccents`, not kit `stepData`.**
  Score voices form a virtual `type: 'synth'` patch, so a drum-wave
  voice in a clip plays like a synth-path drum: gates from `steps`,
  probability from `stepProbs`, velocity scaled by `stepAccents`.
  `stepData` (with its per-step pitch/decay) belongs to `type: 'kit'`
  patches alone. Author score drums accordingly: pattern in `steps`,
  dynamics in `stepAccents`.

## Scoring a piece

Everything above is the field reference — how to build sounds. This section
is compositional guidance: how to use those fields to write an actual piece
of music. Nothing here adds new spec; it builds on the `chain`,
`weightedPool`, `key`, `melody`, and `steps` fields documented above.

The scoring half of this document is a contract, not advice. Before writing
any patch, write the `scoreIntent` block (documented above). If you cannot
fill it in concrete musical terms, you do not have a score yet — keep
thinking. The checklist at the end is a set of requirements, not tips.

### The three layers

A CALORIS score is built in three layers:

1. **Voices** (oscillators) — individual instruments. Each has its own
   rhythm (`steps`), melody (`melody`), and sound design.
2. **Patches** — groups of voices that play together. Think of a patch as
   one section of your arrangement: "verse" (drums + bass), "chorus"
   (drums + bass + lead), "bridge" (pads only).
3. **Structure** (`chain` / `weightedPool`) — the order patches play in.
   `chain` is a fixed arrangement: intro → verse → chorus → outro.
   `weightedPool` is generative: the engine picks the next patch by weight,
   so the piece is different every playthrough. A `score` (see **The
   score**) is the third shape: tracks layer simultaneously instead of
   taking turns.

### Fixed arrangement with `chain`

Each entry is `{key, bars, trans}`:

```json
"chain": [
  {"key": "intro",  "bars": 4, "trans": "fade"},
  {"key": "verse",  "bars": 8, "trans": "fade"},
  {"key": "chorus", "bars": 8, "trans": "cut"},
  {"key": "verse",  "bars": 8, "trans": "fade"},
  {"key": "outro",  "bars": 4, "trans": "fade"}
]
```

- `key` selects which patch plays. It must match a patch key in `patches`.
- `bars` is how many bars the patch plays before the chain advances.
- `trans` is the handoff character: `'fade'` crossfades into the next patch,
  `'cut'` switches hard. Use `'fade'` for smooth sectional flow, `'cut'`
  for dramatic drops, hits, or genre switches.

The chain loops from the top when it reaches the end. `S.chainPos` resets
to 0 on import, so playback always starts at the first entry.

### Generative arrangement with `weightedPool`

Instead of a fixed order, assign each patch a weight and let the engine
choose what comes next:

```json
"weightedPool": [
  {"key": "verse",  "weight": 3},
  {"key": "chorus", "weight": 2},
  {"key": "bridge", "weight": 1}
]
```

Higher weight = more likely to be selected. Good for ambient pieces,
installations, or game music where you want variety without writing a
fixed arrangement. The `verse` above plays roughly three times as often
as the `bridge`.

### Keeping patches in the same key

The `key` section (`rootNote` + `scaleKey`) is imported once and applies
globally. Every patch in your `chain` or `weightedPool` shares it. Write
all your patches' melodies against the same root and scale, or the piece
will sound like it's modulating at every section change. If you want a
genuine key change mid-piece, that currently requires a second import.

### Writing melodies that fit together

- Each voice's `melody` array is read cyclically at `step % length`. A
  4-note melody over a 16-step bar repeats four times per bar; a 16-note
  melody plays once per bar. Longer arrays = slower harmonic rhythm.
- `vel` (0–1) shapes per-note dynamics; `prob` (0–100) makes individual
  notes probabilistic — the melody stays recognizable but breathes.
- Keep voices in separate octave ranges (bass at octave 1–2, chords at
  2–3, lead at 3–5) so they don't mask each other in the mix.
- `stepProbs` on the rhythm layer adds human-feel timing variation without
  changing the written melody.
- With `arpPool: 'bar'`, the arpeggiator follows chord changes you write
  into `melody` — so a patch whose melody moves Am → F → C → G will
  arpeggiate each chord in turn.

### A chain is not a score

A chain orders patches. A score gives them an arc: where the energy
starts, how it rises, what breaks it, where it lands. If your piece is
"intro → verse → chorus → verse → outro" but every section sits at the
same density, register, and intensity, you have written a chain. The
rest of this section is about intent — the difference between ordering
patches and composing a piece.

### Shaping energy over time

Before writing any patch, sketch an energy curve — even roughly:
"low → building → peak → break → resolve". Then assign patches to
curve points, not the other way around. The levers you have:

- **Density**: how many steps fire. A voice on `[1,0,0,0,…]` versus
  `[1,1,1,1,…]` is a different energy at the same tempo.
- **Register**: a lead dropping an octave reads as weight; rising an
  octave reads as lift. Same melody, different octave, different
  meaning.
- **Voice count**: a patch with 2 voices versus 6 is a crescendo you
  didn't need automation for.
- **Harmonic rhythm**: longer `melody` arrays = slower chord change =
  calmer. A 4-note cell repeats four times per 16-step bar; a 16-note
  melody plays once per bar. Speed of change is a mood.
- **Dynamics and brightness**: `vel`, `stepAccents`, and `filterFreq`
  on a filtered lead — a brighter patch feels louder at the same
  volume.

### Tension and release

Tension is sustained expectation. Build it by: raising density without
resolving (driving 16ths on the hats), holding one patch past its
comfort point (8 bars where 4 would do), narrowing a melody to a
repeated cell (obsessive repetition builds pressure), delaying the
section the listener is waiting for. Release it with: the `cut` into
the chorus, a strip-back to a single voice, a held open fifth with a
long `release`. Every score needs at least one moment where something
the listener was waiting for actually arrives — and one where it
deliberately doesn't.

### Motives: say it, then say it differently

A melody heard once is decoration; heard twice with a change, it is a
motive. Develop it by: moving the same `melody` array to a different
voice in a different octave (call and response across patches),
thinning it (same notes, sparser `steps`), changing one note (the
changed note is the emotional event), or stretching it (a 4-note cell
answered by an 8-note line). The `melody` field is cyclic, so a 4-note
cell over 16 steps repeats four times per bar. Use the repetition,
then break it: the break is what makes the repetition mean something.

### Silence is an instrument

Rests are compositional. `stepProbs` below 100 on a sparse voice makes
it breathe instead of march. A patch with one voice and long rests
reads as a held breath between dense sections. Do not fill every step:
the contrast between full and empty is what makes loud feel loud. The
`cut` transition is the loudest silence available — a hard stop before
a section hits makes the hit land twice as hard.

### Pacing grammar: cuts, fades, and phrase lengths

- `'fade'`: continuity — the same idea deepening. Use between
  variations of a section (verse → verse-with-lead).
- `'cut'`: drama — arrival, rupture, the piece changing its mind. Use
  for drops, section hits, or genre switches. A `cut` that follows a
  sparse bar hits twice as hard.
- `bars` as phrasing: keep patch lengths in musical phrases (2, 4, 8
  bars). Odd lengths (3, 5, 7) create deliberate unease — use sparingly,
  on purpose, never by accident.
- The loop point: the chain loops. The last entry's transition back
  into the first is the most-heard moment in the piece. Compose it: an
  outro that thins out and fades into the intro's key and register, or
  a deliberate hard `cut` that reboots the piece from silence.

### Dynamics without a volume knob

There is no per-voice volume automation, so shape dynamics through:
`vel` and `prob` per melody slot; `stepAccents` for stress patterns;
thinning or thickening `steps`; adding or removing voices between
patches (a verse with 3 voices into a chorus with 6 is a crescendo);
brightness swells via `filterFreq` on a filtered lead.

### The prob field is phrasing

`prob` below 100 on melody slots and `stepProbs` below 100 on rhythm
layers are not randomness for its own sake — they are rubato. A lead
with every note at `prob` 60–90 breathes; the listener hears intention
in the gaps, not noise. Reserve `prob: 100` for the notes that must
land: downbeats, the head of the motive, the arrival.

### Weighted pools want shape too

A generative pool without shape is a slot machine. Shape it: weight
toward home-base patches (a verse at weight 3 is home), keep rare
patches genuinely rare (a weight-1 "thunder" patch is an event, not a
section), and consider separate layers (`weightedPoolLayer`) for
independent strata — drones on layer 1, events on layer 2 — so the
generative texture has depth instead of soup. The pool picks what
plays; you still decide the emotional vocabulary it picks from.

### Scored example (valid — three patches, fixed arrangement)

```json
{
  "version": 5,
  "patches": {
    "init": {
      "type": "synth", "name": "Verse", "color": "#4488ff",
      "oscillators": [
        {
          "wave": "sine", "freq": 82.4, "volume": 0.06,
          "steps": [1,0,0,0, 1,0,0,0, 1,0,0,0, 1,0,0,1],
          "melody": [
            {"note": "E", "octave": 2}, {"note": "E", "octave": 2},
            {"note": "G", "octave": 2}, {"note": "A", "octave": 2}
          ],
          "attack": 0.01, "decay": 0.15, "sustain": 0.6, "release": 0.2
        },
        {
          "wave": "drum-kick", "volume": 0.09,
          "steps": [1,0,0,0, 1,0,0,0, 1,0,0,0, 1,0,0,0]
        },
        {
          "wave": "drum-hat", "volume": 0.03,
          "steps": [0,0,1,0, 0,0,1,0, 0,0,1,0, 0,0,1,1]
        }
      ]
    },
    "chorus": {
      "type": "synth", "name": "Chorus", "color": "#ff6644",
      "oscillators": [
        {
          "wave": "sawtooth", "freq": 164.8, "volume": 0.04,
          "steps": [1,1,1,1, 1,1,1,1, 1,1,1,1, 1,1,1,1],
          "melody": [
            {"note": "E", "octave": 3}, {"note": "G", "octave": 3},
            {"note": "B", "octave": 3}, {"note": "D", "octave": 4},
            {"note": "C", "octave": 4}, {"note": "B", "octave": 3},
            {"note": "G", "octave": 3}, {"note": "A", "octave": 3}
          ],
          "filterEnabled": true, "filterType": "lowpass",
          "filterFreq": 1200, "filterQ": 2,
          "attack": 0.02, "decay": 0.1, "sustain": 0.7, "release": 0.25
        },
        {
          "wave": "drum-kick", "volume": 0.09,
          "steps": [1,0,0,0, 1,0,0,0, 1,0,0,0, 1,0,0,0]
        },
        {
          "wave": "drum-snare", "volume": 0.07,
          "steps": [0,0,0,0, 1,0,0,0, 0,0,0,0, 1,0,0,0]
        },
        {
          "wave": "drum-hat", "volume": 0.03,
          "steps": [0,0,1,0, 0,0,1,0, 0,0,1,0, 0,0,1,0]
        }
      ]
    },
    "outro": {
      "type": "synth", "name": "Outro", "color": "#44ff88",
      "oscillators": [
        {
          "wave": "triangle", "freq": 220, "volume": 0.04,
          "steps": [1,0,0,0, 0,0,0,0, 1,0,0,0, 0,0,0,0],
          "melody": [
            {"note": "E", "octave": 4, "prob": 80},
            {"note": "D", "octave": 4, "prob": 60},
            {"note": "B", "octave": 3, "prob": 70},
            {"note": "A", "octave": 3, "prob": 90}
          ],
          "reverbSend": 0.6,
          "attack": 0.05, "decay": 0.3, "sustain": 0.5, "release": 0.8,
          "resonance": {"enabled": true, "type": "string", "damping": 0.4, "feedback": 0.97, "mix": 0.4}
        }
      ]
    }
  },
  "chain": [
    {"key": "init",   "bars": 4, "trans": "fade"},
    {"key": "chorus", "bars": 4, "trans": "cut"},
    {"key": "init",   "bars": 4, "trans": "fade"},
    {"key": "outro",  "bars": 4, "trans": "fade"}
  ],
  "sequencer": {"bpm": 90, "stepCount": 16, "swing": 0, "stepValueKey": "s"},
  "key": {"rootNote": "E", "scaleKey": "minor"},
  "master": {"volume": 0.8, "clipEnabled": true, "reverbSize": 3, "reverbDecay": 4, "reverbWetMaster": 0.6}
}
```

Three patches, one key (E minor), one chain: a sparse verse (bass + kick +
hats) cuts hard into a full chorus (saw lead + snare), falls back to the
verse, then fades into a probabilistic resonator outro. Paste it, press Play.

### Why the example above is a score, not a chain

Read it again as intent, not fields:

- **Verse** (low energy): sparse bass at E2, kick on quarters, hats
  chattering quietly. The 4-note E–E–G–A cell is the motive, and it
  repeats — obsessively. This is the "low → building" stretch of the
  curve.
- **Chorus** (rupture): the `cut` is the compositional event. The snare
  backbeat arrives for the first time, and the saw lead sits a full
  octave above the bass, stretching the motive into an 8-note answer
  (E–G–B–D–C–B–G–A) — the same E–G–A cell, developed. Motive stated,
  then changed. This is the peak.
- **Verse return** (familiarity): after the peak, the same sparse
  texture reads as relief. Same patch, new meaning — tension released
  by return, not by resolution.
- **Outro** (dying embers): one voice, triangle, heavy reverb, notes at
  `prob` 60–90 so the motive dissolves instead of ending. The resonator
  lets each note ring after note-off — the piece doesn't stop, it
  evaporates. And the loop point: this thinning texture fades straight
  back into the sparse verse. The piece breathes in a circle.

Every choice above serves the arc. That is the whole job.

### Composer checklist

Before you call a piece done:

- Does it have an energy arc, or just an order?
- Is there one moment the listener waits for — and does it arrive?
- Is there a motive that returns changed?
- Is there a silence that means something?
- Does the loop point compose, or just restart?
- Could you remove one voice and lose nothing? Then remove it.
