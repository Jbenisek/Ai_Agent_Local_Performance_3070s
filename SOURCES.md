# Sources

This package was assembled from files already present in `C:\Projects\video_render`. Paths below are the original evidence locations at packaging time.

| Copied file | Original path | Supports |
|---|---|---|
| `sources/MODEL_PERFORMANCE_REPORT.md` | `C:\Projects\video_render\MODEL_PERFORMANCE_REPORT.md` | llama.cpp Qwen3.6 measurements, clock correction, serving flags, VRAM, and Qwen3.8 comparison notes |
| `sources/system_hardware.md` | `C:\Projects\video_render\system_hardware.md` | current machine, PCIe topology, driver, RAM, and H3 placement notes |
| `sources/optimization_report.md` | `C:\Projects\video_render\_local_video_pipeline\h3_setup\optimization_report.md` | H3 baseline, Turbo6/Turbo4, four-GPU x16, Toad timings, and interpretation limits |

Selected raw artifacts are copied without rewriting. Their filenames identify the corresponding run; the report files remain the authoritative narrative for units and caveats.

The source reports contain historical notes and a RAM discrepancy (56 GiB in the LLM report versus 48 GB / 47.87 GiB in the current hardware report). The README keeps that discrepancy visible.
