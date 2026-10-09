# Measured results and evaluation limits

These are saved measurements from Rocinante, reviewed October 9, 2026. [The entry page](README.md) records hardware and engine provenance. Exact numbers and selected per-task pairs are in [the evidence JSON](evidence/benchmarks-2026-10-09.json). Raw requests, outputs, tool traces and private configurations remain local.

## What each speed number means

- **Prefill:** processing input tokens; report actual token count, reuse and first-token latency.
- **Observed decode tok/s:** all generated tokens divided by recorded decode time, including reasoning and tool-call text. This excludes prefill and tool execution.
- **End-to-end tok/s:** generated tokens divided by request wall time, including transport and prompt processing. This still does not measure task correctness.
- **Task wall time:** time to complete and verify a task, including reasoning, tools and follow-up requests. Record failed or incomplete tasks separately.

Do not rank models by mixing these metrics, prompt depths, quantizations or speculative acceptance rates.

## Isolated kernel round before the quality rerun

Opus's October 8 round ran fresh processes twice in mirrored order, totaling eight warm 100K requests and four short requests per configuration. Warm requests allowed 1,536 output tokens and short requests 2,048, at medium effort, temperature zero and seed 123. These are narrower timing probes than the completed agent checks below.

| Configuration | Decode at 100K | Decode at short context | Cold 100K prefill |
|---|---:|---:|---:|
| Patched executable, candidate switches off | 173.3 tok/s | 192.7 tok/s | 23.61 s |
| Five-switch stack | 182.4 tok/s | 205.6 tok/s | 22.55 s |
| Stack plus two drafter settings | 184.8 tok/s | 205.8 tok/s | 22.49 s |

The five-switch stack improved those observed decode rates 5.3% and 6.7%. A separate exactness pass used reproducibility settings: `STRATA_IQ_MT_MIN=1`, adaptive swaps off, PCIe fraction zero and prompt cache zero. Each arm ran two cold 100K and two short requests. The stack's four outputs matched baseline byte for byte, with matched-text decode time reduced 3.2% and 2.8% in the two context categories. The new up-projection kernel alone reduced matched-text decode time 1.6% and 2.1%; its isolated bench passed 32/32 bitwise comparisons plus the parity self-test.

`GR_UP8` loads weights earlier and uses eight lanes per output row while preserving the reduction order. Its first version spilled registers for larger verification windows under a two-block launch bound; removing that bound improved the actual SM120 path. `KV_HOST_DMA` replaces slow mapped-host stores during streamed prefill with DMA and protects partially filled KV blocks. `NO_PCIE_NODES` removes unused graph work only when PCIe share is fixed at zero. The patch also guards the PLE copy's ordering on the device-plan path. See [the as-built diff](evidence/opus-as-built-2026-10-08.diff).

Device-side planning helped the high-hit warm cache but slowed the lower-hit reproducibility arms. The English draft vocabulary and 8K draft attention window reduced per-pass work but also reduced acceptance and changed output. Their net benefit was small, and they were not selected for serving. The serving subset's approximately 3% isolated decode gain was originally an estimate from individual switches, not a separately measured combined arm. The later agent comparison below measures that subset directly, with the output-length caveat.

These results establish some kernel headroom after settings tuning; they do not prove that the space is exhausted. CUDA was the measured backend. The local patch is not validated on HIP, SYCL, other CUDA cards, or general concurrent workloads, and is not a current-upstream-ready contribution.

## Completed 100K-context kernel comparison

All configurations used original Qwen3.8-Flash-Next GSQ-RCO IQ3_S, MTP `--spec 4 --spec-min-p 0.5`, INT8 KV, a 32,768-token GPU KV window, maximum context 131,072, prompt cache 16, tokenizer piece cache, and no message-boundary flag.

| Arm | Engine switches | PCIe fraction |
|---|---|---:|
| Original | Original executable; the five candidate switches off | 0.55 |
| Serving subset | Opus executable: `GR_UP8`, `KV_HOST_DMA`, `DF_BRANCH` on | 0.55 |
| Full candidate | Same Opus executable: those three plus `NO_PCIE_NODES`, `VERIFY_DEVICE_PLAN` | 0 |

Switch names have the `STRATA_` prefix. The full candidate also changes placement settings, so it is not a pure kernel-only comparison.

The protocol was registered at `2026-10-09T05:26:58.187073+00:00` (October 8 locally), before inference. Six fresh processes ran in mirrored order: original, subset, full, full, subset, original. The two halves used alternate variants of ten repair tasks, with reverse ordering in the second half. Repairs used medium effort, temperature zero, seed 123, an approximately 100K-token background prefix and guarded read/write/check tools. Hidden fixtures were unavailable to the model.

**Output allowance was 24,576 tokens, including reasoning, with at most 12 model turns per repair.** A pass required a complete saved solution, all visible and hidden cases passing, and checks after the final successful edit. An independent audit matched saved solutions to tool writes and frozen fixtures.

| Arm | Completed repairs | Hidden cases | Completed source answers | Repair wall time | Repair output tokens | Observed decode tok/s |
|---|---:|---:|---:|---:|---:|---:|
| Original | 20/20 | 164/164 | 2/2 | 252.5 s | 24,820 | 179.9 |
| Serving subset | 20/20 | 164/164 | 2/2 | 244.1 s | 24,008 | 179.9 |
| Full candidate | 20/20 | 164/164 | 2/2 | 237.0 s | 23,419 | 186.9 |

Each arm also passed all 102 visible cases. No output-budget exhaustion, tool-turn exhaustion or engine failure occurred. Repair wall time excludes engine startup and the separate source-facts requests.

The subset used 3.36% less aggregate repair time than original; the full candidate used 6.13% less. Each was faster on 12/20 matched tasks. **Outputs differed:** the full candidate generated 5.64% fewer tokens. These wall-time changes therefore do not isolate kernel speed. Matched fixed-work timing probes are still needed for causal attribution, and more varied tasks are needed for a general quality claim.

The repeated source-facts request used 99,700 input tokens, xhigh effort, temperature zero and seed 123:

| Arm | Output tokens, two runs | Request wall time, two runs |
|---|---|---|
| Original | 3,930; 4,840 | 43.80 s; 49.53 s |
| Serving subset | 4,093; 4,093 | 43.70 s; 43.75 s |
| Full candidate | 3,192; 7,204 | 39.29 s; 58.42 s |

All six finished with `stop` and eight correct typed facts. Repeating one fixed request in fresh processes is not independent sampling of task quality. The full candidate's reasoning-length variation is a reason to preserve completion and budget classifications.

The serving subset was restored after the comparison; **the full candidate was not promoted**. Restoration hashes and a smoke request were checked at the time. These small pure-function repairs with a large background prefix do not establish general repository coding quality, statistical noninferiority, concurrency or long-duration stability.

## Correcting the earlier capped run

An earlier full-candidate run exhausted an 8,192-token output allowance while reasoning and produced no final answer. That was an output-budget failure; completed-answer accuracy was unassessed. Increasing the registered allowance prevented a repeat under the new protocol, but does not explain why that earlier trace reasoned longer.

8,192 was a request cap, not the server's hard output limit. For the evaluated server, usable output room was `131072 - 8 - prompt_tokens`, further limited by the requested allowance. A 150K-token input cannot fit that configured maximum. A larger configured context needs its own capacity, state-correctness and speed evaluation.

## Separate dense-model baseline: Qwen3.8-27B NVFP4 + DFlash

Our September 30 assessment used SGLang 0.5.20, Torch 2.13.0+cu130, Transformers 5.12.1 and FlashInfer 0.6.18. The target was the native Hugging Face NVFP4 checkpoint through FlashInfer CUTLASS, with DFlash speculation. This differs from our native GGUF conversion, which retained many BF16/F32 tensors.

| Workload, warmed trials 2–3 | Requests | Pooled end-to-end tok/s | Median TTFT | Median request wall time |
|---|---:|---:|---:|---:|
| Three short prompts | 6 | 181.4 | 0.155 s | 1.555 s |
| Synthetic approximately 8K retrieval | 2 | 105.1 | 1.127 s | 2.026 s |

Requests used a 512-token maximum, temperature zero, seed 123, thinking on and natural EOS. Prompt reuse was disabled. Short outputs were 239–358 tokens, not sustained multi-thousand-token runs. Four short requests passed narrow checks and two chat requests were ungraded; both retrieval requests passed. Code checking parsed syntax without executing the code.

This demonstrates a useful dense-model runtime/representation candidate, not 100K agent usability or quality parity with Flash-Next. Natural-EOS outputs differ across profiles. TTFT is measured at the first visible streaming chunk and includes transport; it is not a GPU timestamp. Compare both models using the same realistic tasks before choosing a serving replacement.

## Where the remaining work is

Earlier profiling notes suggest a large fixed GPU cost per speculative verification step and small host time, with the GPU busy near its 300 W limit. That is a dated, workload-specific observation, not proof that code optimization is exhausted. Re-measure under controlled accepted-token workload, recording draft acceptance, actual GPU clocks, transfer wait, memory use and cache reuse.

The next useful comparison should separate cold prefill from warm turns at 32K, 64K and 100K; use sufficient output headroom; require final answers and executed task tests; and include a multi-turn cache/state stress check. Record reasoning effort and generated reasoning separately. Treat extra RAM primarily as capacity until measurements show a RAM bandwidth or residency bottleneck.
