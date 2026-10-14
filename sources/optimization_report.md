# H3 speed work - 2026-09-09

## Original render

The saved ComfyUI history records 6,648.834 seconds (110 minutes 49 seconds) for the user's 608 x 352, 124-frame clip. At 24 FPS, that is approximately 5.17 seconds of video. The workflow used 20 sampling steps and Spectrum forecasting/replay on one GPU with DynamicVRAM/disk offloading.

The log shows full model steps initially taking about 70-100 seconds, a 2,201-second stall, and subsequent full steps taking roughly 636-1,057 seconds. This is consistent with severe memory/offload stalls, but the historical log alone does not prove their exact cause. The original log and API history were preserved in `single_gpu_before_optimization.log` and `history_before_multigpu.json`.

## Implemented changes

- Installed [ComfyUI-MultiGPU 2.6.4](https://github.com/pollockjj/ComfyUI-MultiGPU), pinned in `source_versions.txt`.
- Configured DisTorch2 to store the H3 model across all five GPUs: `cuda:1,3gb;cuda:2,6gb;cuda:3,6gb;cuda:0,1gb;cuda:4,*`. GPU 1 performs sampling; GPU 0 handles the small text encoder and VAE. Other GPU allocations serve as weight storage. This is model-memory distribution, not five-way simultaneous tensor parallelism.
- Disabled DynamicVRAM and asynchronous offload for this fixed allocation profile. Retained PyTorch attention because Comfy Kitchen's attention kernel failed against the installed driver in the initial setup.
- Patched the installed MultiGPU extension's `p2p_registry.py` to use `torch.cuda.can_device_access_peer` when available. Its upstream fallback hardcodes Linux's `libcudart.so`, which is unsuitable on Windows. The source archive remains unmodified.
- Added [H3 Turbo v4](https://huggingface.co/larryvrh/MiniMax-H3-Turbo-Lora) and [its ComfyUI nodes](https://github.com/Larryvrh/ComfyUI-MiniMax-H3-Turbo). The 779,849,816-byte adapter passed SHA256 verification: `5f3a626cd72c93a8b9318d6760c510bc5092d2ab13aaba1f932c5bab07a416d3`.
- Created six-step Turbo workflows with strength 1.0, low_vram off, Simple scheduling, the Turbo sampler, and Spectrum disabled. Also saved 20-step non-Turbo multi-GPU variants, with Spectrum replay disabled.

Turbo changes the generation trajectory and may alter detail or motion. The author's suggested range is 4-8 steps; six was selected as a speed/quality starting point. This is not an assertion that the output is identical to the original 20-step video.

## Validation

The five-GPU diagnostic successfully rendered a 256 x 256, five-frame, one-step MP4. Its sampling step took 48.22 seconds, compared with 85.23 seconds in the previous single-GPU diagnostic. Total time was 215.76 seconds because static initial loading cost more. These tiny tests establish execution, not full-video speed. The first diagnostic used reordered CUDA indices; the final launch uses `--cuda-device all` to preserve physical numbering.

The full Turbo benchmark uses the original clip's prompt, seed, 608 x 352 resolution, and 124 frames. It changes model placement, sampling from 20 steps to six with the Turbo adapter, and disables Spectrum. See `turbo_benchmark_result.json` and `turbo_benchmark_gpu_samples.json` for the result when complete.

## Files and launch

- Launch: `Start H3 MultiGPU.cmd` in the project root.
- Workflows: `H3_MultiGPU_Turbo6_Text_To_Video.json` and `H3_MultiGPU_Turbo6_First_Last_Frame.json` in ComfyUI's Workflows sidebar.
- Output: `ComfyUI_windows_portable/ComfyUI/output/video/`.
- Runtime logs: `multigpu.stderr.log` and `multigpu.stdout.log`.
- The single-GPU startup script and workflows are retained for reference.

The tested peer-access probe returned false between two RTX 3070s. The attention-sharding implementation reviewed requires bidirectional peer access and a working Comfy Kitchen CUDA attention backend; it is not an immediate match for this Windows setup. The chosen DisTorch2 path can use host-staged GPU transfers.

## Paused for RAM upgrade
The owner requested stopping on 2026-09-09 to add RAM. ComfyUI was already stopped when checked; the monitoring process was stopped/checked as well. The full Turbo benchmark did not complete successfully. The five-GPU tiny diagnostic passed, but full-clip speed remains unverified. Resume configuration and benchmarking after the hardware upgrade.

## Resumed with 48 GB RAM

Windows now reports 47.87 GiB usable RAM. The requested near-720 test uses 1280 x 704, 124 frames at 24 fps, and six Turbo steps. The near-720 presets reduce GPU 1's model allocation to 1 GB to leave sampling headroom: `cuda:1,1gb;cuda:2,6gb;cuda:3,6gb;cuda:0,1gb;cuda:4,*`.

The first attempt loaded the model successfully but failed on a Turbo LoRA cross-device operation. The installed Turbo extension's `_FrugalLoRA.bypass_forward` now moves adapter tensors to the input device as well as its dtype. Other runtime AdaLN adapters already do this. The unmodified source copy is retained. Failure evidence is in `near720_adapter_failure.json` and `near720_adapter_failure.log`.

The patched retry is recorded in `near720_submission.json`; monitor samples and completion history are written to `near720_gpu_samples.json` and `near720_result.json`. Until a new completion result is written, the result file can still contain the previous failure. Near-720 rendering and full-clip speed remain unverified while this retry runs.

### Follow-up failures and memory fix

The adapter-fixed 1280 x 704 attempt failed with CUDA OOM in eager `int8_linear` scaling after 326.46 seconds. Reducing to 1024 x 576 still failed at `torch.cat(scaled_parts)` requesting another 1.18 GiB, after 282.62 seconds. Evidence: `near720_vram_failure.json`, `576_result.json`, `576_vram_failure.log`.

The installed Comfy Kitchen eager `int8_linear` now bounds INT32/FP32 intermediates to row chunks and writes into a preallocated output, removing the full accumulated INT32 matrix and final concatenation copy. The upstream file is preserved as `quantization_before_chunk_fix.py`. `test_int8_chunking.py` compares upstream and patched calculations on CPU and CUDA with scalar/per-channel scales, ConvRot, bias, SwiGLU and a partial final chunk. All six cases passed bit-exact parity. Full pipeline validation is a separate retry recorded under `576_chunked_*`.

### Shared computation investigation

DisTorch2's current profile uses GPUs as model storage, with sampling on GPU 1. It does not aggregate compute or activation memory. The text encoder and VAE run on GPU 0; there is no separate 3D rendering stage.

- [AesSedai attention sharding](https://github.com/AesSedai/ComfyUI-MiniMaxH3-Parallel) requires bidirectional GPU peer access and working Comfy Kitchen CUDA attention, which the current configuration has not provided.
- [buqi sequence parallelism](https://github.com/buqi-code/buqi-minimax-h3-multigpu) replicates approximately 21 GB of model weights per GPU and requires Linux/WSL NCCL. It is unsuitable as supplied for these 8 GB cards.
- [Raylight](https://github.com/komikndr/raylight) provides sequence parallelism plus FSDP weight sharding and lists both H3 variants as supported. Source commit `9a7c33d52b3d35e29f75ecff3c227de987f0d4cf` was downloaded under `sources/raylight` for inspection, not installed. It includes INT8 distributed layout support. Upstream states that stock native Windows PyTorch does not work and recommends WSL. This is a candidate for a separate Linux runtime, not a verified working configuration on this rig.

Windows `Win32_Processor.VirtualizationFirmwareEnabled` reports false; second-level address translation support reports true. WSL2 therefore needs firmware virtualization enabled before this route can be evaluated. The optional-feature state could not be queried without an elevated Windows administrator token. No BIOS setting, Windows feature, or driver was changed. A four-worker H3 sequence split plus the fifth GPU for auxiliary stages is a candidate configuration, not an established performance result.

## Successful 48 GB / 576p benchmark

- Prompt ID: `a63a3518-5994-4136-a495-8d233730a76a`; ComfyUI status success, completed true.
- End-to-end: **1126.389 seconds (18 minutes 46 seconds)**. Six-step sampling: **813 seconds (13 minutes 33 seconds)**.
- MP4: `ComfyUI_windows_portable/ComfyUI/output/video/H3_Turbo6_1024x576_48GB_00001_.mp4`.
- PyAV verified **1024 x 576, 124 decoded frames, 24 fps, 5.1667 seconds**. First/middle/last frames show rendered scene content and camera/scene changes rather than empty or corrupted frames; this is a visual spot check, not a complete motion-quality review.
- Validation: `576_chunked_result.json`, `576_chunked_gpu_samples.json`, `576_chunked_media.json`, and `verify_benchmark_video.py`.
- During sampling the five GPUs used approximately 32 GiB combined; GPU 1 stayed around 7222 MiB and Windows had about 34 GiB of RAM free.
- Compared with the prior 6648.834-second 608 x 352 result, this is about **5.9 times faster**, at **2.76 times the pixels per frame**. It combines RAM, placement, INT8 memory, and six-step Turbo changes; it does not isolate a multi-GPU compute speedup and does not promise identical quality.
- Reusable preset: `H3_Turbo6_576p_Text_To_Video.json`. First/last-frame preset remains untested. Near-720 was not retried after the INT8 chunking fix.

## Successful four-step 704p benchmark

- Prompt ID: `1b6406c1-8cb6-4ce7-be92-d0031fbb7ea9`; status success, completed true.
- End-to-end: **863.328 seconds (14 minutes 23 seconds)**, including warm model preparation, four-step sampling, VAE decode and export.
- PyAV verified **1280 x 704, 124 frames, 24 fps, 5.1667 seconds** in `H3_Turbo4_1280x704_48GB_00001_.mp4`.
- Four-step sampling took 710 seconds (11 minutes 50 seconds). The 704p image has 1.53 times as many pixels as 1024 x 576, but using four rather than six steps made the complete render 263 seconds faster than the validated 576p run.
- Evidence: `700_turbo4_result.json`, `700_turbo4_gpu_samples.json`, `700_turbo4_media.json`, and `700_turbo4_success.log`.
- Four-step Turbo is a speed/quality choice. It can lose motion consistency or fine detail relative to six steps; the successful export does not establish subjective equivalence.

H3 accepts lengths on a 17k+5 frame grid and labels approximately 124-362 frames as its trained range. Presets were added for 175 frames at 1024 x 576 and 243 frames at 832 x 480. They are plausible memory-balanced longer options but have not been rendered yet.

## Z-Image-Turbo starting-image setup

The official ComfyUI Z-Image-Turbo INT8 model set is installed and hash-verified. The diffusion model is 6,201,001,296 bytes, the Qwen text encoder is 5,631,994,051 bytes, and the VAE is 335,304,388 bytes. Exact SHA256 values are in `z_image_models_manifest.json`.

At 1280 x 704 and the official eight-step `res_multistep` settings, the first run completed in 53.151 seconds and a second seed completed in 30.948 seconds with cached weights. Sampling itself was about 21 seconds. Both PNGs decoded at the intended dimensions and were visually inspected. The workflow is `Z_Image_Turbo_INT8_Starting_Image.json`; `H3_Turbo4_700p_ZImage_First_Frame.json` consumes the second generated image as H3's first-frame anchor.

## Ten-second first/last-frame story test

Z-Image generated the accepted room anchor `Toad_Gaming_Room_Start_V2_00001_.png` at 1280 x 704 in 55.461 seconds. A separate endpoint anchor, `Toad_FPS_Battlefield_End_00001_.png`, established a full-frame first-person battlefield.

The first H3 attempt used only the room anchor. It completed at 832 x 480, 243 frames and 24 fps in 1073.298 seconds, but its final frame remained outside the monitor. The second attempt conditioned both endpoints and completed successfully in **1053.176 seconds (17 minutes 33 seconds)**; four-step sampling took approximately **670 seconds (11 minutes 10 seconds)**. PyAV verified 243 decoded frames and a 10.125-second duration. Evidence is in `toad_video_v2_result.json`, `toad_video_v2_gpu_samples.json`, `toad_video_v2_media.json`, and `toad_video_v2_success.log`.

Representative-frame review shows a continuous room-to-monitor camera push: smoke entrance, PC seating, body pixelization, monitor vortex, partial bezel crossing, then a full-frame FPS battlefield. The two cats remain distinct and anatomically coherent. H3 did not reliably execute every requested action in one pass: contact with both cats is unclear, full-body suction is implied rather than shown, and no combat-equipped toad appears inside the battlefield. Reliable completion of those beats requires separate shot generation with anchor frames, followed by interpolation or editing.


## Four-GPU x16 benchmark - 2026-09-11

After GPU `GPU-cc8d63cd-c1d3-07e7-9a6c-60dd7566cd60` disappeared on its mining riser, the owner moved a remaining RTX 3070 into the CPU-connected primary slot. Windows now sees four cards: GPU 0 at PCIe Gen3 x16 and GPUs 1-3 at Gen3 x1.

- Validated allocation: `cuda:0,0.5gb;cuda:1,6.5gb;cuda:2,6.5gb;cuda:3,*`, compute `cuda:0`. CLIP and VAE also run on cuda:0 in separate phases.
- Corrected benchmark: 1280 x 704, 124 frames, 24 fps, H3 Turbo four steps, seed 0.
- Four-step sampling: 11m47s / 176.88 seconds average. Previous five-card/all-x1 result: 11m50s / 177.66 seconds average. Sampling is effectively tied (about three seconds faster, below 1%).
- End-to-end cold time: 15m30s versus the previous warm 14m23s; cold redistribution across four cards accounts for most of the difference.
- Verified MP4: `H3_Turbo4_1280x704_4GPU_x16_V2_00001_.mp4`, 1280 x 704, 124 decoded frames, 24 fps, 5.166667 seconds.
- A test with `eject_models=False` retained donor VRAM after export, but the next Turbo render still discarded and rebuilt the distributed model during its patch/load cycle. The attempted warm run was stopped once this behavior was proven.
- The active local agent now uses `x16_4gpu_benchmark_v2_prompt.json` as its base. V2 resumed from clip 11 with ten prior clips preserved.

## Unexpected reboot evidence - 2026-09-11

Windows System logs show NVIDIA event 153, `BusReset TDR occurred on GPUID:600`, followed by bugcheck `0x116 VIDEO_TDR_FAILURE`. Kernel-Power event 41 records an unclean crash with no power-button timestamp. No WHEA hardware errors were found in the prior seven days. `C:\Windows\MEMORY.DMP` was written. The owner reports an air conditioner was switched on on the same circuit immediately before the GPU drop; this is plausible context but the event log alone does not prove the electrical root cause.
