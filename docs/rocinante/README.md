# Rocinante: local inference research handoff

Updated October 9, 2026. This is the entry point for reviewing our RTX PRO 5000 experiments and proposing the next useful optimization.

Our priority is **completed, correct agent tasks at 32K–100K context**, including prompt processing, reasoning, tool calls and follow-up turns. A short decode benchmark is a useful diagnostic, but does not establish that experience.

## Read these first

1. [Measured results and evaluation limits](MEASUREMENTS-2026-10-09.md): the completed 100K-context kernel check, its output-budget correction, and our separate dense-model baseline.
2. [Research candidates and questions](RESEARCH-2026-10-09.md): Swift IQ3_S, a second GPU as an expert tier, GLM, Radiance and Zyphra expert coupling.
3. [Selected benchmark evidence](evidence/benchmarks-2026-10-09.json): aggregate measurements, matched repair pairs, protocol settings and source hashes.
4. [External source revisions](evidence/research-sources-2026-10-09.json): pinned source URLs and hashes for the inspected GLM and Radiance code.
5. [Opus as-built kernel patch](evidence/opus-as-built-2026-10-08.diff): exact source changes against the evaluation base, supplied as a review artifact.

## Machine and serving snapshot

| Component | Recorded configuration |
|---|---|
| CPU | AMD EPYC 7452, 32 cores / 64 threads |
| System memory | 128 GB, eight 16 GB ECC RDIMMs, eight channels, 2667 MT/s |
| GPU | NVIDIA RTX PRO 5000 Blackwell, 48 GB, SM120, 300 W limit |
| Additional GPU available | RTX 4070 Ti Super, 16 GB; a proposed second-device experiment, not part of these measurements |
| Current model in the saved serving snapshot | Original Qwen3.8-Flash-Next GSQ-RCO IQ3_S |
| Engine base used in our evaluation | Strata 0.1.40.3, commit `d5ea7133741e67743c0e886bb426c0ce8d69cf6c` |
| Speculation | MTP, `--spec 4 --spec-min-p 0.5` |
| Memory settings | INT8 KV, 32,768-token GPU KV window, 131,072 maximum total context |
| Host/cache settings | 31 CPU workers, PCIe fraction 0.55, automatic expert-cache sizing, prompt cache 16 |
| Other serving settings | Tokenizer piece cache enabled; message-boundary flag disabled |
| Local engine switches | `STRATA_GR_UP8=1`, `STRATA_KV_HOST_DMA=1`, `STRATA_DF_BRANCH=1` |

The saved restoration evidence identifies executable `strata-opus-kernel-20261008` with SHA256 `987a2492211830ccb31e17b2962bca732023add32ebee0dcb59872dffda69824`. This is a dated snapshot, not a live health check. The known `DF_BRANCH` stall risk was accepted for that serving configuration; the bounded comparison observed no engine failures, which does not prove long-duration stability.

**Source-version distinction:** this GitHub fork was created from upstream main at `fb58e0dbc8399662c0e47c76578c6e878b14f6cf`. The archived patch applies to evaluation base `d5ea713`, not necessarily to that newer main. It is included as an evidence file, not applied to the fork's engine source. Publishing this handoff did not upgrade Rocinante. Review the patch with the pinned evaluation base; reproducing the executable also requires the original build settings and dependencies.

## Request for GPT-6 Pro or another reviewer

Please read both documents and the evidence before proposing changes. Rank the most promising next experiments for **this one model and this SM120 card**, then for the asymmetric 48+16 GB pair. For each proposal, give:

- The suspected bottleneck and evidence needed to confirm it.
- The exact engine revision, relevant source functions, and a minimal opt-in implementation.
- The expected benefit, clearly marked as a prediction, and conditions that would falsify it.
- A matched timing check plus completed-answer, multi-turn cache/state, and task-quality checks.
- Whether it helps cold prefill, warm turns, accepted decode, total task time, capacity or concurrency.

In particular, assess whether expert coupling can improve placement or predictive prefetching here, whether existing fixed GPU/host work offers more headroom than another settings sweep, and whether Swift's shorter reasoning improves task time without losing useful correctness. Keep the actual router authoritative in any prefetch experiment.

Useful upstream context: [engine details](../DETAILS.md), [multi-GPU placement](../MULTI_GPU.md), [community benchmark protocol](../COMMUNITY_BENCHMARKS.md), and [open test requests](../TEST_REQUESTS.md). Their current contents describe the fork's upstream snapshot, which is newer than our evaluation base.
