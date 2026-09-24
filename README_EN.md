# guard-kit Integration Guide (SettlePal Anti-Fraud Engine)

guard-kit is an **on-device scam-scoring engine**: zero dependencies, fully offline, <0.1ms per message, four scripts (Simplified/Traditional/Cantonese/English). It outputs decisions and probabilities only — never text — and serves as one layer of a fusion architecture alongside your rules channel and LLM semantics channel. This guide is for integrators: from package choice to production.

> Current release: `@p3nn4l/guard-kit@7.0.0` (Radar v7 · Referee v6). The v8 distilled student and the five-engine committee have landed in the source repo (see metrics); install channels are unchanged until the combined package ships.

## Which package, in 30 seconds

| Package | Contents | Size | For |
|---|---|---|---|
| `npm install @p3nn4l/guard-kit` | Radar + Referee (same package, splittable via subpaths) | ~1.7MB | apps needing both models (drill/education) |
| `npm install @p3nn4l/guard-kit-radar` | **Radar only** (engine + weights) | ~770KB | risk-control only (banking/community/chat) |

Version rollback: `@7.0.0` (Radar 7 · Referee 6) / `@6.2.0` (M3 distillation) / `@6.1.0` (HK hardening) / `@6.0.0` (first release).

## TS / RN / Expo (3 lines)

```bash
npm install @p3nn4l/guard-kit@7.0.0
```

```ts
import { mlScore, mlConversation, replyScore } from '@p3nn4l/guard-kit';

// Single-message risk (Radar)
const r = mlScore('【香港郵政】包裹已扣留，請登記：hkpost-redelivery.top');
// r.band: 'high' | 'medium' | 'low'   r.pCal: calibrated probability   r.topType: scam macro-class

// Multi-turn conversation (v6+, same API on six platforms)
mlConversation(messages);   // grooming openers never fire; intent triggers immediately; trigger='context' → label "judged with context"

// Reply quality (Referee, drill/education scenarios)
replyScore(reply);
```

The radar outputs three bands mapping directly to product actions: **high** → interstitial warning with reasons; **medium** → subtle badge; **low** → silent. Band thresholds encode FPR semantics (0.75/0.35 + dual confirmation) — no tuning required.

## Full API reference (TS)

| API | Purpose | Returns |
|---|---|---|
| `mlScore(text)` | single-message risk | `{ pRaw, pCal, band, topType, probs }` |
| `mlConversation(messages)` | multi-turn context (last 4 turns concatenated, max score) | same + `trigger: 'direct' \| 'context'` |
| `replyScore(reply)` | reply-quality 4-class (zh/en/yue) | `{ topClass, confCal }` |
| `mlScoreV62 / mlScoreV6 / replyScoreV5` (`/legacy`) | mount legacy generations (canary/rollback) | per-generation |
| `createRadar(modelJson)` / `createReferee(modelJson)` | factories: build from a weights string | engine objects |

Mini-program / cloud exports share the same names; native constructors below.

## Six-platform integration

| Platform | How |
|---|---|
| TS / RN / Expo / Node | `npm install @p3nn4l/guard-kit` (GitHub Packages; invited token) |
| iOS (Swift · SPM) | private repo URL `https://github.com/P3NN4L/guard-kit-code`; resolvable in Xcode once invited |
| Android (Kotlin · Maven) | `maven { url = uri("https://maven.pkg.github.com/P3NN4L/guard-kit-code") }` + `implementation("com.settlepal:guardkit:7.0.0")` |
| Flutter (Dart) | pubspec git dependency (invited) or Release attachment |
| WeChat mini-program | Release attachment `guard-kit-miniprogram.zip` (`guard-kit-radar.js` = radar-only, smaller) |
| Cloud Worker | Release attachment `guard-kit-worker.zip` (dist-only, zero deps) |

All artifacts are **dist-only**: minified JS / native libs + type declarations + inlined weights — no TS source, no training pipeline, no corpus.

## Metrics: five evolution milestones (all fresh blind sets, measured)

| Engine | Evolution role | v12 | v10 | v11 | public120 (120 real SMS) |
|---|---|---|---|---|---|
| v5 | origin: pre-real-corpus era | — | — | — | 60.2 / 15.7 (external real-world bench) |
| v6.2 | generation change: corpus rebase | — | 85.7/42.9 | 76.9/33.3 | 87.3/4.6 |
| v7 (current package) | evolution: multi-turn + institutional whitelist | 93/12 | 86/29 | 77/8 | **89.1/1.5** |
| Committee | decision layer: five heterogeneous seats, majority vote | 100/12 | 100/43 | 85/8 | 94.5/1.5 |
| v8 | engine swap: distilled-student fusion (landed in source repo) | 92.9/6.2 | 85.7/0 | 84.6/0 | 78.2/1.5 |
| LLM few-shot | external reference: flagship M3 with 4 examples | 100/0 | 100/0 | 100/0 | 58.2/3.1 |

(recall/FPR@mid. v8 vs v7: 10 wins out of 11 sets; the only concession is public120 recall, where v7 keeps contributing inside the fusion at 0.3 weight.) Cross-scale law: 0.6B frozen 41.8 → 8B fine-tuned 49.1 → **20KB linear v7: 89.1** — domain data + calibration beat parameter scale. Efficiency: radar 852KB / mean 0.078ms (4,900-call microbenchmark).

## Who sees what (permission boundaries)

| Audience | Visibility |
|---|---|
| npm package installers | dist-only artifacts (minified JS + types + inlined weights) |
| Release attachment downloaders | per-platform dist bundles |
| Fine-grained token (single-repo Contents:read) | resolves the SPM private URL; still no corpus/ledger |
| Source-repo collaborators | source & training pipeline (core members only) |

## FAQ

**Q: Do I tune the band thresholds?**
No. 0.75/0.35 + dual confirmation encode FPR semantics; identical across platforms.

**Q: Offline?**
Fully offline. Zero network code, zero telemetry; messages never leave the device.

**Q: Cantonese / Traditional / mixed zh-en?**
Natively supported — four-script vocabulary quotas; Cantonese multi-turn appears natively in the training corpus.

**Q: Multi-turn usage?**
Pass recent messages in time order to `mlConversation(messages)`. Grooming openers never fire; intent triggers immediately; label `trigger='context'` as "judged with context" in the UI.

**Q: Commercial licensing?**
Two-tier licence (LICENSE): **free for personal and non-commercial use** (study, research, teaching, competitions, non-commercial projects); **commercial use requires a paid licence** — contact [github.com/P3NN4L](https://github.com/P3NN4L). Weights must not be used standalone to train other models.

**Q: Benchmark dataset?**
[HK-ScamBench](benchmark/HK-ScamBench.md) (12 real scam families + 8 institutional notices, CC BY 4.0) is available for citation and retesting.

---

Architecture details, the training protocol and full ablations live in the source-repo README (invited): `P3NN4L/guard-kit-code`.
