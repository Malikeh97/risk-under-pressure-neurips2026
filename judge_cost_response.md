# Dependence of the Compute Cost on the Judge
---

## 1. Rebuttal text

> This is right that the judge term is non-negligible, and we thank you for pushing on
> it. Two clarifications.
>
> **(a) The per-attack formulas are instantiations, not the definition.** The cost model in Sec. 3
> is a sum of per-component forward/backward passes; the GCG, PAIR and Jailbroken formulas we give
> are proof-of-concept instantiations of that sum for three representative attacks. The released
> framework is dynamically configurable: every component (target, attacker, judge) is a separate
> accounted term, model sizes come from config, and each term can be included or excluded at
> analysis time. Adding an attack means adding its per-step pass count, not changing the metric.
>
> **(b) Which terms to charge is an accounting choice, and it depends on whose budget is being
> measured.** We make this explicit in the revision: (i) *deployer*: only the target's forward
> passes, since attacker and judge run off the deployer's infrastructure; (ii) *evaluator*:
> the end-to-end pipeline (target + attacker + judge), i.e. the cost of reproducing the
> measurement; (iii) *adversary*: target queries plus their own
> attacker model, and, for attacks whose next step is chosen automatically from a judge signal
> (PAIR's refinement, RL attacker's reward, etc.), the judge as well. The judge is therefore a real
> adversary-side cost for in-the-loop attacks, and a pure evaluation overhead for GCG and
> Jailbroken. 
>
> To isolate the judge-size effect we now report every FLOP metric on a **judge-excluded** axis
> (target + attacker only) alongside the default. Removing an 8B judge — larger than or equal to
> the target in 6 of 9 models — rescales the cost axis but leaves every conclusion intact:
> per-attack Spearman ρ between judge-included and judge-excluded rankings is 0.92–1.00
> (pooled ρ = 0.97 for AE, 0.95 for C@0.5). Table R1 reports both.

---

## 2. Table R1 — `tab:cost_summary` with judge-excluded rows added

The published **w/ judge** numbers are the original ones, unchanged. Each model gains one **w/o judge** row carrying the two FLOP metrics on the target + attacker axis. ASR is not repeated on those rows: it is defined over attack steps, the trajectories are identical, and only the cost axis is rescaled.


### Jailbreak Robustness Metrics on HarmBench

| Model | Compute | C@0.5 GCG | C@0.5 PAIR | C@0.5 JB | AE GCG | AE PAIR | AE JB | ASR GCG | ASR PAIR | ASR JB |
|---|---|---|---|---|---|---|---|---|---|---|
| Tulu3 Base | w/ judge | 59.3 | 11.2 | 9.2 | 8.4 | 39.0 | 53.3 | 1.00 | 1.00 | 1.00 |
| Tulu3 Base | w/o judge | 55.7 | 6.4 | 3.8 | 8.6 | 68.4 | 129.2 | — | — | — |
| Tulu3 SFT | w/ judge | ∞ | ∞ | 52.4 | 0.5 | 3.5 | 8.9 | 0.31 | 0.42 | 0.50 |
| Tulu3 SFT | w/o judge | ∞ | ∞ | 13.3 | 0.6 | 6.4 | 37.7 | — | — | — |
| Tulu3 DPO | w/ judge | 521.2 | 79.9 | 40.9 | 1.0 | 6.0 | 10.4 | 0.52 | 0.75 | 0.67 |
| Tulu3 DPO | w/o judge | ∞ | 39.3 | 14.5 | 1.0 | 11.5 | 31.4 | — | — | — |
| Tulu3 RLVR | w/ judge | 503.6 | 72.4 | 25.7 | 1.0 | 6.7 | 18.9 | 0.54 | 0.79 | 0.90 |
| Tulu3 RLVR | w/o judge | 447.2 | 42.7 | 10.0 | 1.2 | 12.1 | 48.1 | — | — | — |
| Qwen2.5 0.5B | w/ judge | 20.0 | 15.5 | 8.2 | 25.6 | 30.6 | 59.6 | 0.99 | 0.99 | 0.99 |
| Qwen2.5 0.5B | w/o judge | 8.5 | 6.1 | 0.3 | 58.7 | 74.3 | 1677.8 | — | — | — |
| Qwen2.5 3B | w/ judge | 173.7 | 33.9 | 13.4 | 3.3 | 15.9 | 36.8 | 0.81 | 0.97 | 0.98 |
| Qwen2.5 3B | w/o judge | 146.5 | 16.5 | 2.5 | 3.9 | 33.6 | 217.8 | — | — | — |
| Qwen2.5 7B | w/ judge | 399.7 | 38.9 | 22.8 | 1.3 | 13.6 | 23.0 | 0.73 | 0.97 | 0.94 |
| Qwen2.5 7B | w/o judge | 337.5 | 22.5 | 6.7 | 1.5 | 21.8 | 76.8 | — | — | — |
| Qwen3 4B | w/ judge | ∞ | 31.3 | 21.2 | 0.9 | 16.6 | 22.1 | 0.36 | 0.98 | 0.86 |
| Qwen3 4B | w/o judge | ∞ | 15.5 | 4.6 | 1.0 | 33.4 | 105.7 | — | — | — |
| Qwen3 4B-SafeRL | w/ judge | 189.0 | 44.8 | 24.5 | 2.1 | 7.6 | 16.0 | 0.67 | 0.75 | 0.83 |
| Qwen3 4B-SafeRL | w/o judge | 299.5| 21.5 | 6.4 | 1.6 | 16.0 | 67.4 | — | — | — |
---

## 3. Supporting numbers to quote in the response

**Judge share of total FLOPs at $\lambda=10$** (across the 9 HarmBench targets, judge =
Llama-3.1-8B-Instruct):

| Attack | judge share of total FLOPs (min – max) | median | judge in the loop? |
|---|---|---|---|
| GCG | 8.6% – 56.6% | 11.4% | no (evaluation only) |
| PAIR | 40.2% – 58.4% | 43.1% | **yes** (score drives refinement) |
| Jailbroken | 59.4% – 96.4% | 74.2% | no (evaluation only) |
| RL (ours) | 35.3% – 63.7% | 48.6% | **yes** (judge = reward) |

**Rank stability** (with-judge vs. no-judge orderings over the 9 targets):

| Attack | Spearman ρ (AE) | Kendall τ (AE) | Spearman ρ (C@0.5) |
|---|---|---|---|
| GCG | 0.967 | 0.889 | 1.000 (n=6 finite) |
| PAIR | 0.967 | 0.889 | 0.952 (n=8 finite) |
| Jailbroken | 0.933 | 0.778 | 0.917 (n=9 finite) |
| **pooled** | **0.973** | — | **0.953** (n=23 finite) |

**Cost-axis rescaling factors** (no-judge / with-judge, median over targets):

| Attack | C@0.5 ratio | AE ratio |
|---|---|---|
| GCG | 0.86 | 1.13 |
| PAIR | 0.54 | 1.76 |
| Jailbroken | 0.26 | 3.88 |

Reading: dropping the judge is close to a **per-attack multiplicative rescaling of the cost
axis**, because the judge contributes one forward pass per step on a model whose size is fixed
across targets. It shifts absolute FLOP levels, and it compresses the *between-attack* gaps; it
does not reorder the *between-model* comparisons that carry the paper's claims.
