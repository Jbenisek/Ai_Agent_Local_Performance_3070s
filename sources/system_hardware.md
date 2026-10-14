# System Hardware and Setup

Scanned 2026-09-08 (read-only PowerShell and NVIDIA inspection, owner approved).

## Summary
AMD Ryzen 5 / B550 render system with four currently visible EVGA RTX 3070 XC3 Ultra GPUs and 48 GB RAM. GPU 0 is installed in the CPU-connected PCIe Gen3 x16 slot; GPUs 1-3 remain on Gen3 x1 mining risers. A fifth matching RTX 3070 is currently spare. H3 uses GPU 0 for sampling, CLIP and VAE while the x1 cards store model weights.

## Setup additions recorded 2026-09-09
- Owner upgraded RAM to 48 GB. Runtime verification reports 47.87 GiB usable total, with 42.71 GiB available before the restarted H3 test. Module sizes, slot layout, and configured speed have not been rescanned; the September 8 DIMM details below are historical.
- LM Studio 0.4.23+1 installed; Qwen3.8-27B GGUF downloaded and listed locally (17.74 GB).
- ComfyUI 0.34.0 portable installed under `_local_video_pipeline/h3_setup/ComfyUI_windows_portable/`, with embedded Python 3.12.10 and PyTorch 2.13.0+cu126. This does not replace the system Python command or install a system CUDA Toolkit.
- Before the hardware move, PyTorch detected all five RTX 3070s. After the move/reboot it detects four cards; the removed fifth card remains available for the dev workstation.
- H3 runtime configuration and validation are documented in [_local_video_pipeline/h3_setup/README.md](_local_video_pipeline/h3_setup/README.md).
- Storage free-space figures below remain the historical scan values, before model downloads.

## CPU
- AMD Ryzen 5 5500 @ 3.60 GHz
- 6 cores / 12 threads

## Memory
- Current: 48 GB installed, 47.87 GiB reported by Windows/runtime after the owner upgrade.
- Historical September 8 scan: 16 GB total (2 x 8 GB DDR4).
- Modules: 2 x `8GX-D4-3000-MR`
- Current configured speed: 2133 MT/s (module-rated speed not verified by this scan)

## Motherboard / BIOS
- Gigabyte B550 EAGLE WIFI6
- BIOS: American Megatrends International, LLC. F3
- BIOS release date: 2025-10-29

## GPUs
- Four currently visible EVGA GeForce RTX 3070 XC3 Ultra cards, 8192 MiB each.
- Installed-card board part: `699-1G142-0010-TS3`; card size 285.37 mm long and 2.2 slots thick.
- Driver: NVIDIA Studio 616.92 (Windows package 32.0.16.1692; signed driver date 2026-09-03), WDDM mode, verified active on all four cards.
- Model memory is distributed for H3 weight storage; the cards do not become one ordinary 32 GB CUDA device.

| GPU | UUID prefix | Bus ID | Current link | H3 role |
|---|---|---|---|---|
| 0 | `56d68c13` | `00000000:01:00.0` | PCIe Gen3 x16 | sampling, CLIP, VAE; 0.5 GB H3 weights |
| 1 | `0aa2fddf` | `00000000:04:00.0` | PCIe Gen3 x1 | 6.5 GB H3 weights |
| 2 | `75e95c88` | `00000000:06:00.0` | PCIe Gen3 x1 | 6.5 GB H3 weights |
| 3 | `38737a5b` | `00000000:07:00.0` | PCIe Gen3 x1 | remaining H3 weights |

The former GPU UUID prefix `cc8d63cd` is absent and is the spare fifth card.

### Verified x16 remap result

- The corrected four-GPU mapping is `cuda:0,0.5gb;cuda:1,6.5gb;cuda:2,6.5gb;cuda:3,*`.
- A native 1280 x 704 / 124-frame / four-step H3 benchmark completed cold in 15m30s. Sampling took 11m47s, essentially tied with the earlier five-card/all-x1 sampling time of 11m50s.
- The fifth 8 GB card did not improve sampling throughput in the tested weight-storage configuration, so it can be reassigned to the RTX 3090 dev workstation.
- V3 retains the production 1280 x 704 / 124-frame first-and-last-frame shot profile.
## Driver, firmware and crash status
- Installed NVIDIA driver: 560.94 / Windows package 32.0.15.6094, dated 2024-08-13. This is obsolete for the current H3/ComfyUI stack; update to a current WHQL NVIDIA Studio Driver before another long render.
- GPU 0 and GPU 2 VBIOS: `94.04.3A.40.5E` (newer EVGA Resizable BAR firmware).
- GPU 1 and GPU 3 VBIOS: `94.04.25.80.6A` (older EVGA firmware).
- Do not flash VBIOS while bus/power stability is unresolved. The newer firmware mainly adds Resizable BAR support, which is not required for x1 H3 weight donors. Reassess after the driver update and a stable benchmark.
- At 2026-09-11 17:16:07, Windows logged nvlddmkm event 153: `BusReset TDR occurred on GPUID:400`, corresponding to GPU 1 at bus `04:00.0`. At the following reboot Windows recorded bugcheck `0x116 VIDEO_TDR_FAILURE`; no restart/shutdown command was issued.
- The live GPU monitor interval is now 60 seconds.
## Storage
- Samsung SSD 970 EVO Plus 500 GB NVMe SSD, Healthy; installed firmware `2B2QEXM7`. Samsung publishes `4B2QEXM7` as the current 970 EVO Plus firmware, but this is not related to the GPU TDR and is deferred until the system is stable and backed up.
- C: NTFS, 500 GB total, ~388 GB free at scan
- No other drives detected

## Network
- Realtek 8851BE Wireless LAN WiFi 6 PCI-E NIC, connected at 433.3 Mbps
- Realtek PCIe GbE Family Controller #2, connected at 1 Gbps

## OS
- Windows 10 Pro 10.0.19045 (64-bit)
- PowerShell 5.1.19041.6456
- Last boot: 2026-09-08

## Installed tooling
| Tool | Status |
|---|---|
| python / python3 | Only Microsoft Store stubs in WindowsApps (not real installs) |
| pip | not found |
| git | not found |
| ffmpeg | not found |
| nvcc / CUDA Toolkit | not found |
| docker | not found |
| wsl | Binary present; distribution / feature status not checked |
| node / npm | not found |
| conda | not found |
| ollama | not found |
| choco | not found |
| winget | v1.29.290 |

## Constraints for AI workloads
- Each RTX 3070 is limited to 8 GB VRAM; model memory cannot be combined across cards for ordinary single-GPU tools.
- Four GPUs are currently dedicated to one H3 render: GPU 0 computes while GPUs 1-3 hold overflow weights. Independent simultaneous jobs would compete for that required model memory.
- The sampling card now has a Gen3 x16 link. Three Gen3 x1 donor links still limit cross-device weight transfers and remain unsuitable for tightly coupled parallel training.
- System RAM has increased from 16 GB to 48 GB. Large model loading, CPU offload, and Windows GPU backing allocations still require monitoring; larger system RAM does not change each card's 8 GB VRAM.
- About 388 GB free disk remains for models, song files, caches, and rendered video.

## Not readable by software (owner to supply)
- Power supply wattage and available PCIe power connectors
- Exact physical GPU/riser layout and cooling arrangement
- Whether additional storage is installed but offline

### Post-update verification (2026-09-11)

- NVIDIA Studio driver **616.92** is active on all four RTX 3070s.
- All four GPUs are visible to nvidia-smi; idle temperatures were 30-35 C and board power 8-15 W.
- GPU 0 advertises PCIe x16 capability. Its current idle state was Gen1 x16, which is normal link power saving; verify Gen3 x16 during the controlled load test.
- GPUs 1-3 enumerate through x1 risers.
- No new nvlddmkm, Display, or WHEA faults were present after the update check.
- The OS had not received a planned reboot after the driver installation. Reboot once before the next stress/render test to establish a clean baseline.
- GPU VBIOS revisions remain unchanged; do not flash firmware until driver/load stability is established.



