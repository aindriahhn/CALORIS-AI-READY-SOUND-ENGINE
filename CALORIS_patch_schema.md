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
- `stepCount`: steps per bar for bar 0 and the default for any bar not
  covered by `meterChanges` (UI offers 2–14, 16, 24, 32; import accepts any,
  clamped to 1–512 in v19, 1–64 in v18 and earlier).
- `swing`: ms added to odd steps, 0–80. Applied when `!== undefined`.
- `stepValueKey`: one of `w h q qd tq e ed te s ts f` (whole → triplet-16th,
  plus v19 `f`, the fine grid — see **Fine grid (v19)** below).
  Unknown keys are silently ignored.
- `stepValueMult`: **IGNORED on import** (re-derived from `stepValueKey`). Do not write it.
- `meterChanges` (v18): optional array of time-signature changes, one
  entry per change: `{"bar": 0, "num": 4, "den": 4, "steps": 16}`. See
  **Meter changes (v18)** below for the full contract.

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
  `step % length`. `note` ∈ sharps list; flat spellings (`Db`, `Eb`, `Gb`,
  `Ab`, `Bb`, also `Cb`/`Fb`/`E#`/`B#`) are accepted and read as their sharp
  equivalents, normalised to sharps on import. The same applies to
  `key.rootNote`. `vel` 0–1, `prob` 0–100.
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
  `absStep` is the monotonic global step running from the score's
  start. With a uniform meter that step is `bar × stepCount + step`;
  with meter changes (v18) it is `meterStepForBar(bar) + step` — see
  **Meter changes (v18)**. A clip boundary swaps *which* pattern
  is read, never the counter, so a clip that starts mid-pattern enters
  the pattern mid-cycle. Corollary: a pattern of length L sits at its
  own step 0 under a clip starting at bar B only when
  `(meterStepForBar(B)) % L == 0`. At the usual 16 steps per bar, **64-step
  loops must start on multiples of 4 bars** (bars 0, 4, 8, …) or they
  enter rotated.
- **End and loop.** `endBar` > 0 cuts everything past it; null = the
  score ends after its last clip. With `loop: true` the global counter
  wraps at the meter total for the end bars (`meterTotalSteps(endBars)`;
  with a uniform meter that is `endBars × stepCount`), which is the
  only phase reset there is.
- **Drums in a score use `steps` and `stepAccents`, not kit `stepData`.**
  Score voices form a virtual `type: 'synth'` patch, so a drum-wave
  voice in a clip plays like a synth-path drum: gates from `steps`,
  probability from `stepProbs`, velocity scaled by `stepAccents`.
  `stepData` (with its per-step pitch/decay) belongs to `type: 'kit'`
  patches alone. Author score drums accordingly: pattern in `steps`,
  dynamics in `stepAccents`.

## Meter changes (v18)

Before v18 there was no meter at all: one global `stepCount` (steps per
bar) and one global `stepValueKey` (duration per step), and every bar
was assumed to be `stepCount` steps. v18 adds a meter timeline to the
sequencer, and bar boundaries — in live patch playback, in SCORE
playback and rendering, in chain timing, in the sequencer info line,
and in MIDI export — come from that timeline instead of from
`bar × stepCount`.

- **Field format.** `sequencer.meterChanges` is an array of entries,
  sorted by bar, each `{"bar": <0-based bar>, "num": <numerator>,
  "den": <denominator>, "steps": <steps per bar>}`. `bar` is 0-based:
  the first bar of the piece is bar 0, and a change at `"bar": 2`
  takes effect from the third bar onward.
- **How steps are computed.** A bar lasts `num/den` whole notes, which
  is `num × 4 / den` quarter notes. Divide by quarters per step — the
  `stepValueMult` for the current `stepValueKey` (`s` = 0.25, `e` = 0.5,
  `q` = 1, `w` = 4, `f` = 1/48, and so on) — and round to an integer, clamped to
  1–512 in v19 (1–64 in v18 and earlier). So 4/4 at sixteenths is `4 × 4 / 4 / 0.25 = 16` steps, 3/4 at
  sixteenths is `3 × 4 / 4 / 0.25 = 12` steps, and 6/8 at eighths is
  `6 × 4 / 8 / 0.5 = 6` steps. If an entry gives only `steps`, that
  value is used directly; if it gives `num`/`den`, the computed value
  wins on import. `num`/`den` are kept alongside `steps` in the file
  because the UI shows them and MIDI time-signature events need them;
  after import, stored `steps` are what playback uses.
- **Bar 0 default.** Normalisation always inserts an entry at bar 0
  if one is missing, with `steps` equal to the current `stepCount`.
  `stepCount` itself remains the bar-0 / default bar length: changing
  it in the UI updates the bar-0 meter entry, and importing a file
  whose bar-0 meter entry disagrees with its `stepCount` resolves in
  favour of `stepCount` for bar 0.
- **Normalisation on import.** Entries are sorted by bar; entries
  sharing a bar collapse to the last one in the file; entries with no
  usable `num`/`den` and no finite `steps` are dropped; every surviving
  `steps` is rounded and clamped to 1–512 in v19 (1–64 in v18 and earlier). The helpers are named
  `meterStepsAtBar(bar)`, `meterStepForBar(bar)` (steps before bar),
  `meterBarForAbsStep(absStep)`, and `meterTotalSteps(bars)`.
- **Files without `meterChanges` behave exactly as before.** Importing
  a file with no `meterChanges` resets the timeline to a single bar-0
  entry equal to `stepCount`, so every bar is `stepCount` steps and all
  the pre-v18 behaviour (including the score phase rule above, with
  `meterStepForBar(bar) = bar × stepCount`) is unchanged. This was
  verified against the Greensleeves and Jerusalem patch and score
  files: all four import clean into v18 and their simulated note
  sequences are identical to v17.12.6 (Greensleeves 72/32/32 gates,
  Jerusalem 133/74/74).
- **What meter does and does not move.** Only bar boundaries move.
  Patterns stay step-indexed: a pattern is still read at
  `floor(absStep × rateMult) % patternLength`, with `absStep` the
  monotonic global step. A meter change mid-pattern does not restart
  or re-phase the pattern; it only changes where the next bar starts.
  A score clip starting at a bar where the meter changes starts at
  that bar's meter offset, not at `startBar × stepCount`. A meter
  change after the last clip does not affect the loop length, because
  the loop total is `meterTotalSteps(endBars)` over the bars the
  score actually covers. A clip spanning a meter change counts its
  bars in bars, and its steps come from the meter map bar by bar.
  Chain entries advance by bars, and their durations are the meter
  steps of the bars they cover, not `bars × stepCount`.
- **Export.** `getExportJSON` writes `sequencer.meterChanges` only when
  it differs from a single bar-0 entry equal to `stepCount`; a uniform
  meter is therefore omitted, exactly as a file without the field
  would read. The sequencer panel has a Meter section (bar, numerator,
  denominator, computed steps per row, add/remove) that rebuilds on
  every render, so an imported timeline shows up in the panel.
- **Worked example — a piece that changes meter mid-score.** Two
  bars, sixteenths (`stepValueKey: "s"`), 300 bpm in the test file:
  bar 0 in 4/4 (16 steps), bar 1 in 3/4 (12 steps), 28 steps total.
  One lead track, one clip per bar, both looping the same 28-step
  pattern (melody C–D–E–F–G–A–B cycling):

```json
"sequencer": {"bpm": 300, "stepCount": 16, "swing": 0, "stepValueKey": "s",
  "meterChanges": [{"bar": 0, "num": 4, "den": 4, "steps": 16},
                   {"bar": 1, "num": 3, "den": 4, "steps": 12}]},
"score": {"loop": true, "endBar": null, "tracks": [{
  "name": "Lead", "timbre": {"patchKey": "meterpat", "oscIdx": 0},
  "clips": [{"id": "c1", "loop": {"patchKey": "meterpat", "oscIdx": 0},
             "startBar": 0, "bars": 1, "fadeIn": 0, "fadeOut": 0},
            {"id": "c2", "loop": {"patchKey": "meterpat", "oscIdx": 0},
             "startBar": 1, "bars": 1, "fadeIn": 0, "fadeOut": 0}]}]}
```

  Verified in v18: the file imports clean; `meterStepsAtBar` gives
  16, 12, 12…; `meterStepForBar` gives 0, 16, 28; `meterTotalSteps(2)`
  is 28; simulated score playback gives 28 gates, with bar 1 starting
  at absolute step 16 on E4 (the fifth note of the cycle); live SCORE
  playback passes absolute step 28 and keeps playing with no page
  errors; and its score MIDI export carries time-signature events at
  tick 0 (4/4) and tick 1920 (3/4, i.e. 16 steps × 120 ticks per
  sixteenth at tpq 480), with end-of-track at tick 3360 (28 × 120).
  A four-bar version of the same idea (4/4 for two bars, then 3/4 for
  two bars: 16, 16, 12, 12 = 56 steps) was verified the same way,
  including a clip spanning the change and a playhead that wraps
  from bar 4 back to bar 0.

## Fine grid (v19) and sequencer readout fix (v19, G1)

v19 adds one step value and fixes one readout. Nothing else about the
contract changes, and every v18 file plays identically in v19.

- **The fine grid.** `stepValueKey: "f"`, label `F`, name "Fine 1/48
  beat", `stepValueMult` = 1/48 quarter notes per step. In beats
  (quarter notes): a straight 16th is 1/4, a triplet 16th is 1/6, a
  32nd is 1/8, a 64th is 1/16. The smallest step dividing all four is
  1/48 of a beat, so on this grid a straight 16th is 12 steps, a
  triplet 16th is 8 steps, a 32nd is 6 steps, and a 64th is 3 steps —
  all integers in one file, with no rhythm stretching. A 4/4 bar at
  this grid is 192 steps (`4 × 4 / 4 / (1/48) = 192`); step seconds are
  still `(60 / bpm) × stepValueMult`, so a 4/4 bar still lasts four
  beats. MIDI ticks per step are `round(480 × 1/48) = 10` exactly, so
  tick positions do not drift: a note at step N starts at tick
  `10 × N`, and a gate of D steps lasts `round(10 × D × gateTime ×
  0.85)` ticks in score MIDI export, exactly as for every other step
  value.
- **Caps raised, and why.** The v18 caps of 64 steps per bar and 64
  for `stepCount` cannot hold a 192-step 4/4 bar, so v19 raises both
  to 512: `meterStepsFromNumDen`, meter normalisation (computed and
  steps-only entries, plus the bar-0 fallback), `meterStepsAtBar`,
  and the `stepCount` clamp on import. 512 covers a 192-step bar with
  headroom for longer meters (up to an 8/4 bar at the fine grid is
  384 steps) while still bounding the meter map. No other step value
  changes meaning: `w h q qd tq e ed te s ts` keep their v18
  multipliers, and old files import with the same gates, bar
  boundaries, and MIDI ticks as in v18.
- **Sequencer readout fix (G1).** In v18 the sequencer info line
  showed `S.stepCount` steps, which is bar 0's length — so Greensleeves
  v18, whose first bar is a 2-step 1/8 pickup, read "2 steps" for a
  tune whose main bars are 12 steps. In v19 the info line shows the
  main meter steps instead: `meterMainSteps()`, the most common bar
  length over the first 64 bars of the meter map (the map's tail entry
  extends, so a short file's main meter still wins). Greensleeves v18
  now reads "96 bpm · S · 12 steps · meter 1/8 @ bar 1 · 6/8 @ bar 2".
  Playback bar-boundary logic is unchanged; only the displayed number
  changes, and a single-meter file reads exactly as before.
- **Long-pattern editor (v19 render fix).** An oscillator card builds
  one DOM button per step for its grid, duration, probability, accent,
  lock-selector, and melody rows. At the fine grid, Bach's patterns are
  27,456 steps each, so one patch card set would need over two million
  DOM nodes — before this fix, importing the Bach v19 patch did not
  return within 30 seconds. In v19, cards for patterns over 2048 steps
  render a summary line (steps and gate count) instead of those
  per-step editors; all other oscillator controls still render, and
  playback, score, and MIDI are untouched. After the fix the same
  import returns in about 2.6 seconds in the headless harness.
  Patterns up to 2048 steps render exactly as before. Editing a long
  pattern's individual steps is therefore done by importing a new JSON
  in v19, not in the card editor.
- **Worked example — one bar, four rhythm values, exact.** A proof
  file at `stepValueKey: "f"`, 120 bpm, one 4/4 bar of 192 steps:
  straight 16th C4 at step 0 (12 steps), triplet-16th group D4/E4/F4
  at steps 12/20/28 (8 steps each), 32nd G4 at step 36 (6 steps), 64th
  A4 at step 42 (3 steps). Verified in v19: the file imports clean,
  the gates simulate at exactly those steps with those durations,
  live SCORE playback passes step 192, and its score MIDI export
  parses back with note-ons at ticks 0, 120, 200, 280, 360, 420 and
  note-offs at 102, 188, 268, 348, 411, 446 — that is, start tick =
  step × 10 and gate = round(steps × 10 × 0.85), with no rounding
  drift.

## Import normalisation (v19): steps/melody alignment and score colour de-duplication

Two import-time fixes, both in the v19 build, neither of which changes
any file on disk — they fix what the engine does with a file on import,
so any imported file benefits.

**Steps/melody alignment.** `steps` is the gate grid every consumer
indexes by (live playback, SCORE playback, WAV render, MIDI export), so
it is canonical on import. If an oscillator arrives with a melody
longer than its steps (the reported case: 22 steps, 23 melody slots),
the melody is tiled/truncated to the steps length:
`newMelody[i] = oldMelody[i % oldLength]` for `i < steps.length`. The
first pass then sounds exactly as written, and later passes repeat
instead of drifting or leaving the extra slot dead. A melody shorter
than steps is tiled the same way, not null-padded, so a looping melody
(e.g. 4 slots cycling over 8 steps, as in the worked example below)
keeps cycling. Per-step arrays are resized to the same length
(`stepDurs` pad 1, `stepProbs` pad 100, `stepAccents` pad 1; empty arrays
stay empty, since every consumer already treats a missing entry as its
default); `stepLocks` at or past the new length are dropped. If `steps`
is empty and a melody is present, `steps` is filled with gates (1s) to
the melody length. The in-app steps-length buttons apply the same
tiling, so editing a pattern length in the editor cannot re-open the
drift either.

**Score colour de-duplication.** `sanitize()` (which `importScore()`
calls) now assigns a fresh colour, via `_uniqueColor()` in track order,
to any track whose colour duplicates an earlier track's colour, and to
any colourless track. The first track of each colour keeps its colour,
already-unique colours are never changed, and the file's stored colours
are untouched. Lanes and VIS both read `track.color`, so after import
each lane renders in its own colour. Note the palette limit:
`_uniqueColor()` draws from a 12-colour palette, so de-duplication is
guaranteed only while the number of distinct colours needed fits the
palette plus the already-unique stored colours; beyond that it falls
back to the requested colour.

## MIDI export (v18)

Two exporters, one file format. Both write SMF Type 1, tpq 480, with a
tempo meta event at tick 0 from `S.bpm` (microseconds per quarter =
`round(60000000 / bpm)`), track-name metas, pan (CC 10) and volume
(CC 7, voice volume scaled from its 0–0.15 range to 0–127) like the
patch exporter, and end-of-track on every track. Ticks per step are
`480 × stepValueMult` (120 per sixteenth, 240 per eighth, 480 per
quarter). Velocities are step level × slot velocity × step accent ×
100, clamped to 1–127, exactly as in patch export; clip fades in score
export multiply that velocity further (see below).

- **Patch MIDI export (`exportMIDI`, the "Export MIDI" button).**
  Loop-based, as before: it exports the currently selected patch, one
  track per oscillator (melody voices first if any exist, otherwise
  all voices), and repeats the loop `min(128, lcm(steps, melody)) × 4`
  steps. Note timing is `tick = step × ticks-per-step`; gate ticks are
  `stepDurs × gateTime × 0.85 × ticks-per-step` (arpeggiated voices use
  `arpGate` instead). Arps follow the patch exporter (including
  sub-step arps at rate > 1), and glides are emitted in a second pass:
  pitch-bend runs between consecutively sounded steps only, rests
  break the chain, bend range `S.midiBendRange`. **New in v18:**
  time-signature meta events (`0xFF 0x58`, `nn dd cc bb` with
  `dd = log2(denominator)`, `cc = 24`, `bb = 8`) are emitted at the
  tick of every meter change inside the exported range, including
  bar 0. Note timing is otherwise unchanged from v17.12.6.
  Patch export still refuses while a score is playing (the live
  `__score` patch is a virtual patch, not an arrangement) — it now
  says so and points at Export Score MIDI instead. That refusal is
  not a gap any more, because score export below does not need
  playback at all.
- **Score MIDI export (`exportScoreMIDI`, the "Export Score MIDI"
  button in the MIDI FILE section).** One pass through the score
  arrangement, never a loop repetition. It works from `S.score`
  directly, without requiring playback, and mirrors
  `renderScoreToWAV`: virtual voices are built from each track's
  `timbre` plus the active clip's pattern keys (the same PAT list as
  the score section), clips follow the latest-`startBar`-wins overlap
  rule, gaps are silence, and the exported range is bars 0 to
  `endBars − 1`, where `endBars` is `endBar` if set, otherwise the end
  of the last clip. Bars use the meter map: a bar's steps come from
  `meterStepsAtBar(bar)`, and a note's tick is
  `absStep × ticks-per-step`, where `absStep` is the meter offset
  (`meterStepForBar(bar) + step`). Notes come from the melody slot at
  `floor(absStep × rateMult) % patternLength` — the same phase rule
  as live playback, so a clip starting after a meter change phases
  exactly as it sounds. Kit/drum voices in a score use `stepData`
  where present (GM drum note, channel 10 unless the voice overrides
  `midiChannel`); other score voices use `steps` plus `stepAccents`,
  as in the score section. Muted tracks are skipped; when any track
  is soloed, only solo tracks are exported. Clip fades scale note
  velocity linearly (fade-in by distance into the clip, fade-out by
  distance to its end, both in bars); a fade value of 0 means that
  step is silent and no note is emitted. Arps are handled like patch
  export, except `arpPool: "bar"` pools the gated melody notes of the
  current bar — meter-aware bar boundaries, with the active pattern
  constant within a bar because clips start on bar boundaries.
  Glides reuse the patch exporter's two-pass approach per track.
  Time-signature metas are emitted at the tick of every meter change
  inside the exported range, including bar 0, so a score that changes
  meter exports its meter, not just its notes.
- **How the two differ, in one line.** Patch export exports a loop,
  repeated; score export exports an arrangement, once. If a piece
  lives in the SCORE tab, Export Score MIDI is the exporter for it;
  Export MIDI is for a patch loop, and its time-signature events now
  describe the patch's meter too.
- **Determinism note.** Both exporters treat `stepProbs` below 100
  the way live playback does — probabilistically, with a fresh random
  draw per export — so a MIDI file is only a faithful record of a
  piece when every gate is prob 100. Proof-of-concept conversions
  (Greensleeves, Jerusalem) are prob 100 throughout for exactly this
  reason.

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

## Render envelope guard (v19.1)

In `renderToWAV`, the sustain-level event is now scheduled only when its gate time falls at or after the end of the attack+decay ramps; otherwise it is skipped and the release ramp is scheduled at the later of its old time and the ramp end, so no automation event is ever inserted inside the attack/decay ramps. Effect: 32nd/64th notes in renders no longer have their attack suppressed (about 9 ms flat gain at short-note onsets in v19.0 renders); long notes are unchanged. Live playback was already unaffected (it uses `adsr-off` ramps, not this scheduling path).

### Composer checklist

Before you call a piece done:

- Does it have an energy arc, or just an order?
- Is there one moment the listener waits for — and does it arrive?
- Is there a motive that returns changed?
- Is there a silence that means something?
- Does the loop point compose, or just restart?
- Could you remove one voice and lose nothing? Then remove it.

## Music theory completion (v19.2)

Four notation-level features that were previously flattened, plus two
round-trip features, all optional: a file that uses none of them imports
and plays exactly as in v19.1 (regression gates: Greensleeves 72/32/32,
Jerusalem 133/74/74, Bach 3,697 gates, no page errors).

### Ornaments

A melody slot may carry `orn: {type, note?, octave?, step?, rate?}`.

- `trill`: starts on the upper neighbour and alternates principal /
  neighbour at `rate` sub-notes per second (default 12, clamped 2..32),
  counted over the note's gate duration.
- `mordentU` / `mordentL`: principal, neighbour, principal in equal
  thirds of the gate.
- `graceAcc` (acciaccatura): a grace note taking the first 25% of the
  gate, or 0.05 s, whichever is smaller, then the principal for the
  remainder. The grace steals time from the principal note, never from
  the previous note.
- `graceApp` (appoggiatura): the grace takes half the gate, the
  principal the other half.
- Neighbour pitch: `orn.note`/`orn.octave` when given; `orn.step` in
  semitones when given; otherwise the diatonic neighbour in a major
  scale (a half step above E and B, below F and C, a whole step
  elsewhere). Ornaments render in live playback (pitch sequence on the
  step's gate, with MIDI note-ons per sub-note), in `renderScoreToWAV`
  (each sub-note retriggered with its own envelope), and in score MIDI
  export (one note-on/off pair per sub-note). An existing conversion
  that flattened an ornament is updated by adding `orn` to that slot
  only: principal note counts are unchanged, sub-notes are added.
  Verified: The Ash Grove's three source trills (bars 11, 13, 15)
  become 9 + 9 + 27 sub-notes, lead 83 -> 125 notes, bass/harmony
  24/24 unchanged, render duration unchanged at 60.0 s.

### Tempo curves

`sequencer.tempoMap` and `score.tempoMap` are arrays of
`{bar, bpm}`, sorted by bar. The tempo at a bar position is linearly
interpolated between neighbouring points and flat outside the first
and last point. One point, or no map, is exactly the old constant bpm.
Live playback, WAV render, and score MIDI export all honour the map;
in MIDI the curve is written as tempo meta events at each map point
(the first point replaces the track's tick-0 tempo), so a player that
honours tempo metas follows the curve. Verified: a 60 -> 120 bpm map
over bars 0..2 renders in 9.67 s (constant 60 would be 16 s, constant
120 would be 8 s) and exports tempo metas 1,000,000 us at tick 0 and
500,000 us at the bar-2 tick.

### Repeat structure and key changes as data

- `score.repeats`: `[{fromBar, toBar, times, endings}]`. The region is
  `[fromBar, toBar)`; `times` is the total number of plays (default 2,
  clamped 1..8); `endings` is one list of source bars per pass, in
  order, written after `toBar` in the source. Pass i plays
  `endings[i]` (the last list repeats if `times` exceeds its length).
  Repeats do not nest and do not overlap; overlapping entries keep
  the first and drop the rest. Playback, render, and MIDI export all
  walk the expanded bar list; an unfolded file (no repeats) expands
  to itself and plays identically.
- `score.key` is the opening key `{root, scale}` (`scale` is
  `"major"` or `"minor"`); `score.keyChanges` is
  `[{bar, root, scale}]`. Key changes emit MIDI key-signature metas
  at the change bar and re-map degree slots (below); absolute note
  slots are untouched by them.
- Degree slots: a melody slot may use `deg` (1..7) and `degAcc`
  (-1, 0, 1) instead of `note`. The pitch is the scale degree in the
  active key, with `octave` as the octave of degree 1. Verified: C
  major deg 1/5 = C4/G4, then a key change to G major makes deg 1/5 =
  G4/D5, with MIDI key metas (0 sharps, major) then (1 sharp, major).

### Microtones and alternate tuning

- A melody slot may carry `cents` (-100..100), an explicit detune
  applied on top of the equal-tempered pitch.
- A score (or oscillator) may carry `tuning`: `"equal"` (default),
  `"just"`, `"pythagorean"`, or `"meantone"` (quarter-comma). Named
  tunings offset diatonic degrees only, in cents relative to equal
  temperament (just: 0, +3.91, -13.69, -1.96, +1.96, -15.64, -11.73;
  Pythagorean: 0, +3.91, +7.82, -1.96, +1.96, +5.87, +11.73;
  meantone: 0, -6.84, -13.69, +3.42, -3.42, -10.26, -17.11);
  chromatic notes are unaffected. Live and rendered audio use the
  exact detuned frequency. Score MIDI export writes the nearest MIDI
  note plus a pitch-bend event (bend range 2 semitones, reset to
  centre at note-off); this is per-channel, so it is exact for
  monophonic tracks and shared across simultaneous notes on one
  channel. Patch MIDI export uses the nearest note without bend.
  Verified: A4 + 50 cents renders at 452.89 Hz and exports as note 69
  with bend value 10239 (0.50 semitones) reset to 8192 at note-off;
  just-intonation degree 3 in C resolves to 327.03 Hz (-13.69 cents).

### In-engine video export (v19.2)

`exportScoreVideo({title, composer, credits, preset})` exports the
loaded score as one video file, entirely in the browser: a 6 s title
card drawn by the VIS canvas itself, then the whole tune with the
engine's own visualiser, driven by the score's real note events
(including ornament sub-notes), over audio from `renderScoreToWAV`
(never a MIDI synth). Title fields live on the score (`title`,
`composer`, `credits`, `videoPreset`) and are saved with it; the
title card's "played by CALORIS" line takes the version from the
document title, never typed by hand. Defaults pre-fill from the
score's existing metadata, so an existing tune exports with a
sensible card without retyping. The preset list comes from VIS
itself (Bloom, Rain, Tunnel, Lattice, Plasma, Orbit, Nebula, Spiral,
Cascade, Kaleidoscope); no preset is hard-coded into the export.
Recording is browser-native: the VIS canvas stream (30 fps
requested) plus the rendered audio through a MediaStreamDestination,
combined in a MediaRecorder. The container is the best the browser
supports, MP4 first, then WebM; the returned info and the UI state
the actual container, resolution, and fps. Recorded resolution is
the VIS canvas size (viewport x devicePixelRatio, capped at 1.5 by
VIS), so a 1920x1080 export needs a 1920x1080 viewport at scale 1.
Gain is built in: the rendered audio is measured before recording
(sample peak, 4x-interpolated true-peak approximation, RMS-based
LUFS-I approximation without K-weighting, labelled as such) and
normalised gain-only, never with dynamics or EQ, so no export
leaves with peak above -1 dBFS; the applied gain is reported.
Verified in headless Chrome: 1920x1080, MP4 container with VP9 video
and Opus audio (Chrome's MP4 muxer, not H.264), requested 30 fps,
measured 25.4 fps in headless, 8.11 s file for a 6 s title + 2 s tune
(0.11 s recorder overhead), title card legible with all five lines
including the auto-filled version, first sound at 6.034 s (one frame
after the 6.000 s title end), peak -2.56 dBFS, true peak -2.56 dBFS,
LUFS-I (approx) -7.85, gain 0 dB, 13 visual events. Arp sub-notes are
not expanded into visualiser events (they still sound in the audio);
that is the one known visualiser gap.

### MIDI to score import (v19.2)

`importMIDIToScore(arrayBuffer, fileName)` parses Standard MIDI Files
(Type 0 and Type 1; Type 2 and SMPTE time division are rejected with
an error) and builds a playable score: one patch (`midi_import`) of
per-bar, per-voice oscillators, one score track per MIDI-track voice,
one clip per bar. Tempo metas become the v19.2 tempo map and
time-signature metas become the meter map, so tempo/meter changes
survive the import. Quantisation: the fine grid is 48 steps per
beat (192 per 4/4 bar), ticksPerStep = tpq/48; onset step =
round((tick - barStart) / ticksPerStep), duration steps =
max(1, round(length / ticksPerStep)). Notes whose onset is not
exactly on a step are counted in the import report (off-grid count
and max deviation in ticks) and rounded, never silently dropped;
exact duplicate notes in one voice are counted as merged. Polyphony
is partitioned greedily into voices per bar (a note joins the first
voice whose previous note has ended); each voice becomes its own
score track, because a score track holds one pitch per step. Pitch
bend in force on a note's channel at its onset becomes a cents
offset on that slot (nearest MIDI note + cents); bends that move
mid-note are not represented. Program changes survive as the
oscillator label and `midiProgram` field; the timbre itself is a
neutral sine, not a sampled GM instrument. Channel 10 drum notes are
imported as pitched notes and counted in the report (GM drum mapping
is not implemented). Durations are gate lengths: on re-export the
gate is 0.85 x the imported step duration, so a round trip matches
onsets, pitches, and counts, not raw gate ticks. Verified:
Greensleeves exported from v19.2 and re-imported gives 72/32/32
notes, 0 off-grid, 0 merged, with pitches and onsets matching in
played order (note: Greensleeves playback uses absolute pattern
phase, so the comparison is against the exported note order, not
the melody-slot order in the original file); the fine-grid bar
lengths become 24 steps (1/8 pickup) and 144 steps (6/8 bars), and
step positions scale by 12 from the original 16th grid. An
independent SMF (Type 1, conductor tempo/meter changes, program,
bend, chord) imports as 3 score tracks from 2 MIDI tracks, 6 notes,
tempo map 120 @ bar 0 and 60 @ bar 2, meter 4/4 @ bar 0 and 3/4 @
bar 2, with the bent A4 arriving as A#4 at -50 cents and the chord
split across two voices.
