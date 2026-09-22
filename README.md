# guard-kit 接入指南

[English](README_EN.md) · 简体中文

**端内反诈打分引擎（雷达 v6 · 裁判 v5）**——两个自训小模型（雷达 + 裁判），**可实现无缝嵌入各类 App**：原生 iOS/Android、RN/Expo、Flutter、小程序、网页与云端。纯函数 + 内置权重，零依赖、完全离线、单条推理 <0.1ms，同一份权重六个平台分数逐位一致。本仓库只放接入文档；**引擎源码、模型权重与训练管线在主仓库**（见文末链接）。

## 两种接入层面

| 层面 | 适用宿主 | 打包内容 |
|---|---|---|
| **双模型打包**（默认） | 需要消息风险打分 + 演练/回复评分（教育、陪练类 App） | 雷达 + 裁判（约 1.14MB） |
| **只接雷达** | 只需要消息风险检测/内容风控（银行、社区、聊天类 App） | 仅雷达权重（约 852KB，裁判不进包） |

裁判后续可能暂停版本更新（雷达独立演进），只接雷达的宿主不受影响。TS 侧用子路径导入 `import { mlScore } from 'guard-kit/radar'` 即只解析雷达模块图；原生侧对应地只放 `radar.json` 资源。

## 它能做什么

| 模型 | 输入 | 输出 | 典型场景 |
|---|---|---|---|
| **雷达** Radar | 任意消息文本（或多轮消息数组） | 校准诈骗概率（0-1）+ 三档结论（高危/可疑/放行）+ 骗术大类；多轮语境下 `trigger` 标注命中来源 | 聊天消息风险提示、粘贴内容检测、内容审核 |
| **裁判** Referee | 用户的回复文本 | 四类意图（反击/拖延/配合/无效）+ 星级 + 校准置信度 | 反诈演练评分、教育类 App 互动反馈 |

设计特点：
- **零幻觉**：只输出概率与枚举结论，不生成任何文本
- **离线**：权重内置（雷达 v6 852KB + 裁判 v5 285KB），无网络请求、无隐私外泄
- **跨平台一致**：六个平台对同一句话打出的分数逐位一致（≤1e-6，黄金向量验收）
- **校准**：模型说 90% 就是约 90% 的把握（Platt 校准），不是黑盒置信度
- **多轮语境（v6）**：单句铺垫（「换号了」）不误报，第二句露出意图（「先垫 8000」）即「结合上文」触发

## 六平台接入速查

### React Native / Expo / 浏览器 / Node（TypeScript）
```bash
npm install guard-kit
```
```ts
import { mlBandLabel, replyStar, mlConversation } from 'guard-kit';

mlBandLabel('把验证码发我一下，不发给我就冻结账户');
// → { band: 'high', label: '高危', typeLabel: '冒充身份' }

replyStar('我不会转钱的，我要打110');
// → { star: 3, label: '稳住反击', reliable: true }

// 多轮语境（v6）：铺垫句 + 意图句一起送，trigger='context' 表示结合上文命中
mlConversation([
  ' mum this is my new number, my phone fell in the water.',
  'Ok son, is everything alright?',
  'Please send HK$8,000 to this account urgently',
]);
// → { band: 'high', pCal: 1.0, trigger: 'context', current: {...} }
```
需要 `resolveJsonModule`（Expo/Next 默认开启）。

### iOS（Swift，SPM）
Xcode → File → Add Package Dependencies → Add Local… → 选主仓库 `swift/` 目录：
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

### Android（Kotlin/Java）
把主仓库 `models/radar.json`、`models/referee.json` 放进 `app/src/main/assets/`，`GuardKit.kt` 拖进源码树（依赖 `org.json:json`，Android 自带）：
```kotlin
val radar = Radar(assets.open("radar.json").readBytes().decodeToString())
val score = radar.score("把存款转入安全账户配合调查")
```

### Flutter / 纯 Dart
`pubspec.yaml` 依赖主仓库 `dart/`（路径依赖或发布包），NFKC 由 `unorm_dart` 提供：
```dart
final radar = Radar(File('radar.json').readAsStringSync());
final score = radar.score('把存款转入安全账户配合调查');
```

### 微信 / 支付宝小程序
把主仓库 `miniprogram/` 目录放进分包（两份权重共约 1.14MB，注意主包 2MB 限制；只接雷达约 852KB）：
```js
const guard = require('../../miniprogram/guard-kit.js');
const r = guard.mlScore('可疑短信文本');   // { pCal, band, topType, ... }
const s = guard.replyScore('我不会转钱的'); // { topClass, star, ... }
```

### 云端 HTTP（任何能发请求的端）
```bash
# 部署（Cloudflare Workers，免费额度内）
npx wrangler deploy   # 在主仓库 api/ 目录
```
```
POST /score        {"text": "..."}         → 雷达结果 JSON
POST /reply        {"text": "..."}         → 裁判结果 JSON
POST /conversation {"messages": ["..."]}   → 雷达多轮语境 JSON（trigger 标注来源）
GET  /health                              → {"status":"ok","radar":6,"referee":5}
```
⚠️ 云端仅用于无法本地跑模型的环境；端内集成才保有离线与隐私优势。

## 结果字段说明（雷达）

| 字段 | 含义 | 用法建议 |
|---|---|---|
| `pCal` | 校准后的诈骗概率 | 直接展示「AI 判定风险 87%」 |
| `band` | `high` / `medium` / `low` | high=强预警可拦截；medium=软提醒；low=放行 |
| **百分值** | **0-100 整数**（嵌入方对外呈现的统一口径） | `pCal × 100`；各平台均有 `mlPercent(text)` 便捷方法 |
| `topType` | `impersonation` 冒充身份 / `pay_first` 先转账后兑现 / `bait` 兼职投资诱饵 | 展示「疑似冒充身份类骗局」 |
| `probs` | 全类概率分布 | 需要自定义阈值时用 |

**官方提示白名单**：反诈宣传文本（如警方提示、防骗贴士）通篇是骗术词汇，模型会误报。引擎内置确定性规则（反诈主题词 + 官方建议词同时出现 → 压到可疑档以下），各平台行为一致，无需调用方处理。

## 裁判结果字段

| 字段 | 含义 |
|---|---|
| `topClass` | `counter` 稳住反击 / `stall` 拖延周旋 / `agree` 配合危险 / `meaningless` 无信息 |
| `star` | counter=3 / stall=2 / agree=1 / meaningless=1；无词表命中为 null |
| `confCal` | 校准置信度；**<0.5 时建议 UI 显示中性结果，不评星** |

## 质量指标（雷达 v6）

**真实世界外部基准**（模型从未见过的公开真实短信，held-out）：

| 切片 | 指标 | v5 | v6 |
|---|---|---|---|
| 英文真实诈骗 | 召回@可疑 / @高危 | 81.3% / 67.0% | **88.3% / 83.7%** |
| 英文真实正常短信 | 高危档误报 | 11.4% | **0.2%** |
| 中文真实诈骗 | 召回@可疑 / @高危 | 52.2% / 36.1% | **89.8% / 84.8%** |
| 中文纯广告 | 高危档误报 | 25.1% | **5.8%** |
| 中文真实正常短信 | 高危档误报 | 10.6% | **0.5%** |

**同一测试集三方对照**（120 条真实短信分层抽样）：

| 方案 | 召回 | 误报 | 准确率 |
|---|---|---|---|
| 关键词规则 | 1.8% | 0% | 55.0% |
| **雷达 v6（端内 0.04ms）** | **92.7%** | **3.1%** | **95.0%** |
| 大模型 zero-shot（云端） | 74.5% | 13.8% | 80.8% |
| 大模型 few-shot（云端） | 54.5% | 4.6% | 76.7% |

**真实香港案例卷（零样本）**：12 个真实 HK 骗案家族（ADCC/警方/新闻逐案重构并标注出处）+ 8 条真实机构通知——雷达 v6 **召回 100%（全部高危档）**、误报 12.5%；关键词规则召回 **0%**；大模型 few-shot 同为 100% 但需秒级联网。

| 内部指标 | 雷达 v6 | 裁判 v5 |
|---|---|---|
| 测试集召回/准确率 | 94.9%（高危 91.5%） | 95.8% |
| 最新诚实盲测卷 | 召回 92.9% / 高危 0 误报 | 准确率 100% |
| 推理延迟 | 0.04ms/条 | 0.03ms/条 |

训练数据三源可审计：手写话术模板 247 个（含 ADCC/警务处/公安部通报真实案例）+ **LLM 蒸馏语料 1,392 条**（MiniMax 五轮，按误报画像精准补粤语/HK 场景）+ **公开真实数据训练侧 9,274 条**（UCI 英文真实短信 + 80 万中文真实短信分层切片），程序化增强 2,142 条，简繁粤英四语。v6 经五轮「混训→双基准→按误报画像蒸馏补齐」迭代晋升，错误台账全程记录。已知边界与完整模型卡见主仓库。

## 常见问题

**Q：换新版模型要改接入代码吗？**
不用。权重 JSON 是稳定契约（文件名与版本解耦），把新的 `radar.json` / `referee.json` 换进资源目录即可，API 不变。

**Q：能自定义风险阈值吗？**
可以。`probs` 给了全类分布，`pCal` 是连续值，档位（0.75/0.35）只是默认产品策略。

**Q：误报了怎么办？**
真实世界基准高危误报 0.2-5.8%（v5 为 10.6-25.1%）；若线上仍出现，主仓库带错误台账与进化管线（`evolve.py`），纠错样本可自动进入下一次训练。

**Q：多轮对话怎么接？**
把最近几条消息按时间序传给 `mlConversation(messages)`（六平台同 API）。单句铺垫不会误报，第二句露出诈骗意图即触发，`trigger='context'` 时 UI 建议标注「结合上文判定」。

**Q：商用授权？**
MIT。权重自训自有，无第三方模型依赖。

---

**主仓库**（源码 + 权重 + 训练管线 + 六平台移植）：见仓库链接或联系维护者。
