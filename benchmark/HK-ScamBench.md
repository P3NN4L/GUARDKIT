# HK-ScamBench v1 · 香港真实骗案零样本评测基准

[English](HK-ScamBench_EN.md) · 简体中文

**HK-ScamBench** 是一个面向香港场景的反诈检测小型评测基准：**12 个真实骗案家族逐案重构 + 8 条真实机构通知硬负例**，每条标注公开出处。用途：在模型从未见过的真实香港话术上，零样本地检验「可疑消息分类器」的召回与误报。

> 数据集：[HK-ScamBench-v1.jsonl](HK-ScamBench-v1.jsonl)（20 条，JSON Lines，含 `id` / `label`（scam|normal）/ `type`（imp 冒充 · pay 先付 · bait 诱饵）/ `src`（出处）/ `text`）
> 许可：CC BY 4.0（案例事实取材于公开警方案情与新闻报道，逐条见 `src` 字段；文本为按案情的忠实重构，非逐字转录）。

## 案例家族与出处

| # | 家族 | 出处 |
|---|---|---|
| S01 | 偽冒 WhatsApp 釣魚短訊 | ADCC 骗案警示 |
| S02 | 冒充 ADCC「回款」雙重詐騙 | 政府資科辦 IT2 通報（2026-02） |
| S03 | 轉數快「過錯錢」退款騙局 | Lapcom 用戶報料 / HK01 / HKET（2020-06） |
| S04 | 香港郵政假郵費釣魚（騙款 220 萬） | SCMP（2021-03） |
| S05 | Keeta 冇訂餐假扣款 | ADCC 骗案警示 |
| S06 | 淘寶「客服」退款屏幕共享 | 真實個案（Threads，2026-08） |
| S07 | 每日截圖 3 次兼職（入侵銀行） | HK01 |
| S08 | 猜猜我是誰（海關扣證件） | ADCC 經典騙案 |
| S09 | 冒充公檢法資金清查 | 香港警方警示 |
| S10 | 假冒 AlipayHK 釣魚 | CyberDefender 守網者 |
| S11 | WhatsApp 騎劫向親友借錢 | 匯豐 HSBC 警示 |
| S12 | 假投資導師（真實個案被騙 120 萬） | SafeCity.hk 罪案資料庫 |
| N01-N08 | 郵政取件 / 轉數快成功通知 / 消費券 / 大學繳費 / 屋苑維修 / 家人日常 / 功課討論 / 月結單 | 各機構真實通知格式 |

## 评测协议

1. **零样本**：被测系统不得使用本基准任何条目及其同义改写做训练/提示调优；
2. 输出二分类（scam / not_scam）或风险分（≥0.35 视为命中）；
3. 报告三项：召回（12 条骗案）、误报（8 条正常）、整体准确率；
4. 多轮语境系统可选用 S11 类铺垫+意图两段式评测（拼接后判定）。

## 基线结果（2026-09）

| 系统 | 召回 | 误报 | 准确率 | 备注 |
|---|---|---|---|---|
| 关键词规则 | 0% | 0% | 40.0% | HK 话术全面失明 |
| **guard-kit 雷达 v6.1（端内 852KB）** | **100%**（12/12，全高危档，校准分≥0.9） | 12.5%（1/8） | 95.0% | 0.04ms · 离线 · 免费 |
| MiniMax abab6.5s few-shot（云端） | 100% | 12.5% | 95.0% | 秒级 · 联网 · 按次计费 |
| MiniMax-M3 few-shot（旗舰，云端） | 100% | 0% | 100% | 秒级推理 · 联网 · 按次计费 |

> 结论供参考：旗舰大模型在无歧义真实案例上可达完美，但延迟/成本/隐私不可用于端内海量筛查；端内小模型 + 领域语料 + 校准可在 0.04ms 内达到同等召回。欢迎用你的系统复测并引用。

## 引用

```bibtex
@misc{hkscambench2026,
  title = {HK-ScamBench v1: A Zero-Shot Benchmark of Real Hong Kong Scam Case Families},
  author = {SettlePal / guard-kit team, Hong Kong Baptist University},
  year = {2026},
  howpublished = {\url{https://github.com/P3NN4L/GUARDKIT/tree/main/benchmark}},
  note = {Case sources: ADCC, ITHB/HK Government, SCMP, HK01, CyberDefender, HSBC, SafeCity}
}
```
