# guard-kit Integration Guide

English · [简体中文](README.md)

**On-device anti-scam scoring engine (Radar v6.1 · Referee v5)** — two self-trained compact models (Radar + Referee) that **embed seamlessly into any app**: native iOS/Android, RN/Expo, Flutter, mini programs, web and cloud. Pure functions + built-in weights: zero dependencies, fully offline, <0.1 ms per inference, bit-identical scores across six platforms. This repo hosts integration docs only; **the engine source, model weights and training pipeline live in the main repo** (link at the bottom).

## Two integration tiers

| Tier | Target hosts | Bundle |
|---|---|---|
| **Both models** (default) | Apps needing message risk scoring + drill/reply scoring (education, training apps) | Radar + Referee (~1.14MB) |
| **Radar-only** | Apps needing message risk detection / content moderation only (banking, community, chat) | Radar weights only (~852KB; referee stays out) |

Referee version updates may pause (radar evolves independently); radar-only hosts are unaffected. On TS, the subpath import `import { mlScore } from 'guard-kit/radar'` resolves only the radar module graph; on native, ship only the `radar.json` resource accordingly.

## What it does

| Model | Input | Output | Typical use |
|---|---|---|---|
| **Radar** | Any message text (or a multi-turn message array) | Calibrated scam probability (0–1) + three-band verdict (high/suspicious/pass) + scam category; in conversation mode `trigger` labels the hit source | Chat risk warnings, pasted-content checks, content moderation |
| **Referee** | The user's reply | Four intents (counter/stall/agree/meaningless) + star rating + calibrated confidence | Anti-scam drill scoring, interactive feedback in education apps |

Design properties:
- **Zero hallucination**: outputs probabilities and enum verdicts only; never generates text
- **Offline**: weights embedded (Radar v6.1 852KB + Referee v5 285KB); no network calls, no privacy leakage
- **Cross-platform consistency**: the same sentence scores bit-identically on all six platforms (≤1e-6, golden-vector acceptance)
- **Calibrated**: when it says 90% it means ~90% confidence (Platt calibration) — not a black-box confidence score
- **Multi-turn context (v6)**: a grooming opener ("this is my new number") alone doesn't fire; the moment intent appears ("please send HK$8,000") it triggers "with context"

## Six-platform quick start

### React Native / Expo / browser / Node (TypeScript)
```bash
npm install guard-kit
```
```ts
import { mlBandLabel, replyStar, mlConversation } from 'guard-kit';

mlBandLabel('把验证码发我一下，不发给我就冻结账户');
// → { band: 'high', label: '高危', typeLabel: '冒充身份' }

replyStar('我不会转钱的，我要打110');
// → { star: 3, label: '稳住反击', reliable: true }

// Multi-turn context (v6): send the tail of the conversation; trigger='context' means the hit came from context
mlConversation([
  ' mum this is my new number, my phone fell in the water.',
  'Ok son, is everything alright?',
  'Please send HK$8,000 to this account urgently',
]);
// → { band: 'high', pCal: 1.0, trigger: 'context', current: {...} }
```
Requires `resolveJsonModule` (on by default in Expo/Next).

### iOS (Swift, SPM)
Xcode → File → Add Package Dependencies → Add Local… → select the main repo's `swift/` directory:
```swift
import GuardKit

let radar = try Radar()
let score = try radar.score("把存款转入安全账户配合调查")
let percent = radar.mlPercent("可疑消息")  // Int 0-100
// score.pCal: Double, score.band: "high"/"medium"/"low", score.topType: String?

let referee = try Referee()
let rs = try referee.score("好的我马上转账")
// rs.topClass: "agree", rs.star: 1, rs.confCal: Double
```

### Android (Kotlin/Java)
Put the main repo's `models/radar.json` and `models/referee.json` into `app/src/main/assets/`, and drop `GuardKit.kt` into your source tree (depends on `org.json:json`, bundled with Android):
```kotlin
val radar = Radar(assets.open("radar.json").readBytes().decodeToString())
val score = radar.score("把存款转入安全账户配合调查")
```

### Flutter / pure Dart
Depend on the main repo's `dart/` in `pubspec.yaml` (path dependency or published package); NFKC is provided by `unorm_dart`:
```dart
final radar = Radar(File('radar.json').readAsStringSync());
final score = radar.score('把存款转入安全账户配合调查');
```

### WeChat / Alipay mini programs
Copy the main repo's `miniprogram/` directory into a subpackage (both weights ~1.14MB, mind the 2MB main-package limit; radar-only ~852KB):
```js
const guard = require('../../miniprogram/guard-kit.js');
const r = guard.mlScore('可疑短信文本');   // { pCal, band, topType, ... }
const s = guard.replyScore('我不会转钱的'); // { topClass, star, ... }
```

### Cloud HTTP (any client that can send a request)
```bash
# Deploy (Cloudflare Workers, within the free tier)
npx wrangler deploy   # in the main repo's api/ directory
```
```
POST /score        {"text": "..."}         → Radar result JSON
POST /reply        {"text": "..."}         → Referee result JSON
POST /conversation {"messages": ["..."]}   → Radar multi-turn context JSON
GET  /health                              → {"status":"ok","radar":6,"referee":5}
```
⚠️ Cloud is only for environments that cannot run the models locally; on-device integration keeps the offline & privacy advantages.

## Radar result fields

| Field | Meaning | Suggested use |
|---|---|---|
| `pCal` | Calibrated scam probability | Display "AI risk: 87%" directly |
| `band` | `high` / `medium` / `low` | high = strong warning, may block; medium = soft reminder; low = pass |
| **Percent** | **0–100 integer** (the unified consumer-facing contract) | `pCal × 100`; every platform has an `mlPercent(text)` helper |
| `topType` | `impersonation` identity impersonation / `pay_first` pay-first-pay-later / `bait` part-time & investment bait | Display "suspected impersonation scam" |
| `probs` | Full class probability distribution | For custom thresholds |

**Official-notice whitelist**: anti-fraud advisories (police tips, fraud-prevention posts) are full of scam vocabulary and would false-positive the model. The engine ships a deterministic rule (anti-fraud topic word + official advice word appearing together → capped below the suspicious band); behavior is identical on every platform and callers need no special handling.

## Referee result fields

| Field | Meaning |
|---|---|
| `topClass` | `counter` steady counter / `stall` buy time / `agree` comply (dangerous) / `meaningless` no information |
| `star` | counter=3 / stall=2 / agree=1 / meaningless=1; null when no keyword hit |
| `confCal` | Calibrated confidence; **when <0.5, show a neutral result without stars** |

## Quality metrics (Radar v6.1)

**Real-world external benchmark** (public real SMS the model never saw, held-out):

| Slice | Metric | v5 | v6 |
|---|---|---|---|
| Real English fraud | recall@suspicious / @high | 81.3% / 67.0% | **88.3% / 83.7%** |
| Real English normal SMS | high-band FPR | 11.4% | **0.2%** |
| Real Chinese fraud | recall@suspicious / @high | 52.2% / 36.1% | **89.8% / 84.8%** |
| Chinese pure ads | high-band FPR | 25.1% | **5.8%** |
| Real Chinese normal SMS | high-band FPR | 10.6% | **0.5%** |

**Three-way comparison on the same test set** (120 stratified real SMS):

| Approach | Recall | FPR | Accuracy |
|---|---|---|---|
| Keyword rules | 1.8% | 0% | 55.0% |
| **Radar v6.1 (on-device, 0.04ms)** | **92.7%** | **3.1%** | **95.0%** |
| LLM zero-shot (cloud) | 74.5% | 13.8% | 80.8% |
| LLM few-shot (cloud) | 54.5% | 4.6% | 76.7% |

**Real Hong Kong case set (zero-shot)**: 12 real HK scam case families (reconstructed case-by-case from ADCC / police / news, source-tagged) + 8 real institutional notices — Radar v6.1 scores **100% recall (all high band)** at 12.5% FPR; keyword rules get **0%** recall; the LLM few-shot also reaches 100% but needs seconds of cloud round-trips.

| Internal | Radar v6.1 | Referee v5 |
|---|---|---|
| Test-set recall / accuracy | 94.9% (high band 91.5%) | 95.8% |
| Latest honest blind set | recall 92.9% / high-band 0 FPR | accuracy 100% |
| Inference latency | 0.04 ms/item | 0.03 ms/item |

Training data, three auditable sources: 247 handwritten script templates (incl. real cases from ADCC / HK Police / MPS advisories) + **1,392 LLM-distilled items** (MiniMax, five rounds targeted at the FP profile, covering Cantonese/HK scenarios) + **9,274 public real-world training items** (UCI real English SMS + stratified slices of 800k real Chinese SMS), plus 2,142 programmatic augmentations, four languages (SC/TC/Cantonese/English). v6 was promoted after five rounds of "mix-train → dual benchmark → FP-profile-targeted distillation", with the error ledger maintained throughout. Known boundaries and the full model card live in the main repo.

## FAQ

**Q: Do I need code changes to swap in a new model version?**
No. The weight JSON is a stable contract (filenames decoupled from versions) — drop the new `radar.json` / `referee.json` into the resource directory; the API is unchanged.

**Q: Can I customize risk thresholds?**
Yes. `probs` gives the full class distribution and `pCal` is a continuous value; the bands (0.75/0.35) are just the default product policy.

**Q: What about false positives?**
Real-world high-band FPR is 0.2-5.8% (v5: 10.6-25.1%). If one still appears in production, the main repo's error ledger and evolution pipeline (`evolve.py`) fold corrections into the next training run automatically.

**Q: How do I integrate multi-turn chat?**
Pass the recent messages in time order to `mlConversation(messages)` (same API on six platforms). A grooming opener alone won't fire; the moment a follow-up reveals scam intent it triggers, and when `trigger='context'` the UI should label it "judged with context".

**Q: Commercial licensing?**
MIT. The weights are self-trained and owned; no third-party model dependencies.

---

**Main repo** (source + weights + training pipeline + six-platform ports): see the repo link or contact the maintainer.
