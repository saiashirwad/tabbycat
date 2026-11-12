# Hand-profile defaults for an average adult hand

Scope: default per-finger-pair stretch limits and grip rules for the tabbycat fingering solver. Everything is **tuning- and string-count-independent**: limits are in **mm** on the fingerboard plus per-instrument geometry (scale length, string spacing), never in "strings" or pitch.

**Evidence legend:** `[V]` verified by reading the source text or data. `[S]` secondary/summary only. `[own]` my engineering proposal, no literature. **No published guitar study gives per-pair stretch limits in mm.** Section 5 numbers are calibrated defaults, not measurements. Treat them as a starting point to tune against the user's own hands.

---

## 1. Fret spans per finger pair, and how they change up the neck

### 1.1 Fret-width model `[V]` (pure geometry)

Equal temperament, scale length `L` (nut to bridge saddle), fret `n`:

```
x(n)      = L · (1 − 2^(−n/12))          distance from nut to fret wire n
w(n)      = x(n) − x(n−1)
          = L · (1 − 2^(−1/12)) · 2^(−(n−1)/12)
          = L · 0.056126 · 2^(−(n−1)/12) = (L − x(n−1)) / 17.817
span(a,b) = x(b) − x(a) = L · (2^(−a/12) − 2^(−b/12))     wire-to-wire mm, a < b
```
Source: https://liutaiomottola.com/formulae/fret.htm (17.817 = 1/0.056126). Fret width falls by a factor of 2^(1/12) ≈ 5.9% per fret, and halves every 12 frets.

Cross-check: Heijink & Meulenbroek say the interfret space "ranges from 12.9 to 36.5 mm on a classical guitar" `[V]`. For L = 650 my formula gives w(1) = 36.5 and w(19) = 12.9. So their guitar is 650 mm with 19 frets.

| fret n | w(n), L=650 | L=628 | L=648 (25.5") | 
|---|---|---|---|
| 1 | 36.5 | 35.2 | 36.4 |
| 5 | 29.0 | 28.0 | 28.9 |
| 9 | 23.0 | 22.2 | 22.9 |
| 12 | 19.3 | 18.7 | 19.3 |
| 15 | 16.3 | 15.7 | 16.2 |
| 19 | 12.9 | 12.5 | 12.9 |
| 24 | 9.7 | 9.3 | 9.6 |

Wire-to-wire span in mm covered by *k* frets starting at fret *a* (L = 650, computed):

| start fret a | 1 fret | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|
| 1 | 34 | 67 | 98 | 127 | 154 | 180 |
| 5 | 27 | 53 | 77 | 100 | 122 | 143 |
| 9 | 22 | 42 | 61 | 80 | 97 | 113 |
| 12 | 18 | 35 | 52 | 67 | 82 | 95 |
| 15 | 15 | 30 | 43 | 56 | 69 | 80 |

**Consequence:** a hand that manages a 4-fret index-little span (127 mm) at fret 1 manages about 6 frets at fret 9 for the same mm. So a fixed fret limit is too strict high on the neck and too loose in first position. **Use mm limits and convert the fingers' fret positions to mm with `span(a,b)`.**

### 1.2 What the literature actually gives in frets

**Radicioni & Lombardo 2005, Table 1 `[V]`** (read from the eScholarship PDF text, https://escholarship.org/content/qt7cp5z258/qt7cp5z258.pdf). Fingers numbered index = 1 ... little = 4. "Max span over finger pairs: for each finger pair, there exists a maximum span that can never be exceeded." Verbatim table layout, row = finger, column = finger:

| maxSpan | index | middle | ring | little |
|---|---|---|---|---|
| little | 4 | 3 | 1 | - |
| ring | 3 | 0 | - | |
| middle | 2 | - | | |

Decoded:

| pair | 1-2 | 1-3 | 1-4 | 2-3 | 2-4 | 3-4 |
|---|---|---|---|---|---|---|
| Radicioni max (frets) | 2 | 3 | 4 | **0** | 3 | **1** |

Notes:
- This **confirms** the values read (with doubt) in `fingering-algorithms.md` §2: I-M 2, I-R 3, I-L 4, M-R 0, M-L 3, R-L 1.
- **Indexing is not defined in the paper.** Footnote 1 only says frets are numbered from I to XVII and fingers 1-4 from index to little. The most plausible reading is **fret-number difference** (index on fret 1, little on fret 5 = 4), `[own inference]`. It matches Heijink's "four frets" index-little span (below). A value of 0 for middle-ring then means both fingers on the same fret.
- A minimum-span table exists but "is not reported here for space reasons".
- The authors say the **middle-ring 0 and ring-little 1 are too strict**: "The 6 missed fingerings are due to a heavy restriction of the Max span constraint, which forbids large stretches between the middle and ring fingers although these seem realizable by human experts." They also admit the model "does not take into account the diminishing width of frets towards the body end... thus making the difficulty associated with stretching constant all along." `[V]` This is exactly why we want mm.
- Source of values: classical handbooks (Aguado, Duarte, Giuliani, Sor) plus the authors' experience. No measurements. `[V]`

**Heijink & Meulenbroek 2002 `[V]`** (read the PDF text, https://www.socsci.ru.nl/meulenbroek/Publications/Heijink%20en%20Meulenbroek%202002.pdf). Six professional classical guitarists, scale fingerings at index-little span of **four frets ("small") vs five frets ("large")**, with the first note on fret 1, 4 or 8, at 5 notes/s. Large span rated harder than small. High position (fret 8) rated harder than low or middle (low and middle judged about equal). Hand repositioning was the strongest complexity factor, then span, then position. The authors' reason: joints are most comfortable in mid-range, so large spans and extreme positions cost more. So **4 frets = ordinary, 5 frets = demanding but playable by professionals at tempo**, at 650 mm in 1st-8th position.

**Others:** Sakai et al. 2023 allow span ≤ 3 in chord states `[P per existing doc]`. Parncutt et al. 1997 (piano) give a six-level span table per finger pair `[V]`, see 1.3.

### 1.3 Parncutt's piano table as a structural template `[V]`

https://static.uni-graz.at/fileadmin/_Persoenliche_Webseite/parncutt_richard/Pdfs/PaSlClRaDe97_FingeringModel.pdf, Table 1 (semitones, right hand, "average adult male pianist"; values "determined in an ad hoc manner"):

| pair | MinRel | MaxRel | MaxComf | MaxPrac |
|---|---|---|---|---|
| 2-3 (index-middle) | 1 | 2 | 3 | 5 |
| 2-4 (index-ring) | 3 | 4 | 5 | 7 |
| 2-5 (index-little) | 5 | 6 | 8 | 10 |
| 3-4 (middle-ring) | 1 | 2 | 2 | **4** |
| 3-5 (middle-little) | 3 | 4 | 5 | 7 |
| 4-5 (ring-little) | 1 | 2 | 3 | 5 |

- Ratios: MaxComf ≈ 0.6-0.8 × MaxPrac. MaxRel ≈ 0.4-0.6 × MaxPrac.
- **Middle-ring has the smallest practical span** (4 vs 5 for the other adjacent pairs). Same pair Radicioni made most restrictive. Both are classical-pedagogy based.
- Rules: Stretch Rule = 2 points per semitone beyond MaxComf (or below MinComf). Large-Span Rule = 2 points per semitone beyond MaxRel for non-thumb pairs. Weak-Finger Rule = 1 point per use of finger 4 or 5. Three-Four-Five Rule = 1 point for each run of three weak-side fingers in a row. Quote: the table "may be adjusted to suit different hand sizes", "all aspects of the model based on physical distance" are adjustable. This is the template for our soft cost.

### 1.4 Position-dependence: what to model

1. **Geometry (hard fact):** convert fret pairs to mm. Limits are mm, so higher on the neck the same limit allows more frets. `[V]`
2. **Hand-position cost (soft):** Heijink: cost lowest mid-neck, higher toward the extremes, with high position (fret 8+ on 650 mm) rated clearly harder than low `[V]`. Keep it a separate cost term (already in `fingering-algorithms.md`), not part of the span limit.
3. **Narrow-fret floor (soft) `[own]`:** adjacent fingers on adjacent frets need centre-to-centre room. Below about 12-13 mm per fret (fret ≥ 19 on 650 mm, per the table above) fingers must roll or overlap. Default: soft cost when `w(n) < 13 mm` and two different fingers press frets n−1 and n. This is the **minimum-span** side that Radicioni left unpublished. Fingertip width is typically ~14-18 mm `[unverified]`.
4. **Fret-count cap `[own]`:** mm limits alone permit absurd fret counts at fret 15+ (150 mm = 11 frets). Add a per-pair fret-count cap as a second impossible limit. See 5.

---

## 2. Across strings, barre and thumb-over

### 2.1 Cross-string geometry

- Lateral distance between strings *i* and *j* at fret n: `|i−j| · spacing(n)`, where `spacing(n) = s_nut + (s_bridge − s_nut) · x(n)/L`. Linear taper is a geometric approximation `[own]`.
- Typical string spacing (centre to centre), from manufacturer figures in a search `[S]`: electric 6-string, Strat-style nut 41.9 mm wide with ~7.1 mm spacing. Classical 52 mm nut with ~11.8 mm spacing at the bridge side; 48 mm nut with 10.6 mm. Bridge spacing ~51-58 mm total for 6 strings. Nut and bridge figures were mixed up in the sources, so **take spacing as a per-instrument input**, not a constant. Defaults: nut 8 mm, bridge 10.5 mm `[own]`.
- **Do not use string counts as the unit.** Use `Δstring × spacing` in mm. A 7- or 8-string guitar then works automatically (narrower spacing, same hand).

### 2.2 Combining axial and lateral reach `[own]`

Finger separation along the neck is limited by finger abduction (spread). Separation across strings is limited by finger flexion/extension and hand angle, which is easier. Proposed effective distance for a finger pair:

```
d_eff = sqrt( Δaxial_mm² + (λ · Δlateral_mm)² )     λ = 0.5 default (user-tunable)
```
Compare `d_eff` with the pair limits. At fret 1, a 5-string-gap lateral reach is ~40 mm, so it adds <10% to a 127 mm axial span but dominates when the axial span is 0 (same fret). Cross-string-only reach at the same fret is normally fine: open G (middle on low string, ring/little on the top strings, all on fret 3) is a standard shape `[own, common knowledge]`.

### 2.3 Barre `[own, with anatomy support]`

- Radicioni: barre only with the index finger; all barre positions on the same fret; all other positions on higher frets `[V]`. Keep as a hard rule, with a per-profile option to allow other-finger barres.
- **Index barre reach:** the index finger bone chain is ~78 mm (pd 15.8 + pm 22.4 + pp 39.8, Buryanov & Kotiuk `[V]`, see 3.2), so the whole fingerboard width (45-55 mm over 6 strings) is within reach. Full barre across all strings is allowed up to a lateral width of ~55 mm (soft cost above that, scaled by hand length). Barre fatigue and string-count effects are a **soft cost, not a ban**.
- **Partial barres:** teaching sources say partial barres (2-3 strings) are a standard stepping stone `[S]`: https://classicalguitarshed.com/half-barre-chord/ . Mini-barre by a non-index finger: contact length ≈ flattened distal phalanx + tip ≈ 20-22 mm (pd 15.8-17.4 + tip 3.8-4.0, Buryanov `[V]`), so covers ~`1 + floor(21 / spacing)` strings: **3 strings at 8-10 mm spacing, 2 strings at 12 mm** `[own derivation]`. Default: middle/ring may cover up to 3 adjacent strings with a soft cost for 3, little up to 2.
- The barre finger counts as one finger in the finger-order rule, so the barre must be on the lowest fret in the grip. Combine with the "no other finger on a lower fret" rule.

### 2.4 Thumb-over `[S]`

- History: Carulli's later edition uses it "on the sixth string and sometimes on the fifth"; von Held paired it with barre chords; the mainstream classical tradition (Sor onward) rejected it. Molino: "very uncomfortable for those with small hands". Rusanov: only "when it is not possible to perform without it". https://www.digitalguitararchive.com/2020/12/a-short-history-of-the-use-of-the-left-hand-thumb/
- In rock/blues it is routine (Hendrix grip on the lowest string `[own, common knowledge]`).
- Proposed default `[own]`: thumb can fret **only the lowest 1 string (2 if user enables), at frets 1-7**, with the other four fingers' hard spans reduced by ~1 fret-equivalent while the thumb is over. Off in a "classical" preset. Soft cost 3-5 points per use, higher for small hands. Thumb is **not** counted as finger 5 in the span table.

---

## 3. Hand anthropometrics and scaling

### 3.1 Hand dimensions `[V]` computed from raw data

ANSUR II (US Army, 2010-12, 4,082 men, 1,986 women). I downloaded the public CSVs from https://tools.openlab.psu.edu/publicData/ANSUR_II_MALE_Public.csv and `..._FEMALE_Public.csv` (linked from https://www.openlab.psu.edu/ansur2/) and computed the stats (mm). Hand length = wrist crease to middle fingertip; hand breadth = across metacarpals; palm length = wrist crease to proximal finger crease.

| mm | sex | mean | SD | P5 | P50 | P95 |
|---|---|---|---|---|---|---|
| Hand length | M | 193.3 | 9.9 | 177 | 193 | 210 |
| | F | 181.1 | 10.1 | 165 | 180 | 199 |
| | **50/50 mix** | **187.2** | ~11.7 | ~168 | ~187 | ~206 |
| Hand breadth | M | 88.3 | 4.4 | 81 | 88 | 96 |
| | F | 78.2 | 3.8 | 72 | 78 | 85 |
| | **50/50 mix** | **83.3** | ~6.5 | ~72 | ~83 | ~94 |
| Palm length | M | 116.5 | 6.2 | 107 | 116 | 127 |
| | F | 108.7 | 5.9 | 100 | 108 | 119 |
| Middle finger ≈ hand − palm length | M / F | 76.8 / 72.4 | 6.3 / 6.0 | | | |

- 50/50 mix is my arithmetic (equal weights; SD adds the between-sex term). ANSUR is a young, fit, US-military sample, so it skews slightly large. Sanity: other adult datasets agree within ~5% (Colombian adults: hand length 183 mm M / 166 mm F median, breadth 85 / 75 mm; the 2016 comparison table lists ~188 / 177 mm length and 86 / 80 mm breadth) `[S]` from a search snippet, https://pmc.ncbi.nlm.nih.gov/articles/PMC10308009/table/T1 and https://pmc.ncbi.nlm.nih.gov/articles/PMC9225906/table/tab3.
- ANSUR II public data has **no per-finger lengths**, only hand length, breadth, palm length, circumference, thumb-tip reach.
- Within sex, length and breadth correlate only r ≈ 0.57-0.58 (computed). So **breadth and length are not interchangeable**; breadth matters for lateral spread and a classical-guitarist remark that hand width matters more than finger length for stretches is anecdotal `[S]`.

### 3.2 Finger proportions `[V]`

Buryanov & Kotiuk 2010, *Int. J. Morphol.* 28(3) (X-ray of 66 adults, sexes pooled, right hand; text read from https://scielo.conicyt.cl/pdf/ijmorphol/v28n3/art15.pdf). Bone lengths in mm: tip = soft-tissue tip.

| finger | tip | distal | middle | proximal | metacarpal | MCP-to-tip (sum without metacarpal) | ratio to middle finger |
|---|---|---|---|---|---|---|---|
| index (II) | 3.8 | 15.8 | 22.4 | 39.8 | 68.1 | 81.8 | 0.89 |
| middle (III) | 4.0 | 17.4 | 26.3 | 44.6 | 64.6 | 92.3 | 1.00 |
| ring (IV) | 4.0 | 17.3 | 25.7 | 41.4 | 58.0 | 88.3 | 0.96 |
| little (V) | 3.7 | 16.0 | 18.1 | 32.7 | 53.7 | 70.5 | 0.76 |

MCP-to-tip sums are my arithmetic. A pooled-sex sample, so use the **ratios** and scale by the user's hand length.

### 3.3 Finger spread (abduction) `[S]`

Erdogan & Bakirci 2023, *Medicine Science* (photo angles at maximal spread, right hand; from a search summary): index-middle 20.0° (M) / 20.8° (F), middle-ring 17.3 / 18.2, ring-little 27.7 / 29.4, thumb-index 53.9 / 48.4. A second dataset (*Anat Cell Biol* 57(2)): 17.2, 14.3, 32.1, 53.4. Take-away: **middle-ring has the least spread, ring-little the most angular spread** but the little finger is short. https://synapse.koreamed.org/articles/1516087497

**Independence:** Häger-Ross & Schieber 2000, *J. Neurosci.* 20(22): the middle and ring fingers were the least individuated, thumb/index/little more so `[S]`: https://www.jneurosci.org/content/jneuro/20/22/8542.full.pdf . Parncutt cites the juncturae tendinum linking fingers 3-4-5 and Ortmann: "no amount of practice can overcome entirely this physiological limitation" `[V]`.

**Cross-check of my mm defaults `[own, rough]`:** index/middle/ring/little MCP spaced evenly over the hand breadth (83 mm), abducted at the angles above with the Buryanov finger lengths, gives a *passive flat-hand* tip-to-tip spread of roughly I-M 56, M-R 54, R-L 52, I-R 109, M-L 106, I-L 161 mm. My guitar "max" values below (65, 65, 65, 115, 115, 155) are within ~15% of these, except that the real adjacent-pair guitar reach is somewhat *higher* than flat abduction (the hand pivots and fingers angle). Use this only as a sanity check on the proportions, not as a measurement.

### 3.4 Scaling defaults to a user

Reference hand: **hand length 187 mm, breadth 83 mm (50/50 adult mix)**. All profile numbers below are at this reference. Scale:

```
k_len = handLength / 187          k_wid = handBreadth / 83
axial limit (mm) = base × (0.5·k_len + 0.5·k_wid)       if both entered
                 = base × k_len                          if only length entered
lateral/barre limit = base × k_wid
```
Rationale: stretch limits are lengths, and pure isometric scaling by one dimension is the simplest defensible model. Error is about ±10% for one measurement (r ≈ 0.57 between length and breadth within sex) `[own]`.

**Best one-number calibration (recommended, cheap for the user):** ask for the **largest index-to-little span they can comfortably play in first position**, as a number of frets (default 4 at 650 mm), then set `I-L.hard = span(1, 1+n)`, `I-L.comfortable = 0.75 × that`, and scale the other pairs by the same ratio `k = I-L.hard / 125`. This directly encodes real hand + real guitar and beats an anthropometric guess. Optional second entry: **max stretch frets for middle-ring** (the weakest, most variable pair) to override M-R and R-L.

Optional measurement: flat-hand thumb-to-little span is a common pianist measure (~226 mm M / 200 mm F averages `[S]`, https://paskpiano.org/measuring-hand-spans) but no source gives the 2-5 span. Do not ask for it.

---

## 4. Hard grips in pedagogy, and the weakest pairs

All pedagogy items below are `[S]` (blogs and forums), with the anatomy backing from sections 1.3 and 3.3.

- **Ring-little (3-4) is the usual trouble pair.** Common claim: they "share a single tendon", which is imprecise (it is the juncturae tendinum), but the effect is real. The little finger wants to "fly" and spreads badly from the ring finger ("snakebite"): https://classicalguitarshed.com/left-hand-little-finger/ . The common **D-chord pinky stretch** is "quite difficult at first": https://songnotes.net/lessons/378 .
- **Middle-ring** is the least independent pair and has the smallest angular spread (3.3). Radicioni and Parncutt both restrict it most, but Radicioni's own results show the restriction was too tight `[V]`.
- **Index-little** is the largest absolute reach and the one pedagogy gives a number for: roughly four frets in first position for most players. "Some players struggle to span four frets on all six strings": https://community.justinguitar.com/t/beginner-finger-stretches/2279?page=3 . Reaching the little finger is also limited by the index joint pressing the neck, so the hand must keep finger joints parallel to the neck `[S]`.
- **Hardest grips in practice** (all `[own, common guitar knowledge]` unless linked): 4- and 5-fret index-little stretches in low positions; middle-ring skips (two frets apart); ring-little two frets apart (the D-chord pinky stretch above).
- **Scale length matters little:** one report says going from 650 to 640 mm cuts the stretch by only 1-2 mm per fret; **nut width and string spacing matter as much** `[S]`: https://classicalguitarcorner.com/guitar-for-small-hands . Study evidence that hand size predicts guitar ability is weak and unverified `[S]`.
- **Ranking, easiest to hardest** (my synthesis): I-M ≈ I-R < M-L < I-L(large) ≈ M-R(2-fret) < R-L(2-fret). Use the profile's pair-weight vector to let users reorder.

---

## 5. Proposed default profile

Reference hand 187 × 83 mm, reference instrument L = 650 mm. Distances are **wire-to-wire along the neck** between the frets the two fingers sit on, i.e. `span(a, b)`; frets in brackets are the equivalent count at frets 1-5 (≈ 31.7 mm per fret). Levels:
- `comfortable`: no span cost up to here.
- `hard`: allowed but costed steeply (quadratic) beyond; this is the "demanding grip" level.
- `max`: infinite cost (impossible for this hand). A "stretch" technique flag raises `hard` and `max` by 10%.

Calibration: `hard` = Radicioni's max (frets) converted to mm for I-M, I-R, I-L, M-L (2, 3, 4, 3) and = Heijink's "small" span for I-L. `max` = one fret beyond Radicioni for I-L (5 = Heijink "large") and roughly +0.5 to +1 fret for the others. M-R and R-L are **deliberately looser than Radicioni's 0 and 1** (the authors report this was too strict) but kept the tightest. All `[own]` calibration of those numbers.

```jsonc
{
  "version": 1,
  "reference": { "handLengthMm": 187, "handBreadthMm": 83 },   // 50/50 ANSUR II mix

  "pairs": {                       // finger 1=index 2=middle 3=ring 4=little
    "1-2": { "comfortableMm": 35, "hardMm": 65,  "maxMm": 80,  "maxFrets": 3, "weight": 1.0 },
    "1-3": { "comfortableMm": 65, "hardMm": 95,  "maxMm": 115, "maxFrets": 4, "weight": 1.0 },
    "1-4": { "comfortableMm": 95, "hardMm": 125, "maxMm": 155, "maxFrets": 6, "weight": 1.2 },
    "2-3": { "comfortableMm": 30, "hardMm": 50,  "maxMm": 65,  "maxFrets": 3, "weight": 1.5 },
    "2-4": { "comfortableMm": 65, "hardMm": 95,  "maxMm": 115, "maxFrets": 4, "weight": 1.3 },
    "3-4": { "comfortableMm": 30, "hardMm": 50,  "maxMm": 65,  "maxFrets": 3, "weight": 1.8 }
  },
  // frets @ref:  1-2: 1 / 2 / 2.5   1-3: 2 / 3 / 3.6   1-4: 3 / 4 / 5
  //              2-3: 1 / 1.6 / 2   2-4: 2 / 3 / 3.6   3-4: 1 / 1.6 / 2
  // The span is also bounded below by the fret-count cap (maxFrets), since mm limits alone
  // allow 8+ frets above fret 12. Order pairs by anatomy: min span for adjacent fingers is
  // 1 fret (fingers may share a fret only for barre / same-fret chords).

  "minSeparation": {                // narrow-fret floor
    "fretWidthMm": 13,              // below this adjacent-fret finger pairs get a cost
    "costPerMmBelow": 0.5
  },

  "lateral": {                      // across strings
    "lambda": 0.5,                  // d_eff = hypot(axial, lambda * lateral)
    "stringSpacingNutMm": 8.0,      // per-instrument input; override from the instrument
    "stringSpacingBridgeMm": 10.5
  },

  "barre": {
    "fingers": [1],                 // Radicioni: index only
    "maxLateralMm": 55,             // soft cost above (≈ 6 strings @ 9-10 mm)
    "partialByOthers": { "2": 3, "3": 3, "4": 2 }  // max contiguous strings; 3rd string costs extra
  },
  "thumbOver": {
    "enabled": false,               // true in rock/blues presets
    "lowestStrings": 1,             // 2 if the user opts in
    "maxFret": 7,
    "cost": 4,
    "otherFingersHardSpanScale": 0.8
  },

  "fingerStrength": {               // cost per use of the finger (Parncutt Weak-Finger Rule)
    "1": 0.0, "2": 0.1, "3": 0.5, "4": 1.0,
    "streak345": 1.0                // extra when ring+little are used in a run of three
  },

  "costWeights": {
    "stretchBeyondComfortable": 1.0, // per mm over comfortable, up to hard (linear)
    "stretchBeyondHard": 4.0,        // per mm over hard (quadratic term)
    "positionExtremes": 0.5,         // Heijink: convex penalty away from mid-neck
    "narrowFret": 0.5,
    "crossStringOnly": 0.2           // lateral reach adds little
  },

  "hardLimits": [                    // only these are infinite cost
    "pair.maxMm exceeded (after scaling and stretch flag)",
    "pair.maxFrets exceeded",
    "finger order violated, unless thumbOver or crossing flag",
    "barre on non-index finger when barre.fingers = [1]",
    "more fretted fingers than fingers (4 + thumb if enabled)"
  ]
}
```

### What to make user-tunable (in order of value)

1. **Largest comfortable index-little span in frets** at fret 1 (one number; scales the whole table, see 3.4). Highest value, lowest effort.
2. **Hand length, optionally breadth** (mm) as an alternative to 1.
3. **Middle-ring / ring-little hard limits** (weakest, most variable pairs).
4. **Thumb-over** on/off and string count.
5. **Instrument geometry**: scale length, fret count, nut width / string spacing (comes from the `Instrument`, not the hand profile; keep them out of `HandProfile`).
6. **Stretch tolerance** (per-passage flag): raises `hard`/`max` by ~10% for deliberate wide-stretch lines.
7. **Weights**: per-pair weight vector. Fit from accept/reject feedback later (Radisavljevic path-difference learning).

### Independence from tuning and string count

- Limits use only fret positions, mm and per-instrument spacing. `span(a,b)` needs `L`; `lateral` needs `stringSpacing*`. No pitch or string-count constants.
- For 7-8 string guitars, spacing shrinks and cross-string cost falls. A 4-string bass (34" = 864 mm scale) has wider frets (`w(1)` ≈ 48.5 mm). The mm limits stay the same, and the number of frets in reach shrinks automatically. That is the point of using mm. `maxFrets` is a separate cap and does not need rescaling.
- All-fourths (E A D G C F) changes the *shapes* needed for a chord or scale but not the hand: the same span limits apply. The reachable fingerings differ, which the solver handles from pitch, not from the profile.

---

## 6. Gaps and caveats

- **No guitar-specific measured data** for per-pair stretch in mm. The `hard` column rests on two classical-guitar sources of different quality: Radicioni (handbook-derived, "can never be exceeded", flagged too strict) and Heijink (six professionals, 4 vs 5 frets). The `comfortable` and `max` columns are my interpolation using Parncutt ratios.
- **Radicioni's fret-difference indexing is inferred**, not stated. Treat my reading ("difference between fret numbers") as likely but unconfirmed. The 2007 *Constraints* journal paper by the same group may define it and has the minimum-span table; I did not obtain it.
- **Hand data is military (ANSUR II)**, young and fit. The 50/50 mix is not a population mix. SD for the mix is approximate.
- ANSUR II has no finger-length data. Finger proportions are from a 66-patient X-ray study, sexes pooled.
- Abduction angles and Häger-Ross findings are from search summaries `[S]`, not the full texts. The anatomical cross-check in 3.3 is my model.
- Pedagogy claims in section 4 come from blogs and forums. The tendon-sharing claim is a simplification.
- Thumb-over limits (lowest string(s), fret ≤ 7), the lateral weight λ = 0.5, partial-barre string counts and the narrow-fret floor (13 mm) are `[own]` and untested against real players. **Calibrate with the user's own hands** (the all-fourths 6-string player) before trusting defaults.
- Position cost beyond "low/middle ≈ equal, high harder" is not quantified by Heijink. I did not extract the per-position ratings.

## Sources

- Radicioni & Lombardo 2005 (Table 1, max spans): https://escholarship.org/content/qt7cp5z258/qt7cp5z258.pdf
- Heijink & Meulenbroek 2002: https://www.socsci.ru.nl/meulenbroek/Publications/Heijink%20en%20Meulenbroek%202002.pdf
- Parncutt et al. 1997 (span table, rules): https://static.uni-graz.at/fileadmin/_Persoenliche_Webseite/parncutt_richard/Pdfs/PaSlClRaDe97_FingeringModel.pdf
- Fret formula: https://liutaiomottola.com/formulae/fret.htm
- ANSUR II data: https://www.openlab.psu.edu/ansur2/ (CSVs at https://tools.openlab.psu.edu/publicData/)
- Buryanov & Kotiuk 2010: https://scielo.conicyt.cl/pdf/ijmorphol/v28n3/art15.pdf
- Abduction angles: https://synapse.koreamed.org/articles/1516087497
- Finger individuation (Häger-Ross & Schieber 2000): https://www.jneurosci.org/content/jneuro/20/22/8542.full.pdf
- Pianist spans: https://paskpiano.org/measuring-hand-spans
- Little-finger technique: https://classicalguitarshed.com/left-hand-little-finger/
- Left-hand thumb history: https://www.digitalguitararchive.com/2020/12/a-short-history-of-the-use-of-the-left-hand-thumb/
- Small hands and scale length: https://classicalguitarcorner.com/guitar-for-small-hands
- Partial barres: https://classicalguitarshed.com/half-barre-chord/
- Hand-size tables (snippets): https://pmc.ncbi.nlm.nih.gov/articles/PMC10308009/table/T1, https://pmc.ncbi.nlm.nih.gov/articles/PMC9225906/table/tab3
