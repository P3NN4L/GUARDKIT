# guard-kit 接入指南（漂管家 SettlePal 反诈引擎）

guard-kit 是**端内反诈打分引擎**：零依赖、完全离线、单条 <0.1ms、四语（简/繁/粤/英）。它只输出决策与概率、不生成文本，作为融合架构的一层与你的规则通道、LLM 语义通道配合。本指南面向接入方：从选包到上线，一站读完。

> 当前发布：`@p3nn4l/guard-kit@7.0.0`（雷达 v7 · 裁判 v6，含机构白名单 v3 的源码仓对应文档）。v8 蒸馏学生与五引擎委员会已在源码仓落地（见指标节），组合包发布前安装通道不变。

## 30 秒决定装哪个包

| 包 | 内容 | 体积 | 适合 |
|---|---|---|---|
| `npm install @p3nn4l/guard-kit` | 雷达 + 裁判（同包，可子路径拆用） | ~1.7MB | 需要双模型（陪练/教育类） |
| `npm install @p3nn4l/guard-kit-radar` | **只有雷达**（引擎+权重） | ~770KB | 只要风控（银行/社区/聊天类） |

版本回退直装：`@7.0.0`（雷达 7 · 裁判 6）/ `@6.2.0`（M3 蒸馏版）/ `@6.1.0`（HK 加固版）/ `@6.0.0`（首发）。

## TS / RN / Expo（3 行上手）

```bash
npm install @p3nn4l/guard-kit@7.0.0
```

```ts
import { mlScore, mlConversation, replyScore } from '@p3nn4l/guard-kit';

// 单条消息风险（雷达）
const r = mlScore('【香港郵政】包裹已扣留，請登記：hkpost-redelivery.top');
// r.band: 'high' | 'medium' | 'low'   r.pCal: 校准概率   r.topType: 骗术大类

// 多轮对话（v6+ 新增，六端同 API）
mlConversation(messages);   // 铺垫句不误报；意图一出现即触发；trigger='context' 时 UI 标注「结合上文判定」

// 回复质量（裁判，陪练/教育场景）
replyScore(reply);
```

雷达输出三档 `band`，直接映射产品动作：**high** → 弹窗拦截并展示理由；**medium** → 角标提醒；**low** → 静默。档位阈值即误报率的产品语义（0.75/0.35 + 双重确认），无需自行调参。

## 全 API 参考（TS）

| API | 作用 | 返回 |
|---|---|---|
| `mlScore(text)` | 单条消息风险 | `{ pRaw, pCal, band, topType, probs }` |
| `mlConversation(messages)` | 多轮语境（最近 4 轮拼接，取最大分） | 同上 + `trigger: 'direct' \| 'context'` |
| `replyScore(reply)` | 回复质量四分类（中英粤） | `{ topClass, confCal }` |
| `mlScoreV62 / mlScoreV6 / replyScoreV5`（`/legacy`） | 挂载旧代（灰度/回退） | 同各代 |
| `createRadar(modelJson)` / `createReferee(modelJson)` | 工厂：传入权重字符串自建实例 | 引擎对象 |

小程序/云端导出名一致；原生端见下节构造器。

## 六端接入

| 端 | 接入方式 |
|---|---|
| TS / RN / Expo / Node | `npm install @p3nn4l/guard-kit`（GitHub Packages 私有包，需受邀 token） |
| iOS（Swift · SPM） | 私有仓 URL：`https://github.com/P3NN4L/guard-kit-code`，受邀 GitHub 授权后 Xcode 直接解析 |
| Android（Kotlin · Maven） | `maven { url = uri("https://maven.pkg.github.com/P3NN4L/guard-kit-code") }` + `implementation("com.settlepal:guardkit:7.0.0")` |
| Flutter（Dart） | pubspec git 依赖（受邀）或 Release 附件 |
| 微信小程序 | Release 附件 `guard-kit-miniprogram.zip`（`guard-kit-radar.js` 只含雷达，更小） |
| 云端 Worker | Release 附件 `guard-kit-worker.zip`（dist-only，零依赖） |

发布物全部 **dist-only**：压缩 JS/原生库 + 类型声明 + 内联权重——无 TS 源码、无训练管线、无语料。

## 指标：演化五里程碑（全部 fresh 盲测卷实测）

| 引擎 | 演化身份 | v12 | v10 | v11 | public120（120 条真实短信） |
|---|---|---|---|---|---|
| v5 | 起点：真实语料落地前 | — | — | — | 60.2 / 15.7（外部真实基准） |
| v6.2 | 换代：语料重基 | — | 85.7/42.9 | 76.9/33.3 | 87.3/4.6 |
| v7（现役包） | 进化：多轮语境+机构白名单 | 93/12 | 86/29 | 77/8 | **89.1/1.5** |
| 群集 | 决策层：五席异构多数票 | 100/12 | 100/43 | 85/8 | 94.5/1.5 |
| v8 | 换发动机：蒸馏学生主从融合（源码仓已落地） | 92.9/6.2 | 85.7/0 | 84.6/0 | 78.2/1.5 |
| LLM few-shot | 外部参照：旗舰 M3 给 4 例 | 100/0 | 100/0 | 100/0 | 58.2/3.1 |

（召回/误报@mid 档。v8 相对 v7：11 卷 10 胜，唯 public120 召回让位——v7 以 0.3 权重留在融合内贡献该域。）跨规模铁律：0.6B 冻结 41.8 → 8B 微调 49.1 → **20KB 线性 v7：89.1**——域数据+校准 > 参数规模。效率：雷达 852KB / mean 0.078ms（4,900 次微基准）。

## 谁能看到什么（权限边界）

| 对象 | 可见范围 |
|---|---|
| npm 包安装者 | dist-only 产物（压缩 JS + 类型 + 内联权重） |
| Release 附件下载者 | 各端 dist 包 |
| Fine-grained token（单仓 Contents:read） | 可解析 SPM 私有仓 URL；仍无语料/台账 |
| 引擎仓 Collaborator | 源码与训练管线（仅限核心成员） |

## FAQ

**Q：档位阈值要自己调吗？**
不。0.75/0.35 + 双重确认是产品语义（阈值=误报率），各端内置一致。

**Q：离线可用？**
完全离线。零网络代码、零遥测，消息不出设备。

**Q：粤语/繁体/中英夹杂？**
原生支持。词表四语配额，粤语多轮对话在训练语料中原生出现。

**Q：多轮怎么用？**
把最近消息按时间序传 `mlConversation(messages)`。铺垫句不误报，意图出现即触发；`trigger='context'` 时 UI 标注「结合上文判定」。

**Q：商用授权？**
双档许可（LICENSE）：**个人与非商业用途免费**（学习、研究、教学、竞赛、非商业项目）；**商用须付费授权**，联系 [github.com/P3NN4L](https://github.com/P3NN4L)。权重不得单独用于训练其他模型。

**Q：基准数据集？**
[HK-ScamBench](benchmark/HK-ScamBench.md)（12 真实骗案家族 + 8 机构通知，CC BY 4.0）可引用复测。

---

架构细节、训练协议与完整消融见源码仓 README（受邀可见）：`P3NN4L/guard-kit-code`。
