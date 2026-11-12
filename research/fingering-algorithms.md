# Guitar fingering algorithms and playability: research for tabbycat

Scope: how to validate and repair an LLM-proposed (string, fret, finger) assignment, for any string count and tuning (default E A D G C F, all fourths). Stack is TS + Effect, so "reuse" mostly means "learn from and port".

**Evidence legend:** `[P]` read the paper text. `[S]` secondary source only (abstract, citing paper, or search snippet). `[own]` my engineering proposal, no literature backing. Numbers marked `[P]` are quoted from the source.

---

## 1. Algorithms in the literature

The field is small and mostly 2004-2016 for classical methods, with a 2024-25 wave of transformer models. All of it targets **standard-tuned 6-string guitar and assigns a string/fret to given pitches**. None validates a *proposed* fingering, which is our case.

| Approach | Key work | What it models | How good |
|---|---|---|---|
| **Shortest path / DP over a layered graph** | Sayegh 1989, *Fingering for String Instruments with the Optimum Path Paradigm*, Computer Music J. 13(3) `[S]` | One layer per note, a node per (string, fret). Edge weight = transition difficulty. Min-cost path = fingering. Built on expert-guitarist rules. | Foundation for everything below. No quantitative eval that I could verify. |
| **Phrase segmentation + shortest path** | Radicioni, Anselma, Lombardo 2004, CIM04 `[S]` | Splits the score into phrases, solves each as a shortest path in an acyclic graph. | No numbers verified. |
| **Learned cost weights** | Radisavljevic & Driessen 2004, ICMC, *Path Difference Learning* [pdf](https://www.mistic.ece.uvic.ca/publications/2004_icmc_pdl.pdf) `[P]` | Gradient descent on the cost-function coefficients so the model's optimal path matches a human's. | Useful trick: **fit weights from examples**. Small-scale. |
| **A\* search** | Burlet & Fujinaga 2013, *Robotaba*, ISMIR [pdf](https://archives.ismir.net/ismir2013/paper/000217.pdf) `[P]` | DAG of candidate (string, fret) per note or chord, with **capo, tuning and fret count as inputs**. Edge weight = sum of 3 factors from Heijink & Meulenbroek: fretwise hand movement, fretwise finger span for chords, +1 penalty if the hand passes fret 7. | Normalized fitness vs human tabs: mean 1.11, sd 0.17, i.e. about as hard as human-made tabs. Fitness is a cost-model score, not a playability study. Monophonic and chord events only (no voice tracking). |
| **Genetic algorithms** | Tuohy & Potter 2005, ICMC, *A Genetic Algorithm for the Automatic Generation of Playable Guitar Tablature* `[S]`. Tuohy PhD thesis, UGA, [pdf](https://www.ai.uga.edu/sites/default/files/inline-files/tuohy_daniel.pdf) `[S]`. Ramos et al. add subpopulations `[S]` | Fitness function encodes playability limits. Also arranging (choosing which notes to drop). Later hybridised with neural nets. | Slow and stochastic. Dominated by DP for pure fingering. Interesting only for *arrangement* (dropping or transposing notes). |
| **HMM + Viterbi** | Hori et al. 2013 (input-output HMM) `[S]`. Hori & Sagayama 2016, *Minimax Viterbi* [pdf](https://archives.ismir.net/ismir2016/paper/000285.pdf) `[P]` | Hidden state = left-hand form `(string, fret, finger)`, output = pitch (deterministic, so prob 0 or 1). Transition prob = Laplace in index-finger-position distance, scaled by inter-onset time (`ifp = fret - finger + 1`, i.e. fingers assumed on consecutive frets). Probabilities are **hand-designed, not trained** (no data). | Minimax variant maximises the *minimum* transition probability, which "makes the most difficult move easier". Qualitative eval on a C major scale only. Monophonic. |
| **Viterbi over chord-vector states** | Sakai, Segawa, Kitahara 2023, CMMR, melody+bass [pdf](https://zenodo.org/records/10113396/files/cmmr2023_3c-D3.pdf) `[P]` | State = 6-vector of frets (`-1` = unplayed, `0` = open). Hard filter: span ≤ 3, ≤ 2 fretted strings, no duplicate pitch. Costs: transition 0 / 5 / 30 for index-finger-position jump ≤3 / ≤4 / else; extra cost for high positions (5 / 10 / 20). | A preliminary demo, no numbers. **A clear template for vector states.** CC BY 4.0. |
| **Position-aware Viterbi, all voices** | `knoguchi/fretpath` (MuseScore plugin, 2026) `[P]` README | Merges all voices into onset events. State = (fingering combo, hand position). Costs: shift distance and count, quadratic penalty past the body joint, chord stretch, open-string bonus. User annotations are hard constraints. Reads tuning and fret count from the instrument. | The closest *engineering* analogue to our backend. No published eval. |
| **CSP for chords** | Radicioni & Lombardo 2005, CogSci, *Computational Modeling of Chord Fingering* [pdf](https://escholarship.org/content/qt7cp5z258/qt7cp5z258.pdf) `[P]` | Variables = chord notes. Domain = (string, fret, finger) triples, so ≤16 per note. Constraints: one note per string, no finger overlap, per-finger-pair max span, index barré. Arc-consistency + backtracking. | 34 chords vs 3 guitarists. 100% precision (every model fingering was judged viable). 97.25% recall (the model covered the experts' fingerings except 6 cases). 66.6% of top-ranked fingerings matched the experts' top pick. Experts agreed with each other only 67%, so ~67% is near the ceiling. |
| **Neural: sequence labelling** | Edwards et al. 2024, *MIDI-to-Tab* (arXiv 2408.05024) `[S via fetch]` | BART-style masked LM on string tokens. Fret is derived from pitch minus open string. Trained on DadaGP, fine-tuned on 281 tabs. | Note-string agreement with ground truth 73.6% vs MuseScore 62.5%, Guitar Pro 8 62.3%, TuxGuitar 55.4%. Playability rating from 15 guitarists (1-10): GT 7.45, model 6.04, GP8 4.69. **Standard tuning only, no capo.** Max chord stretch 12 frets, which is *unplayable*. A model with no hard checks can output impossible shapes. |
| **Neural: T5** | Hamberger et al. 2025, *Fretting-Transformer* (arXiv 2506.14223) `[S via fetch]` | T5 encoder-decoder, trained with tuning and capo tokens and augmentation. | Tab accuracy 72.2% on Leduc jazz and 81.6% on DadaGP vs 62.6% and 78.9% for A\*. Difficulty is *not* lower. No official code found. Two third-party reimplementations exist (§3). |

Takeaways:
- **DP/Viterbi with a hand-designed cost is the proven core.** It is cheap, deterministic and explainable.
- Everything is validated against *human tabs* (agreement) or a *self-defined cost*. No standard playability benchmark exists. We must define our own.
- Inter-annotator agreement is poor (67% on first-choice chord fingerings), so **expect "valid" fingerings to be a set, not a point**.
- The ML papers show learned models drift into impossible shapes, which is an argument for **hard-constraint checks regardless of who proposes the fingering**.
- None of the papers handle **legato, tapping, hybrid picking, sustain/let-ring or voice independence**. Those are our gaps (§2, §4).

Supporting biomechanics: Heijink & Meulenbroek 2002, *J. Motor Behavior* [pdf](https://www.socsci.ru.nl/meulenbroek/Publications/Heijink%20en%20Meulenbroek%202002.pdf) `[P]`. Six professional classical guitarists, 3D kinematics. Complexity rose with (a) hand position toward the *extremes* of the neck (rated lowest around the middle), (b) finger span (4 frets vs 5), (c) hand repositioning. Joints are cheapest mid-range, and cost rises **non-linearly** toward extremes. This is what Robotaba's three factors come from.

Piano analogue worth stealing: Parncutt, Sloboda, Clarke, Raekallio, Desain 1997 *Music Perception* 14(4) [pdf](https://static.uni-graz.at/fileadmin/_Persoenliche_Webseite/parncutt_richard/Pdfs/PaSlClRaDe97_FingeringModel.pdf) `[P]`. Twelve additive rules, each with a point value. Stretch, Small-Span, Large-Span, Position-Change-Count (2 pts per change) and Position-Change-Size. Each finger pair has MinComf / MaxComf / MinRel / MaxRel limits. **The shape of the rule set is the template for our diagnostics**: per-pair comfort and relaxed ranges, additive points, and per-rule attribution.

---

## 2. Playability metrics

There is no guitar standard. The table gives literature values where they exist and otherwise says so. **Treat all thresholds as defaults in a per-player profile**, with hard limits (impossible) separate from soft limits (costly).

Convention: spans are in **frets** and measured on the finger-position axis. Fret width shrinks up the neck, so real reach is longer in low positions. **A single fret threshold is a first-order approximation.** Better: convert to mm using scale length (`distance_n = L · (1 - 2^(-n/12))`) and threshold in mm `[own]`.

| Metric | Definition | Literature value | Notes for tabbycat |
|---|---|---|---|
| **Per-finger-pair max span** | Max fret gap between two fingers pressing at once | Radicioni CSP `[P]`, as I read the (garbled) table: index↔middle 2, index↔ring 3, index↔little 4, middle↔ring 0 (?), middle↔little 3, ring↔little 1. The extraction was messy, so check the original table and its indexing (frets of difference vs frets covered) before porting. A minimum-span table also exists but isn't reproduced in the paper. | Make this a 4×4 matrix, **data, not code**. |
| **Hand span (index-to-little)** | Max fret gap in one hand position | Sakai: ≤3 for chord states (classical, conservative) `[P]`. Heijink: 4 = "small", 5 = "large" span `[P]`. Teaching sources: index-little ≈ 5 frets is the practical stretch max `[S, blogs]`. | Soft limit at 4, hard at 5 for a default adult hand. 6+ only with explicit "stretch" flag (Holdsworth-style). |
| **Hand position** | `ifp = fret - finger + 1` (index-finger fret) | Hori `[P]`. Heijink: cost lowest mid-neck, nonlinear at extremes `[P]`. | Use it as the **hand-state variable** in the Viterbi layer. |
| **Position shift** | |Δ ifp| between consecutive events | Sakai `[P]`: free ≤3, small cost ≤4, big cost beyond. Robotaba: fretwise distance as the edge weight `[P]`. Parncutt: 2 pts per change plus a size term `[P]`. | A shift is cheaper when the **onset gap is longer** (Hori: scale by inter-onset time `[P]`). Take a time-aware cost. |
| **Shift with no free time** | Shift distance vs available time and whether a string is sustaining | Not in the literature as a threshold. | `[own]` Flag: shift > 1 fret between onsets < ~100 ms unless via guide-finger/same finger sliding or legato. Calibrate on tempo. |
| **Simultaneous notes** | Count of fretted notes at one onset | CSP: ≤4 fingers, barré with index only, one note per string `[P]`. Sakai: ≤2 fretted strings in a melody+bass state `[P]`. | Hard: fretted notes ≤ 4 unless a barré covers the rest. Barré = one finger on one fret across contiguous strings, with every other fretted note at higher frets. |
| **Finger order (no crossing)** | Higher-numbered finger on higher fret on the **fret axis** | CSP "NoOverlaps" `[P]`. | Hard. Exception: thumb-over, and deliberate finger-crossing in stretch lines, which are flagged and not forbidden. |
| **One note per string** | | CSP `[P]` | Hard. Applies per instant, not per bar. |
| **Open-string preference** | Open strings are free and don't use a finger | Fretpath gives a bonus `[P]`. Robotaba: open-string vertices weigh 0 `[P]`. | Cheap to add. Caveat: tuning-dependent in all-fourths. Open strings are only free if the pitch is actually open. |
| **High-fret penalty** | Fret > ~7 | Robotaba +1 above fret 7 `[P]`. Sakai: higher cost for positions beyond fret 5 `[P]`. Fretpath: quadratic penalty past the "body joint" `[P]`. | Make "neck end" per-instrument: use the **fret where the body joins**, not a fixed 7. |
| **String crossings (right hand)** | Number of strings skipped between consecutive picked notes; same-string vs. cross-string alternation | Not found in the guitar fingering literature. | `[own]` Cost for skips ≥2 under alternate picking, and a lower cost under finger-style p-i-m-a. |
| **Sustain / voice conflicts** | A held note's string gets re-used by another voice before it ends | Not found. | `[own]` The key polyphonic check (§4): a note whose duration extends past a later onset on the **same string** is cut off. |
| **Legato (hammer-on / pull-off)** | Frets between slurred notes on one string | Not found in the fingering literature. Teaching sources only. | `[own]` Slur chain must stay on **one string**, each fretted note needs a free finger within span, an upward hammer needs the target fret > source. Distinct stretch limit from picked notes (Holdsworth-style lines routinely use 5-6 frets, with the hand re-anchoring). |
| **Finger strength / independence** | Ring and little penalised | Parncutt weak-finger rule `[P]`. Radicioni: prefers index/middle on bass strings `[S]`. | Per-finger weight vector in the profile. |
| **Reach in all fourths** | In all-fourths, a 3-note-per-string scale pattern spans differently | Not in the literature. | `[own]` The model must be a **function of the string pitch table** (§5). All-fourths has a uniform 5-semitone gap between adjacent strings (standard has 5,5,5,4,5), so the same shape is the same fret pattern on every string set. |

Other checks worth adding `[own]`:
- **Pitch correctness**: `open_pitch[string] + fret == note`. Ground truth, always check first.
- **Fret range**: 0 ≤ fret ≤ fret count.
- **Capo / nut**: finger on fret 0 is invalid. Capo makes lower frets unavailable.
- **Duration vs. gap**: onset-to-onset time must exceed a minimum for the move (see shift).
- **Repeated finger**: same finger at two different frets within a time too short to lift it.
- **Harmonics, bends, tapping**: out of scope initially. Mark as explicit techniques the validator trusts or skips.

---

## 3. Open-source implementations

Nothing here is a reusable *validator*. Most are DP solvers. Licenses checked on GitHub on 2026-11-12.

| Project | Lang | License | Status | Use for us |
|---|---|---|---|---|
| [knoguchi/fretpath](https://github.com/knoguchi/fretpath) | MuseScore plugin (QML/JS) | GPL-3.0 | Active, pushed 2026-10-25, 5★ | **Best cost-term reference.** Position-aware Viterbi with hard annotation constraints and tuning from instrument data. GPL means read and re-implement, don't copy. |
| [jgollub1/guitar_dp](https://github.com/jgollub1/guitar_dp) | Python (Flask) | MIT | Dormant, 2019 | Small DP per (note, finger) with an editable cost in `guitar_util.py`. Monophonic. Fine to port. |
| [twschiller/optimal-guitar](https://github.com/twschiller/optimal-guitar) | Java | GPL-2.0 | Dormant, 2013, 3 commits | Historical. Author says "real-guitar" is slow. |
| [gburlet/robotaba](https://github.com/gburlet/robotaba) | Python/C++ | none stated | Dormant, 2013 | A\* arrangement. **No license, so read-only.** The paper has the cost terms. |
| [rattleh83d/FretOPT](https://github.com/rattleh83d/FretOPT) | Python | none stated | Active, 2026-11, 2★ | DP with physical fret-distance costs. Self-described as AI-assisted. No license so read-only. |
| [blue-jam/guitar-fingering-optimizer](https://github.com/blue-jam/guitar-fingering-optimizer) | unverified | Apache-2.0 | Pushed 2026-08, 0★ | Not inspected. Candidate to check. |
| [Jazvie/t5_fretting_transformer](https://github.com/Jazvie/t5_fretting_transformer), [gakonspark/fullhorn-fret](https://github.com/gakonspark/fullhorn-fret) | Python | none / "MIT" claimed in description | 2026, third-party | Reimplementations of Hamberger 2025. **Not the authors' code**; verify the LICENSE file. Wrong shape for us anyway (standard tuning, no validation). |
| [maximoskp/midi2tab_Deconvolutions](https://github.com/maximoskp/midi2tab_Deconvolutions) | Python | none | Dormant, 2022 | Code for Kaliakatsos-Papakostas et al. 2025 (arXiv 2510.10619). Assumes 6 strings, 24 frets, standard tuning. |
| [dada-bots/dadaGP](https://github.com/dada-bots/dadaGP) | Python | MIT (code) | 2022, 176★ | GP-to-token pipeline. The **dataset** is built from Ultimate Guitar tabs, so check the data terms before any training use. |
| [Perlence/PyGuitarPro](https://github.com/Perlence/PyGuitarPro) | Python | LGPL-3.0 | Active, 2026-11, 374★ | GP3-5 read/write. Useful only for importing test corpora. |
| [slundi/guitarpro](https://github.com/slundi/guitarpro) | Rust | MIT | Active, 2026-11 | Same, in Rust. |
| [CoderLine/alphaTab](https://github.com/CoderLine/alphaTab) | TypeScript | MPL-2.0 | Active, 2026-11, 1.9k★ | Rendering and alphaTex. Already flagged in SPEC as a reference. It's a model of what a tuning/string-count-aware score IR looks like. |
| [helge17/tuxguitar](https://github.com/helge17/tuxguitar) | Java | LGPL (repo has no machine-readable license, verify) | Active, 1.6k★ | Its tab conversion was the *worst-rated* baseline in Edwards 2024 (3.32/10) `[S]`, so it is a negative example. |

**Verdict:** no library is worth depending on. Write a small TypeScript module. The cost terms from fretpath, Robotaba, Sakai and Parncutt give us the design.

---

## 4. Chords and polyphony vs monophonic lines

**Mono (single line)**
- State = `(string, fret, finger)` per note, or just `(string, fret)` plus a hand-position variable.
- DP over a layered graph. Complexity is O(N · k²) with k ≈ 4-8 candidates per note. Trivial.
- Costs depend on **consecutive pairs only**: the Markov property that Viterbi needs.

**Chord (simultaneous notes)**
- **Joint state**: all notes in one onset assigned at once. Domain size explodes if each note is independent (≤16 each → 16^n). Use constraint propagation (Radicioni) or enumerate **grips**. Strings are ordered and fingers are ordered, so valid grips are few.
- Hard constraints from Radicioni `[P]`: one note per string, no finger crossing, per-pair span, barré on the index only.
- Rank the survivors by comfort (lowest fret, fewest fingers, smallest span, open strings).
- Expert agreement on the *top* grip is only ~67%. Accept any grip that passes hard constraints and let the cost only break ties.

**Polyphony / counterpoint (several voices with different durations)**
This is not covered by the classic papers (they treat events as *onset-aligned chords*). Fretpath merges all voices into onset events, which loses sustain information. Our requirements:
1. **Event model**: sort all notes by onset. At each onset, the *sounding set* = notes started now + notes still held. Held notes **keep occupying their string and their finger**, so they constrain the new notes. A chord is not only what starts together.
2. **String occupancy timeline**: per string, intervals `[on, off)`. Overlap on the same string = conflict (a hard error unless the earlier note is intentionally cut, which the idea must say).
3. **Finger occupancy timeline**: per finger, `[press, release)`. A finger can't be in two places. A held note pins its finger, so a held finger can't shift.
4. **Hand position is a function of the *fingers currently down***, not of the new notes alone. A shift is only possible if the pinned fingers are compatible with the new `ifp`.
5. **Voice independence** `[own]`: when voice A holds while voice B moves, the held finger and B's fingers must co-exist within the span limit. Reject moves that need the hand to leave a pinned finger behind.
6. **Right hand** `[own]`: p-i-m-a or pick assignment per voice, with the max simultaneous right-hand fingers (thumb + 3). Chords with more than 4-5 plucked strings need a strum or pick flag.
7. Voice separation (Cambouropoulos 2008, cited in Kaliakatsos 2025 `[S]`) is for *input without voices*. Our LLM emits voices, so skip it.

**Polyrhythm**: not a fingering problem. It only matters through the *timeline*: onsets are rationals (e.g. 3:2). Use exact rationals so overlap and gap checks stay exact. Avoid float milliseconds until a tempo is applied.

**Complexity**: state = (finger→(string,fret) map, hand position). Practical: beam search or Viterbi over onset events with top-K grips per event.

---

## 5. Recommended design: validate + repair

Our case differs from every paper: **the LLM has already chosen fingers**. So the primary job is a *checker that explains*, and the solver is the *repair oracle*.

### 5.1 Pipeline

```
IR (notes, onsets, durations, voices, techniques, proposed (string,fret,finger))
 │
 ├─ L0 Structural     pitch == tuning[string]+fret, ranges, schema      (hard, fatal)
 ├─ L1 Hard physical  per-instant grip legality                         (hard)
 ├─ L2 Hard temporal  string/finger occupancy timeline, shifts vs time (hard)
 ├─ L3 Soft cost      Viterbi score of the proposed path vs. optimum    (graded)
 └─ Repair oracle     constrained Viterbi, pinned to LLM choices where legal
```

- **Check in order**; stop reporting L1+ errors for notes that already fail L0, to avoid noise.
- **Pass = no hard errors and soft cost within budget.** Soft issues alone don't fail the loop after a retry limit (avoids endless revisions of acceptable lines).

### 5.2 Cost model

Additive, per-rule, with **named terms** so each can be reported. Use Parncutt's shape (points per rule) with Viterbi's DP structure.

- **State at event t**: `(grip: finger → (string, fret) | none, handPos: ifp)`.
- **Hard filters** (cost = ∞): pitch match, one note per string, no finger crossing (unless flagged), per-pair span matrix, ≤ 4 fingers, barré rules, fret range, held-note pins.
- **Soft terms** (all weights are config):
  - `shift` = f(|Δifp|) × g(onset gap). Convex (Laplace-like per Hori, or 0/5/30 steps per Sakai).
  - `span` = penalty growing as the span passes a comfort threshold and then the max (Parncutt's MaxComf → MaxRel).
  - `position` = convex penalty away from mid-neck (Heijink) or past the body joint (fretpath).
  - `finger weakness` per finger.
  - `open string` bonus.
  - `string crossing` (right hand).
  - `minimax` term: also track the **max single-move cost**, not just the sum (Hori). Report both. The "worst move" is what actually makes a passage unplayable.
- Weights are initially hand-set. Radisavljevic's path-difference learning is a route to fit weights **from the user's own accept/reject feedback** later.

### 5.3 Tuning and string-count agnosticism

- The whole model sees the instrument only as `Instrument { tuning: number[] /* MIDI pitch per string, low→high */, frets: number, scaleLength?: number, capo?: number, hand: HandProfile }`.
- **Never hard-code 6, standard tuning, or "E4".** `pitchAt(s, f) = tuning[s] + capo + f`. Candidate generation: for each pitch, all `(s, f)` with `0 ≤ f ≤ frets`.
- **String indices are ordinal, with 0 = lowest pitch.** Say so in the diagnostics, and always include a human label (the pitch name of the open string: "string 3 (G)").
- The only physical constants (span matrix, comfort limits, finger weights) live in `HandProfile`, **not tuning-dependent**. Position costs are in frets or mm, independent of pitch.
- In all-fourths tuning, adjacent strings are a uniform 5 semitones apart, so chord shapes and scale patterns move across strings unchanged. A *checker* needs nothing special, but a **repair step may exploit it**: the same shape may exist one string up.
- Fret-width: take `scaleLength` and use mm for reach if present, otherwise fall back to a frets-only approximation.
- **Property tests** (cheap and valuable): random tuning and string count, then solver output must pass the checker, and `pitch == tuning[s]+f`.

### 5.4 Diagnostics the LLM can act on

Principles: **machine-parseable + a one-line prose fix hint**. Locate with the *same coordinates the LLM wrote*: bar and beat, voice, note id. State the **rule, the measured value, the limit, and a concrete option** that would satisfy it.

```json
{
  "id": "D12", "severity": "error", "rule": "span.pair",
  "where": { "bar": 3, "beat": "2.5", "voice": 1, "notes": ["n17", "n18"] },
  "measured": { "fingers": [1, 4], "fretGap": 6 },
  "limit": { "fretGap": 4, "soft": 4, "hard": 5 },
  "message": "Fingers 1 and 4 must cover 6 frets (index on 5, little on 11) in the same grip; max is 5.",
  "fixes": [
    { "kind": "restring", "note": "n18", "to": { "string": 4, "fret": 6 }, "cost": +2 },
    { "kind": "octave", "note": "n18", "by": -12 },
    { "kind": "drop", "note": "n17", "reason": "inner voice" }
  ]
}
```

- **Severity**: `error` (impossible, must fix), `warn` (hard but legal with effort, e.g. shift near the limit), `info` (cost summary). Cap the number returned per round: **fix the root causes first**, group by cause (one over-long span can cascade into 5 shift errors).
- **Machine-suggested fixes** from the repair oracle: "if you change n18 to (string 4, fret 6) the line is playable." Give the LLM 1-3 *verified* alternatives rather than only a complaint. LLMs repair much better when handed candidates that already pass.
- **Say what is unchanged**: "bars 1-2 and 4 pass. Don't touch them." Reduces regressions.
- **Round summary**: counts by severity, and the **total soft cost with its top 3 terms**, so the LLM sees progress between iterations.
- Distinguish *fingering* faults (fixable by re-fingering, no musical change) from *musical* faults (the notes themselves can't be played together on this instrument). The latter require the LLM to change pitches or rhythm: **tell it which kind**, because the first kind is free.
- Do **not** return raw numeric costs alone: "cost 47" tells the model nothing.

### 5.5 Repair

1. **Locate** failing events from L0-L2.
2. **Re-solve with pinning**: run constrained Viterbi where every *valid* LLM choice is kept as a strong prior (a low cost for staying, a high cost for deviation, as fretpath does with annotated fingers) and only failing windows plus one event of context are free.
3. **Minimal change wins**: prefer the repair that changes fewest notes and keeps the LLM's *shape* (the idiomatic fingering is part of the idea per SPEC). Measure "distance from proposal" and report it.
4. If a fingering-only repair exists, apply it automatically and tell the LLM what changed. Only escalate to the LLM when the *musical* content must change.
5. **Fallback**: full re-solve from scratch (code-only fingering) per SPEC decision 10.
6. Cap the loop at N rounds. Return best-so-far with remaining warnings if it doesn't converge.

### 5.6 Sustain, voices and technique (our differentiator)

No literature to lean on, so build the timeline model from §4 early:
- Rational onsets and durations, exact.
- Per-string and per-finger interval trees.
- Technique tags (`legato`, `tap`, `harmonic`, `let-ring`, `stretch`) that **relax or retarget** specific rules (e.g. `legato` raises the span limit and requires same-string, `stretch` raises the hard limit by 1).
- Let the LLM **declare intent** (`let-ring`, `stretch`) so the checker does not flag deliberate choices as errors.

### 5.7 Validation of the validator

- **Ground truth is thin.** Hori used a C major scale, Radicioni 34 chords. Build a small **golden set**: known-playable passages (Bach, Holdsworth-style licks, user-confirmed lines) and known-impossible ones, with expected rule IDs.
- Log accept/reject from the actual user. It is a source for weight fitting, per Radisavljevic.
- Report the checker's agreement with the user's own judgment, not with other people's tabs. The user plays all fourths and is the target.

### 5.8 Suggested build order

1. IR + `Instrument` + L0 (cheap, kills most LLM errors).
2. L1 grip legality with the span matrix and barré.
3. Timeline occupancy (L2), the highest-value polyphony check.
4. Viterbi cost + minimax, then the repair oracle with pinning.
5. Diagnostic format + fix candidates, then wire the loop.
6. Techniques (legato first), then weight fitting.

---

## 6. Gaps and caveats

- Most sources could not be read in full: Sayegh 1989, Radicioni 2004, Tuohy & Potter 2005 and Hori 2013 are cited from secondary sources `[S]`. Verify before citing them in anything external.
- The **Radicioni span table's exact indexing** (frets of difference vs. frets covered) needs checking against the original before reuse.
- The MIDI-to-Tab and Fretting-Transformer numbers come from the paper HTML via a summarising fetch, and the two sources disagree slightly within themselves (e.g. 73.18% vs 73.58%). Quote them as approximate.
- Teaching-blog stretch limits (5 frets index-little) are rules of thumb, not measurements.
- No source covers **non-standard tunings, 7+ strings, legato stretch limits, or voice independence**. §2 rows marked `[own]` are judgments to be calibrated against the user's hands.
- Licenses: all repo licenses were read via the GitHub API on 2026-11-12. tuxguitar and the third-party Fretting-Transformer repos need a manual LICENSE check.

## Sources

- Sayegh 1989: https://doi.org/10.2307/3680014
- Radisavljevic & Driessen 2004: https://www.mistic.ece.uvic.ca/publications/2004_icmc_pdl.pdf
- Burlet & Fujinaga 2013: https://archives.ismir.net/ismir2013/paper/000217.pdf
- Radicioni & Lombardo 2005: https://escholarship.org/content/qt7cp5z258/qt7cp5z258.pdf
- Heijink & Meulenbroek 2002: https://www.socsci.ru.nl/meulenbroek/Publications/Heijink%20en%20Meulenbroek%202002.pdf
- Parncutt et al. 1997: https://static.uni-graz.at/fileadmin/_Persoenliche_Webseite/parncutt_richard/Pdfs/PaSlClRaDe97_FingeringModel.pdf
- Hori & Sagayama 2016: https://archives.ismir.net/ismir2016/paper/000285.pdf
- Sakai, Segawa, Kitahara 2023: https://zenodo.org/records/10113396/files/cmmr2023_3c-D3.pdf
- Tuohy dissertation: https://www.ai.uga.edu/sites/default/files/inline-files/tuohy_daniel.pdf
- Edwards et al. 2024, MIDI-to-Tab: https://arxiv.org/abs/2408.05024
- Hamberger et al. 2025, Fretting-Transformer: https://arxiv.org/abs/2506.14223
- Kaliakatsos-Papakostas et al. 2025: https://arxiv.org/abs/2510.10619
