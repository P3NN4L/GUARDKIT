# guard-kit — On-Device Anti-Fraud Engine

**Weighted Radar — a five-engine on-device anti-fraud scoring system**: every message is scored independently by four engines (the v8 distilled-student primary + Radar v7 + SVM + NB) and fused into the final verdict by weighted voting, with the rules layer (three institutional-whitelist families) holding veto power; a Referee additionally grades reply quality. **Zero dependencies, fully offline, <0.1ms per message**, supporting Simplified/Traditional Chinese, Cantonese and English — embeddable into any app: banking, community, chat, payments, education. It outputs risk probabilities and bands only; it never generates text, never goes online, never uploads data.

- Current version: `@p3nn4l/guard-kit@8.0.0` (five-engine system; npm package carries the on-device base — radar/referee/rules layer/multi-turn — primary-engine weights via Release attachments/hot-update)
- Distribution: GitHub Packages private registry + Release attachments (invitation-based; see [permissions](#who-sees-what-permission-boundaries))
- License: **free for personal & non-commercial use; paid licence for commercial use** (see [LICENSE](LICENSE.txt))
- This repo hosts the integration docs and public benchmark; it contains no engine source or raw weights

## What it does

| Capability | Description |
|---|---|
| Five-engine fused verdict | Four heterogeneous scoring engines in weighted fusion + rules-layer veto — single-engine errors are absorbed by the seats; 10 wins out of 11 evaluation sets vs the single-model era |
| Message risk scoring | Calibrated probability + three bands (high/medium/low) for any SMS/chat message; band thresholds encode FPR semantics |
| Multi-turn context detection | Pass recent conversation — grooming openers never fire; the moment scam intent appears it triggers, labeled "judged with context" |
| Reply-quality grading | Four-class quality rating of the user's replies (S/T/Cantonese) for drill & education scenarios |
| Institutional whitelist | Legitimate notices (banks/government/schools) recognized via a triple condition — low FPR, zero false passes (0 across 2,570+ scam samples) |
| Six-platform consistency | The same weights produce bit-identical scores on TS/native iOS/native Android/Flutter/mini-program/cloud |

## Install & usage (TS / React Native / Expo / Node)

```bash
npm install @p3nn4l/guard-kit@8.0.0
```

```ts
import { mlScore, mlConversation, replyScore } from '@p3nn4l/guard-kit';

// Single-message risk
const r = mlScore('【香港郵政】包裹已扣留，請登記：hkpost-redelivery.top');
// r.band → 'high' | 'medium' | 'low' (maps directly to product actions)

// Multi-turn conversation
mlConversation(messages);

// Reply quality (drill/education)
replyScore(reply);
```

Risk control only? Install the smaller radar-only package: `npm install @p3nn4l/guard-kit-radar`. Rollback: `@6.2.0 / @6.1.0 / @6.0.0` remain on the registry.

## Six platforms at a glance

| Platform | How |
|---|---|
| TS / RN / Expo / Node | npm private package (GitHub Packages; invited token) |
| iOS | Swift Package (private repo URL; resolvable in Xcode once invited) |
| Android | Maven (`com.settlepal:guardkit:8.0.0`) |
| Flutter | pubspec git dependency (invited) or Release attachments |
| WeChat mini-program | Release attachment `guard-kit-miniprogram.zip` |
| Cloud Worker | Release attachment `guard-kit-worker.zip` (dist-only) |

All artifacts are **dist-only**: minified code + type declarations + inlined weights — no source, no training pipeline, no corpus.

## API reference (TS, isomorphic across platforms)

| API | Purpose | Returns |
|---|---|---|
| `mlScore(text)` | single-message risk | `{ pRaw, pCal, band, topType, probs }` |
| `mlConversation(messages)` | multi-turn context (last 4 turns) | same + `trigger: 'direct' \| 'context'` |
| `replyScore(reply)` | reply-quality 4-class | `{ topClass, confCal }` |
| `mlScoreV62()` / `mlScoreV6()` (`/legacy`) | mount legacy weights | per-generation, isomorphic |
| `createRadar(json)` / `createReferee(json)` | factories: build from a weights string | engine objects |

## Performance (v8 five-engine system, fully reproducible)

| Metric | Value |
|---|---|
| Five-engine system vs previous single model (v7) | 10 wins out of 11 evaluation sets (conversation-set FPR to zero; institutional-notice FPR halved) |
| 120 real public SMS (mixed zh/en) | radar seat solo: recall 89.1% @ FPR 1.5% |
| Reconstructed real HK case set (12 scam families) | high-band FPR 0% |
| Ultra-short variant probes (zh/en) | 2/2 detected |
| Reference: flagship cloud LLM few-shot (same real SMS) | recall 58.2% @ FPR 3.1% — plus network, per-call cost, seconds of latency |

Multi-turn context, the institutional whitelist and the full evaluation protocol are documented in the engine docs (visible to invitees). The public benchmark [HK-ScamBench](benchmark/HK-ScamBench.md) (CC BY 4.0) enables independent retesting.

## Who sees what (permission boundaries)

| Audience | Visibility |
|---|---|
| npm installers / Release downloaders | dist-only artifacts |
| Fine-grained token (Contents:read) | resolves the SPM private repo URL |
| Engine-repo collaborators | source & training pipeline (core members only) |

## FAQ

**Q: Does it need network?**
Fully offline. Zero network code, zero telemetry; messages never leave the device.

**Q: Cantonese / Traditional / mixed zh-en?**
Natively supported — four-script vocabulary quotas; Cantonese multi-turn appears in the training corpus.

**Q: Do I tune thresholds?**
No. The 0.75/0.35 dual-condition bands are built in and identical across platforms; thresholds encode FPR semantics.

**Q: Commercial use?**
Two-tier licence: free for personal & non-commercial use (study/research/teaching/competitions/non-commercial projects); for commercial use contact [github.com/P3NN4L](https://github.com/P3NN4L) for a paid licence. Weights must not be used standalone to train other models.

**Q: Is there a benchmark dataset?**
[HK-ScamBench](benchmark/HK-ScamBench.md): 12 real scam families + 8 institutional-notice categories, CC BY 4.0, citable and retestable.

---

SettlePal is the first host app of guard-kit; any app can integrate the same capability via this guide.
