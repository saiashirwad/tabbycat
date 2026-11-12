# Decision models for tabbycat

Researched 2026-11-12. Marked **[unverified]** where I only found secondary sources or nothing. Many sources are blogs/aggregators; primary docs cited where I reached them.

## 1. What the user meant

**Jev, by TypeSafe AI.** Confidence ~95%. "Type Safe AI" → TypeSafe AI; "System 1" → TypeSafe's own category name, "System One models".

Evidence:
- TypeSafe's launch post: Jev is "the first System One Model", "unstructured state in, typed probabilistic decisions out" — https://typesafe.ai/blog/introducing-system-one-models-and-jev
- InfoQ coverage (Oct 2026): TypeSafe AI is a San Francisco lab; Jev returns typed probabilities, not text — https://www.infoq.com/news/2026/10/typesafe-ai-jev-released/
- Curated ecosystem list calls these "System One models (also called decision models)" — https://github.com/andyrewlee/awesome-system-one
- Survey paper (single author, arXiv 2610.33775): System 1 models map input to a typed decision in one forward pass — https://arxiv.org/abs/2610.33775

Not the match:
- **BAML (BoundaryML)** is a typed-prompt language that makes an LLM *generate* structured output. Different category, no decision-model features in its docs — https://github.com/BoundaryML/baml
- Open model **GEV-26B-Decide** (Apache 2.0) is the only "Gev" I found. It's a Jev-style model, so the user might have meant it, but "Jev by TypeSafe" is far more likely — https://huggingface.co/autotrust/GEV-26B-Decide

"System 1" is **not a product**. It's the Kahneman-derived label for fast, single-pass, classifier-like models.

## 2. What these things do

| Category | What it does | Examples |
|---|---|---|
| **Structured generation** | LLM writes free text; tooling coerces/validates it into a schema. Slow, flexible. | BAML, Effect AI `generateObject`, Vercel AI SDK |
| **Decision / System One models** | You send `state` + typed `questions`. One forward pass returns a *probability distribution over your options*. Can't invent an option or return an invalid type. Can't write strings. | Jev, Clef, OpenAI Decisions API |
| **Classic small classifiers** | Encoder or embedding model with fixed labels or trained heads. | SetFit, GLiClass, semantic-router |

Jev question types (docs: https://docs.typesafe.ai, SDK: https://github.com/typesafe-ai/typesafe-sdk-js):
- `choice`: pick one of 1–255 options → choice, per-option probabilities, confidence.
- `score`: rate on an ordered rubric (1–10 levels in the AI SDK provider) → fractional score.
- `noul`: yes/no → P(true). Called `boolean` in the AI SDK and `predicate` in OpenAI's API.
- `box`: image region (experimental; irrelevant here).

All questions run in parallel and in isolation against the same `state`.

Claims and caveats:
- Latency: TypeSafe says 70–500 ms. A user-reported median of 76 ms (OpenChamber via InfoQ) is real-world data. Headline "193x faster" is vendor marketing; independent median was ~7x. Cost: $0.042/M input tokens, output free.
- Availability: launched as early access Sept 15, 2026; Jev 1.14 shipped Oct 14. Weights unpublished. Whether it's fully GA **[unverified]** — https://openrouter.ai/blog/insights/what-is-jev/
- Calibration is aggregate only: 0.8 means right ~80% across many answers, not for any single one. Probabilities drift a few hundredths between runs, so threshold on bands. Wording changes confidence a lot. Set thresholds from a few hundred labelled examples (same OpenRouter post).
- "Jaggedness": counting, arithmetic and date comparison are unreliable; keep math in code (via InfoQ summary of TypeSafe's jev-1.13 doc **[secondary]**).
- A decision model can't give an invalid type but can still give a valid wrong answer (Armin Ronacher: it "delegates the hallucination problem a little bit to the user"; InfoQ).
- Context: 32k tokens for `state` + one question at base limits; long-context mode is slower and less accurate.

## 3. Alternatives (late 2026)

Hosted, Jev-compatible or similar:
- **Cloudflare Clef / Clef-Flash / Clef-Omni** (Oct 1, 2026). 27B / 9B. Open weights (Apache 2.0; training data not public). Run via the Workers AI binding `env.AI.run("@cf/cloudflare/clef", { model, state, questions })`. Jev-compatible; migrating is "change the endpoint and model". 64K context, 64 questions per request. $0.24 / $0.038 per M input tokens (clef / clef-flash). Median latency 209 ms in Cloudflare's own run. Benchmarks self-reported.
  https://developers.cloudflare.com/changelog/post/2026-10-01-clef-workers-ai/ · https://developers.cloudflare.com/workers-ai/platform/pricing/ · https://huggingface.co/Cloudflare/clef
- **Cloudflare Clairic** (preview). Extraction/estimation: typed fields, dates, numbers, forecasts with p10–p90 intervals. `@cf/cloudflare/clairic`, $0.10/M. Terms say it may be withdrawn. https://developers.cloudflare.com/workers-ai/models/clairic/
- **OpenAI Decisions API**. `POST /v1/decisions`, model `gpt-6-luna`. 1–400 questions, `choice`/`predicate`/`score`. $0.10/M uncached input, free output, **prompt caching on shared input** (≥1,024-token prefix). Official `openai` SDK. https://developers.openai.com/api/docs/guides/decisions
- **OpenRouter decisions endpoint** (alpha): `POST /api/alpha/decisions`, slugs `typesafe/jev-1.14`, `~typesafe/jev-latest`, plus others (Perplexity Decider, Inception Mercury Decide, Jev Router). https://openrouter.ai/blog/insights/what-is-jev/
- **Vercel AI SDK**: `experimental_decide` plus `@ai-sdk/typesafe-ai` provider (`createTypeSafeAi({ baseURL, fetch })`). https://ai-sdk.dev/providers/ai-sdk-providers/typesafe-ai
- Others in the list **[unverified]**: Snowflake Decision, Parallel Decisions, Microsoft-Decision-1, Solar Decide, Instinct (ZooWork).

Open / self-host (all need a GPU or CPU server, not Workers):
- **Laya** — 150–421M encoder, Apache 2.0, ~33–40 ms/question on a T4, custom criteria, `agent.fit(rows)` to fit light heads on labelled data; `laya-ts` runs ONNX in JS. Workers support **[unverified]**. https://github.com/NandhaKishorM/laya
- **xor** (Juspay, Qwen3.6-35B-A3B), **GLiNER2.5-Decide** (encoder, labels at call time), **Jev-compatible servers** (vLLM, SGLang, Ollama ≥0.35 serve `/v1/systemone` or `/v1/decisions` per the list). All via https://github.com/andyrewlee/awesome-system-one

Structured-generation tooling:
- **BAML**. Apache-2.0, own DSL, codegen client. 0.19.0 (Sept 11, 2026, canary) fixed web SDK init under workerd; trace IDs there use zero random bytes, so isolates can collide. https://boundaryml.com/blog/baml-0.19.0 · https://github.com/BoundaryML/baml
- **Effect AI** (`effect/unstable/ai`): `LanguageModel.generateObject({ prompt, schema })` with Effect Schema; v4 beta adds native structured output for Claude 4. Modules are **unstable**. https://effect.website/docs/v4/api/effect/unstable/ai/LanguageModel
- **Effect Agent**: I only found it described in a GitHub issue (schema-in/schema-out agents, policy budgets, subagents, durability up to one Durable Object per thread). Couldn't find a repo or docs **[unverified]**. https://github.com/rocketflare-dev/rocketflare/issues/9

Concepts worth knowing (survey paper, arXiv 2610.33775): System 1 → System 2 **cascades** (act on confident fast answer, escalate low-confidence to an LLM); calibration training; known issues such as ordinal-scale bias and option-order effects.

## 4. Fit with TypeScript + Effect v4 + Cloudflare Workers

| Option | Workers/DO fit | Effect v4 fit | Verdict |
|---|---|---|---|
| **Clef-Flash / Clef** | Best. Native `env.AI` binding inside the DO. No keys, same bill and network. | Wrap `env.AI.run` in `Effect.tryPromise`, decode with `Schema`. Alchemy has Workers AI binding → Effect AI layers for text, not decisions **[no decision support seen]** | **Default choice.** Preview-era; pin the model ID. |
| **Jev (TypeSafe)** | Fetch-based API. Official SDK README requires Node 22 and doesn't mention Workers **[verify]**; use raw `fetch` or the AI SDK provider's custom `fetch`. | Thin `HttpClient` service + Schema. | Strong second; swap-compatible with Clef by endpoint. Best-documented calibration story. |
| **OpenAI Decisions** | Fine over fetch. | Same. | Pick if question count per idea is large: shared-input caching. |
| **BAML** | Works on workerd since 0.19.0 canary; adds codegen + DSL build step. | No Effect integration known. | **Skip for v1.** Effect Schema + native structured output covers the same job. |
| **Effect AI generateObject** | Fine. | Native. | Use for the *generator* (slow path), not decisions. |
| **Laya / open models** | Needs a server (ONNX/GPU). | Callable over HTTP. | Later: distill from logged decisions. |

Design rule: put decisions behind one Effect service, e.g. `Decide.choice / score / noul`, with Schema-typed question sets and two layers (Clef via binding, Jev via fetch). The API is nearly the same, so swapping is a layer change. Return `{ value, probabilities, confidence }` and make "below threshold → escalate to LLM" a typed branch.

## 5. Concrete uses in tabbycat

Rule of thumb: decision models handle **fuzzy labels and gates**. Anything countable (intervals, beats, stretch, pitch classes) stays in deterministic code (the jaggedness caveat above).

1. **Request router** (`choice`). Input: user text. Options: single-line / chord-melody / counterpoint / polyrhythm / riff-on-selection / tune-lookup. Plus `noul`: "does this need a named tune's changes?" Below threshold → default generator (spec wants few choices for the user).
2. **Style and harmonic-intent mapping** (`choice`/`score`). "Very minor-sounding", "like Holdsworth" → style tags and a mode-colour target (Lydian ♭7, Dorian, altered…). Feeds the LLM prompt. The *implied harmony shown in the UI* should come from code analysing the notes, not the DM.
3. **Repair picker** (`choice`). State: the static-checker diagnostics (as text/JSON). Options: re-finger / shift position / move to another string set / octave-displace note / simplify rhythm / drop inner voice / full LLM rewrite. Confident → run the code repair; unsure → LLM rewrite. Log every pick plus the outcome (did the check pass?).
4. **Variety and novelty gate** (`noul` + `score`). First compute cheap features in code (interval histogram, rhythm hash, contour, embedding via Workers AI + Vectorize **[standard Cloudflare stack, not searched]**). Then ask the DM about the top-3 nearest saved ideas: "Is this idea essentially the same as any of these?" Reject or re-prompt for a variation. Also `score` rubrics for "rhythmic interest" / "idiomatic for fingerstyle", each as its own atomic question (TypeSafe docs advise splitting multi-factor judgments).
5. **Pre-generation guard** (`noul`). "Is this prompt ambiguous enough to need a default pick?" or "does the selected range have enough context to riff on?" Cheap, runs before the expensive call.
6. **Selection-riff scoping** (`choice`). Which of the selected bars hold the harmonic anchor vs free material.
7. **Self-calibration loop.** User actions (Next / Save / replay) are free labels. Store `{state, question, probs, outcome}` in the Durable Object's SQLite, derive thresholds from them, and later distil to a small local model. Several repos do this kind of distillation (jimothy, Jevstiller in the awesome list **[unverified]**).

Cost sanity: a 5-question call on ~500 tokens ≈ $0.00002 on Jev or ≈ $0.0001 on Clef. Free enough to run on every generation and every repair attempt; the latency (~100–300 ms) matters more than the money.

## 6. Suggested v1

- Build the `Decide` Effect service with Schema-typed questions; implement the **Clef-Flash** layer first (Workers AI binding, zero new infra), keep a **Jev** layer for A/B.
- Start with three uses: router (1), repair picker (3), novelty gate (4).
- Log every decision with a version-pinned model ID. Threshold from logged outcomes, not from the confidence field alone.
- Skip BAML. Revisit if you need strict parsing from models without native structured output.

## Open questions to verify before committing
- Is Jev GA or still waitlist? Does `@typesafe-ai/sdk` run on workerd?
- Is Clef out of preview on Workers AI, and what are its rate limits? Is `clef-flash` accurate enough on music-theory-flavoured labels? (No benchmark covers this; build a small eval set.)
- Effect v4 stable release date; `effect/unstable/ai` API churn.
- What "Effect Agent" really is and where it lives.
