# Community benchmark on NVIDIA GeForce RTX 3060

Measured on 2026-10-22 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 3060, 12 GiB VRAM
- CPU: Intel(R) Core(TM) i7-7700K CPU @ 4.20GHz (4 cores, 8 threads); installed RAM: 46.9 GiB; storage: not measured
- OS: Linux 6.12.113+deb13-amd64; GPU driver: 615.71.10; CUDA: not read
- Strata: project 0.1.47, commit 011c9e60; engine 0.1.47
- Background workloads and power limits: not recorded

## Model and configuration

- Model and quantization: qwen3.8-flash-next-iq2_xs
- Context: 131072 tokens; parallel slots: 1
- Settings: the install's own configuration, unchanged; thinking off for the suite; temperature 0

```text
strata --pack iq2_xs --native Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf --ple-gguf Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf --expert-profile expert-profile.bin --expert-cache auto --prefill auto --spec 4 --mtp rt --max-context 131072 --kv int8 --kv-resident 32768 --vision --vram-reserve-mib 700 --conversation-cache-mib 2048 --conversation-cache-slots 2 --adapt-async 1 --pcie-frac 0.34 --spec-min-p 0.70 --adapt-every 1 --adapt-swaps 80 --adapt-decay 0.97
```

## Method

Run by `python tools/strata_bench.py --long` against the HTTP API of a Strata server (started by the benchmark from the install configuration); model loading is not timed. One warm-up chat and a 1K prompt are discarded first. Every prompt starts with a unique tag, so the conversation cache cannot help the fresh-prompt tests.

- short chat: the three questions of `tools/ab_engine.py`, up to 100 new tokens, 3 runs
- 4K / 16K prompts: generated word lists sized by a calibration request, 24 new tokens, 3 fresh prompts each
- agent session: system prompt and four tools, then 5 turns that each add about 1500 words of a project file and continue from the model's own earlier answers (the way agent clients resend a conversation), 64 new tokens per turn, 3 sessions; the cache numbers come from the answer's usage (`cached_tokens`) and timings (`prompt_n`)
- concurrency (when run): N requests sent at the same moment, total tokens over the wall time of the burst, latency over all requests

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 46 (43-51) | 0 | 100 (100-100) | 9 | not measured | 52.7 (47.3-69.5) | 0.68 (0.58-0.79) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 3 | 962.5 (956.4-968.5) | not measured | 4.20 (4.17-4.23) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 3 | 1103.4 (1100.9-1106.7) | not measured | 14.59 (14.54-14.62) |
| 32K prompt, fresh | 32049 (32049-32049) | 0 (0-0) | 24 cap | 3 | 1094.4 (1093.7-1095.2) | not measured | 29.33 (29.30-29.35) |
| agent session, follow-up turns | 14599 (14555-14619) at the last turn | see below | up to 64 | 3 | see below | 58.3 (44.3-64.1) | 3.76 (3.06-4.58) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 69.5 % (69.4-69.5); earlier-turn tokens read again per session: 74 (74-74) (earlier turns that the cache did not hold; the 13729 (13728-13731) new tokens of the files are read once); first turn (nothing cached) first token: 1.89 s (1.88-1.97).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 326.9 | 1.974 | 36.4 |
| 1 | 4099 | 628 | 3471 | 798.7 | 4.576 | 50.9 |
| 2 | 5632 | 4163 | 1469 | 481.7 | 3.126 | 58.2 |
| 3 | 9065 | 5625 | 3440 | 799.6 | 4.382 | 44.3 |
| 4 | 11802 | 9094 | 2708 | 716.3 | 3.84 | 63.9 |
| 5 | 14555 | 11838 | 2717 | 730.9 | 3.758 | 45.5 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load and power limits are not recorded. Total time of the suite: 256 s.
