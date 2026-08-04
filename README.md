Вот обновленная версия `README.md`, интегрирующая результаты независимого аудита (эксперименты John6666) и уточняющая архитектурные ограничения проекта. Текст составлен на английском языке для соответствия оригинальному языку репозитория, но с учетом всех смысловых нюансов запроса.

---

# Qxern — latent semantic channel + symbolic AST sidecar for inter-LLM code communication

> **One honest sentence:** Continuous latents are a fast *semantic* channel between two LLMs, but they are structurally unable to carry *exact symbols*; independent probing confirms that identifier signals exist in the latent space but are too weak to overcome receiver priors without a deterministic AST sidecar. Every claim below is locked behind guard gates, paired bootstrap CIs, and external replication attempts.

**English (this file) · [Русская версия → README.ru.md](README.ru.md)**

🤗 **Weights & int4 deployment bundle:** [huggingface.co/aximi/qxern-v6-deploy-int4](https://huggingface.co/aximi/qxern-v6-deploy-int4)

![Quality–latency–payload trade-off](assets/qxern_v6_tradeoff.png)

---

## 👋 About the author

I'm **15 years old, from Russia**.

- Everything up to **v5** was built in **one week on free Kaggle GPUs**.
- **v5 → v6** ran on a **rented RTX 5090 32 GB** (vast.ai).
- Right now I can't afford GPU rent, so the project may not continue — everything measured so far is published here and reproducible from the notebook.
- Tooling honesty: first drafts were sketched with **Qwen-3.7max**, and the code was polished with **Claude**. The architecture decisions, the experiment design and every measurement in this repo are mine.

**Why "Qxern"?** **Q** — Qwen was the first model that helped (and Q reads quantum-ish, science-y) · **x** — like *axim* · **ern** — my name is Ernest.

## Why? What for?

**The question that started this project: why should AI agents talk to each other in a human language that isn't their native one?**

Models think in continuous vectors; text is a lossy, token-by-token serialization imposed on them for *our* convenience. When two models talk in text, the sender pays generation latency, and rich internal semantics get flattened into a string the receiver must re-encode. If models could exchange *meaning* directly, multi-model pipelines — agent swarms, code-review bots, model cascades — would get faster and cheaper.

But measurements (both internal and external) show exactly where the dream breaks: a continuous latent channel trained by distillation carries **behavioral semantics** well and **discrete symbols** not at all. Qxern v6 is the pragmatic answer: **meaning through latents, symbols through a deterministic symbolic channel, text only as a rare fallback.** Don't fight the geometry — route around it.

## What is this?

Qxern is an experiment in **latent inter-model communication**: model A (`Qwen2.5-Coder-1.5B`) encodes a code snippet into **32 latent tokens** via a trained Q-Former adapter; a frozen decoder (`Qwen3.5-0.8B`) answers questions about the code using only that latent packet — no code text is transmitted.

The project went through an honest arc:

- **v4 claim:** “latents are better and faster than text” — **did not survive** a properly strengthened text relay baseline.
- **v5 finding:** pure latents win on speed (**6.0×**) and binary behavioral facts (`returns` 0.90), but **completely lose exact symbols** (function names 0.00 vs 0.70 for relay, param count 0.13 vs 0.63). Channel width is irrelevant: the 8→64-token capacity curve is flat.
- **v6 (this repo):** don't force a continuous channel to be a lossless codec. The packet becomes `[semantic latents] + [deterministic AST sidecar]` (signature / arity / behavioral flags / literals — parsed with `ast`, no LLM, microseconds, ~30 tokens). An **adaptive router** picks latent-only / latent+sidecar / relay per question type.
- **v6 Independent Audit (John6666):** Confirmed that the sidecar does the heavy lifting for exact facts. Crucially, refined the understanding of symbol loss: identifier information *is* present in the latent packet and weakly affects the receiver's probability distribution, but it is too uneven to overcome lexical priors. Pure-latent open-vocabulary identifier recovery remains at 0/96. This validates the hybrid approach as an architectural necessity, not just a performance hack.

## Key results (measured, from the raw run data)

All numbers are computed from [`results/qxern_v6_results.json`](results/qxern_v6_results.json) (n=30 held-out functions for AST facts, n=50 for SemSim/latency; CodeSearchNet, repo-level split). Accuracy/SemSim are means, latency is p50, payload is the median prefix length.

| System | func name | param count | returns | SemSim | p50 ms | payload (prefix tok) |
|---|---|---|---|---|---|---|
| baseline (no code) | 0.00 | 0.00 | 0.10 | 0.12 | 678 | 51 |
| relay (strengthened text) | 0.70 | 0.63 | 0.83 | 0.43 | 1222 | 122 |
| qxern_v5 (pure latents) | 0.00 | 0.13 | 0.90 | 0.24 | **202** | 92 |
| **hybrid** (latents + sidecar, no retrain) | **0.87** | **0.93** | **0.90** | **0.44** | 511 | 122 |
| qxern_struct (retrained, no sidecar) | 0.20 | 0.80 | 0.80 | 0.30 | 223 | 92 |
| **hybrid_struct** (both) | **0.87** | 0.73 | **0.90** | 0.42 | 420 | 122 |
| oracle (code as text) | 1.00 | 0.90 | 0.93 | 0.50 | 736 | 170 |

**Guard gates: PASSED for both hybrids.** The self-imposed “don't make anything worse” gates (`returns ≥ 0.88`, SemSim drop vs pure latents ≤ 0.01, `names ≥ 0.65`, `params ≥ 0.60`, speedup vs relay ≥ 2×) pass for `hybrid` and `hybrid_struct`, and **fail for `qxern_struct`** (names 0.20 < 0.65). That failure is the point of the ablation: **retraining alone does not restore exact symbols — the sidecar does.**

**Paired bootstrap vs relay** (10,000 resamples, 95% CI):

- zero-shot **hybrid**: param count **+0.30 [+0.10, +0.50] — significant**; names +0.17 [−0.03, +0.37] and returns +0.07 [0.00, +0.17] — positive but not significant at n=30; SemSim indistinguishable — all at **2.39× lower p50 latency**.
- **hybrid_struct**: all accuracy diffs statistically indistinguishable from relay, at **2.91× lower p50 latency**.

### 🔬 Independent Replication & Refined Geometry (John6666 Audit)

An independent follow-up using the released v6 adapter (12 Python functions, seed 42, Colab T4) validated and refined the core claims:

1.  **Exact recovery is sidecar-driven:** Replacing the latent while keeping the correct sidecar preserved 11-12/12 function names. Replacing the sidecar while keeping the correct latent dropped names to 0/12. Pure-latent name generation remained at 0/96.
2.  **Latents carry coarse sample-specific semantics:** Correct latent vs. wrong-example latent showed a significant semantic similarity gap (+0.261, 95% CI [+0.168, +0.382]). The packet is not a generic soft prompt.
3.  **Identifier signal exists but is weak:** Cosine similarity between original and renamed packets is high (0.987), yet candidate-free likelihood tests show the receiver's distribution *does* shift positively when the encoded identifier changes (+0.0519 self-score lift, p<0.002). However, raw top-1 accuracy is chance-like (13.5% vs 12.5%), and open-vocab generation fails. **Conclusion:** Identifier information occupies a small direction in latent space that is decodable but insufficient to overcome pretrained lexical priors.
4.  **Fine-grained behavior edits are directional but unreliable:** Packet swaps for localized behavior changes (> vs >=, asc vs desc) produce positive mean shifts in the correct direction, but no pair passed the robust discrimination gate. Free-form behavior recovery was 0/24.
5.  **Cross-receiver transfer requires more than linear bridges:** A lightweight linear bridge from 0.8B to 2B failed to preserve the known semantic control. Packets are currently receiver-specific; a Shared Latent Hub will need validated receiver-specific adapters or alignment mechanisms, not just dimensional matching.

> **Revised Geometric Interpretation:** Identifiers do not "physically never reach the decoder." They reach it as a weak, high-dimensional signal that the current readout cannot reliably amplify above the noise floor of lexical priors. The sidecar is architecturally necessary because continuous interpolation fundamentally struggles with discrete symbol recovery at this scale.

### Benchmark plots

| Plot | Image |
|---|---|
| v6 AST facts per system (main ablation) | ![v6 AST accuracy](assets/qxern_v6_ast_accuracy.png) |
| v6 first zero-shot hybrid check | ![v6 hybrid check](assets/qxern_v6_ast_accuracy_hybrid.png) |
| v6 SemSim with 95% CIs | ![v6 SemSim](assets/qxern_v6_semsim.png) |
| v5 main benchmark analysis | ![v5 analysis](assets/qxern_v5_analysis.png) |
| v5 channel capacity 8→64 tokens (flat!) | ![v5 capacity](assets/qxern_v5_capacity_curve.png) |

## Honest limitations (read before quoting numbers)

- **The names gain is not statistically significant.** hybrid beats relay on names 0.87 vs 0.70, but the 95% CI (+0.17 [−0.03, +0.37]) crosses zero at n=30. Don't quote “significantly better names”; the *significant* win is param count (+0.30) at 2.4× lower latency, with nothing degraded.
- **Retraining is not a free win:** `hybrid_struct` is faster (420 vs 511 ms) but drops param count to 0.73 vs 0.93 for the zero-shot hybrid. Architecture does the heavy lifting; training buys speed, not accuracy.
- **Identifier/behavior signals are sub-threshold:** While externally probed and shown to affect receiver likelihoods, neither supports reliable free-form generation or fine-grained state discrimination in the current release.
- **Cross-receiver compatibility is unproven:** Linear bridges fail semantic controls. Heterogeneous deployment requires receiver-native adapters or learned alignment beyond simple projection.
- **Multi-turn stability is unevaluated:** Fresh-state discrimination gates have not been passed reliably; drift measurements are therefore not yet interpretable.
- SemSim is computed against **teacher answers**, so relay (same model family as teacher) has a built-in advantage; SemSim also favors verbose answers.
- n=30 eval functions (50 for SemSim/latency), one GPU, one seed, small models (1.5B + 0.8B). Directional evidence, not a paper-grade eval.
- `transformers` was installed from `main` during the week; pin `QXERN_TRANSFORMERS_REF` to a commit SHA to reproduce exactly.
- Latency numbers are single-GPU p50 with warm-up, CUDA sync and randomized order — but still hardware-specific (RTX 5090).

## How is this different from C2C and LatentMAS?

I designed and built Qxern **before I found out that anyone else was working on latent inter-model communication**. Then it turned out the field exists — which is honestly great news: the direction isn't crazy.

| | **Qxern (this repo)** | **[C2C — Cache-to-Cache](https://arxiv.org/abs/2510.03215)** (ICLR'26) | **[LatentMAS](https://arxiv.org/abs/2511.20639)** (ICML'26) |
|---|---|---|---|
| Medium | explicit compact packet: 32 latent tokens from a trained Q-Former + deterministic AST sidecar | KV-cache projection & fusion via a trained fuser network | last-layer hidden states as “latent thoughts” + shared latent working memory |
| Training | small adapter distilled from a teacher | fuser network trained | training-free |
| Domain | code understanding; the exact-symbol problem | general multi-LLM QA | multi-agent reasoning |
| Core question | *why should AI agents talk to each other in a human language that isn't their native one?* | can LLMs communicate through KV-caches? | can agents collaborate without text? |
| Exact symbols | explicitly measured (names 0.00), restored by a symbolic AST channel; refined by external audit as "weak but present" | not the focus | not the focus |

Code: [thu-nics/C2C](https://github.com/thu-nics/C2C) · [Gen-Verse/LatentMAS](https://github.com/Gen-Verse/LatentMAS)

**Why Qxern is not a copy of either:**

- **Different medium.** Qxern transmits an explicit, compact, inspectable packet (32 latents + ~30 sidecar tokens) between two *different* small models. C2C fuses KV-caches; LatentMAS shares raw hidden states in working memory. Three different points in the design space.
- **Different question.** C2C and LatentMAS argue latents are *enough* (and show impressive gains). Qxern measures **where they are not**: the flat 8→64 capacity curve, `names = 0.00`, and the refined geometric explanation of symbol loss. The conclusion is a *hybrid*: continuous channel for meaning, symbolic channel for exactness.
- **The negative results are the contribution.** Neither paper has an AST sidecar, guard gates, a quality↔latency↔payload trade-off curve, or the externally-validated channel-geometry explanation of symbol loss. Convergent evolution, independent execution.

## Adaptive router

Mode is chosen per question type: semantics/behavior → pure latents (p50 202 ms); exact symbols → latents + sidecar (p50 511 ms); verbatim reproduction → rare relay fallback (p50 1222 ms). Exactness where needed, speed everywhere else. See [`sidecar_router.py` on HF](https://huggingface.co/aximi/qxern-v6-deploy-int4/blob/main/sidecar_router.py) (pure `ast` + routing, runs on CPU, no LLM needed).

## Repository layout

```
qxern/
├── README.md                        # English (this file)
├── README.ru.md                     # Russian version
├── Qxern_v6_hybrid_sidecar.ipynb    # the whole pipeline: cells 1–10 = v5, 11–17 = v6
├── assets/                          # benchmark plots (linked above)
└── results/
    ├── qxern_v6_results.json        # all raw benchmark numbers (per-example)
    └── teacher_answers.json         # cached teacher generations
```

> ⚖️ **Weights are not in git.** All trained adapters (8/16/32/64-tok, struct) and the int4-quantized models live on Hugging Face: [aximi/qxern-v6-deploy-int4](https://huggingface.co/aximi/qxern-v6-deploy-int4).

## Reproducing

1. Rent a GPU (tested: RTX 5090 32 GB on vast.ai; everything before v5 ran on free Kaggle T4/P100).
2. Open `Qxern_v6_hybrid_sidecar.ipynb`. Set `QXERN_DIR` (default `/workspace/Qxern`) and pin `QXERN_TRANSFORMERS_REF`.
3. Run cells 1–10 (v5 pipeline: teacher distillation → adapter training → v5 benchmark). Checkpoints are cached, re-runs are cheap.
4. Run cells 11–17 (v6: sidecar → zero-shot hybrid → contrastive probing → struct retrain ablation → main v6 benchmark with guard gates → router).

Optional flags: `RUN_CAPACITY_CURVE` (reproduce the flat 8–64 curve), `RUN_LLM_JUDGE`, `RUN_STRUCT_RETRAIN`.

Or skip training entirely and grab the trained weights:

```python
from huggingface_hub import snapshot_download
path = snapshot_download("aximi/qxern-v6-deploy-int4")
```

## What NOT to do (lessons locked in)

- Don't grow latent width — the capacity curve is flat.
- Don't retrain end-to-end from scratch — you'll lose the working `returns` behavior.
- Don't push the whole code through the sidecar — that's just relay in disguise.
- Don't average metrics into one number — better names must not mask degraded behavior facts.
- Don't claim “lossless quality” without measuring latency and payload size.
- **Don't assume latent identifiers are usable without explicit readout training** — the signal exists but is buried under lexical priors.
- **Don't assume cross-receiver compatibility via linear projection** — functional validation with semantic controls is mandatory.

## 🔮 Future & Shared Latent Hub Roadmap

Ordered next steps — all of them need GPU money, so the project is frozen for now. Based on the v6 audit, the path toward a reusable hub protocol requires:

1.  **Fresh-state discrimination gates:** Before evaluating multi-turn drift, establish that receivers can reliably distinguish current states from stale/neighboring ones via counterfactual controls.
2.  **Receiver-specific capability contracts:** Treat the hub as a versioned protocol. For each field (identifier, behavior, literal), document: encoded? used by this receiver? reliably recoverable? fallback route?
3.  **Scale the eval:** n ≥ 300, several seeds — the names gain (+0.17) needs a bigger n to reach significance; then bigger models and languages beyond Python.
4.  **Fusion adapter** (copy/pointer mechanism) trained with the semantic path frozen — to see if the decoder can use the sidecar even better than zero-shot.
5.  **VQ / discrete bottleneck** — a “pure” latent solution for symbols: continuous latents interpolate, symbols don't. The audit shows weak continuous signals exist, so discrete bottlenecks are not logically mandatory but remain promising.
6.  **Confidence gate** on decoder entropy for automatic relay fallback (the router is currently question-type based).
7.  **Atomic state envelopes:** Bundle packet + sidecar + schema version + generation ID into a single lifecycle-managed object to enable staleness detection and synchronization.

If you have spare GPU hours and find this interesting — open an issue.

## License

Apache-2.0.
