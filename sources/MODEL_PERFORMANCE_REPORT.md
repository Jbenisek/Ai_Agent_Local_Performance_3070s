# Local LLM testing on four RTX 3070s: measured results

September 2026 · Windows · one PCIe x16 card and three x1 cards

**Best recorded result: Qwen3.6-35B-A3B Q4_K_M produced 66.72 generated tokens/second across four concurrent requests.** The largest improvement came from correcting GPU clocks: the same fixed request improved from 5.71 to 38.58 decode tokens/second.

## The machine

- Four RTX 3070 GPUs, 8 GB each: 32 GB total VRAM across separate cards.
- GPU 0 connected at x16; GPUs 1–3 connected at x1.
- Ryzen 5 5500, six physical CPU cores.
- Updated configuration: 56 GiB RAM at 2133 MT/s.
- Samsung 970 EVO Plus 500 GB system NVMe; SK hynix BC711 256 GB model NVMe; two Samsung 850 EVO 250 GB SATA SSDs.
- September 27 clock tests: NVIDIA driver 616.92; llama.cpp build 11146, commit 7fe450e19.

## Qwen3.6-35B-A3B: Q4 and Q5

Both quantizations ran across all four GPUs using layer splitting. Thinking stayed at the model default and appeared in the responses. MTP/speculative decoding was off.

| Test | Q4_K_M | Q5_K_M |
|---|---:|---:|
| Before clock correction: isolated decode | 5.68 tok/s | 5.18 tok/s |
| Before correction: one four-request batch, aggregate | 9.40 tok/s | 8.61 tok/s |
| Before correction: sustained queued/rolling work, aggregate | 13.72 tok/s, 40 requests | 13.67 tok/s, 16 requests |
| After correction: isolated decode | 34.94 tok/s | No corresponding benchmark recorded |
| After correction: 16 requests, concurrency 4, aggregate | **66.72 tok/s** | No corresponding benchmark recorded |

Aggregate speed means **total generated tokens divided by elapsed wall time**, including the final requests finishing. Isolated decode uses the server's generation timer. Generated tokens include thinking tokens.

The fast Q4 run generated **10,045 tokens in 150.55 seconds**. All 16 requests finished with `stop`. The earlier Q5 rolling run generated **11,932 tokens in 872.56 seconds**: 15 `stop`, one `length`.

### The clock finding

During the slow Q4 request, average GPU clocks were approximately **219, 210, 211 and 338 MHz**. Applying a **1500–1900 MHz GPU clock range** changed the fixed 1,024-token test from **5.706 to 38.584 tok/s: 6.76 times faster**. Its answer and reasoning text were byte-identical between those two tests.

The commands used, from an elevated terminal, were:

```powershell
nvidia-smi -lgc 1500,1900
# Restore default clock control:
nvidia-smi -rgc
```

No automatic startup task was installed for this setting. Check clocks again after reboot. The same x1 connections were present during the fast result.

### Settings and memory

```text
-ngl 999 --split-mode layer --tensor-split 1,1,1,1
-c 16384 --parallel 4 -ctk q8_0 -ctv q8_0
-fa on -b 512 -ub 256
```

That context setting gave 4,096 tokens per slot. Logs placed the 40 ordinary decoder layers across GPUs as **11 / 10 / 11 / 8**, with no CPU decoder layers. Input embeddings retained a CPU-mapped buffer; the output and unused next-token-prediction layer were assigned to GPU 3.

| Peak VRAM in the recorded four-request serving tests | GPU 0 | GPU 1 | GPU 2 | GPU 3 |
|---|---:|---:|---:|---:|
| Q4 | 6,362 MiB | 5,156 MiB | 5,694 MiB | 4,824 MiB |
| Q5 | 7,185 MiB | 6,068 MiB | 6,676 MiB | 5,436 MiB |

## Qwen3.8-27B: Hugging Face and llama.cpp

These warm tests used four small lab jobs per batch: select a probe, run it, and interpret the result. Loading and warm-up were excluded. Thinking was **off**.

| Setup | Four-job batch, run 1 | Run 2 |
|---|---:|---:|
| Hugging Face NF4, distributed computation | 83.51 seconds | 80.85 seconds |
| Hugging Face NF4, custom donor staging | 1,764.36 seconds | 1,662.30 seconds |
| llama.cpp Q3_K_XL, three GPUs, MTP draft length 2 | 25.20 seconds | 21.29 seconds |

The Hugging Face pair used the same checkpoint, NF4 double quantization, BF16 computation and greedy decoding. The checkpoint loaded once. Distributed mode computed layers on their storage GPUs. Donor mode kept 22 decoder layers on GPU 0 and staged 42 through a reusable GPU 0 buffer.

Each donor batch transferred **2,100 layers, totaling 412.23 GB**. The second batch included **331.99 seconds of deliberate pacing** and **1,327.98 seconds of generation, transfers and associated overhead**. Transfer time was not timed separately. These are measurements of that specific staging implementation. The Q3 run changed both quantization and engine and enabled MTP.

All three setups completed four valid interpreted jobs and exposed **six of eight planted defects per batch**. A later adaptive donor session exposed **all eight cumulatively over four batches**, saved 15 reference-pair experiments, then stopped with GPU 0 out-of-memory after roughly 90 minutes.

## Other tests

- **Qwen3-8B Q4_K_M:** completed a two-candidate generation/review/database test using llama.cpp b11118. No comparable throughput figure was recorded in the cited summary.
- **H3 video donor workflow:** the existing four-card setup used GPU 0/x16 for computation and the x1 cards as weight donors. It rendered 124 frames at 1280×704, with approximately 707 seconds sampling and 930 seconds cold end-to-end. This was a video workload, measured in rendering time.

## Practical findings

1. **Watch actual GPU clocks during generation.** Loaded weights and low utilization did not reveal the clock problem by themselves.
2. **Measure sustained concurrency.** Q5's single four-request batch gave 8.61 aggregate tok/s; rolling 16 requests gave 13.67.
3. **Keep load, transfer, pacing and inference times separate.** A warm model can still transfer weights repeatedly during generation.
4. **Record thinking and finish reason.** At 512 tokens, two of four thinking requests exhausted the budget without final answers. At 2,048, all four completed in the amended serving test.
5. **Measure completed work as well as tokens.** The later research supervisor stopped after about 23 minutes when all three branches failed; the server stayed loaded. The following ten-hour window contained no new experiments.

## Saved evidence

- Q4/Q5 serving and concurrency: `orders/02_thing1/REPORT.md`
- Clock measurements and identical-output comparison: `orders/02_thing1/diag/DIAG_REPORT.md`
- Fast Q4 run: `orders/02_thing1/diag/G_Q4_fixed_20260927_191212/result.json`
- Warm 27B comparisons: `logs/removed-model-testing-2026-09-27/warm-comparison/REPORT.md`, `warm-results.json`, `production-results.json`
- Hardware: `runtime/storage-setup/hardware.json`, `orders/02_thing1/diag/A1_gpu_query.csv`
- H3 render record: `GROK_FABLE_HANDOFF_2026-09-27.md`, H3 four-card evidence section
