# guard-kit 接入指南

[English](README_EN.md) · 简体中文

**端内反诈打分引擎（雷达 v7 · 裁判 v6）**——两个自训小模型，**可实现无缝嵌入各类 App**：原生 iOS/Android、RN/Expo、Flutter、小程序、网页与云端。纯函数 + 内置权重：零依赖、完全离线、单条推理 <0.1ms，同一份权重六平台分数逐位一致（≤1e-6）。

> 本仓库（公开）只放接入文档与公开基准；**引擎源码、权重与训练管线在私有仓分发**（受邀制，见 [私有分发](#私有分发六端)）。
> 🧠 一图看懂：[中文脑图](docs/guard-kit-mindmap.html) · [英文脑图](docs/guard-kit-mindmap-en.html)（GitHub 页内显示源码，下载后双击打开即为渲染图）
> 📊 我们发布的公开基准 **[HK-ScamBench](benchmark/HK-ScamBench.md)**：12 个真实香港骗案家族 + 8 机构通知，出处逐条标注，欢迎用你的系统复测引用。

---

## 能力总览

| 模型 | 输入 | 输出 | 典型场景 |
|---|---|---|---|
| **雷达** Radar | 单条消息 **或多轮消息序列** | 校准概率 + 三档（高危/可疑/放行）+ 骗术大类 + 百分值(0-100) | 聊天风控、粘贴检测、内容审核 |
| **裁判** Referee | 用户的一条回复 | 四意图（反击/拖延/配合/无效）+ 星级 + 校准置信 | 反诈演练评分、教育互动 |

设计特点：**零幻觉**（只输出概率与枚举，不生成文本）· **离线**（权重内置，无网络无隐私外泄）· **校准**（说 90% 就是约九成把握）· **多轮语境**（铺垫不误报、意图一出现即触发）· **确定性规则层**（反诈讨论白名单 + 官方域名规则，全平台逐位一致）。

**质量速览（v7）**：HK 真实案例卷零样本 **100% 召回 / 0% 高危误报**；公开 120 条真实短信 **89.1% / 1.5% / 94.2%**；0.04ms/条。完整指标与四代对比见[下文](#质量指标与版本对比)。

---

## 安装（私有分发，受邀制）

引擎经 GitHub Packages 私有通道分发，需要团队发放的 token（只需 `read:packages`）：

```bash
# 一次性配置（~/.npmrc 需含 //npm.pkg.github.com/:_authToken=YOUR_TOKEN）
npm config set "@p3nn4l:registry" https://npm.pkg.github.com
```

**两种包任选**：

| 包 | 内容 | 体积 | 适合 |
|---|---|---|---|
| `npm install @p3nn4l/guard-kit` | 雷达 + 裁判（同包，可子路径拆用） | ~1.7MB | 需要双模型（陪练/教育类） |
| `npm install @p3nn4l/guard-kit-radar` | **只有雷达**（引擎+权重） | ~770KB | 只要风控（银行/社区/聊天类） |

**按版本回退安装**（`latest` 永远指向最新代）：

```bash
npm install @p3nn4l/guard-kit@7.0.0     # 最新（雷达 7 · 裁判 6）
npm install @p3nn4l/guard-kit@6.2.0     # M3 蒸馏版（雷达 6.2 · 裁判 5）
npm install @p3nn4l/guard-kit@6.1.0     # HK 加固版（雷达 6.1 · 裁判 5）
npm install @p3nn4l/guard-kit@6.0.0     # 首发（雷达 6.0 · 裁判 5）
```

雷达独立包同样支持按版本回退（`@p3nn4l/guard-kit-radar@6.x.0`，发布时同步）。

---

## API 参考

### 雷达：单条打分

```ts
import { mlScore, mlPercent, mlBandLabel } from '@p3nn4l/guard-kit';
// 只接雷达：import { mlScore, mlPercent } from '@p3nn4l/guard-kit-radar';

mlScore('我是王警官，你涉嫌洗钱，把存款转入安全账户');
// → { pCal: 0.998, pRaw: 0.904, band: 'high', topType: 'impersonation', probs: {...} }

mlPercent('我是王警官，你涉嫌洗钱，把存款转入安全账户');  // → 100（整数 0-100，对外展示口径）
mlBandLabel('把验证码发我一下，不发给我就冻结账户');
// → { band: 'high', label: '高危', typeLabel: '冒充身份' }
```

| 字段 | 含义 | 用法建议 |
|---|---|---|
| `pCal` | 校准诈骗概率 | 直接展示「AI 判定风险 87%」 |
| `pRaw` | 原始证据分 | 双条件档位的第二道闸，一般不直接展示 |
| `band` | `high` / `medium` / `low` | high=强预警可拦截；medium=软提醒；low=放行 |
| `topType` | `impersonation` / `pay_first` / `bait` | 展示「疑似冒充身份类骗局」 |
| `probs` | 全类概率分布 | 自定义阈值时用 |
| **百分值** | `mlPercent()` → 0-100 整数 | 嵌入方对外呈现的统一口径 |

### 雷达：多轮语境（核心特性）

单句铺垫（「我是你儿子，换号了」）证据不足不硬判；对话里意图一出现即触发。`trigger` 标注命中来源，干净对话零误报：

```ts
import { mlConversation } from '@p3nn4l/guard-kit';

mlConversation([
  ' mum this is my new number, my phone fell in the water.',
  'Ok son, is everything alright?',
  'Please send HK$8,000 to this account urgently',
]);
// → { band: 'high', pCal: 0.9999, trigger: 'context', current: {…当前句单独分…} }
```

| `trigger` | 含义 | UI 建议 |
|---|---|---|
| `direct` | 当前句单独命中 | 常规强预警 |
| `context` | 当前句平淡、**结合上文**命中 | 标注「结合上文判定」 |
| `none` | 无风险 | 放行 |

默认窗口取最近 4 条（`ML_CONVERSATION_WINDOW` 可覆写）；「粘性升级」（会话内不自动降级）由宿主持有会话状态实现。

### 裁判：回复评分

```ts
import { replyScore, replyStar } from '@p3nn4l/guard-kit';

replyScore('我不会转钱的，我要打110');
// → { topClass: 'counter', confCal: 0.98, star: 3, probs: {...} }

replyStar('我不会转钱的，我要打110');  // → { star: 3, label: '稳住反击', reliable: true }
```

| 字段 | 含义 |
|---|---|
| `topClass` | `counter` 反击 / `stall` 拖延 / `agree` 配合（危险） / `meaningless` 无信息 |
| `star` | counter=3 / stall=2 / agree=1 / meaningless=1 |
| `confCal` | 校准置信；**<0.5 时建议显示中性、不评星**（`REPLY_RELIABLE_THRESHOLD`） |

### 旧版本并行挂载（灰度/回退开关）

7.0.0 包内同进程可跑多代引擎，用于灰度对照或一键回退，不必换安装：

```ts
import { mlScore, mlScoreV62, mlScoreV6 } from '@p3nn4l/guard-kit';
mlScore(msg);    // v7（默认最新）
mlScoreV62(msg); // v6.2
mlScoreV6(msg);  // v6.0
// 裁判旧代：replyScoreV5()；百分值：mlPercentV62() / mlPercentV6()
```

### 官方提示白名单（内置，无需处理）

反诈宣传/防骗贴士通篇是骗术词汇，文本层无法与实施骗局区分。引擎内置确定性规则（主题词+建议词双条件；官方域名与可疑域名互斥），各平台行为逐位一致——**调用方零处理**。

---

## 六端接入

### TS / React Native / Expo / Node（主通道）

即上文 API 参考的用法。子路径拆用（装整包但不加载裁判运行时）：

```ts
import { mlScore } from '@p3nn4l/guard-kit/radar';        // 裁判不进模块图
import { replyScore } from '@p3nn4l/guard-kit/referee';
```

### iOS（Swift · SPM）

Xcode → Add Package Dependency → `https://github.com/P3NN4L/guard-kit-code`（受邀 GitHub 授权）：

```swift
import GuardKit

let radar = try Radar()
let s = try radar.score("我是王警官，你涉嫌洗钱，把存款转入安全账户")  // s.pCal, s.band, s.topType
let pct = radar.mlPercent("可疑消息")                        // Int 0-100
let conv = try radar.mlConversation([" mum this is my new number.", "Please send HK$8,000 urgently"])
// conv.band / conv.trigger / conv.current
let rs = try Referee().score("好的我马上转账")                // rs.topClass == "agree", rs.star == 1
// 只接雷达：不初始化 Referee 即可（权重按 target 资源打包）
```

### Android（Kotlin/Java · Maven）

受邀凭据配置 Maven 仓库（`gpr.user`=GitHub 用户名，`gpr.key`=token，写 `~/.gradle/gradle.properties`）：

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
val pct = radar.mlPercent("可疑消息")            // Int 0-100
```

权重放 `app/src/main/assets/`（`radar.json` / `referee.json`，随 Release 附件提供）。只接雷达：只放 `radar.json`、只用 `Radar` 类。

### Flutter / Dart

pubspec git 依赖（受邀）或 Release 附件：

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

### 微信 / 支付宝小程序

受邀获取 `miniprogram/` 目录（或 Release 附件 `guard-kit-miniprogram.zip`），放分包：

```js
const guard = require('../../miniprogram/guard-kit.js');        // 双模型
const r = guard.mlScore('我是王警官，你涉嫌洗钱，把存款转入安全账户');
const c = guard.mlConversation(['铺垫句', '要钱句']);             // { band, trigger, ... }
const s = guard.replyScore('我不会转钱的');                      // { topClass, star, ... }
// 只接雷达：require('.../guard-kit-radar.js')（裁判权重零加载，含 mlPercent/mlConversation）
```

### 云端 HTTP（任何能发请求的端）

受邀 clone 后自行部署（Cloudflare Workers，免费额度内 `wrangler deploy`）：

```
POST /score        {"text": "..."}         → 雷达（含 percent 字段）
POST /reply        {"text": "..."}         → 裁判
POST /conversation {"messages": ["..."]}   → 多轮语境（含 trigger）
GET  /health                              → {"status":"ok","radar":7,"referee":6}
```

⚠️ 云端仅是无法本地跑模型时的补充通道；端内集成才保有离线与隐私优势。

---

## 质量指标与版本对比

### 四代同卷对比（全部 fresh 盲测卷，训练前盲写，实测）

| 评测集 | v6 | v6.1 | v6.2 | **v7** |
|---|---|---|---|---|---|
| blind-v9 多轮卷：召回/高危误报 | 87.5/27.3 | 87.5/18.2 | 93.8/45.5 | **100/0** |
| blind-v10：召回/高危误报 | 85.7/42.9 | 85.7/57.1 | 85.7/42.9 | 85.7/**28.6** |
| blind-v11 新骗面：召回/高危误报 | 76.9/8.3 | **84.6**/8.3 | 76.9/33.3 | 76.9/**8.3** |
| HK-ScamBench 高危误报 | 12.5 | 12.5 | **0** | **0** |
| 极短变体探针（中英） | 0/2 | 1/2 | 2/2 | 2/2 |
| 公开 120 条：召回/误报/准确 | 92.7/3.1/95.0 | 92.7/3.1/95.0 | 87.3/4.6/91.7 | **89.1/1.5/94.2** |
| 裁判（同卷 fresh：v9/v10） | —（v5：65/41） | — | — | **90/65** |
读法：每一代收敛一个前沿——v6 落地真实语料，v6.1 HK 加固，v6.2 关闭极短变体与机构误报，v7 关闭多轮盲区并把公开集误报压到 1.5%。全部数字固定种子、锁定样本、单线程官方构建，可复现。

### 多方对照（同一批真实短信，120 条）

| 方案 | 召回 | 误报 | 准确率 | 成本 |
|---|---|---|---|---|
| 关键词规则 | 1.8% | 0% | 55.0% | — |
| **雷达 v7（端内 0.04ms）** | **89.1%** | **1.5%** | **94.2%** | 离线 · 免费 |
| 大模型 zero-shot（云端） | 74.5% | 13.8% | 80.8% | 秒级 · 联网 · 按次计费 |
| 大模型 few-shot（云端） | 54.5% | 4.6% | 76.7% | 秒级 · 联网 · 按次计费 |
| MiniMax-M3 zero-shot（旗舰） | 56.4% | 4.6% | 77.5% | 秒级推理 · 联网 · 按次计费 |
| MiniMax-M3 few-shot（旗舰） | 58.2% | 3.1% | 79.2% | 秒级推理 · 联网 · 按次计费 |

> LLM 对照配置：MiniMax abab6.5s-chat / MiniMax-M3（`chatcompletion_v2`）· temperature 0.1 · few-shot 内嵌 4 个标注例 · 严格 JSON 判定输出。全参数见私有引擎仓 README「LLM 配置」。

### 训练量（v7，全部自建/公开可复现）

| 数据 | 雷达 v7 | 裁判 v6 |
|---|---|---|
| 手写模板 | 247 | 174 |
| LLM 蒸馏（MiniMax 9 轮） | 1,530 条 | 314 条（四意图 × 中英粤） |
| 多轮对话语料（M3） | 156 段 | — |
| 公开真实数据 | 9,274 条（UCI + 80 万中文分层） | — |
| 程序化增强 | 2,142 条 | 同配方 |
| 判错台账强化 | 896 行（×2） | 185 行（×2） |
| **总量** | **13,547 行** | **1,448 条** |

四语（简/繁/粤/英）；正常侧刻意收录反诈宣传/紧急合法/促销分期三类强干扰负例。已知边界与完整模型卡见私有引擎仓。

---

## 私有分发（六端）

| 端 | 通道 |
|---|---|
| TS/RN/Expo/Node | GitHub Packages npm（本指南「安装」节） |
| iOS (SPM) | 私有仓 URL + 受邀 GitHub 授权 |
| Android (Gradle) | GitHub Packages Maven（`com.settlepal:guardkit`） |
| Flutter/Dart | git 依赖（受邀）或 Release 附件 |
| 小程序 | Release 附件（`guard-kit-miniprogram.zip`） |
| 云端 Worker | Release 附件（`guard-kit-worker.zip`） |

### 谁能看到什么（权限边界）

| 身份 | 可见范围 |
|---|---|
| 公开访客（无 token） | 仅本仓库（文档/脑图/基准）；无源码无权重 |
| `read:packages` token 持有者 | 只能下载安装包（编译产物+权重）；**看不到任何仓库内容** |
| 引擎仓 Collaborator | 引擎仓全部内容（含训练管线/语料）——个人仓协作者默认写权限，仅限核心成员 |
| Fine-grained token（单仓 Contents:read） | 可解析 SPM 私有仓 URL；仍无语料/台账 |

---

## 常见问题

**Q：换新版模型要改代码吗？**
不用。权重 JSON 是稳定契约（文件名与版本解耦），换 `models/` 里的权重文件即可；npm 通道则是改依赖版本号。API 跨代不变。

**Q：能自定义阈值吗？**
可以。`pCal` 是连续值、`probs` 是全分布，档位（0.75/0.35 × 原始证据双条件）只是默认产品策略。

**Q：误报了怎么办？**
公开 120 条上误报 1.5%；若线上出现，私有引擎仓的错误台账与进化管线会把纠错样本折进下一次训练。

**Q：多轮对话怎么接？**
把最近消息按时间序传 `mlConversation(messages)`。单句铺垫不误报，意图出现即触发；`trigger='context'` 时 UI 标注「结合上文判定」。

**Q：商用授权？**
代码与模型权重**保留一切权利**（[LICENSE.txt](LICENSE.txt)）——受邀书面授权使用；HK-ScamBench 数据集单独 CC BY 4.0。权重自训自有，无第三方模型依赖。

---

**引擎私有仓**（源码 + 权重 + 训练管线 + 六平台移植）：受邀访问 `guard-kit-code`。
