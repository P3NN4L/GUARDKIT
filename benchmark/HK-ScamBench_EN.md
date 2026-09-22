# HK-ScamBench v1 · A Zero-Shot Benchmark of Real Hong Kong Scam Case Families

English · [简体中文](HK-ScamBench.md)

**HK-ScamBench** is a small evaluation benchmark for scam-message detection in the Hong Kong context: **12 real scam case families reconstructed case-by-case + 8 real institutional notices as hard negatives**, each item source-tagged. Purpose: zero-shot testing of a suspicious-message classifier's recall and false positives on real Hong Kong scripts the system has never seen.

> Dataset: [HK-ScamBench-v1.jsonl](HK-ScamBench-v1.jsonl) — 20 items, JSON Lines, fields: `id` / `label` (scam|normal) / `type` (imp impersonation · pay pay-first · bait) / `src` (source) / `text`)
> License: CC BY 4.0. Case facts are drawn from public police advisories and news reports (per-item sources in the `src` field); texts are faithful reconstructions of the reported modus operandi, not verbatim transcripts.

## Case families and sources

| # | Family | Source |
|---|---|---|
| S01 | Fake WhatsApp phishing SMS | ADCC scam alert |
| S02 | Fake-ADCC "refund recovery" double scam | HK Government IT2 advisory (2026-02) |
| S03 | FPS "wrong transfer" refund trick | Lapcom user report / HK01 / HKET (2020-06) |
| S04 | Hongkong Post fake mailing fee phishing (HK$2.2M stolen) | SCMP (2021-03) |
| S05 | Keeta "order you never placed" fake charge | ADCC scam alert |
| S06 | Taobao "customer service" refund with screen sharing | real case (Threads, 2026-08) |
| S07 | "Screenshot 3×/day" part-time job (bank intrusion) | HK01 |
| S08 | Guess-who (customs detention) | ADCC classic |
| S09 | Fake law-enforcement fund inspection | HK Police warning |
| S10 | Fake AlipayHK phishing | CyberDefender |
| S11 | WhatsApp hijack borrowing from relatives | HSBC warning |
| S12 | Fake investment mentor (real victim lost HK$1.2M) | SafeCity.hk |
| N01-N08 | Post pickup / FPS success notice / voucher scheme / university fees / estate maintenance / family chat / coursework / bank statement | real institutional formats |

## Protocol

1. **Zero-shot**: the system under test must not train or prompt-tune on any item (or close paraphrases) of this benchmark;
2. Output binary (scam / not_scam) or a risk score (≥0.35 counts as flagged);
3. Report: recall (12 scam items), FPR (8 normal items), overall accuracy;
4. Multi-turn systems may additionally evaluate S11-style grooming+intent two-step conversations (judged on the concatenation).

## Baselines (2026-09)

| System | Recall | FPR | Accuracy | Notes |
|---|---|---|---|---|
| Keyword rules | 0% | 0% | 40.0% | completely blind to HK scripts |
| **guard-kit Radar v6.1 (on-device, 852KB)** | **100%** (12/12, all high band, calibrated ≥ 0.9) | 12.5% (1/8) | 95.0% | 0.04ms · offline · free |
| MiniMax abab6.5s few-shot (cloud) | 100% | 12.5% | 95.0% | seconds · online · per-call cost |
| MiniMax-M3 few-shot (flagship, cloud) | 100% | 0% | 100% | seconds of reasoning · online · per-call cost |

> Reading: a flagship LLM can be perfect on unambiguous real cases, but its latency/cost/privacy profile rules out on-device mass screening; a small on-device model with domain corpus and calibration reaches the same recall at 0.04ms. Re-run with your own system and cite freely.

## Citation

```bibtex
@misc{hkscambench2026,
  title = {HK-ScamBench v1: A Zero-Shot Benchmark of Real Hong Kong Scam Case Families},
  author = {SettlePal / guard-kit team, Hong Kong Baptist University},
  year = {2026},
  howpublished = {\url{https://github.com/P3NN4L/GUARDKIT/tree/main/benchmark}},
  note = {Case sources: ADCC, ITHB/HK Government, SCMP, HK01, CyberDefender, HSBC, SafeCity}
}
```
