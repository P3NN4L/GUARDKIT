# guard-kit 接入指南

[English](README_EN.md) · 简体中文

🧠 **一图看懂引擎**：[在线渲染·中文脑图](https://htmlpreview.github.io/?https://raw.githubusercontent.com/P3NN4L/GUARDKIT/main/docs/guard-kit-mindmap.html) · [在线渲染·英文脑图](https://htmlpreview.github.io/?https://raw.githubusercontent.com/P3NN4L/GUARDKIT/main/docs/guard-kit-mindmap-en.html)（点开即看；[源文件·中](docs/guard-kit-mindmap.html) / [源文件·英](docs/guard-kit-mindmap-en.html)）

📊 **HK-ScamBench v1**：我们发布的香港真实骗案零样本评测基准（12 案例家族 + 8 机构通知，出处逐条标注，含基线结果与引用格式）——[中文](benchmark/HK-ScamBench.md) · [English](benchmark/HK-ScamBench_EN.md)

**端内反诈打分引擎（雷达 v7 · 裁判 v6）**——两个自训小模型（雷达 + 裁判），**可实现无缝嵌入各类 App**：原生 iOS/Android、RN/Expo、Flutter、小程序、网页与云端。纯函数 + 内置权重，零依赖、完全离线、单条推理 <0.1ms，同一份权重六个平台分数逐位一致。本仓库只放接入文档；**引擎源码、模型权重与训练管线在主仓库**（见文末链接）。

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
- **离线**：权重内置（雷达 v7 852KB + 裁判 v6 285KB），无网络请求、无隐私外泄
- **跨平台一致**：六个平台对同一句话打出的分数逐位一致（≤1e-6，黄金向量验收）
- **校准**：模型说 90% 就是约 90% 的把握（Platt 校准），不是黑盒置信度
- **多轮语境（v6）**：单句铺垫（「换号了」）不误报，第二句露出意图（「先垫 8000」）即「结合上文」触发

## 六平台接入速查

### React Native / Expo / 浏览器 / Node（TypeScript）
引擎以私有通道分发（源码仓保持私有）：受邀者配置 GitHub Packages 凭据后安装——

```bash
# 一次性配置（我们提供 token 与仓库名，非公开包）
npm config set "@p3nn4l:registry" https://npm.pkg.github.com
npm install @p3nn4l/guard-kit
```

安装指定旧版本（整包回退）：

```bash
npm install @p3nn4l/guard-kit@7.0.0   # 最新（雷达 7 · 裁判 6）
npm install @p3nn4l/guard-kit@6.2.0   # 回退（雷达 6.2 · 裁判 5）
npm install @p3nn4l/guard-kit@6.0.0   # 首发六代（雷达 6.0 · 裁判 5）
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

只接雷达装独立包更省：`npm install @p3nn4l/guard-kit-radar`（仅雷达引擎与权重，约 770KB）。

### iOS（Swift，SPM）
> 私有分发：引擎源码仓为私有。受邀团队以 GitHub 账号授权后可直接 Add Package Dependency；其他接入方使用我们提供的 Release 附件（含预编译包与权重）。
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
> 私有分发：受邀访问源码仓或获取 Release 附件，随后按下列方式集成。
把主仓库 `models/radar.json`、`models/referee.json` 放进 `app/src/main/assets/`，`GuardKit.kt` 拖进源码树（依赖 `org.json:json`，Android 自带）：
```kotlin
val radar = Radar(assets.open("radar.json").readBytes().decodeToString())
val score = radar.score("把存款转入安全账户配合调查")
```

### Flutter / 纯 Dart
> 私有分发：以 git 依赖（受邀 token）或 Release 附件提供。
`pubspec.yaml` 依赖主仓库 `dart/`（路径依赖或发布包），NFKC 由 `unorm_dart` 提供：
```dart
final radar = Radar(File('radar.json').readAsStringSync());
final score = radar.score('把存款转入安全账户配合调查');
```

### 微信 / 支付宝小程序
> 私有分发：受邀获取 `miniprogram/` 目录（或 Release 附件）后放入分包。
把主仓库 `miniprogram/` 目录放进分包（两份权重共约 1.14MB，注意主包 2MB 限制；只接雷达约 852KB）：
```js
const guard = require('../../miniprogram/guard-kit.js');
const r = guard.mlScore('可疑短信文本');   // { pCal, band, topType, ... }
const s = guard.replyScore('我不会转钱的'); // { topClass, star, ... }
```

### 云端 HTTP（任何能发请求的端）
> 私有分发：受邀 clone 引擎仓后自行部署。
```bash
# 部署（Cloudflare Workers，免费额度内）
npx wrangler deploy   # 在主仓库 api/ 目录
```
```
POST /score        {"text": "..."}         → 雷达结果 JSON
POST /reply        {"text": "..."}         → 裁判结果 JSON
POST /conversation {"messages": ["..."]}   → 雷达多轮语境 JSON（trigger 标注来源）
GET  /health                              → {"status":"ok","radar":7,"referee":5}
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

## 版本对比（v6 → v6.2 → v7，三代同卷实测）

| 评测集（fresh，训练前盲写） | v6 | v6.1 | v6.2 | **v7** |
|---|---|---|---|
| blind-v9 多轮卷：召回 / 高危误报 | 87.5% / 27.3% | 87.5% / 18.2% | 93.8% / 45.5% | **100% / 0%** |
| blind-v10：召回 / 高危误报 | 85.7% / 42.9% | 85.7% / 57.1% | 85.7% / 42.9% | 85.7% / **28.6%** |
| blind-v11 新骗面：召回 / 高危误报 | 76.9% / 8.3% | 84.6% / 8.3% | 76.9% / 33.3% | 76.9% / **8.3%** |
| HK-ScamBench 高危误报 | 12.5% | 12.5% | **0%** | **0%** |
| 极短变体探针（中英） | 0/2 | 1/2 | 2/2 | 2/2 |
| 公开 120 条：召回/误报/准确 | 92.7 / 3.1 / 95.0 | 92.7 / 3.1 / 95.0 | 87.3 / 4.6 / 91.7 | **89.1 / 1.5 / 94.2** |
| 裁判（同卷 fresh：v9 / v10） | —（v5：65% / 41%） | — | — | **90% / 65%** |

读法：每一代收敛一个前沿——v6 完成真实语料落地，v6.2 关闭极短变体与机构误报，v7 关闭多轮盲区并把公开集误报压到 1.5%；单点波动在下一代收敛。所有数字固定种子、锁定样本、单线程官方构建，可复现。

## 质量指标（雷达 v7）

**真实世界外部基准**（模型从未见过的公开真实短信，held-out）：

| 切片 | 指标 | v5 | v6 |
|---|---|---|---|
| 英文真实诈骗 | 召回@可疑 / @高危 | 81.3% / 67.0% | **88.3% / 83.7%** |
| 英文真实正常短信 | 高危档误报 | 11.4% | **0.2%** |
| 中文真实诈骗 | 召回@可疑 / @高危 | 52.2% / 36.1% | **89.8% / 84.8%** |
| 中文纯广告 | 高危档误报 | 25.1% | **5.8%** |
| 中文真实正常短信 | 高危档误报 | 10.6% | **0.5%** |

**同一测试集多方对照**（120 条真实短信分层抽样）：

| 方案 | 召回 | 误报 | 准确率 |
|---|---|---|---|
| 关键词规则 | 1.8% | 0% | 55.0% |
| **雷达 v7（端内 0.04ms）** | **89.1%** | **1.5%** | **94.2%** |
| 大模型 zero-shot（云端·按次计费） | 74.5% | 13.8% | 80.8% |
| 大模型 few-shot（云端·按次计费） | 54.5% | 4.6% | 76.7% |
| MiniMax-M3 zero-shot（旗舰·云端按次计费） | 56.4% | 4.6% | 77.5% |
| MiniMax-M3 few-shot（旗舰·云端按次计费） | 58.2% | 3.1% | 79.2% |

> 公开集「诈骗类」切片含关键词定义的灰区促销文本，大模型推理后倾向判「广告」（标签语义分歧，如实披露）；**语义无歧义的 HK 真实案例上 M3 few-shot 达 100%/0%**——大模型适合做语义兜底，雷达 0.04ms 离线免费守住海量第一道筛查，两者是融合分工而非替代关系。

**真实香港案例卷（零样本）**：12 个真实 HK 骗案家族（ADCC/警方/新闻逐案重构并标注出处）+ 8 条真实机构通知——雷达 v7 **召回 100%（12/12 全高危档）/ 高危误报 0%**；关键词规则召回 **0%**；大模型 few-shot 同为 100% 但需秒级联网。
> LLM 对照配置：MiniMax `abab6.5s-chat`（`chatcompletion_v2`）· temperature 0.1 · max_tokens 40 · system 内嵌 4 个标注例 · 严格 JSON 输出；**旗舰 `MiniMax-M3`**（推理模型）：默认 temperature · max_tokens 1500（容纳思考），zero-shot 用同一无例子任务描述、few-shot 用同一组 4 例，判定取正文 JSON（空则取推理尾结论）——上表两行 M3 即此两种模式。few-shot/zero-shot/蒸馏全参数见主仓库 README「LLM 配置」。

| 内部指标 | 雷达 v7 | 裁判 v6 |
|---|---|---|
| 测试集召回/准确率 | 95.7%（高危 94.0%） | 95.8% |
| 最新诚实盲测卷 | 召回 100% / 高危 0 误报（blind-v9 含多轮） | 准确率 100% |
| 推理延迟 | 0.04ms/条 | 0.03ms/条 |

### 训练量（v7，全部自建/公开可复现）

| 数据 | 雷达 v7 | 裁判 v6 |
|---|---|---|
| 手写话术/回复模板 | 247 个 | 174 个 |
| LLM 蒸馏语料（MiniMax，9 轮） | 1,530 条（骗 718 / 正 812） | 314 条回复（四意图 × 中英粤） |
| 多轮对话语料（M3 生成） | 156 段对话（骗 52 / 正 104） | — |
| 公开真实数据训练侧 | 9,274 条（UCI 英文短信 + 80 万中文短信分层） | — |
| 程序化增强语料 | 2,142 条 | 同配方 |
| 判错台账强化 | 896 行（含 ×2 权重） | 185 行（含 ×2 权重） |
| **语料总量** | **雷达 13,547 行** | **裁判 1,448 条** |

语料覆盖简体/繁体/粤语/英文四语；正常侧刻意收录官方反诈宣传、紧急但合法、促销分期三类强干扰负例。已知边界与完整模型卡见主仓库。

### 谁能看到什么（私有分发权限边界）

| 身份 | 能看到 |
|---|---|
| 公开访客（无 token） | 仅本仓库（文档/脑图/HK-ScamBench 基准），无源码无权重 |
| 持 `read:packages` token 的受邀者 | 只能下载安装包（编译产物 + 权重）；**看不到任何仓库内容** |
| 引擎仓 Collaborator | 能看到引擎仓全部内容（含训练管线/语料）——⚠️ 个人仓协作者默认写权限，仅限团队核心成员 |
| Fine-grained token（限单仓 Contents:read） | 可解析 SPM 私有仓 URL（iOS 场景），仍看不到语料与台账 |

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
代码与模型权重**保留一切权利**（All Rights Reserved，见 [LICENSE.txt](LICENSE.txt)）——本仓库仅公开接入文档；引擎以私有分发通道（受邀 + GitHub Packages 私有包 / Release 附件）提供，按书面授权使用。基准数据集 HK-ScamBench 单独以 CC BY 4.0 发布。权重自训自有，无第三方模型依赖。

---

**主仓库**（源码 + 权重 + 训练管线 + 六平台移植）：见仓库链接或联系维护者。
