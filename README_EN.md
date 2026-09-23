# guard-kit Integration Guide

English · [简体中文](README.md)

**On-device anti-scam scoring engine (Radar v7 · Referee v6)** — two self-trained compact models that **embed seamlessly into any app**: native iOS/Android, RN/Expo, Flutter, mini programs, web and cloud. Pure functions + built-in weights: zero dependencies, fully offline, <0.1 ms per inference, bit-identical scores across all six platforms (≤1e-6).

> This repo (public) hosts integration docs and the public benchmark only; **engine source, weights and training pipeline are distributed from a private repo** (invitation-based, see [Private distribution](#private-distribution-six-channels)).
> 🧠 At a glance: [online mind map · zh](https://htmlpreview.github.io/?https://raw.githubusercontent.com/P3NN4L/GUARDKIT/main/docs/guard-kit-mindmap.html) · [online mind map · en](https://htmlpreview.github.io/?https://raw.githubusercontent.com/P3NN4L/GUARDKIT/main/docs/guard-kit-mindmap-en.html) ([source · zh](docs/guard-kit-mindmap.html) / [source · en](docs/guard-kit-mindmap-en.html))
> 📊 Our published benchmark **[HK-ScamBench](benchmark/HK-ScamBench_EN.md)**: 12 real Hong Kong scam case families + 8 institutional notices, source-tagged — re-run it with your own system and cite freely.

---

## Capability overview

| Model | Input | Output | Typical use |
|---|---|---|---|
| **Radar** | A single message **or a multi-turn message sequence** | Calibrated probability + three bands (high/suspicious/pass) + scam category + percent (0-100) | Chat risk control, paste checks, content moderation |
| **Referee** | One user reply | Four intents (counter/stall/comply/meaningless) + star rating + calibrated confidence | Anti-scam drill scoring, education apps |

Design properties: **zero hallucination** (outputs probabilities and enums only, never text) · **offline** (weights built in; no network, no privacy leakage) · **calibrated** (90% means ~90%) · **multi-turn context** (grooming openers don't fire; intent triggers immediately) · **deterministic rule layer** (anti-fraud-discussion whitelist + official-domain rule, bit-identical on every platform).

**Quality at a glance (v7)**: zero-shot **100% recall / 0% high-band FPR** on the real HK case set; **89.1% / 1.5% / 94.2%** on 120 real SMS; 0.04ms per message. Full metrics and the four-generation comparison [below](#quality-metrics--generation-comparison).

---

## Installation (private distribution, invitation-based)

The engine ships through GitHub Packages; you need a team-issued token (`read:packages` only):

```bash
# one-time setup (~/.npmrc must contain //npm.pkg.github.com/:_authToken=YOUR_TOKEN)
npm config set "@p3nn4l:registry" https://npm.pkg.github.com
```

**Choose one of two packages**:

| Package | Contents | Size | For |
|---|---|---|---|
| `npm install @p3nn4l/guard-kit` | Radar + Referee (one package, subpath-split usable) | ~1.7MB | Both models (drills/education) |
| `npm install @p3nn4l/guard-kit-radar` | **Radar only** (engine + weights) | ~770KB | Risk control only (banks/community/chat) |

**Install a specific older version** (`latest` always points to the newest generation):

```bash
npm install @p3nn4l/guard-kit@7.0.0     # latest (Radar 7 · Referee 6)
npm install @p3nn4l/guard-kit@6.2.0     # M3-distilled (Radar 6.2 · Referee 5)
npm install @p3nn4l/guard-kit@6.1.0     # HK-hardened (Radar 6.1 · Referee 5)
npm install @p3nn4l/guard-kit@6.0.0     # first v6 release (Radar 6.0 · Referee 5)
```

The radar-only package supports version rollback the same way (`@p3nn4l/guard-kit-radar@6.x.0`, published in sync).

---

## API reference

### Radar: single-message scoring

```ts
import { mlScore, mlPercent, mlBandLabel } from '@p3nn4l/guard-kit';
// radar-only: import { mlScore, mlPercent } from '@p3nn4l/guard-kit-radar';

mlScore('This is Officer Chan. Your account is involved in money laundering. Transfer your savings to a safe account.');
// → { pCal: 0.99, pRaw: 0.90, band: 'high', topType: 'impersonation', probs: {...} }

mlPercent('This is Officer Chan. Transfer your savings to a safe account.');  // → 99 (int 0-100)
mlBandLabel('Send me the verification code or your account will be frozen');
// → { band: 'high', label: '高危', typeLabel: '冒充身份' }
```

| Field | Meaning | Suggested use |
|---|---|---|
| `pCal` | Calibrated scam probability | Display "AI risk: 87%" |
| `pRaw` | Raw evidence score | Second gate of the dual-condition band; usually not shown |
| `band` | `high` / `medium` / `low` | high = strong warning / may block; medium = soft reminder; low = pass |
| `topType` | `impersonation` / `pay_first` / `bait` | "Suspected impersonation scam" |
| `probs` | Full class distribution | For custom thresholds |
| **Percent** | `mlPercent()` → int 0-100 | The unified consumer-facing number |

### Radar: multi-turn context (core feature)

A single grooming opener carries insufficient evidence and never fires; the moment intent appears in the conversation it triggers. `trigger` labels the hit source; clean conversations stay clean:

```ts
import { mlConversation } from '@p3nn4l/guard-kit';

mlConversation([
  ' mum this is my new number, my phone fell in the water.',
  'Ok son, is everything alright?',
  'Please send HK$8,000 to this account urgently',
]);
// → { band: 'high', pCal: 0.9999, trigger: 'context', current: {…score of the current message alone…} }
```

| `trigger` | Meaning | UI suggestion |
|---|---|---|
| `direct` | The current message alone hits | Regular strong warning |
| `context` | Current message is plain but **with context** it hits | Label "judged with context" |
| `none` | No risk | Pass |

Default window = last 4 messages (override via `ML_CONVERSATION_WINDOW`); sticky escalation (no auto-downgrade within a session) is host-side session state.

### Referee: reply scoring

```ts
import { replyScore, replyStar } from '@p3nn4l/guard-kit';

replyScore("I will not transfer any money. I am calling 999 right now.");
// → { topClass: 'counter', confCal: 0.98, star: 3, probs: {...} }

replyStar("I will not transfer any money. I am calling 999.");  // → { star: 3, label: '稳住反击', reliable: true }
```

| Field | Meaning |
|---|---|
| `topClass` | `counter` steady counter / `stall` buy time / `agree` comply (dangerous) / `meaningless` no information |
| `star` | counter=3 / stall=2 / agree=1 / meaningless=1 |
| `confCal` | Calibrated confidence; **below 0.5 show a neutral result without stars** (`REPLY_RELIABLE_THRESHOLD`) |

### Legacy generations in-process (canary / kill switch)

The 7.0.0 package runs multiple generations side by side — no reinstall needed:

```ts
import { mlScore, mlScoreV62, mlScoreV6 } from '@p3nn4l/guard-kit';
mlScore(msg);    // v7 (default latest)
mlScoreV62(msg); // v6.2
mlScoreV6(msg);  // v6.0
// legacy referee: replyScoreV5(); percents: mlPercentV62() / mlPercentV6()
```

### Official-notice whitelist (built in, zero handling)

Anti-fraud advisories are full of scam vocabulary and cannot be separated textually from real scams. The engine ships deterministic rules (topic+advice dual condition; official vs suspicious domains mutually exclusive), identical on every platform — **callers do nothing**.

---

## Six integration channels

### TS / React Native / Expo / Node (primary)

Exactly the API reference above. Subpath split (full package, referee never loaded at runtime):

```ts
import { mlScore } from '@p3nn4l/guard-kit/radar';
import { replyScore } from '@p3nn4l/guard-kit/referee';
```

### iOS (Swift · SPM)

Xcode → Add Package Dependency → `https://github.com/P3NN4L/guard-kit-code` (invited GitHub auth):

```swift
import GuardKit

let radar = try Radar()
let s = try radar.score("我是王警官，你涉嫌洗钱，把存款转入安全账户")  // s.pCal, s.band, s.topType
let pct = radar.mlPercent("suspicious message")               // Int 0-100
let conv = try radar.mlConversation([" mum this is my new number.", "Please send HK$8,000 urgently"])
// conv.band / conv.trigger / conv.current
let rs = try Referee().score("好的我马上转账")                  // rs.topClass == "agree", rs.star == 1
// radar-only: simply never initialize Referee
```

### Android (Kotlin/Java · Maven)

Configure the Maven repo with invited credentials (`gpr.user` = GitHub username, `gpr.key` = token, in `~/.gradle/gradle.properties`):

```kotlin
repositories {
    maven {
        url = uri("https://maven.pkg.github.com/P3NN4L/guard-kit-code")
        credentials {
            username = project.findProperty("gpr.user") as String?
            password = project.findProperty("gpr.key") as String?
        }
    }
}
dependencies { implementation("com.settlepal:guardkit:7.0.0") }
```

```kotlin
val radar = Radar(assets.open("radar.json").readBytes().decodeToString())
val s = radar.score("我是王警官，你涉嫌洗钱，把存款转入安全账户")
val conv = radar.mlConversation(listOf(" mum this is my new number.", "Please send HK\$8,000 urgently"))
val pct = radar.mlPercent("suspicious message")                // Int 0-100
```

Weights go into `app/src/main/assets/` (`radar.json` / `referee.json`, shipped as Release attachments). Radar-only: bundle only `radar.json` and use only `Radar`.

### Flutter / Dart

pubspec git dependency (invited) or Release attachment:

```yaml
dependencies:
  guard_kit:
    git:
      url: https://github.com/P3NN4L/guard-kit-code.git
      path: dart
```

```dart
final radar = Radar(File('radar.json').readAsStringSync());
final s = radar.score('我是王警官，你涉嫌洗钱，把存款转入安全账户');
final conv = radar.mlConversation([' mum this is my new number.', 'Please send HK\$8,000 urgently']);
```

### WeChat / Alipay mini programs

Receive the `miniprogram/` directory (or the Release attachment `guard-kit-miniprogram.zip`), place it into a subpackage:

```js
const guard = require('../../miniprogram/guard-kit.js');        // both models
const r = guard.mlScore('我是王警官，你涉嫌洗钱，把存款转入安全账户');
const c = guard.mlConversation(['opener', 'money ask']);        // { band, trigger, ... }
const s = guard.replyScore('我不会转钱的');                      // { topClass, star, ... }
// radar-only: require('.../guard-kit-radar.js') (referee weights never load; includes mlPercent/mlConversation)
```

### Cloud HTTP (any client)

Invited users clone and deploy themselves (Cloudflare Workers, free tier, `wrangler deploy`):

```
POST /score        {"text": "..."}         → Radar (incl. percent)
POST /reply        {"text": "..."}         → Referee
POST /conversation {"messages": ["..."]}   → multi-turn context (incl. trigger)
GET  /health                              → {"status":"ok","radar":7,"referee":6}
```

⚠️ Cloud is only a fallback for environments that cannot run the models locally; on-device integration keeps the offline & privacy advantages.

---

## Quality metrics & generation comparison

### Four generations on the same sets (all fresh, blind-written before training, measured)

| Eval set | v6 | v6.1 | v6.2 | **v7** |
|---|---|---|---|---|
| blind-v9 multi-turn: recall/high-FPR | 87.5/27.3 | 87.5/18.2 | 93.8/45.5 | **100/0** |
| blind-v10: recall/high-FPR | 85.7/42.9 | 85.7/57.1 | 85.7/42.9 | 85.7/**28.6** |
| blind-v11 new surfaces: recall/high-FPR | 76.9/8.3 | **84.6**/8.3 | 76.9/33.3 | 76.9/**8.3** |
| HK-ScamBench high-FPR | 12.5 | 12.5 | **0** | **0** |
| Ultra-short probes (zh/en) | 0/2 | 1/2 | 2/2 | 2/2 |
| Public 120: recall/FPR/accuracy | 92.7/3.1/95.0 | 92.7/3.1/95.0 | 87.3/4.6/91.7 | **89.1/1.5/94.2** |
| Referee (same fresh sets: v9/v10) | — (v5: 65/41) | — | — | **90/65** |

Reading: each generation converges one frontier — v6 landed real-world corpora, v6.1 hardened HK, v6.2 closed ultra-short variants and institutional FPs, v7 closed the multi-turn blind spot and pushed public-set FPR to 1.5%. All numbers reproducible (fixed seeds, pinned samples, single-thread official build).

### Multi-way comparison (same 120 real SMS)

| Approach | Recall | FPR | Accuracy | Cost |
|---|---|---|---|---|
| Keyword rules | 1.8% | 0% | 55.0% | — |
| **Radar v7 (on-device, 0.04ms)** | **89.1%** | **1.5%** | **94.2%** | offline · free |
| LLM zero-shot (cloud) | 74.5% | 13.8% | 80.8% | seconds · online · per-call |
| LLM few-shot (cloud) | 54.5% | 4.6% | 76.7% | seconds · online · per-call |
| MiniMax-M3 zero-shot (flagship) | 56.4% | 4.6% | 77.5% | seconds of reasoning · online · per-call |
| MiniMax-M3 few-shot (flagship) | 58.2% | 3.1% | 79.2% | seconds of reasoning · online · per-call |

> LLM comparison config: MiniMax abab6.5s-chat / MiniMax-M3 (`chatcompletion_v2`) · temperature 0.1 · 4 labeled few-shot examples · strict JSON verdict. Full parameters in the private engine README ("LLM configuration").

### Training volume (v7, all self-built / publicly reproducible)

| Data | Radar v7 | Referee v6 |
|---|---|---|
| Handwritten templates | 247 | 174 |
| LLM-distilled (MiniMax, 9 rounds) | 1,530 items | 314 replies (4 intents × zh/en/Cantonese) |
| Multi-turn corpus (M3) | 156 dialogues | — |
| Public real-world data | 9,274 items (UCI + 800k Chinese stratified) | — |
| Programmatic augmentation | 2,142 | same recipe |
| Error-ledger hardening | 896 rows (×2) | 185 rows (×2) |
| **Total** | **13,547 rows** | **1,448 items** |

Four languages (SC/TC/Cantonese/English); the normal side deliberately includes anti-fraud advisories, urgent-but-legitimate, and promo/installment hard negatives. Known boundaries and the full model card live in the private engine repo.

---

## Private distribution (six channels)

| Channel | Route |
|---|---|
| TS/RN/Expo/Node | GitHub Packages npm (see Installation) |
| iOS (SPM) | private repo URL + invited GitHub auth |
| Android (Gradle) | GitHub Packages Maven (`com.settlepal:guardkit`) |
| Flutter/Dart | git dependency (invited) or Release attachment |
| Mini programs | Release attachment (`guard-kit-miniprogram.zip`) |
| Cloud Worker | Release attachment (`guard-kit-worker.zip`) |

### Who can see what (permission boundaries)

| Identity | Visible |
|---|---|
| Public visitor (no token) | This repo only (docs / mind maps / benchmark); no source, no weights |
| `read:packages` token holder | Downloadable packages (compiled artifacts + weights) only; **cannot see any repository contents** |
| Engine repo collaborator | Full engine repo (training pipeline / corpus) — collaborators on personal repos get write access by default; core team only |
| Fine-grained token (single repo, Contents:read) | Can resolve the SPM private repo URL; still no corpus or ledger |

---

## FAQ

**Q: Do I need code changes to upgrade?**
No. The weight JSON is a stable contract (filenames decoupled from versions) — swap the file in `models/`, or change the dependency version on npm. The API never changes across generations.

**Q: Custom thresholds?**
Yes. `pCal` is continuous and `probs` is the full distribution; the bands (0.75/0.35 × raw-evidence dual condition) are just the default product policy.

**Q: False positives?**
1.5% on the public 120. If one appears in production, the private repo's error ledger and evolution pipeline fold corrections into the next training run.

**Q: Multi-turn chat?**
Pass recent messages in time order to `mlConversation(messages)`. Grooming openers never fire; intent triggers immediately; label `trigger='context'` as "judged with context" in the UI.

**Q: Commercial licensing?**
Code and weights are **All Rights Reserved** ([LICENSE.txt](LICENSE.txt)) — use under written invitation; the HK-ScamBench dataset is separately CC BY 4.0. Weights are self-trained and owned; no third-party model dependencies.

---

**Private engine repo** (source + weights + training pipeline + six-platform ports): invited access to `guard-kit-code`.
