# tabbycat IR

One representation for five readers: the LLM writes it, the solver validates and repairs it, the renderer draws it, the player plays it, and the library stores it.

Two forms:

- **Canonical IR** (`Idea` → `Revision` → `Piece`). Flat, explicit, stable IDs, absolute time. Everything except the LLM reads and writes this.
- **Authoring form** (`AuthoredPiece`, `EditOp`). What the LLM emits. Bar by bar, durations in sequence, no IDs, no onsets. Code expands it into the canonical IR.

## 1. Scalars

Types are written as plain TypeScript. `// enc:` gives the JSON encoding where it differs, so each maps to one Effect Schema (brand, literal union, or string transform).

```ts
/** Exact time in WHOLE NOTES. "1/4" = a quarter, "1/12" = a triplet eighth, "3" = three whole notes. */
interface Rational { readonly num: number; readonly den: number }   // enc: "3/8" | "2", always reduced, den > 0

/** Half-open [start, end), measured from the start of the piece. */
interface TimeSpan { readonly start: Rational; readonly end: Rational }

type Step = "C" | "D" | "E" | "F" | "G" | "A" | "B";
/** Spelled pitch class. `alter` in semitones, -2..2. */
interface PitchClass { readonly step: Step; readonly alter: number }      // enc: "C#", "Db", "F##", "Bbb"
/** Spelled pitch. MIDI number is derived: (octave + 1) * 12 + stepOffset + alter. */
interface Pitch extends PitchClass { readonly octave: number }            // enc: "C#4" (C4 = 60)

/** Interval above a chord root, as a chord-degree label. */
type Interval = string;   // enc: /^(bb|b|#)?(1|2|3|4|5|6|7|9|11|13)$/, e.g. "b3", "#11"

type NoteId = string & { readonly NoteId: unique symbol };       // "n17"
type VoiceId = string & { readonly VoiceId: unique symbol };     // "v1"
type HarmonyId = string & { readonly HarmonyId: unique symbol }; // "h3"
type IdeaId = string & { readonly IdeaId: unique symbol };
type RevisionId = string & { readonly RevisionId: unique symbol }; // "r2"

/** Who produced a value. */
type Actor = "llm" | "solver" | "code" | "user";
```

**Why**

- **Whole notes, not beats.** The prototype used quarter = 1. "Beat" changes meaning in 6/8 and 12/8; a whole-note fraction reads as the note value in every meter (`"1/8"` is always an eighth).
- **Rationals as strings.** `"3/8"` is one short token group, LLMs write it reliably, and it decodes to `{num, den}` at the boundary. Exact arithmetic means a 5:4 polyrhythm never drifts.
- **Spelled pitch, MIDI derived.** The prototype stored MIDI, which cannot tell C♯ from D♭. Scientific pitch names are the form LLMs know best.

## 2. Instrument

```ts
interface Instrument {
  readonly name: string;
  /** Open strings, LOW to HIGH. String numbers everywhere are indices into this array (0 = lowest). */
  readonly strings: readonly Pitch[];
  readonly frets: number;
  readonly scaleLengthMm: number;
}
```

Default: `{ name: "6-string all fourths", strings: ["E2","A2","D3","G3","C4","F4"], frets: 24, scaleLengthMm: 648 }`.

**Why**

- **Embedded in the piece, not referenced.** Fingerings mean nothing without the tuning they were solved for, and a library entry should be self-contained. To move a piece to another instrument: swap `instrument`, drop every `fingering`, re-solve. Pitches survive.
- **String 0 = lowest (kept from the prototype).** It is an array index, so it works for any string count. Guitarists and alphaTex count from the highest string, so an LLM may slip. That slip is caught: `strings[string] + fret` must equal the note's pitch, so a wrong convention is a structural error, never a silently wrong tab.
- The hand profile is per user and is not part of the IR.

## 3. Piece

```ts
interface Piece {
  readonly instrument: Instrument;
  /** Non-empty; first entry at 0; every entry on a barline. Bars are derived from this. */
  readonly meter: readonly MeterChange[];
  /** Non-empty; first entry at 0. */
  readonly tempo: readonly TempoChange[];
  /** Length of an anacrusis bar, if any. */
  readonly pickup?: Rational;
  /** Total length. Explicit, so trailing rests and looping are exact. */
  readonly length: Rational;
  readonly voices: readonly Voice[];
  /** Sorted by onset, then by voice order. */
  readonly notes: readonly Note[];
  readonly tuplets: readonly Tuplet[];
  readonly harmony: readonly HarmonySpan[];
  readonly intents: readonly Intent[];
  readonly findings: readonly Finding[];
}

interface MeterChange {
  readonly at: Rational;
  readonly beats: number;                 // 7
  readonly unit: number;                  // 8
  readonly groups?: readonly number[];    // [2, 2, 3], for beaming and click
}

interface TempoChange {
  readonly at: Rational;
  readonly bpm: number;
  /** The note value that gets the beat: "1/4", or "3/8" for a dotted quarter. */
  readonly per: Rational;
}

interface Voice {
  readonly id: VoiceId;
  readonly role?: "melody" | "bass" | "inner";
}

interface Note {
  readonly id: NoteId;
  readonly voice: VoiceId;
  readonly pitch: Pitch;
  readonly onset: Rational;
  readonly duration: Rational;      // sounding length; may cross barlines and overlap later notes
  readonly fingering?: Fingering;   // absent = the solver has not placed it yet
  readonly attack?: Attack;         // absent = unspecified; v1 solver ignores it
}

/** Display hint only. Time lives in the notes; a tuplet says how to bracket them. */
interface Tuplet {
  readonly voice: VoiceId;
  readonly span: TimeSpan;
  readonly actual: number;   // 3
  readonly normal: number;   // in the time of 2
}
```

**Why**

- **A flat list of notes with absolute onsets**, not bars containing voices containing beats. Held notes and overlaps are exact with no ties to reassemble. The solver's timeline is one sort. Bars, beams, rests and ties are derived by the renderer from `meter` plus the gaps in each voice.
- **A voice is one line.** At most one note may *start* in a voice at any instant (a structural check). A chord is several voices sharing an onset. This makes voice leading explicit with no guessing, makes "the previous note in this voice" well defined (legato needs it), and makes "select the bass voice" meaningful. Notes in a voice may still overlap in duration, which is how let-ring arpeggios are written. The renderer merges voices that share onset and duration onto one stem.
- **No rest events.** A rest is a gap. Fewer things to keep consistent.
- **Tuplets are a notation layer.** Exact durations already carry the rhythm for the player and solver. The renderer cannot always tell which bracket was meant (five notes of `1/20` could be 5:4 under a quarter or part of something larger), so the bracket is stored. Spans may nest. A check confirms the notes inside fit the ratio.
- **`length` is explicit** because the last note's end is not always the end of the bar.

## 4. Fingering and technique

```ts
type Hand = "fret" | "pick";
/** Thumb, index, middle, ring, little: the same numbering for either hand. */
type Finger = "T" | 1 | 2 | 3 | 4;

interface Placement {
  readonly string: number;      // index into instrument.strings
  readonly fret: number;        // 0 = open
  /** Which hand stops the string. "pick" = that hand is on the fretboard (tapping). */
  readonly hand: Hand;
  /** Absent on an open string. Absent on a fretted note = string and fret proposed, finger left to the solver. */
  readonly finger?: Finger;
}

interface Fingering extends Placement {
  /** Who set the placement now in force. "user" is a pin the solver must not move. */
  readonly by: Actor;
  /** Present when the solver overrode the LLM. Keeps what was proposed and the rule that rejected it. */
  readonly replaced?: { readonly was: Placement; readonly rule: string };
}

/** How the note is started. Legato and slide refer to the previous note in the same voice. */
type Attack =
  | { readonly kind: "pick"; readonly stroke?: "down" | "up" }
  | { readonly kind: "pluck"; readonly finger: "p" | "i" | "m" | "a" | "c" }
  | { readonly kind: "legato" }   // hammer-on or pull-off; direction follows from the pitches
  | { readonly kind: "slide" }
  | { readonly kind: "tap" };     // the hand is `fingering.hand`
```

How the required techniques are carried:

| Technique | In the IR | What the solver does with it later |
|---|---|---|
| Legato / slur chain | A run of notes in one voice whose `attack` is `legato` (the chain starts at the note before the run) | Every note in the chain shares a string |
| Slide | `attack: slide` | Same string as the previous note in the voice; one finger |
| Sweep | Consecutive `pick` notes with the same `stroke` | Costs anything but one note per adjacent string |
| Hybrid picking | `pick` and `pluck` (`m`, `a`, `c`) mixed in a passage | Costs string distance between pick and fingers |
| Two-hand tapping | `attack: tap` plus `fingering.hand: "pick"` | Adds pick-hand placements as candidates |

**Why**

- **One `Fingering` in force, with provenance on it.** The renderer reads one field. The library can still answer "what did the LLM propose and what did the solver change" from `by` and `replaced`, with no parallel "proposed" array to keep in step.
- **`hand` on the placement** fixes the prototype gap. A tapped note and the pull-off after it differ only in `hand` and `attack`, so tapping needs no special structure.
- **`finger` optional on a fretted note.** The LLM often knows the shape (string and fret) better than the exact finger. A partial proposal is still a useful constraint.
- **Attack per note, not technique spans.** The prototype had `techniques: string[]`, which no code could interpret. A per-note attack is compact for the LLM and survives edits: deleting a note cannot leave a dangling span. Chains and sweeps are derived as maximal runs.
- **Expression later.** Dynamics, vibrato and bends become an optional `expr` field on `Note` and, for shapes over time, a `spans` list on `Piece` keyed by `NoteId`. Both are additions; nothing above changes. Swing feel is the same kind of addition on `TempoChange`.

## 5. Harmony

```ts
interface HarmonySpan {
  readonly id: HarmonyId;
  /** backing = what the pad plays. implied = what the line says. */
  readonly lane: "backing" | "implied";
  readonly span: TimeSpan;
  /** Non-empty. Entry 0 is the default reading. */
  readonly readings: readonly ChordReading[];
  /** Index the user switched to. Absent = the default. */
  readonly picked?: number;
}

interface ChordReading {
  readonly root: PitchClass;
  /** Suffix after the root: "m7", "7#11", "13sus", "" for major. Display only. */
  readonly quality: string;
  readonly bass?: PitchClass;
  /** Chord tones above the root, in chord-degree names. The verified content of the reading. */
  readonly tones: readonly Interval[];
  /** Optional scale or mode: "lydian b7". */
  readonly scale?: string;
  readonly by: Actor;   // "llm" for the default, "code" for enumerated alternatives
}
```

Rules:

- Spans in one lane do not overlap. The backing lane has no gaps; the implied lane may.
- A note's harmony is the implied span containing its **onset**, else the backing span. The toggle in the renderer forces the backing lane.
- A chord-melody chord is an implied span exactly as long as the chord.
- The pad plays the backing reading's `root` + `tones` (+ `bass`). Voicing is the player's job.

**Deriving a note's interval** (pure function, nothing stored per note):

```ts
declare const intervalOf: (pitch: Pitch, reading: ChordReading) => Interval;
```

1. If the note's pitch class equals `root + t` for some `t` in `reading.tones`, the label is `t`. So D against a C13sus reading is `9`, not `2`.
2. Otherwise use the spelling: letter distance from the root gives the number, semitones give the accidental. Non-chord seconds, fourths and sixths are named `9`, `11`, `13`.
3. If the spelling does not fit the root (it can happen right after switching readings), fall back to the pitch class with a fixed table: `b9 9 b3 3 11 #11 5 b13 13 b7 7`.

**Why**

- **Two lanes on one timeline** is the direct model of "backing says C7, line says C7♯11". The two are independent: different roots, different boundaries.
- **`tones` is the truth, `quality` is a label.** Code can check every reading against the actual notes without parsing chord names, and interval labels come from `tones`. That is what lets the LLM name the default freely while code guarantees it.
- **Default by position (entry 0), user choice by `picked`.** The LLM's choice and the user's choice are different facts; both are kept.
- **Intervals are derived, not stored.** Switching a reading re-colours every note with no writes.

## 6. Intents and findings

```ts
/** Something the LLM declares so the solver does not "fix" it. */
interface Intent {
  readonly kind:
    | "intentional-stretch"   // a wide reach is the point: allow up to the hand's max without a warning
    | "keep-shape"            // the fingering is the idea: if it fails, change notes rather than re-finger
    | "position";             // stay near `fret`
  readonly notes: readonly NoteId[];
  readonly fret?: number;     // for "position"
  readonly why?: string;
}

/** A solver result that outlives the repair loop: a warning that was accepted, and why. */
interface Finding {
  readonly rule: string;              // stable rule id, e.g. "cost.stretch-max"
  readonly severity: "warn" | "info";
  readonly notes: readonly NoteId[];
  readonly summary: string;
  /** Index into `intents` that excused it, if any. */
  readonly excusedBy?: number;
}
```

**Why**

- The full `Diagnostic` (measured value, limit, verified fixes) stays in the solver and the agent loop. It is transient. Only what the user should still see on the saved tab is persisted, as a `Finding`.
- Diagnostics should address notes by `NoteId` only. The prototype also used array indices, which break as soon as an edit lands.
- An intent never overrides a hard-impossible rule. It only moves the line between "warn" and "fine".

## 7. Library: ideas, revisions, selections

```ts
interface Idea {
  readonly id: IdeaId;
  readonly title: string;
  readonly tags: readonly string[];
  readonly request: Request;
  /** Immutable snapshots. A tree through `parent`, so "go back and try another direction" is free. */
  readonly revisions: readonly Revision[];
  readonly head: RevisionId;
  readonly flags: readonly Flag[];
  /** Counter for note and harmony IDs. IDs are never reused within an idea. */
  readonly nextId: number;
  readonly createdAt: string;   // ISO 8601
  readonly savedAt?: string;    // absent = not in the library yet
}

interface Request {
  readonly prompt: string;                 // the user's words, verbatim
  readonly style?: readonly string[];      // ["Allan Holdsworth"]
  readonly tune?: string;                  // "Giant Steps"
  readonly harmonicIntent?: string;        // "imply lydian b7 on the dominants"
  readonly basedOn?: readonly IdeaId[];    // saved ideas this one connects to
}

interface Revision {
  readonly id: RevisionId;
  readonly parent?: RevisionId;
  readonly at: string;
  readonly model?: string;
  /** What produced this revision from its parent. Absent on the first. */
  readonly edit?: Edit;
  readonly piece: Piece;
}

interface Selection {
  readonly span: TimeSpan;
  readonly voices?: readonly VoiceId[];    // absent = all
  readonly strings?: readonly number[];    // absent = all
  /** The notes the selection resolved to when it was made. */
  readonly notes: readonly NoteId[];
}

interface Flag {
  readonly kind: "hard" | "dislike";
  readonly revision: RevisionId;
  readonly selection: Selection;
  readonly comment?: string;
  readonly at: string;
}

interface Edit {
  readonly prompt: string;                 // "make it more fusiony"
  readonly selection?: Selection;
  readonly ops: readonly EditOp[];
}
```

**Why**

- **Snapshots for storage, ops for the LLM.** A practice-sized piece is a few kilobytes, so each revision stores a whole `Piece`. Nothing has to be replayed to render or play. The LLM never rewrites the piece; it emits `EditOp`s and code applies them. The ops are kept on the revision as provenance, and they are the natural events if the backend becomes event-sourced.
- **IDs are assigned by code and never reused.** A note the edit did not touch keeps its ID, so the diff between two revisions is a join on `NoteId`. Short counters (`n17`) cost the LLM few tokens when it is shown the piece to edit.
- **A selection stores both the span and the note IDs.** The span says where new material goes, including empty space. The IDs stay correct after an edit shifts time.
- **A flag points at a revision.** Revisions are immutable, so "this fingering felt hard" still names the exact strings, frets and fingers after ten more edits. That is what the hand profile learns from.

## 8. Authoring form (what the LLM emits)

```ts
interface AuthoredPiece {
  readonly title: string;
  readonly tags: readonly string[];
  readonly meter: string;                  // "4/4"
  readonly tempo: number;                  // bpm, quarter-note beat
  readonly voices: readonly Voice[];
  readonly bars: readonly AuthBar[];
  readonly intents?: readonly AuthIntent[];
}

interface AuthBar {
  readonly meter?: string;                 // only when it changes
  /** Sequential; durations must fill the bar. */
  readonly backing: readonly AuthChord[];
  /** Sequential from the bar start. Use { sym: "-" } for a stretch with no implied harmony. */
  readonly implied?: readonly AuthChord[];
  /** Events per voice, in order. A voice left out is silent for the bar. */
  readonly voices: Readonly<Record<string, readonly AuthEvent[]>>;
}

interface AuthChord {
  readonly d: string;        // "1/2"
  readonly sym: string;      // "C7#11", "Dm7/C"
  readonly scale?: string;
}

type AuthEvent = AuthNote | AuthRest | AuthTuplet;

interface AuthNote {
  readonly p: string;        // "F#4"
  readonly d: string;        // NOMINAL duration: "1/8" even inside a triplet
  readonly str?: number;
  readonly fret?: number;
  readonly fin?: Finger;
  readonly hand?: Hand;      // default "fret"
  readonly atk?: "down" | "up" | "p" | "i" | "m" | "a" | "c" | "legato" | "slide" | "tap";
  readonly tie?: true;       // sustain into the next event of this voice (same pitch); used across barlines
  readonly mark?: string;    // a label an AuthIntent can point at
}
interface AuthRest { readonly r: string }
/** One level only; contents are notes and rests. */
interface AuthTuplet { readonly tuplet: string; readonly of: readonly (AuthNote | AuthRest)[] }   // "3:2"

interface AuthIntent { readonly kind: Intent["kind"]; readonly mark: string; readonly fret?: number; readonly why?: string }

type EditOp =
  /** Remove the notes in the region and write new material there. New notes get new IDs. */
  | { readonly op: "replace"; readonly span: TimeSpan; readonly voices?: readonly VoiceId[];
      readonly content: Readonly<Record<string, readonly AuthEvent[]>>;
      /** If the new material is longer or shorter than the span, shift everything after it. Default false. */
      readonly resize?: boolean }
  /** Change fields of existing notes in place. IDs are kept. */
  | { readonly op: "patch"; readonly notes: readonly ({ readonly id: NoteId } & Partial<Omit<AuthNote, "tie" | "mark">>)[] }
  | { readonly op: "delete"; readonly notes: readonly NoteId[] }
  /** Replace one lane's harmony inside a span. */
  | { readonly op: "harmony"; readonly lane: HarmonySpan["lane"]; readonly span: TimeSpan; readonly chords: readonly AuthChord[] }
  | { readonly op: "meta"; readonly title?: string; readonly tags?: readonly string[] };
```

What expansion does: computes onsets by summing durations; scales tuplet contents and records the `Tuplet`; merges tied events into one `Note`; assigns IDs; parses chord symbols into `root` / `quality` / `tones`; turns contiguous notes with one `mark` into an `Intent`; stamps the user's `instrument`; adds code-enumerated alternative readings.

**Why**

- **Durations in sequence, no onsets.** LLMs miscount absolute positions. Summing is code's job.
- **Bar by bar, voices side by side.** Each bar is a checksum: every voice present must add up to the bar length, and so must `backing`. A miss produces a precise diagnostic ("bar 3, voice v2: 7/8 of 4/4") for the repair loop. It also keeps the vertical alignment in front of the model while it writes counterpoint.
- **Pitch and string/fret are both written.** They are redundant on purpose. Pitch is the truth; a string/fret that does not sound it is reported as a structural error.
- **Nominal durations inside tuplets** match how musicians and notation think (three eighths in the time of two), and they yield the bracket for free. One level keeps the schema non-recursive, which structured output handles better. Nested tuplets are still legal in the canonical IR.
- **Chord symbols as text.** LLMs write `C7#11` fluently and a symbol parser is needed for the backing anyway. Code derives `tones`; an unparseable symbol is a diagnostic.
- **Readable short keys** (`str`, `fret`, `fin`) instead of single letters. They cost about the same in tokens and are harder to mix up.
- **`replace` takes the same event lists as a bar**, so the model learns one syntax for writing and for editing.

## 9. Worked examples

All use the default instrument. Open strings are MIDI 40 45 50 55 60 65.

### 9.1 A line that implies more than the backing (authoring → canonical)

Backing is C7. The F♯ makes the line C7♯11.

```json
{
  "title": "Lydian b7 pickup", "tags": ["lydian b7"], "meter": "4/4", "tempo": 96,
  "voices": [{ "id": "v1", "role": "melody" }],
  "bars": [{
    "backing": [{ "d": "1", "sym": "C7" }],
    "implied": [{ "d": "1", "sym": "C7#11", "scale": "lydian b7" }],
    "voices": { "v1": [
      { "p": "E4",  "d": "1/8", "str": 4, "fret": 4, "fin": 1 },
      { "p": "F#4", "d": "1/8", "str": 4, "fret": 6, "fin": 3 },
      { "p": "G4",  "d": "1/8", "str": 4, "fret": 7, "fin": 4 },
      { "p": "Bb4", "d": "1/8", "str": 5, "fret": 5, "fin": 2 },
      { "p": "C5",  "d": "1/2", "str": 5, "fret": 7, "fin": 4 }
    ] }
  }]
}
```

Expanded (second note and the harmony only):

```json
{ "id": "n2", "voice": "v1", "pitch": "F#4", "onset": "1/8", "duration": "1/8",
  "fingering": { "string": 4, "fret": 6, "hand": "fret", "finger": 3, "by": "llm" } }
```
```json
[
  { "id": "h1", "lane": "backing", "span": { "start": "0", "end": "1" },
    "readings": [{ "root": "C", "quality": "7", "tones": ["1","3","5","b7"], "by": "llm" }] },
  { "id": "h2", "lane": "implied", "span": { "start": "0", "end": "1" },
    "readings": [{ "root": "C", "quality": "7#11", "tones": ["1","3","5","b7","#11"], "scale": "lydian b7", "by": "llm" }] }
]
```

`intervalOf` gives `3 #11 5 b7 1`. Against the implied reading F♯ is a chord tone (rule 1); against the backing it is a non-chord tone that still reads `#11` (rule 2). The renderer can style those differently.

### 9.2 Chord-melody: held bass, voices, switchable readings

A D bass rings for the whole bar under a melody. The pitches D F A C can be read three ways.

```json
"notes": [
  { "id": "n1", "voice": "bass", "pitch": "D3", "onset": "0",   "duration": "1",
    "fingering": { "string": 1, "fret": 5, "hand": "fret", "finger": 2, "by": "llm" } },
  { "id": "n2", "voice": "mel",  "pitch": "F4", "onset": "0",   "duration": "1/4" },
  { "id": "n3", "voice": "mel",  "pitch": "A4", "onset": "1/4", "duration": "1/4" },
  { "id": "n4", "voice": "mel",  "pitch": "C5", "onset": "1/2", "duration": "1/2" }
],
"harmony": [
  { "id": "h1", "lane": "implied", "span": { "start": "0", "end": "1" }, "picked": 1,
    "readings": [
      { "root": "D", "quality": "m7",    "tones": ["1","b3","5","b7"],  "by": "llm" },
      { "root": "F", "quality": "6",     "tones": ["1","3","5","6"],    "by": "code" },
      { "root": "C", "quality": "13sus", "tones": ["1","9","11","13"],  "by": "code" }
    ] }
]
```

`n1` overlaps `n2`–`n4`, so the solver keeps string 1 and finger 2 busy for the bar. The user has clicked to the F6 reading (`picked: 1`): the melody F A C re-colours from `b3 5 b7` to `1 3 5` with no change to the notes.

### 9.3 Polyrhythm: 3 against 2

Authoring, half a bar shown per voice:

```json
"voices": {
  "top":  [{ "tuplet": "3:2", "of": [{ "p": "G4", "d": "1/4" }, { "p": "A4", "d": "1/4" }, { "p": "Bb4", "d": "1/4" }] }, { "r": "1/2" }],
  "bass": [{ "p": "C3", "d": "1/4" }, { "p": "G3", "d": "1/4" }, { "r": "1/2" }]
}
```

Canonical: top onsets `0`, `1/6`, `1/3`, each `1/6` long; bass onsets `0`, `1/4`. One hint is recorded: `{ "voice": "top", "span": { "start": "0", "end": "1/2" }, "actual": 3, "normal": 2 }`. The player needs only the onsets.

### 9.4 Legato chain into a tap

```json
[
  { "id": "n1", "voice": "v1", "pitch": "C4", "onset": "0",    "duration": "1/16", "attack": { "kind": "pick" },
    "fingering": { "string": 3, "fret": 5,  "hand": "fret", "finger": 1, "by": "llm" } },
  { "id": "n2", "voice": "v1", "pitch": "E4", "onset": "1/16", "duration": "1/16", "attack": { "kind": "legato" },
    "fingering": { "string": 3, "fret": 9,  "hand": "fret", "finger": 4, "by": "llm" } },
  { "id": "n3", "voice": "v1", "pitch": "G4", "onset": "1/8",  "duration": "1/16", "attack": { "kind": "tap" },
    "fingering": { "string": 3, "fret": 12, "hand": "pick", "finger": 2, "by": "llm" } },
  { "id": "n4", "voice": "v1", "pitch": "E4", "onset": "3/16", "duration": "1/16", "attack": { "kind": "legato" },
    "fingering": { "string": 3, "fret": 9,  "hand": "fret", "finger": 4, "by": "llm" } }
]
```

`n2` and `n4` are legato, so each must share a string with the note before it. That is the one-string constraint the prototype lacked, and it stops the solver from flattening the shape onto open strings. `n3` is stopped by the picking hand, so it does not count against the fretting hand's stretch.

### 9.5 Solver repair, provenance, and a declared stretch

The LLM put G4 on string 3, fret 12 (little finger) against an F3 at fret 3. The solver moved it, and re-fingered the F3 the same way:

```json
{ "id": "n3", "voice": "v1", "pitch": "G4", "onset": "1/4", "duration": "1/4",
  "fingering": { "string": 5, "fret": 2, "hand": "fret", "finger": 1, "by": "solver",
    "replaced": { "was": { "string": 3, "fret": 12, "hand": "fret", "finger": 4 }, "rule": "cost.stretch-max" } } }
```

Elsewhere the LLM meant a wide reach and said so (`"mark": "wide"` on two notes plus an `AuthIntent`). After expansion:

```json
"intents":  [{ "kind": "intentional-stretch", "notes": ["n7", "n8"], "why": "the 1-4 five-fret shape is the idea" }],
"findings": [{ "rule": "cost.stretch", "severity": "info", "notes": ["n7", "n8"],
               "summary": "Index to little spans 118 mm, past your comfortable 95 mm.", "excusedBy": 0 }]
```

The renderer can mark `n3` as solver-changed and `n7`–`n8` as a deliberate stretch.

### 9.6 Select a bar, riff on it, flag it

The user selects bar 2 and types "intersperse chord-melody here". The LLM sees revision `r1` with IDs and returns ops only:

```json
{
  "prompt": "intersperse chord-melody here",
  "selection": { "span": { "start": "1", "end": "2" }, "notes": ["n9", "n10", "n11", "n12"] },
  "ops": [
    { "op": "replace", "span": { "start": "1", "end": "2" },
      "content": {
        "v1":   [{ "p": "A4", "d": "1/4" }, { "p": "C5", "d": "1/4" }, { "p": "Bb4", "d": "1/2" }],
        "bass": [{ "r": "1/4" }, { "p": "D3", "d": "1/4" }, { "p": "C3", "d": "1/2" }]
      } },
    { "op": "harmony", "lane": "implied", "span": { "start": "5/4", "end": "2" },
      "chords": [{ "d": "1/4", "sym": "Dm7" }, { "d": "1/2", "sym": "C7" }] }
  ]
}
```

Code applies it to make `r2`: `n9`–`n12` are gone, the new notes are `n20`–`n24`, and every note outside bar 2 keeps its ID and fingering. The solver re-solves bar 2 with its neighbours pinned, which keeps the new material in the same region of the neck.

Later the user flags part of it:

```json
{ "kind": "hard", "revision": "r2", "at": "2026-11-12T19:04:00Z",
  "selection": { "span": { "start": "5/4", "end": "3/2" }, "voices": ["bass"], "notes": ["n23"] } }
```

### 9.7 A library entry

```json
{
  "id": "idea_7f3", "title": "Lydian b7 pickup", "tags": ["lydian b7", "chord-melody"],
  "request": { "prompt": "something Holdsworthy over a C7, imply lydian b7",
               "style": ["Allan Holdsworth"], "harmonicIntent": "lydian b7 on C7" },
  "revisions": [ { "id": "r1", "at": "…", "model": "…", "piece": { } },
                 { "id": "r2", "parent": "r1", "at": "…", "edit": { }, "piece": { } } ],
  "head": "r2", "flags": [ ], "nextId": 25,
  "createdAt": "2026-11-12T18:31:00Z", "savedAt": "2026-11-12T19:10:00Z"
}
```

## 10. What changed from the prototype IR

| Prototype | Now | Reason |
|---|---|---|
| `pitch: number` (MIDI) | Spelled `Pitch` | Chord spelling and interval names |
| Time in beats, quarter = 1 | Whole notes | Stable meaning across meters |
| `Passage { tempoBpm, notes }` | `Piece` with meter, tempo map, voices, harmony | Rhythm, harmony and voices had no home |
| `voice?: number` | Required `VoiceId`, one line per voice | Explicit voice leading |
| `techniques?: string[]` | Typed `attack` | Opaque tags could not constrain anything |
| `Fingering { string, fret, finger }` | Adds `hand`, thumb, `by`, `replaced`; finger optional | Tapping, provenance, partial proposals |
| `proposed?: Fingering` | One `fingering` in force | One field for the renderer; history is in `replaced` |
| Diagnostics by array index and ID | ID only | Indices break on edit |

Kept: stable note IDs, exact rational time, overlap as the model of held notes, optional fingering, string 0 = lowest, the instrument as data with scale length in mm.

## 11. Own JSON or alphaTex?

**Recommendation: our own JSON, with the LLM emitting the smaller authoring form in section 8.**

alphaTex does not fit as the IR:

- It is fret-first. A note is `fret.string`; pitch falls out of the tuning. Our rule is the opposite: pitch is the truth and fingering is an optional layer.
- It has no spelled pitch tied to a fretted note, no two-lane harmony, no chord readings, no note IDs, no provenance, no revisions. Every one of those would live in a side file that must stay aligned with a text format, which is worse than one structure.
- It counts strings from the highest, and its syntax has changed between alphaTab versions. What a model "already knows" is a mix of versions. This is a judgment from the format's history, not something measured here.
- It is text. The repair loop works best with schema-constrained structured output, where a malformed piece cannot be emitted and every diagnostic can name a field.

Where alphaTex still helps: as a one-way **export** from the canonical IR (to open an idea in alphaTab or Guitar Pro), and as a reference for which tab features exist.

On the authoring form: yes, the LLM should not emit the canonical IR. Absolute onsets, IDs, provenance fields and parsed chords are either error-prone for a model or simply not its to decide. The authoring form is much smaller (my estimate is about a third; not measured), and its bar checksums and pitch/fret redundancy turn most model mistakes into precise, local diagnostics.

## 12. Structural checks (run at expansion, before the solver)

1. Each bar: every voice present, and `backing`, sum to the bar length.
2. `strings[string] + fret` equals the pitch; `string` and `fret` are in range; no finger on an open string.
3. At most one note starts per voice per instant.
4. A `legato` or `slide` note has a previous note in its voice.
5. A `tap` has a fingering hand if it has a fingering.
6. Every reading's `tones` are consistent with its `root` and `quality`; the default implied reading contains the notes it covers, or the mismatch is reported.
7. Every `NoteId` in intents, findings, selections and ops exists in the revision it refers to.
