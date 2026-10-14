# RTX 3070 local-model and multi-GPU performance data

Measured results from a Windows workstation with four 8 GB RTX 3070 cards. This package is a reproducible starting point for comparing local LLM serving and video-generation workloads on this topology.

## Machine under test

- 4x EVGA RTX 3070 XC3 Ultra, 8192 MiB each (32 GiB aggregate VRAM).
- GPU 0: PCIe Gen3 x16, bus `01:00.0`; used as the compute card in the H3 donor layout.
- GPUs 1-3: PCIe Gen3 x1, buses `04:00.0`, `06:00.0`, and `07:00.0`; used as donor cards.
- AMD Ryzen 5 5500, 6 physical cores / 12 threads.
- Windows, NVIDIA driver 616.92.
- Current hardware notes in `system_hardware.md` report 48 GB / 47.87 GiB system RAM. The older LLM report says 56 GiB, so the RAM value is deliberately called out as a source discrepancy rather than silently reconciled.

The cards are not a single uniform 32 GiB CUDA device. PCIe x1 links constrain tightly coupled transfers, and a workload's placement and transfer pattern matter.

## Headline results

### LLM serving (llama.cpp)

The strongest directly measured result in the supplied report was **66.72 generated tokens/s aggregate** from Qwen3.6-35B-A3B Q4_K_M with four concurrent requests after correcting an unexpectedly low GPU-clock state. The same fixed 1024-token test changed from 5.706 to 38.584 tokens/s after `nvidia-smi -lgc 1500,1900` (6.76x); outputs were byte-identical. The clocks were restored with `nvidia-smi -rgc` after the test.

The serving configuration was `-ngl 999 --split-mode layer --tensor-split 1,1,1,1 -c 16384 --parallel 4 -ctk q8_0 -ctv q8_0 -fa on -b 512 -ub 256`. The measured layer split was 11/10/11/8, with no CPU decoder layers.

| Model / mode | Test shape | Aggregate generated rate | Peak VRAM (GPU0/GPU1/GPU2/GPU3) |
|---|---|---:|---|
| Qwen3.6-35B-A3B Q4_K_M, before clock correction | isolated request | 5.68 tok/s | not recorded in this row |
| Qwen3.6-35B-A3B Q5, before clock correction | isolated request | 5.18 tok/s | not recorded in this row |
| Qwen3.6-35B-A3B Q4_K_M, before clock correction | 4-request batch | 9.40 tok/s | not recorded in this row |
| Qwen3.6-35B-A3B Q5, before clock correction | 4-request batch | 8.61 tok/s | not recorded in this row |
| Qwen3.6-35B-A3B Q4_K_M, before clock correction | sustained queued/rolling, 40 requests | 13.72 tok/s | not recorded in this row |
| Qwen3.6-35B-A3B Q5, before clock correction | sustained queued/rolling, 16 requests | 13.67 tok/s | not recorded in this row |
| Qwen3.6-35B-A3B Q4_K_M, after clock correction | isolated request | 34.94 tok/s | not recorded in this row |
| Qwen3.6-35B-A3B Q4_K_M, after clock correction | 16 requests, concurrency 4 | **66.72 tok/s** | 6362/5156/5694/4824 MiB |
| Qwen3.6-35B-A3B Q5, earlier rolling run | 16 requests | 13.67 tok/s | 7185/6068/6676/5436 MiB |

The Q5 VRAM figures are from the earlier rolling run and are not a like-for-like comparison to the Q4 four-request result. See `data/model_performance.csv` for machine-readable rows and the copied source report for the full test notes.

### H3 video generation and donor-card layout

These rows are end-to-end or sampling measurements from ComfyUI H3 workflows. They are not all the same model, step count, resolution, or warm/cold state, so they must not be treated as one controlled scaling curve.

| Workload | Configuration | Resolution / frames | Sampling | End-to-end |
|---|---|---|---:|---:|
| H3 baseline | original single-GPU run, 20 steps | 608x352 / 124 | not isolated | 6648.834 s |
| H3 Turbo6 | optimized 48 GB run | 1024x576 / 124 at 24 fps | 813 s | 1126.389 s |
| H3 Turbo4 | four-card donor layout | 1280x704 / 124 at 24 fps | 710 s | 863.328 s |
| H3 Turbo4 | GPU0 x16 compute + GPUs1-3 x1 donors, cold | 1280x704 / 124 at 24 fps | 176.88 s average | about 930 s |
| H3 Turbo4 | earlier five-card/all-x1 donor comparison | 1280x704 / 124 at 24 fps | 177.66 s average | warm comparison |
| H3 Toad test | four-card workflow | 832x480 / 243 at 24 fps | about 670 s | 1053.176 s |

The four-card x16 run was about 0.4% faster in sampling than the earlier five-card/all-x1 comparison, effectively tied within this measurement. It does not establish that adding cards scales compute linearly. The H3 Turbo6 result is about 5.9x faster than the old baseline, but it changes resolution, steps, model/optimization, memory placement, and runtime conditions at the same time.

## Files

- `data/model_performance.csv` — LLM measurements.
- `data/render_performance.csv` — H3/video measurements.
- `data/gpu_topology.csv` — observed card and PCIe topology.
- `data/clock_experiment.csv` — fixed-test clock-state comparison.
- `sources/` — copied reports that explain methodology and limitations.
- `raw/` — selected JSON, GPU-sample, and success-log artifacts from the H3 runs.
- `SOURCES.md` — original paths and what each source supports.
- `checksums.sha256` — SHA-256 hashes for the package files.

## Interpretation and limits

The values here are measurements from this machine, software stack, and test harness. They are not universal RTX 3070 performance claims. Throughput numbers use different request lengths and concurrency levels unless the row says otherwise. H3 rows also vary in resolution, steps, and warm/cold state. Use the raw artifacts and source reports before making a new comparison.

The package intentionally preserves a RAM discrepancy found in the source material: `MODEL_PERFORMANCE_REPORT.md` says 56 GiB while `system_hardware.md` records the current machine as 48 GB / 47.87 GiB. Historical and current facts are not merged without evidence.

No benchmark result in this folder should be interpreted as a promise of performance, a claim of unified VRAM, or a proof that PCIe x1 donor cards improve every workload.
