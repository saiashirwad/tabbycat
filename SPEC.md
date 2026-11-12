# tabbycat — Spec

Living spec, refined through conversation. Newest decisions win; superseded ones move to "Rejected / Deferred".

## One-liner
A practice app that generates fresh guitar ideas as playable, Guitar Pro-style tabs over a backing track, in a style and harmony you ask for, so you pick up new vocabulary by playing rather than by transcribing or studying theory.

## The user (v1 = me)
- Self-taught guitarist (also plays piano). Good ear. Strong technique: fingerstyle (two-finger alternation, Matteo Mancuso-style) and pick.
- Finds new sounds by **noodling**, not transcription. Doesn't care about jam-session improv skills, sheet music, or playing other people's songs.
- Programmer with ADHD and little free time. Practice must be quick, varied, and low-friction.
- Reads tab fluently and can almost "sight-hear" it. Tab is the native notation, not standard notation.
- Plays **all fourths, E A D G C F** (low → high), but the app itself is instrument-agnostic (see Decisions).
- Fingerstyle leans toward Bach-inspired jazz fusion: counterpoint, polyrhythms, chord-melody solos that mix linear lines with passing chords.
- Wants ideas to carry over into metal and fusion soloing too.
- Practices on a **laptop**.

## Problem / Why
- Practice keeps circling back to the same ideas.
- The usual advice runs "hear it in your head → learn to play it / learn the theory". I want the **opposite**: get handed interesting shapes, play them, and let the ear catch up.
- Transcribing jazz greats is too slow and grindy for the time and attention I have.

## Core experience (draft)
1. **Ask for ideas with constraints.** A request combines any of:
   - **Context:** a tune or changes ("Giant Steps"), or "pick a random tune".
   - **Harmonic intent:** "imply Lydian ♭7 here", "very minor-sounding". Can be vague.
   - **Style:** "like Allan Holdsworth", "like Julian Lage".
   - **Length:** "a few bars".
2. **Get tab + backing.** Guitar Pro-style tab with rhythm, plus a simple backing track (e.g. pad/strings playing the chords). The app **shows which harmony each passage implies**.
3. **Listen and play.** Built-in player plays the idea and the backing, so I can hear it and then play over the backing.
4. **Next.** Show me another. Fast loop, lots of variety.
5. **Select and riff on it.** Select a range of the tab, type a prompt ("intersperse this with chord-melody ideas"). New material has to fit that region of the neck and stay within reach.
6. **Save.** Keep a library of ideas I liked. Later requests can build on or connect to them.

## Inspiration & design principles
- **An active Slonimsky.** Like *Thesaurus of Scales and Melodic Patterns*, but generated on demand, in context, and playable.
- **Bret Victor-style design.** Tactile and direct: the tool should build understanding as you use it, with AI used in interesting ways. Skip the usual pile of dials and settings.
- **Not overwhelming.** Sensible defaults; I shouldn't have to make many choices.

## Tab renderer (first-class, built in-house)
- **Tab only, no staff notation.** Dropping sheet music makes our own renderer feasible.
- **Guitar Pro / MuseScore-level quality** for tab, with rhythm and our own playback.
- **Note annotations** that build fretboard knowledge by association:
  - Each note is labeled by its **interval relative to the implied harmony** (what the line implies, which may differ from the backing chord). A toggle switches to the backing chord.
  - **Colors are configuration**, not hard-coded, so I can experiment. Starting palette: 3rd = green, ♭3 = light green; 5th = blue; 6th = yellow; 7th = brown, ♭7 = light brown. Major/minor variants share a hue at different lightness.
  - Other modes: subscript note names; highlight only the root; only root + 3rd.
  - Tunable (relative to the root, choice of chord-naming), but **the default must just work**.
- **Voice-leading display** for chord-melody: show how individual voices move between chords.
- **Chord interpretations you can switch between.** Click a chord symbol to cycle through readings of the same pitches; the interval colors re-map live. E.g. D F A C as **Dm7** (D = root), **F6** (F = root), or over a C root as 9-11-13 (C13sus-ish). The LLM supplies the best default reading (it knows jazz-guitar conventions); code enumerates the alternatives from the pitch set.

## Chord naming
- Needs its own small system: pitch set (+ bass, + context) → ranked list of chord names with roots and interval labels.
- LLM picks the default in context; code guarantees every interpretation is consistent with the actual notes.

## A typical session
1. I type a rich prompt: "an interesting progression with altered sounds, Frank Gambale-style sweeps through the changes, resolving like X".
2. It generates a **short, practice-sized piece**. Length follows the request; ~10-12 bars is just a sense of scale, **not a hard-coded limit**. Never pages and pages of music.
3. I practice that for **~30 minutes**. Sessions are about depth on one idea, not rapid-fire browsing.
4. If it's not grabbing me, I **refine it conversationally**: "make it more fusiony", "add some slidey ideas". The idea gets revised, not replaced.
5. Select any passage and flag it: **"feels hard to play"** (feeds the hand profile) or **"not a fan of this sound"** (taste; see Deferred).
6. Partial use is normal: I might play one part of a hard idea and ignore the rest.

## Scope of v1 musical content
- **Notes first.** Pitches, strings, frets, fingers.
- **Rhythm:** the IR must represent it fully, but v1 keeps it simple. Don't overcomplicate.
- **Expression** (dynamics, vibrato, bends, phrasing nuance): out of scope for now.
- **Techniques** (sweep, hybrid picking, slides, two-hand tapping à la Tosin Abasi): v1 solver is **not** technique-aware, and slides are ignored. But the solver needs a **pluggable technique model** that changes which positions are candidates and how they're costed. Tapping especially changes what's possible (a second hand on the fretboard).

## Architecture: a compiler for guitar ideas
The LLM supplies musical ideas; code makes sure they're playable. Rough pipeline:

1. **Front end (LLM):** turns the request into a musical idea: notes, rhythm, the harmony it implies, **and a suggested fingering** (string/fret/finger). It starts from music theory, but fingering is part of the idea because idiomatic lines are shape-driven.
2. **IR:** a music-theory representation (pitches, durations, voices, chord symbols). It isn't tied to any instrument.
3. **Instrument model:** strings, tuning, fret count, hand reach. Any string count and any tuning.
4. **Back end (code):** validates the LLM's fingering and repairs it only where it fails. It can also assign fingerings from scratch as a fallback.
5. **Playability checks ("static analysis"):** stretch, position shifts, impossible simultaneous notes, awkward transitions. Core = dynamic programming (Viterbi) over fretboard positions with a cost model.
6. **Agentic loop:** if checks fail, send the diagnostics back to the LLM, which adjusts the idea, and repeat until it passes. Like compiler errors driving a fix.
   - **Diagnostics must be extremely descriptive**, like a good compiler error: which rule, where, measured value vs. limit, *why* it's hard, and verified alternatives. Enough for the LLM to understand and self-correct.

## Hand profile (my hands, not a generic hand)
- **Per-finger-pair stretch limits in mm** (so reach adapts to fret width up the neck), configurable. E.g. I can stretch index↔pinky wide, but middle↔ring only a little (unlike Holdsworth).
- **Grip rules:** specific shapes can be marked hard. Example: ring finger on fret 6 of the lowest string + middle finger on fret 4 of the highest string + other fingers busy = difficult for me.
- **Hard vs. soft:** a "difficult" rule is a cost penalty, not a ban. The idea can still be shown, but the solver avoids it when it can.
- **Learned over time:** select a passage → "feels hard to play" records a new rule (finger/fret/string pattern) into my profile.
- **Starts from sensible defaults** for an average hand (research running). No calibration step.
- Enforced by the **algorithm itself as constraints and costs**. No decision model needed here.

## Algorithm transparency
- I want to **understand the fingering algorithm**, not just use it. It's fun.
- Idea: a Bret Victor-style explorable view of the solver: see the candidate positions per note, the cost of each path, why it chose this fingering, and how my hand profile changed the result.
7. **Output:** a rendered tab, playback audio and backing audio.

## Constraints the generator must respect
- The instrument model (strings, tuning, reach). Default profile: 6-string, E A D G C F.
- Playability: fret span / hand stretch, position on the neck, transitions between shapes.
- Riffing on a selection has to stay near the selected passage on the neck.
- Variety beats difficulty progression. Diversity in the moment matters more than a curriculum.

## Non-goals
- Sheet music / standard notation.
- Learning existing songs or transcriptions.
- Ear training or theory curriculum.
- Jam-session / real-time improv training.
- Phone / tablet support (v1).
- Rhythm-focused backing (drums, grooves): later.

## Technical shape
- **Hosted website**, used on desktop/laptop. Not a local-only app.
- **Language:** TypeScript 7, Effect v4.
- **Frontend:** FoldKit (latest), with an Elm-style single state tree. Easy to inspect and debug the complex app state. (Svelte is the fallback.)
- **Backend:** Cloudflare Durable Objects, deployed with Alchemy. Look into Alchemy's domain-modeling features.
- **Agent loop:** Yielded Agent (`@yielded/agent`, Effect v4; confirmed this is the "Effect Agent" I meant) on the backend. Repair loop is ours: a completion tool that runs the playability checker, or hand-rolled on `effect/ai` `generateObject` with `@effect/ai-anthropic`.
- **Domain model:** candidate `@alchemy.run/domain` (event-sourced aggregates/events/views on DOs). Very new; evaluate.
- **Decision models:** a `Decide` Effect service for fast typed choices (Jev / Cloudflare Clef style). Candidate uses:
  - **Request router:** pick which generator handles a request.
  - **Repair picker:** choose among the checker's verified alternatives when an idea fails.
  - **Novelty gate:** is this new idea different enough from what's in my library?
  - **Tuning thresholds:** use my Next/Save behavior to tune thresholds.
  Music math (intervals, fingering, spans) always stays in code.
- **Editor/typecheck:** TS 7 with `@effect/tsgo` (the classic Effect language service doesn't run on TS 7).
- **Tab renderer + player:** our own, SVG or canvas.
- **Backing:** synthesized chord pads to start.
- **FoldKit Blocks:** build on the preview now (PR #1655 / #1860). We have a direct line to the FoldKit author, so cutting-edge APIs are fine.
- **Stack risks:** Blocks churn and TS2589 on native TS (raise upstream); beta Alchemy and Yielded Agent; building the repair loop ourselves.

## Open questions
- IR migration: the solver prototype uses quarter-note beats + MIDI + a separate `proposed` fingering; the IR uses whole notes + spelled pitch + one `fingering` with provenance. Migrate the solver.
- Which decision-model provider: Clef-Flash (Workers AI, native to our host) vs. Jev? Behind one `Decide` service either way.
- Library: what metadata gets saved (style, harmony, tune, tags)? How do new ideas "connect" to saved ones?

### From the solver prototype
- **Legato:** the solver doesn't know a slur chain must stay on one string, so from scratch it ruins Holdsworth-style shapes. Needs a minimal legato constraint soon, since it's a core use case.
- **Tapping:** `Fingering` has only fret-hand fingers; tapping needs a `hand` field.
- **Held stretches:** charged once when placed, not per beat held. Is that right? Should weights be fitted from "feels hard" feedback?
- **Beyond-max stretches:** who decides when one is acceptable: my profile, or the LLM declaring intent? How many repair rounds before the loop accepts remaining warnings?

## Decisions
| # | Decision | Why |
|---|----------|-----|
| 1 | Tab is the only notation | Tab is my native notation; sheet music doesn't matter |
| 2 | Instrument-agnostic: any string count and tuning | Covers 6/7/8 strings and all-fourths; the core model is music theory, not one guitar |
| 3 | Optimize for variety, not progression | Fits ADHD and short sessions |
| 4 | Requests are constrained (tune, harmony, style), not free generation | Better and more useful ideas than "just generate something" |
| 5 | LLM supplies the music, code enforces playability, in a loop | LLMs alone write unplayable tab |
| 6 | Guitar Pro-style tab with rhythm, not ASCII | Polyrhythms and counterpoint need durations |
| 7 | Built-in playback + backing track | Need to hear ideas and play over harmony |
| 8 | Laptop / desktop web only | It's where I practice |
| 9 | Saved library of liked ideas | Basis for connecting future ideas to past ones |
| 10 | LLM suggests fingerings; code validates and repairs | Shapes are part of the musical idea |
| 11 | Build our own tab-only renderer + player, early | Core to the learning experience; no staff notation keeps it feasible |
| 12 | Notes colored by interval relative to the current chord | Builds fretboard knowledge by association |
| 13 | Stack: TS 7, Effect v4, FoldKit, Effect Agent, Cloudflare DOs via Alchemy | Elm-style inspectable state; hosted; Effect end to end |
| 14 | Build on the FoldKit Blocks preview now; don't wait for a release | Direct line to the FoldKit author; churn is acceptable |
| 15 | Note colors are config variables | I want to experiment with palettes |
| 16 | Colors follow the implied harmony by default; toggle for backing chord | The line's harmony is what I'm learning |
| 17 | Fingering = DP/Viterbi with a cost model, written in-house in TS | Proven approach; nothing reusable exists |
| 18 | Diagnostics to the LLM are verbose and explanatory | The LLM self-corrects better when it knows why |
| 19 | Per-user hand profile: per-finger-pair limits + grip rules, hard or soft, learned via "doesn't work for me" | Hands differ; my limits aren't Holdsworth's |
| 20 | Hand-profile rules are enforced by the algorithm, not a decision model | Deterministic, explainable constraints |
| 21 | Clickable chord symbols cycle interpretations and re-color notes | Seeing one line from several roots builds understanding |
| 22 | LLM gives the default chord name; code enumerates and validates alternatives | LLMs know jazz naming conventions; code keeps it correct |
| 23 | Hand profile starts from researched average-hand defaults; no calibration step | Get playing immediately; learn from "feels hard" |
| 24 | Session = one rich prompt → short practice-sized piece → ~30 min practice → conversational refinement | That's how I actually practice |
| 28 | IR = own JSON; LLM writes a compact authoring form, code expands to canonical IR | Pitch-first, IDs, harmony lanes, provenance don't fit alphaTex; structured output catches errors |
| 29 | Pitches spelled, time as exact rationals in whole notes | Correct chord spelling; exact polyrhythms in any meter |
| 30 | Harmony = two lanes (backing, implied), each with switchable readings | Powers interval coloring + interpretation switching |
| 31 | Revisions are immutable snapshots; edits are ops on stable note IDs | Conversational refinement + selections that survive edits |
| 27 | Techniques plug into the solver later; v1 is technique-agnostic | Tapping etc. change what's playable; leave the seam now |
| 26 | Output length is flexible, not fixed; always practice-sized, never long-form | Small chunks to drill, not 10 minutes of music |
| 25 | v1 content = notes first; rhythm representable but simple; no expression | Keep scope tight |

## Rejected / Deferred
| Idea | Why not (yet) |
|------|---------------|
| Transcription-based learning | Too slow and grindy; not how I learn |
| Theory-first / "hear it first" pedagogy | Deliberately inverted: play first, ear follows |
| Unconstrained LLM generation | Too random; constraints give better ideas |
| Plain ASCII tab | No rhythm info |
| alphaTex as the IR | Fret-first, no spelled pitch/harmony/IDs/provenance; kept as export only |
| Adopting alphaTab as the renderer | Want a first-class, tab-only renderer with interval coloring; alphaTab stays a reference |
| Code-only fingering (LLM emits pitches only) | Loses idiomatic shapes; kept as a fallback only |
| Phone / iPad | Not where I practice |
| Rhythm/drum backing | Later; harmony comes first |
| Taste learning ("not a fan of this sound" → avoid that resolution next time) | Needs rules that reason about voice leading and harmony, accumulated over time. Much bigger than playability. Capture the feedback now, act on it later |
| Up-front hand calibration exercise | Defaults + learning from feedback is enough |
| Expression (dynamics, vibrato, bends) | Notes first |

## Research & experiments
| Topic | Status | Finding |
|-------|--------|---------|
| Guitar fingering algorithms & playability metrics | Done | [`research/fingering-algorithms.md`](research/fingering-algorithms.md). Viterbi/DP over positions with a hand-made cost is the proven core; nothing reusable as a validator (write a small TS module; `knoguchi/fretpath` GPL-3.0 as reference). Layered checks: structural → hard physical → hard temporal → soft cost. Repair re-solves only failing windows, pinned to the LLM's valid choices. JSON diagnostics: rule, measured value, limit, 1-3 verified alternatives. Polyphony = timeline where held notes keep string+finger. Literature assumes standard tuning and given pitches; legato/crossing/voice thresholds must be calibrated on my hands. |
| IR design | Done (draft, not yet compiled) | [`research/ir.md`](research/ir.md). Canonical `Idea → Revision → Piece`: flat notes with stable IDs, exact rational time in whole notes, spelled pitches (C♯ ≠ D♭). Fingering = one optional field with `hand` + provenance (`by: llm/solver/user`). Voices explicit; chords = voices sharing an onset. Techniques = typed per-note `attack`. Harmony = backing + implied lanes, each span with several verified readings + default + user pick; note intervals derived. Revisions immutable; LLM edits via ops (`replace`, `patch`). LLM emits a smaller bar-by-bar **authoring form**; code expands + runs structural checks. Own JSON; alphaTex = export only. |
| Fingering solver prototype | Done | Branch `prototype/fingering-solver` @ `f909abf`, worktree `/Users/texoport/.t3/worktrees/tabbycat/t3-a1dbb9d0-d7ed292f2b0c2794`. TS 7 (tsgo) + bun; 59 tests pass (verified). Any tuning/string count; rational-time IR; hand profile in mm; only 2 hard-impossible rules; layered validator; diagnostics with verified fixes; window-local repair; trace CLI (`bun run trace <fixture>`) + `ALGORITHM.md`. Gaps: no technique awareness (Holdsworth legato gets flattened to open strings), cost weights are placeholders, fixes not grouped by root cause, no thumb/lateral reach. |
| Average-hand defaults for the hand profile | Done | [`research/hand-defaults.md`](research/hand-defaults.md). Limits in **mm**, not frets: fret span = `L·(2^(−a/12) − 2^(−b/12))`, so the same hand reaches ~4 frets at fret 1 and ~6 at fret 9. Default profile per finger pair as comfortable/hard/max mm (e.g. index-little 95/125/155), based on ANSUR II reference hand 187×83 mm. Weakest pairs: middle-ring, ring-little (matches me). Radicioni max-span in frets: I-M 2, I-R 3, I-L 4, M-R 0, M-L 3, R-L 1 (authors say M-R, R-L too strict). Numbers are calibrated guesses, not measured: tune on my hands. Best one-question calibration: max comfortable index-little span at fret 1. |
| Decision models ("Jev by Type Safe AI", System 1, alternatives) | Done | [`research/decision-models.md`](research/decision-models.md). Jev by TypeSafe AI (~95% sure). "System One" = TypeSafe's label for one-pass decision models returning typed probabilities over `choice`/`score`/yes-no questions; can't return invalid options but can be confidently wrong; no reliable arithmetic (music math stays in code). Alternatives: Cloudflare Clef / Clef-Flash (Workers AI, Jev-compatible), OpenAI Decisions API, OpenRouter, Vercel AI SDK provider; open-weight Laya, xor, GLiNER2.5-Decide. Recommendation: one Effect `Decide` service, Clef-Flash via Workers AI binding, Jev layer as swap-in. Unverified: Jev GA status, Jev SDK on workerd, Clef preview status. |
| Stack: FoldKit + blocks, Effect v4, Effect Agent, Alchemy/DOs, TS 7 | Done | [`research/stack.md`](research/stack.md). FoldKit 0.170 (pre-1.0, weekly releases; Snabbdom VDOM, SVG + `Canvas.view`, `ManagedResource` for AudioContext). Blocks = PR #1655/#1860, big breaking rewrite, no date. Effect 4.1.x stable; use `@effect/ai-anthropic@4.x`. "Effect Agent" = Yielded Agent (`@yielded/agent` beta, ~6 wks old; Schema output, tools, DO host; no built-in repair loop). Alchemy 2.0 beta: DOs supported; `@alchemy.run/domain` is an event-sourced DDD framework on DOs (very new). TS 7.0 stable but no plugin API: use `@effect/tsgo`; Blocks preview hits TS2589 on native TS. |
