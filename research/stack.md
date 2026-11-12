# tabbycat stack research

Date: 2026-11-12. Sources: npm registry, GitHub (`gh`), official docs. Anything not confirmed is marked **UNVERIFIED**.

## TL;DR

| Item | Version (2026-11-12) | Status | Verdict |
|---|---|---|---|
| FoldKit | `0.170.0` (stable `latest`) | pre-1.0, ships ~weekly, breaking changes in minors | Usable. **Blocks is a large breaking rewrite in preview, no release date.** |
| Effect v4 | `4.1.4` (`latest`) | **Stable since 4.0.0 on 2026-10-01**, LTS | Solid. Use `effect/unstable/*` modules (incl. AI) with care. |
| "Effect Agent" | `@yielded/agent@0.1.0-beta.192` | beta, 1 maintainer org, created 2026-10 | Real and on Effect v4. **No documented generate → validate → repair loop.** |
| Alchemy | `2.0.0-beta.85` | v2 beta, Effect-native | DO support is real. **"Domain" is a real DDD framework, not just DNS.** |
| TypeScript 7 | `7.0.2` (`latest`), `7.1.0-beta` | **Stable since 2026-07-08** | Works with all of them via compiler. **No stable plugin/JS API; Effect LSP needs `@effect/tsgo`.** |

Biggest risks: (1) FoldKit Blocks churn, (2) TS7 + Effect/Blocks type-depth errors (TS2589), (3) beta Alchemy and beta agent library, (4) agent repair loop must be built by us.

---

## 1. FoldKit

- **Version**: `foldkit@0.170.0`, released 2026-10-29, npm `latest`. Canary builds ship daily, stable ~weekly. Source: [npm](https://www.npmjs.com/package/foldkit)
- **Repo / docs**: [github.com/foldkit/foldkit](https://github.com/foldkit/foldkit) (MIT, ~1.2k stars, pushed 2026-11-08), [foldkit.dev](https://foldkit.dev)
- **Peers**: `effect ~4.1.0`, `@effect/platform-browser ~4.1.0`. Works with Effect 4.1.4.
- **Maturity**: pre-1.0, no 1.0 date. The [roadmap](https://github.com/foldkit/foldkit/blob/main/packages/website/src/page/roadmap.md) says 1.0 comes after more ambitious apps, SSR/Machine/Query leaving experimental, a perf pass, a docs pass and an RC freeze. The author started June 2025. Stable public APIs will follow semver only at 1.0.
- **Core API** (per [foldkit.dev](https://foldkit.dev)):
  - Model = Effect Schema. Message = tagged union.
  - `update(model, message)` returns the next model plus Commands.
  - Command = a described side effect that the runtime runs and that can dispatch Messages.
  - Subscription = a stream active while a Model condition holds.
  - ManagedResource holds handles such as WebSocket or **AudioContext**, closed automatically.
  - Submodel = nested Model/Message/update/view with typed OutMessages.
  - `Runtime.embed` hosts a widget inside another app.
- **Rendering**: **virtual DOM, built on a vendored Snabbdom** (`packages/foldkit/src/snabbdom`, `vdom.ts`, `patchVNode`). The `foldkit/html` module includes SVG support. There is a first-party `foldkit/canvas` (`Canvas.view` with shapes and pointer handlers on a 2D context), plus `foldkit/mount`, `foldkit/customElement`, `foldkit/port` and `foldkit/managedResource`.
- **Devtools**: logs every Message, shows the Model after each, shows a diff, can rewind. `@foldkit/devtools-mcp` exposes inspect, history, rewind and dispatch to AI agents. Also Story and Scene test harnesses (`foldkit/story`, `foldkit/scene`).
- **Blocks (the upcoming API)** — this is public:
  - Main PR: [#1655 "feat!: build applications from Blocks and Effect Layers"](https://github.com/foldkit/foldkit/pull/1655). It opened 2026-10-12, is still open, and is labeled P2, size/XL, with merge conflicts. A preview is installable (`create-foldkit-app@0.41.0-canary.54f8bcd443b4`, packages `0.170.0-canary.54f8bcd443b4`).
  - Redesign: [#1860 "replace Block authoring with class and Layer Blocks"](https://github.com/foldkit/foldkit/pull/1860) (branch `blocks-simplify`), from [proposal #1858](https://github.com/foldkit/foldkit/issues/1858).
    - A Block is a class declaring a contract: Model, Message, OutMessage and Commands.
    - `make` implements it as an Effect **Layer**.
    - Parents create child Blocks with `yield*` and hold them in their Model. The shape is borrowed from Alchemy bindings.
    - Handlers for Commands are Layers, so `Command.make` has no `execute`.
  - Follow-ups: [#1922](https://github.com/foldkit/foldkit/pull/1922) (open, 2026-11-08), merged #1656 / #1705 / #1729 / #1801 / #1894. About 15+ open `[Blocks preview]` issues, e.g. [#1698](https://github.com/foldkit/foldkit/issues/1698), [#1772](https://github.com/foldkit/foldkit/issues/1772), [#1844](https://github.com/foldkit/foldkit/issues/1844), [#1911](https://github.com/foldkit/foldkit/issues/1911), [#1925](https://github.com/foldkit/foldkit/issues/1925). Discussion: [#1781](https://github.com/foldkit/foldkit/discussions/1781).
  - **Timeline: none published.** The PR says "APIs and conventions may change before the stable release." The roadmap does not mention Blocks. The design was still being rewritten on 2026-11-04, so expect more churn. Estimate: unknown (**UNVERIFIED**).
  - Alchemy's Domain→FoldKit bridge was written against stable FoldKit and is already broken against the Blocks preview: [alchemy#2788](https://github.com/alchemy-run/alchemy/issues/2788).

**For us**
- **Do not wait** for Blocks before starting. Blocks is the biggest foreseeable migration. Mitigate by keeping our tab renderer and player as framework-agnostic modules, with FoldKit only as a thin shell.
- **Tab renderer**: render as SVG in the view (VDOM diff is fine for a tab of a few hundred nodes) or use `Canvas.view` for large tabs. For playback highlight, update a single "current beat" in the Model.
- **Audio**: model WebAudio as a `ManagedResource` plus Subscriptions. First-party pattern, no custom escape hatch needed.
- FoldKit already tests on native TS 7 (`@typescript/native`) and `@effect/tsgo`.

## 2. Effect v4

- **Status**: **stable**. `effect@4.0.0` on 2026-10-01 ([German mirror of the announcement](https://zeli.app/de/story/49925812); the official post was not found — date also matches npm publish time). `4.1.0` 2026-10-29, `4.1.4` 2026-11-11 (npm `latest`). Beta started 2026-02-18; RC announced 2026-08-12 ([RC recap](https://effect.website/blog/effect-v4-rc-august-recap/)). v3 is at `3.22.4` (`prev` tag), in maintenance.
- **Repo / docs**: [Effect-TS/effect](https://github.com/Effect-TS/effect) (the former `effect-smol` repo is archived; v4 lives in the main repo), [effect.website](https://effect.website), [beta post](https://effect.website/blog/releases/effect/40-beta).
- **Support**: 4.x is LTS with fixes "until at least September 2029" (single mirror source, **UNVERIFIED** officially).
- **Changes from v3** ([beta post](https://effect.website/blog/releases/effect/40-beta)):
  - Rewritten fiber runtime. Min Effect+Stream+Schema ~70 kB → ~20 kB. The 6.4x speed / 86% memory figures are Effect's own, not independently reproduced.
  - `@effect/platform`, `@effect/rpc` and `@effect/cluster` merged into `effect`. Provider and platform packages (`@effect/platform-*`, `@effect/sql-*`, `@effect/ai-*`) stay separate. All share one version number.
  - **New features live under `effect/unstable/*`**, with no semver guarantee, so breaking changes are possible in minors.
  - Schema rewritten (see the SCHEMA.md migration guide). Core programming model otherwise the same. Services API reworked: `Effect.services` in LS issue #779 hints at it, details **UNVERIFIED**.
- **AI lives in `effect/ai`** (exports `./ai`: `LanguageModel.generateObject/generateText/streamText`, `Tool`, `Toolkit`, `Chat`, `Prompt`, `McpServer`, plus `AnthropicStructuredOutput`). `@effect/ai` on npm is the **v3-era** package (`0.37.x`), not for v4. For v4, use `@effect/ai-anthropic@4.1.4` (peer `effect ^4.1.4`). Source: npm, [repo](https://github.com/Effect-TS/effect/tree/main/packages/ai/anthropic).
- **Compat**:
  - FoldKit `0.170.0`: peer `~4.1.0` — OK.
  - `@yielded/agent` beta.192: peer `^4.1.0` — OK.
  - Alchemy beta.85: peer `^4.1.0` — OK.
  - All three line up on Effect 4.1.x. Pin `effect` and every `@effect/*` to the same version.
  - Mismatch risk: `effect-agent@0.0.1-beta.3` hard-pins `effect@4.0.0-beta.102` (old name, avoid), and the FoldKit Blocks issue #1772 repro used exact `effect@4.0.0`.

## 3. "Effect Agent"

The exact project is **Yielded Agent** (originally published as `effect-agent`; the old `effect-agent` npm name is stale).
- **npm**: `@yielded/agent@0.1.0-beta.192` (2026-11-03; created 2026-10-02; `latest` = `beta` tag). Peer `effect ^4.1.0`. Source: [npm](https://www.npmjs.com/package/@yielded/agent).
- **Repo / docs**: [github.com/yielded-dev/agent](https://github.com/yielded-dev/agent) (formerly `danieljvdm/effect-agent`; MIT; ~182 stars; pushed 2026-11-08), [yielded.dev/agent](https://yielded.dev/agent/), [getting started](https://yielded.dev/agent/guide/getting-started/).
- **Maturity**: young (about 6 weeks old), ~190 beta releases, one org. Durability docs say plainly there is "no exactly-once external execution guarantee."
- **What it offers**:
  - `Agent.make(id, {input, output, instructions, toolkit, policy, completion})`, where `output` is an Effect Schema and "final output is schema-validated JSON by default" ([guide](https://yielded.dev/agent/guide/agents/)).
  - Native Effect AI `Toolkit` tools and provider Layers. The docs example uses OpenAI. Anthropic is only referenced via the Effect AI provider, with a prompt-caching test in the repo — **Anthropic wiring not documented**, but `@effect/ai-anthropic@4.1.4` is a standard Effect AI provider Layer.
  - Bounded loop: `policy` {`maxTurns`, `maxToolCalls`, `maxDuration`, `modelRetries`, `repeatedFailureLimit`, `onExhaustion`}. `completion: {tool, project}` forces the agent to finish via a tool call.
  - Durable "Threads", with Runs and Attempts, approvals and recovery.
  - Hosts: in-memory, Node+SQLite/Postgres, **Cloudflare Durable Object (SQLite) via `@yielded/agent-platform-cloudflare@beta`** (needs `effect-cf`; `ThreadObject.make`; limits and alarms documented in the [Cloudflare page](https://yielded.dev/agent/platforms/cloudflare/)).
  - A `testing` package with scripted models.
- **Generate → validate → repair loop**: **not documented as a feature.** Output is Schema-validated, but docs do not say what happens on a schema failure (`modelRetries` is documented only for model failures before content arrives, `repeatedFailureLimit` only for tool failures). No custom validator or repair hook appears in the docs. **UNVERIFIED** whether it feeds validation errors back to the model.
  - **Workable pattern**: make a `submit_tab` **completion tool** whose handler runs our playability checker and returns a typed failure message. A failed tool call goes back to the model as a tool result, and `repeatedFailureLimit` / `maxToolCalls` bound the repair loop. This is a standard and cheap way to get the loop using documented features. Needs a spike to confirm failure results are fed back to the model.
- **Alternative with no extra dependency**: write the loop ourselves on `effect/ai` `LanguageModel.generateObject` + Schema + `Effect.retry`/`Effect.iterate`. Small, fully owned, fine for a single-purpose loop.

## 4. Alchemy

- **Version**: `alchemy@2.0.0-beta.85` (2026-11-02, `latest`). v2 beta; 85 betas since ~spring 2026. Apache-2.0. Peer `effect ^4.1.0`. [Repo](https://github.com/alchemy-run/alchemy) (~1.9k stars, pushed 2026-11-11), [docs](https://alchemy.run).
- **Durable Objects: supported.** Docs: [Durable Objects](https://alchemy.run/cloudflare/compute/durable-objects/), plus cross-worker DOs, WebSockets and code updates. Effect-style `RpcDurableObject` helper in v2. SQLite-backed classes, migrations handled automatically. Pages for **Foldkit** frontend and **Effect AI** exist in the docs nav ([Foldkit guide](https://alchemy.run/cloudflare/frontend/) — exact URL **UNVERIFIED**).
- **"Domains" means two different things**:
  1. **Domains & DNS** ([docs](https://alchemy.run/cloudflare/networking/domains)): plain Cloudflare zones, DNS records, DNSSEC, zone settings, and `domain: "app.example.com"` on a Worker for a custom hostname. **This is NOT domain modeling.**
  2. **`@alchemy.run/domain`** ([README](https://github.com/alchemy-run/alchemy/tree/main/packages/domain), [docs](https://alchemy.run/domain)): "A domain-driven design framework for Effect". It is a real feature: Commands, Aggregates (`decide`/`evolve`), Events, Views/Projections, Policies, Rejections, `Domain.Model`. Event-sourced, tested with `Story.given/when/then` in memory, then run on **Cloudflare Durable Objects** unchanged (one DO per aggregate). Version `2.0.0-beta.85`, package created 2026-10-21 (**new, young**).
    - Known issues: first Domain call per isolate can block up to 5 s ([#3217](https://github.com/alchemy-run/alchemy/issues/3217)), `exactOptionalPropertyTypes` failures ([#2959](https://github.com/alchemy-run/alchemy/issues/2959)), DO rename upload limit ([#2976](https://github.com/alchemy-run/alchemy/issues/2976)), migration hooks still being designed ([#2938](https://github.com/alchemy-run/alchemy/issues/2938)), RFC for agents integrated with Domain ([#3080](https://github.com/alchemy-run/alchemy/issues/3080)).
    - FoldKit bridge `@alchemy.run/domain/Foldkit` exists but targets stable FoldKit, not Blocks ([#2788](https://github.com/alchemy-run/alchemy/issues/2788)).

**For us**
- Alchemy + DO is a good fit. For the "saved library of ideas", a Domain aggregate (`Idea`: `Generated`, `Saved`, `Rated`) is a natural but optional fit. Plain DO + SQLite is the lower-risk path while Domain is a month old.
- Both Domain (event-sourced DO) and `@yielded/agent-platform-cloudflare` use their own DO classes. Check they do not both want to own the same DO bindings and alarms (the agent lib says it owns the DO alarm slot) — **UNVERIFIED**, needs a spike.

## 5. TypeScript 7

- **Status**: **stable**. `7.0.0` GA, `7.0.2` is npm `latest` (2026-07-08), `7.1.0-beta` since 2026-10-20 (`next` = 7.1.0-dev). Ships as the normal `typescript` package; `tsc` is the Go compiler. Nightlies moved from `@typescript/native-preview` to `typescript@next`. Source: [announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/), [npm](https://www.npmjs.com/package/typescript).
- **Gotchas** (from the announcement):
  - **No stable programmatic API** until 7.1. Tools that import `typescript` (typescript-eslint, Vue/Astro/Svelte tooling, webpack loaders) need the 6.0 API via the `@typescript/typescript6` package (`tsc6`).
  - Defaults changed: `strict` on, `module: esnext`, `rootDir: ./`, `types: []`. Many 6.0-deprecated options are now hard errors (`baseUrl`, `moduleResolution: node10`, `target: es5`).
  - Language-service plugins do not load. VS Code can fall back to the 6.0 server.
- **Effect**: `@effect/language-service` supports **TS 5 and 6 only** and crashes on 7 ([issue #750](https://github.com/Effect-TS/language-service/issues/750), closed with the advice to use `@effect/tsgo`). Use **[`@effect/tsgo`](https://www.npmjs.com/package/@effect/tsgo) (`0.68.0`, 2026-11-11)** for Effect diagnostics on TS7. There is also [Effect-TS/tsgo](https://github.com/Effect-TS/tsgo), "TSRS, TypeScript-go and Oxlint enhanced with the Effect LSP experience", which builds on [oxc-project/tsrs](https://github.com/oxc-project/tsrs) (Rust port of the TS7 checker; 3.1k stars).
- **FoldKit**: repo itself uses `@typescript/native` (`typescript@^7.0.2` alias), `@effect/tsgo` and keeps a TS6 alias for tooling. **Blocks preview hits TS2589 ("excessively deep") on native TS 7.1-beta** with 12+ bindings ([#1772](https://github.com/foldkit/foldkit/issues/1772), [#1844](https://github.com/foldkit/foldkit/issues/1844)) and has editor latency ([#1709](https://github.com/foldkit/foldkit/issues/1709)); the preview recommends the optional `tsrs` language server.
- **Alchemy**: its own repo uses `@effect/tsgo` + `typescript` via catalog, and typechecks with `tsrs`. So the Alchemy author is on TS7-family tooling. Alchemy compiles via rolldown, not `tsc`. Domain has an `exactOptionalPropertyTypes` issue (#2959) — a TS option, not TS7-specific.

## 6. Risks across the stack

| # | Risk | Severity | Mitigation |
|---|---|---|---|
| 1 | **FoldKit Blocks**: large breaking rewrite, still redesigning (2026-11-04), no date. Foldkit 0.x minors already break. | High | Pin exact versions. Keep domain logic (tab model, renderer, player, playability checker) in framework-free modules. Isolate FoldKit to a thin shell. Decide: start on stable 0.170 API and migrate, or start on the Blocks canary. |
| 2 | **Alchemy is v2 beta** (85 betas) and **Domain is ~3 weeks old**. | High | Use Alchemy for infra and DO. Treat Domain as optional. Pin versions. |
| 3 | **Agent lib is a young beta**; repair loop undocumented; Anthropic path not documented. | Medium-High | Spike the completion-tool repair pattern first. Fall back to a hand-rolled `generateObject` loop on `effect/ai`. |
| 4 | **TS7 + Effect/Blocks**: TS2589 on large compositions; LS plugin gap; no plugin API. | Medium | Use `@effect/tsgo`. Keep a TS6 alias for tooling (eslint). Small Block graphs. |
| 5 | **`effect/unstable/*`** (AI, HTTP, RPC, etc.) has no semver guarantee in 4.x. | Medium | Pin `effect` and all `@effect/*` to one exact version. Read changelogs for each minor. |
| 6 | **Version-skew traps**: old `@effect/ai` (v3), old `effect-agent` (pins 4.0.0-beta), FoldKit `~4.1.0` peer. | Medium | Use `@effect/ai-anthropic` 4.x, `@yielded/agent`, one `effect@4.1.x`. |
| 7 | **DO ownership overlap** between Domain and the agent Cloudflare host (both own alarms/classes). | Unknown | Spike in a throwaway Worker. **UNVERIFIED**. |
| 8 | Effect v4 stable date and LTS window rest on one mirror, not the official post. | Low | Cosmetic. npm confirms `4.0.0` on 2026-10-01. |

## Unverified

- Official Effect 4.0 stable announcement URL and the "supported until Sept 2029" claim.
- Whether `@yielded/agent` feeds Schema/validation failures back to the model, and its Anthropic setup.
- Whether Domain, `@yielded/agent-platform-cloudflare` and `effect-cf` coexist in one Worker.
- Blocks release timeline (nothing public).
- Alchemy Foldkit guide exact URL, and its compatibility with the Blocks preview.
- Services API changes in Effect v4 beyond what the beta post lists.
