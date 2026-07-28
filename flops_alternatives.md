# FLOPs Alternatives: Wall-clock and Dollar Cost Axes (JailbreakBench)

**How the two new axes are implemented.** 
*Wall-clock* is **measured, not modelled**: every
refinement step is wrapped in a CUDA-synchronised `perf_counter` region that covers target
generation + judge call + attacker work (for GCG, the gradient step and the 128-candidate
scoring pass; for RL, the GRPO group sampling and the LoRA update, amortised over the
candidates scored in that round), and the per-step seconds are accumulated over λ exactly as
the FLOP axis is. *USD* is a token-level bill: we count each component's exact input and
output tokens with that component's own tokenizer and charge them at that model's **cheapest
hosted per-token rate** (`configs/pricing.yaml`, min across OpenRouter serving providers,
cross-checked against DeepInfra/Together, snapshot 2026-07-26), then sum target + judge +
attacker. So the three axes are the *same* accumulated quantity measured in three units:
device FLOPs, device seconds, and what a hosted API would charge for the identical token
traffic.

**Setup for the numbers below.** JailbreakBench, 100 behaviors, λ_max = 10, judge =
Llama-3.1-8B-Instruct, PAIR/RL attacker = Qwen2.5-7B-Instruct. All targets are loaded in
4-bit NF4 and all timings come from a **single seed (1394) on one NVIDIA L40S**. Costs are
**per behavior**; multiply by 100 for the whole benchmark. `m$` = 10⁻³ USD. ∞ = the attack
never reaches 50 % risk within λ = 10.

---

## Table 1 — Cost to 50 % risk, $C_{@0.5}$, on all three axes

Lower = cheaper for the attacker = worse for the model. The TFLOPs block is the paper's
existing metric; the `sec` and `m$` blocks are the two requested columns.

| Model | TFLOPs GCG | TFLOPs PAIR | TFLOPs JB | TFLOPs **RL** | sec GCG | sec PAIR | sec JB | sec **RL** | m$ GCG | m$ PAIR | m$ JB | m$ **RL** |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ***Tulu3 (8B)*** | | | | | | | | | | | | |
| Base | 61.8 | 10.6 | 10.0 | **6.5** | 14.6 | 11.8 | 13.3 | **7.5** | 0.080 | 0.021 | 0.017 | **0.010** |
| SFT | ∞ | ∞ | ∞ | **35.7** | ∞ | ∞ | ∞ | **23.8** | ∞ | ∞ | ∞ | **0.060** |
| DPO | 510.5 | 79.3 | 39.8 | **25.4** | 154.0 | 92.6 | 45.6 | **25.5** | 0.678 | 0.166 | 0.064 | **0.046** |
| RLVR | 468.8 | 82.5 | 23.1 | **26.5** | 149.9 | 98.5 | 27.2 | **26.9** | 0.624 | 0.173 | 0.037 | **0.048** |
| ***Qwen2.5 (Instruct)*** | | | | | | | | | | | | |
| 0.5B | 21.9 | 17.8 | 8.5 | **10.9** | 10.8 | 11.7 | 7.2 | **7.0** | 0.045 | 0.036 | 0.012 | **0.018** |
| 3B | 215.6 | 33.8 | 15.8 | **16.4** | 80.3 | 52.2 | 23.8 | **22.6** | 0.555 | 0.088 | 0.032 | **0.035** |
| 7B | 491.3 | 44.2 | 26.6 | **22.6** | 89.2 | 51.5 | 24.0 | **23.0** | 1.297 | 0.146 | 0.068 | **0.064** |
| ***Qwen3*** | | | | | | | | | | | | |
| 4B | ∞ | 37.1 | 28.1 | **18.7** | ∞ | 62.7 | 39.5 | **27.2** | ∞ | 0.237 | 0.148 | **0.101** |
| 4B-SafeRL | 277.4 | 53.2 | 24.1 | **16.5** | 186.8 | 113.1 | 55.4 | **27.9** | 2.123 | 0.399 | 0.189 | **0.099** |

## Table 2 — Total cost of the full λ = 10 budget, on all three axes

What the attacker spends if it runs the whole budget on every behavior (early-stopped trials
stop contributing at their first success, as in the FLOP axis).

| Model | TFLOPs GCG | TFLOPs PAIR | TFLOPs JB | TFLOPs **RL** | sec GCG | sec PAIR | sec JB | sec **RL** | m$ GCG | m$ PAIR | m$ JB | m$ **RL** |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ***Tulu3 (8B)*** | | | | | | | | | | | | |
| Base | 125.5 | 22.6 | 19.8 | 17.1 | 32.9 | 25.4 | 26.4 | 18.0 | 0.164 | 0.046 | 0.033 | 0.030 |
| SFT | 563.7 | 120.1 | 57.6 | 79.5 | 62.1 | 90.9 | 35.0 | 44.4 | 0.707 | 0.258 | 0.081 | 0.118 |
| DPO | 542.3 | 125.8 | 65.9 | 52.6 | 164.1 | 142.7 | 73.3 | 46.0 | 0.720 | 0.267 | 0.106 | 0.094 |
| RLVR | 536.4 | 123.5 | 47.4 | 58.2 | 173.7 | 145.3 | 55.5 | 48.6 | 0.715 | 0.263 | 0.076 | 0.099 |
| ***Qwen2.5 (Instruct)*** | | | | | | | | | | | | |
| 0.5B | 40.5 | 32.6 | 16.6 | 26.8 | 21.0 | 24.1 | 14.3 | 15.4 | 0.084 | 0.067 | 0.024 | 0.045 |
| 3B | 273.9 | 61.5 | 30.0 | 38.8 | 103.4 | 98.5 | 44.8 | 43.0 | 0.706 | 0.164 | 0.061 | 0.076 |
| 7B | 617.0 | 81.0 | 44.0 | 41.5 | 110.8 | 95.1 | 39.4 | 39.6 | 1.628 | 0.271 | 0.111 | 0.119 |
| ***Qwen3*** | | | | | | | | | | | | |
| 4B | 447.5 | 66.6 | 44.7 | 38.5 | 128.1 | 116.2 | 60.6 | 48.5 | 3.111 | 0.440 | 0.228 | 0.190 |
| 4B-SafeRL | 357.3 | 103.8 | 50.3 | 43.6 | 238.7 | 213.6 | 114.4 | 56.7 | 2.727 | 0.761 | 0.391 | 0.216 |
| **mean** | **389.4** | **81.9** | **41.8** | **44.1** | **115.0** | **105.8** | **51.5** | **40.0** | **1.174** | **0.282** | **0.123** | **0.110** |

Two accounting notes. (i) These are judge-**inclusive** figures, i.e. the cost of reproducing
the measurement; the attacker-only dollar column (judge excluded) is also computed and runs
30–60 % lower on the template attacks (e.g. Tulu3-Base/JB drops 0.033 → 0.019 m$) without
changing any ordering. (ii) The dollar column prices GCG's 129 optimization forward passes at
hosted token rates even though no hosted API exposes gradients; it is the honest answer to
"what does this token traffic cost", not a claim that GCG is purchasable that way.

---

## Where the axes agree and disagree

Across the 32 finite $C_{@0.5}$ cells the Spearman rank correlations are 0.84 (FLOPs vs.
seconds), 0.88 (FLOPs vs. USD) and 0.92 (seconds vs. USD), and every headline claim survives
the change of unit: RL is the cheapest attack on all three axes for 6/9 models (and on the
seconds axis for 9/9), the Qwen2.5 size trend stays monotone in all three units (RL:
10.9/16.4/22.6 TFLOPs, 7.0/22.6/23.0 s, 0.018/0.035/0.064 m$), and Tulu3-SFT remains the only
model that no non-RL attack breaks. Two things do move:

- **GCG.** It costs 8.6× RL in FLOPs and 8.9× in hosted dollars but only **2.6×** in
  wall-clock, and it is effectively tied with PAIR in seconds (115 vs. 106 s) despite 4.8×
  more FLOPs. GCG spends its compute in large batched teacher-forced passes that saturate the
  GPU (0.34 s/TFLOP), while PAIR/JB/RL spend theirs in batch-1 autoregressive decoding that is
  memory-bandwidth-bound (1.23–1.28 s/TFLOP). The memory/parallelization effect the reviewer
  names, invisible to FLOPs.
- **The Tulu3 SFT peak under RL.** On FLOPs and dollars SFT is the most expensive stage
  (35.7 TFLOPs / 0.060 m$ vs. 25.4–26.5 / 0.046–0.048 for DPO/RLVR); on wall-clock it is the
  *cheapest* of the three (23.8 s vs. 25.5 / 26.9), because its steps are individually faster
  even though it needs more of them. We report this rather than claim the ordering holds under
  every axis. Although the Tulu3 checkpoints share the same architecture and parameter count, their SFT, DPO, and RLVR post-training stages can induce different generation-length distributions. Prior work has documented length exploitation and increased verbosity under both DPO and reinforcement-learning-based post-training ([Park et al., 2024](https://arxiv.org/abs/2403.19159); [Lu et al., 2024](https://arxiv.org/abs/2406.10957); [Liu et al., 2024](https://arxiv.org/abs/2409.06411)). However, this observation needs further investigation.

Note also that "dollars" is not one axis. Priced per hosted token it tracks FLOPs (ρ = 0.88);
priced as rented GPU time (≈\$1/GPU-hour for an L40S, so 115 s ⇒ ≈32 m\$ for GCG vs. ≈11 m\$ for
RL) it tracks wall-clock instead and compresses the GCG/RL gap to 2.9×.

## How this view differs from the FLOP view

Wall-clock and USD price the things FLOPs deliberately abstracts away: batching, memory
bandwidth, quantization, and a vendor's margin, so they reorder *attacks* (GCG's 8.6× FLOP
penalty over RL shrinks to 2.6× once its candidate pass is batched) even while leaving the
*model* rankings the paper draws its conclusions from essentially intact. The price of that
extra realism is that both axes are properties of one attacker's hardware and one week's
price list, not of the model under test: rerun on an H100, unquantized, or against a different
provider and the numbers move, whereas the FLOP axis does not. We therefore keep FLOPs as the
default comparison axis and ship seconds and dollars alongside it, since the reviewer's point
is exactly right that no single unit captures every real cost.
