# guard-kit Integration Guide

**On-device anti-scam scoring engine (v5)** — two self-trained compact models (Radar + Referee) that **embed seamlessly into any app**: native iOS/Android, RN/Expo, Flutter, mini programs, web and cloud. Pure functions + built-in weights: zero dependencies, fully offline, <0.1 ms per inference, bit-identical scores across six platforms. This repo hosts integration docs only; **the engine source, model weights and training pipeline live in the main repo** (link at the bottom).

## Two integration tiers

| Tier | Target hosts | Bundle |
|---|---|---|
| **Both models** (default) | Apps needing message risk scoring + drill/reply scoring (education, training apps) | Radar + Referee (~720KB) |
| **Radar-only** | Apps needing message risk detection / content moderation only (banking, community, chat) | Radar weights only (~425KB; referee stays out) |

Referee version updates may pause (radar evolves independently); radar-only hosts are unaffected. On TS, the subpath import `import { mlScore } from 'guard-kit/radar'` resolves only the radar module graph; on native, ship only the `radar.json` resource accordingly.

## What it does

| Model | Input | Output | Typical use |
|---|---|---|---|
| **Radar** | Any message text | Calibrated scam probability (0–1) + three-band verdict (high/suspicious/pass) + scam category | Chat risk warnings, pasted-content checks, content moderation |
| **Referee** | The user's reply | Four intents (counter/stall/agree/meaningless) + star rating + calibrated confidence | Anti-scam drill scoring, interactive feedback in education apps |

Design properties:
- **Zero hallucination**: outputs probabilities and enum verdicts only; never generates text
- **Offline**: weights embedded (Radar + Referee v5, ~720KB total); no network calls, no privacy leakage
- **Cross-platform consistency**: the same sentence scores bit-identically on all six platforms (≤1e-6, golden-vector acceptance)
- **Calibrated**: when it says 90% it means ~90% confidence (Platt calibration, ECE 3.5%) — not a black-box confidence score

## Six-platform quick start

### React Native / Expo / browser / Node (TypeScript)
```bash
npm install guard-kit
```
```ts
import { mlBandLabel, replyStar } from 'guard-kit';

mlBandLabel('把验证码发我一下，不发给我就冻结账户');
// → { band: 'high', label: '高危', typeLabel: '冒充身份' }

replyStar('我不会转钱的，我要打110');
// → { star: 3, label: '稳住反击', reliable: true }
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
Copy the main repo's `miniprogram/` directory into a subpackage (both weights ~730KB; mind the 2MB main-package limit):
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
POST /score  {"text": "..."}   → Radar result JSON
POST /reply  {"text": "..."}   → Referee result JSON
GET  /health                    → {"status":"ok","radar":4,"referee":4}
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

## Quality metrics (v5)

| Metric | Radar | Referee |
|---|---|---|
| Test-set recall / accuracy | 97.6% | 95.8% |
| Latest honest blind set | recall 100% / high-band 0 FPs | accuracy 100% |
| False positives | 6.0% (suspicious band) / **0% (high band)** | — |
| Calibration error ECE | 3.5% | 1.9% |
| Inference latency | 0.06 ms/item | 0.03 ms/item |

All training data is self-built and auditable: 421 handwritten script templates (incl. real cases from ADCC / HK Police / Ministry of Public Security advisories), 3,383 augmented corpus items, 240 blind acceptance items, four languages (SC/TC/Cantonese/English). v5 was promoted after 32 gated evolution iterations (error ledger → hardened retrain → gate → version promotion; roll back on failure). Known boundaries and the full model card live in the main repo.

## FAQ

**Q: Do I need code changes to swap in a new model version?**
No. The weight JSON is a stable contract (filenames decoupled from versions) — drop the new `radar.json` / `referee.json` into the resource directory; the API is unchanged.

**Q: Can I customize risk thresholds?**
Yes. `probs` gives the full class distribution and `pCal` is a continuous value; the bands (0.75/0.35) are just the default product policy.

**Q: What about false positives?**
The high band has 0 false positives across all evaluation sets. If one still appears in production, the main repo's error ledger and evolution pipeline (`evolve.py`) fold corrections into the next training run automatically.

**Q: Commercial licensing?**
MIT. The weights are self-trained and owned; no third-party model dependencies.

---

**Main repo** (source + weights + training pipeline + six-platform ports): see the repo link or contact the maintainer.
