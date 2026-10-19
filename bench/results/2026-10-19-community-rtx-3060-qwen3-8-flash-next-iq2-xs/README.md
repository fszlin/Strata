# Community benchmark on NVIDIA GeForce RTX 3060

Measured on 2026-10-19 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 3060, 12 GiB VRAM
- CPU: Intel(R) Core(TM) i7-7700K CPU @ 4.20GHz (4 cores, 8 threads); installed RAM: 46.9 GiB; storage: not measured
- OS: Linux 6.12.111+deb13-amd64; GPU driver: 615.71.09; CUDA: not read
- Strata: project 0.1.45.3, commit e88ee39d; engine 0.1.45.3
- Background workloads and power limits: not recorded

## Model and configuration

- Model and quantization: qwen3.8-flash-next-iq2_xs
- Context: 131072 tokens; parallel slots: 1
- Settings: the install's own configuration, unchanged; thinking off for the suite; temperature 0

```text
strata --pack iq2_xs --native Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf --ple-gguf Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf --expert-profile expert-profile.bin --expert-cache auto --prefill auto --spec 4 --mtp rt --max-context 131072 --kv int8 --kv-resident 32768 --vision --vram-reserve-mib 700 --conversation-cache-mib 2048 --conversation-cache-slots 2 --adapt-async 1 --pcie-frac 0.34 --spec-min-p 0.70 --adapt-every 1 --adapt-swaps 80 --adapt-decay 0.97
```

## Method

Run by `python tools/strata_bench.py --long` against the HTTP API of a Strata server (already running); model loading is not timed. One warm-up chat and a 1K prompt are discarded first. Every prompt starts with a unique tag, so the conversation cache cannot help the fresh-prompt tests.

- short chat: the three questions of `tools/ab_engine.py`, up to 100 new tokens, 3 runs
- 4K / 16K prompts: generated word lists sized by a calibration request, 24 new tokens, 3 fresh prompts each
- agent session: system prompt and four tools, then 5 turns that each add about 1500 words of a project file and continue from the model's own earlier answers (the way agent clients resend a conversation), 64 new tokens per turn, 3 sessions; the cache numbers come from the answer's usage (`cached_tokens`) and timings (`prompt_n`)
- concurrency (when run): N requests sent at the same moment, total tokens over the wall time of the burst, latency over all requests

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 46 (43-51) | 0 | 100 (100-100) | 9 | not measured | 52.6 (47.0-71.9) | 0.68 (0.58-0.79) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 3 | 953.4 (942.4-962.9) | not measured | 4.24 (4.20-4.29) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 3 | 1095.8 (1085.4-1100.5) | not measured | 14.69 (14.62-14.84) |
| 32K prompt, fresh | 32049 (32049-32049) | 0 (0-0) | 24 cap | 3 | 1096.5 (1095.9-1097.8) | not measured | 29.27 (29.24-29.29) |
| agent session, follow-up turns | 14699 (14676-14713) at the last turn | see below | up to 64 | 3 | see below | 54.9 (39.0-68.6) | 3.78 (3.04-4.57) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 69.5 % (69.4-69.6); earlier-turn tokens read again per session: 4 (3-73) (earlier turns that the cache did not hold; the 13811 (13810-13812) new tokens of the files are read once); first turn (nothing cached) first token: 1.90 s (1.87-2.23).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 282.9 | 2.229 | 36.6 |
| 1 | 4120 | 634 | 3486 | 791.6 | 4.571 | 52.0 |
| 2 | 5653 | 4183 | 1470 | 480.4 | 3.123 | 51.8 |
| 3 | 9085 | 5718 | 3367 | 786.7 | 4.345 | 43.5 |
| 4 | 11831 | 9078 | 2753 | 722.5 | 3.866 | 64.6 |
| 5 | 14699 | 11892 | 2807 | 751.4 | 3.777 | 54.9 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load and power limits are not recorded. Total time of the suite: 259 s.
