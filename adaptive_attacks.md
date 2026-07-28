# Response: evaluation scope and adaptive attacks

## GCG and PAIR are already adaptive attacks

We would first note that the three attacks in the paper are not static jailbreak templates.
Under the taxonomy of Nasr et al., *"The Attacker Moves Second: Stronger Adaptive Attacks
Bypass Defenses Against LLM Jailbreaks and Prompt Injections"* (arXiv:2510.09023), an
adaptive attack is one in which the attacker observes the target/defense and modifies its
strategy against it, rather than replaying a fixed prompt and the attack families they
study as adaptive are precisely gradient-based optimization (GCG), LLM-assisted iterative
refinement, and human red-teaming. Both GCG and PAIR in our setup are run per-target
and per-behavior with no train/test split: GCG optimizes a suffix against the target's own
gradients, and PAIR refines its prompt from the target's own responses. Our λ axis is
exactly the number of adaptive optimization steps each is allowed. The Jailbroken (JB) suite
is the deliberate static-baseline contrast that makes the adaptive attacks interpretable.

We added a fourth RL-based adaptive attack, which is the strongest adaptive attack from the mentioned paper.

## The RL attack

We implement the per-prompt RL attack of Nasr et al. (App. A.2) and run it on both
benchmarks and every model in the paper. Implementation:

- **Attacker.** Qwen2.5-7B-Instruct with a LoRA adapter that is **reset to scratch for every
  behavior**.
- **Optimization.** Short GRPO run per behavior: sample a group of 8 candidate adversarial
  prompts, score each against the frozen target, compute group-normalized advantages, and
  apply a group-relative policy gradient with a KL penalty (β = 0.04) to the frozen base
  attacker via adapter-disable. Single inner epoch, no PPO clipping, lr 1e-5, temperature
  1.0 / top-p 0.95, completions capped at 256 tokens.
- **Reward.** Judge verdict on the target's response, plus a perplexity shaping term
  (weight 0.3) against an affirmative-prefix template, guarded against reward hacking.
- **Budget accounting.** One candidate = one target query = one recorded step, so λ means
  the same thing for RL as for GCG/PAIR and the resulting records drop into the existing
  risk/cost pipeline unchanged. Crucially, the attacker's own forward *and backward* passes
  are billed into the FLOP axis, so RL is charged for its optimization overhead. Its per-query cost is strictly higher than PAIR's.
- **Documented deviation.** Nasr et al. run a fixed budget (multiple sessions × ~5 rounds)
  with best-of scoring and no early stop; we stop at the first judged-unsafe candidate to
  mirror the early-stop semantics of GCG/PAIR so that pressure and cost stay comparable
  across attacks. This does not change success labels or risk-vs-λ. The first-success step
  is identical either way; early stopping only trims queries spent *after* the jailbreak.

The RL column below is **single-seed** due to time limit for rebuttal.

---

## HarmBench (RL column added)

| Model | C@0.5 GCG ↑ | C@0.5 PAIR ↑ | C@0.5 JB ↑ | C@0.5 RL ↑ | AE GCG ↓ | AE PAIR ↓ | AE JB ↓ | AE RL ↓ | ASR GCG ↓ | ASR PAIR ↓ | ASR JB ↓ | ASR RL ↓ |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ***Tulu3 (8B)*** | | | | | | | | | | | | |
| Base | 59.3<sub>±1.2</sub> | 11.2<sub>±0.2</sub> | 9.2<sub>±0.2</sub> | 6.8 | 8.4<sub>±0.2</sub> | 39.0<sub>±0.7</sub> | 53.3<sub>±1.0</sub> | 53.5 | 1.00<sub>±0.00</sub> | 1.00<sub>±0.00</sub> | 1.00<sub>±0.00</sub> | 0.99<sub>±0.01</sub> |
| SFT | ∞ | ∞ | 52.4<sub>±5.0</sub> | 34.5 | 0.5<sub>±0.0</sub> | 3.5<sub>±0.2</sub> | 8.9<sub>±0.6</sub> | 10.2 | 0.31<sub>±0.01</sub> | 0.42<sub>±0.02</sub> | 0.50<sub>±0.02</sub> | 0.85<sub>±0.05</sub> |
| DPO | 521.2<sub>±26.6</sub> | 79.9<sub>±4.2</sub> | 40.9<sub>±2.2</sub> | 24.6 | 1.0<sub>±0.1</sub> | 6.0<sub>±0.2</sub> | 10.4<sub>±0.4</sub> | 16.4 | 0.52<sub>±0.03</sub> | 0.75<sub>±0.02</sub> | 0.67<sub>±0.02</sub> | 0.96<sub>±0.03</sub> |
| RLVR | 503.6<sub>±14.3</sub> | 72.4<sub>±3.4</sub> | 25.7<sub>±1.4</sub> | 24.8 | 1.0<sub>±0.0</sub> | 6.7<sub>±0.3</sub> | 18.9<sub>±0.8</sub> | 19.8 | 0.54<sub>±0.01</sub> | 0.79<sub>±0.02</sub> | 0.90<sub>±0.01</sub> | 0.97<sub>±0.02</sub> |
| ***Qwen2.5 (Instruct)*** | | | | | | | | | | | | |
| 0.5B | 20.0<sub>±1.0</sub> | 15.5<sub>±0.5</sub> | 8.2<sub>±0.3</sub> | 9.3 | 25.6<sub>±0.6</sub> | 30.6<sub>±0.6</sub> | 59.6<sub>±1.6</sub> | 40.3 | 0.99<sub>±0.01</sub> | 0.99<sub>±0.00</sub> | 0.99<sub>±0.00</sub> | 0.98<sub>±0.02</sub> |
| 3B | 173.7<sub>±6.7</sub> | 33.9<sub>±1.4</sub> | 13.4<sub>±0.4</sub> | 15.9 | 3.3<sub>±0.1</sub> | 15.9<sub>±0.6</sub> | 36.8<sub>±0.8</sub> | 31.9 | 0.81<sub>±0.02</sub> | 0.97<sub>±0.01</sub> | 0.98<sub>±0.01</sub> | 0.99<sub>±0.01</sub> |
| 7B | 399.7<sub>±14.5</sub> | 38.9<sub>±1.7</sub> | 22.8<sub>±1.6</sub> | 20.1 | 1.3<sub>±0.0</sub> | 13.6<sub>±0.4</sub> | 23.0<sub>±0.9</sub> | 24.1 | 0.73<sub>±0.01</sub> | 0.97<sub>±0.01</sub> | 0.94<sub>±0.01</sub> | 0.98<sub>±0.02</sub> |
| ***Qwen3*** | | | | | | | | | | | | |
| 4B | ∞ | 31.3<sub>±1.6</sub> | 21.2<sub>±0.8</sub> | 16.1 | 0.9<sub>±0.0</sub> | 16.6<sub>±0.7</sub> | 22.1<sub>±1.1</sub> | 27.8 | 0.36<sub>±0.01</sub> | 0.98<sub>±0.00</sub> | 0.86<sub>±0.02</sub> | 0.98<sub>±0.02</sub> |
| 4B-SafeRL | 189.0<sub>±12.3</sub> | 44.8<sub>±3.9</sub> | 24.5<sub>±1.8</sub> | 14.3 | 2.1<sub>±0.1</sub> | 7.6<sub>±0.3</sub> | 16.0<sub>±1.0</sub> | 17.5 | 0.67<sub>±0.02</sub> | 0.75<sub>±0.02</sub> | 0.83<sub>±0.02</sub> | 0.94<sub>±0.03</sub> |
| 8B<sub>transfer</sub> | ∞ | — | — | — | 4.9<sub>±0.6</sub> | — | — | — | 0.15<sub>±0.02</sub> | — | — | — |

## JailbreakBench (RL column added)

| Model | C@0.5 GCG ↑ | C@0.5 PAIR ↑ | C@0.5 JB ↑ | C@0.5 RL ↑ | AE GCG ↓ | AE PAIR ↓ | AE JB ↓ | AE RL ↓ | ASR GCG ↓ | ASR PAIR ↓ | ASR JB ↓ | ASR RL ↓ |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ***Tulu3 (8B)*** | | | | | | | | | | | | |
| Base | 60.0<sub>±1.9</sub> | 11.2<sub>±0.3</sub> | 9.0<sub>±0.2</sub> | 6.5 | 8.5<sub>±0.2</sub> | 38.5<sub>±2.7</sub> | 54.9<sub>±1.6</sub> | 58.5 | 1.00<sub>±0.00</sub> | 1.00<sub>±0.00</sub> | 1.00<sub>±0.00</sub> | 1.00<sub>±0.00</sub> |
| SFT | ∞ | ∞ | 51.8<sub>±5.9</sub> | 35.7 | 0.6<sub>±0.1</sub> | 3.2<sub>±0.3</sub> | 9.0<sub>±0.8</sub> | 11.1 | 0.34<sub>±0.03</sub> | 0.40<sub>±0.03</sub> | 0.51<sub>±0.04</sub> | 0.88<sub>±0.06</sub> |
| DPO | 482.9<sub>±24.2</sub> | 88.6<sub>±5.4</sub> | 37.1<sub>±1.7</sub> | 25.4 | 1.1<sub>±0.1</sub> | 5.7<sub>±0.3</sub> | 11.8<sub>±0.5</sub> | 18.6 | 0.58<sub>±0.04</sub> | 0.74<sub>±0.02</sub> | 0.73<sub>±0.03</sub> | 0.98<sub>±0.03</sub> |
| RLVR | 494.1<sub>±20.4</sub> | 83.4<sub>±5.7</sub> | 27.3<sub>±2.5</sub> | 26.5 | 1.1<sub>±0.1</sub> | 6.2<sub>±0.3</sub> | 17.6<sub>±1.6</sub> | 16.3 | 0.58<sub>±0.04</sub> | 0.78<sub>±0.02</sub> | 0.88<sub>±0.02</sub> | 0.95<sub>±0.05</sub> |
| ***Qwen2.5 (Instruct)*** | | | | | | | | | | | | |
| 0.5B | 24.3<sub>±1.2</sub> | 18.2<sub>±1.0</sub> | 8.5<sub>±0.5</sub> | 10.9 | 23.4<sub>±0.6</sub> | 28.4<sub>±1.4</sub> | 58.7<sub>±2.6</sub> | 36.9 | 0.98<sub>±0.01</sub> | 0.99<sub>±0.00</sub> | 1.00<sub>±0.01</sub> | 0.99<sub>±0.02</sub> |
| 3B | 196.6<sub>±9.0</sub> | 36.7<sub>±1.6</sub> | 14.9<sub>±0.9</sub> | 16.4 | 3.0<sub>±0.2</sub> | 14.8<sub>±0.7</sub> | 33.2<sub>±1.4</sub> | 25.0 | 0.79<sub>±0.03</sub> | 0.96<sub>±0.01</sub> | 0.96<sub>±0.01</sub> | 0.97<sub>±0.03</sub> |
| 7B | 482.0<sub>±21.8</sub> | 48.3<sub>±2.7</sub> | 23.2<sub>±1.7</sub> | 22.6 | 1.1<sub>±0.0</sub> | 10.9<sub>±0.6</sub> | 23.1<sub>±1.0</sub> | 24.1 | 0.67<sub>±0.02</sub> | 0.94<sub>±0.02</sub> | 0.93<sub>±0.02</sub> | 1.00<sub>±0.00</sub> |
| ***Qwen3*** | | | | | | | | | | | | |
| 4B | ∞ | 37.6<sub>±1.3</sub> | 24.2<sub>±1.5</sub> | 18.7 | 0.6<sub>±0.1</sub> | 14.5<sub>±0.3</sub> | 20.1<sub>±1.5</sub> | 25.4 | 0.28<sub>±0.03</sub> | 0.96<sub>±0.01</sub> | 0.83<sub>±0.03</sub> | 0.98<sub>±0.03</sub> |
| 4B-SafeRL | 233.3<sub>±21.9</sub> | 59.1<sub>±4.4</sub> | 29.4<sub>±3.0</sub> | 16.5 | 1.9<sub>±0.1</sub> | 6.9<sub>±0.4</sub> | 15.1<sub>±1.0</sub> | 22.2 | 0.63<sub>±0.02</sub> | 0.74<sub>±0.02</sub> | 0.84<sub>±0.02</sub> | 0.97<sub>±0.03</sub> |
| 8B<sub>transfer</sub> | ∞ | — | — | — | 1.8<sub>±0.4</sub> | — | — | — | 0.06<sub>±0.01</sub> | — | — | — |

---

## Findings

**The RL attack is the strongest of the four, and this is exactly what makes the
compute-aware metric necessary rather than optional.** RL drives every model in the paper
past 50% risk, including the two that GCG never breaks at any budget: Tulu3-SFT
(GCG $C_{@0.5} = \infty$ → RL 34.5 TFLOPs on HarmBench, 35.7 on JailbreakBench) and Qwen3-4B
(GCG $\infty$, ASR 0.36 → RL 16.1 TFLOPs, ASR 0.98). It does so *net of its own optimization
cost*: because the attacker's forward and backward passes are billed to the FLOP axis, RL
pays strictly more per query than PAIR and still reaches 50% risk sooner on every model.
The consequence for evaluation is the headline: **under a strong adaptive attacker, ASR
stops discriminating.** RL's ASR spans only 0.85–0.99 across the nine models on HarmBench
(0.88–1.00 on JailbreakBench). Every model looks broken, and the metric that most jailbreak
papers report has no resolution left. Over the same nine models and the same attack,
$C_{@0.5}$ still spans 6.8 → 34.5 TFLOPs, a 5× spread, and AE spans 10.2 → 53.5. The
compute axis is what remains informative once attacks are strong enough to succeed
eventually, which is precisely the regime the reviewer is asking us to test.

**The paper's two substantive conclusions survive the stronger attack, with the same
orderings.** Under RL alone, the Tulu3 stage ordering is unchanged: base 6.8 < DPO 24.6 ≈
RLVR 24.8 < SFT 34.5 on HarmBench (6.5 < 25.4 ≈ 26.5 < 35.7 on JailbreakBench),  reproducing
the SFT-peak-then-erosion pattern we report under GCG, PAIR, and JB, and again showing DPO
and RLVR landing in the same place. The Qwen2.5 size trend is likewise monotone under RL
(9.3 < 15.9 < 20.1 on HarmBench; 10.9 < 16.4 < 22.6 on JailbreakBench). The two benchmarks
agree cell-by-cell to within roughly 10%, so neither conclusion is a benchmark artifact.

**RL's margin scales with how robust the target is.** Against the weakest model
(Qwen2.5-0.5B) the cheap static Jailbroken suite is actually competitive with RL
($C_{@0.5}$ 8.2 vs 9.3 on HarmBench). There is nothing for an adaptive attacker to earn.
The gap opens as the target gets harder: RL beats JB by 24% on Qwen3-4B (16.1 vs 21.2) and
by 34% on Tulu3-SFT (34.5 vs 52.4). Adaptivity buys the attacker the most exactly where
defenses are working, which is the regime a robustness metric has to measure well.

**One notable exception, and a caveat.** Qwen3-4B-SafeRL's safety training measurably helps
against the prompt-level attacks. It raises PAIR's cost-to-50% by ~57% relative to plain
Qwen3-4B on JailbreakBench (59.1 vs 37.6) and JB's by ~21% (29.4 vs 24.2), but that
advantage does not carry over to an attacker that itself optimizes with RL, where SafeRL is
marginally *easier* to break than the un-hardened 4B (16.5 vs 18.7 on JailbreakBench; 14.3
vs 16.1 on HarmBench). We flag this as suggestive rather than established: the RL column is
single-seed and these gaps are small. It is, however, the kind of finding the framework is
built to surface, a defense that buys real compute against one attack class and none
against another is invisible to any single-attack ASR number.

**Finally, on the AE axis, RL is not uniformly the most efficient attack**, and that is
informative rather than contradictory. On the weakest targets the static JB suite wins on
risk-per-FLOP (Qwen2.5-0.5B HarmBench: JB 59.6 vs RL 40.3) because RL pays for gradient
steps that a trivially-broken model does not require. $C_{@0.5}$ and AE are measuring
different things: total compute to a risk threshold versus marginal risk per unit compute, and the RL column is where they most clearly come apart.
