# Strata on Debian: plan

## Goal
Work from a clean fork of Strata (https://github.com/Niko1221/Strata) on Debian, and adopt selected ideas from
GyroStrata (https://github.com/Gitshop77/GyroStrata) one at a time. We do not use GyroStrata's patch.

## What we found about GyroStrata
- It is not a fork. The repo holds a single 4,971-line `patches/strata.patch`, pinned to Strata commit `a0116b2`,
  plus a Windows-only installer (`.cmd`, MSVC, `nvcc.exe`, `ctypes.windll`) and launch profiles.
- It targets the Gyro-M model (trellis quantization + Hadamard rotations, custom `agentionai/llama.cpp`), which is
  a different model file from the standard Strata models. Those parts do not apply to standard Strata models.
- Its own changes (about 3,600 added lines, 47 files):
  - faster CPU trellis kernels (AVX2/AVX512 dispatch)
  - SSD/CPU overlap (async worker fetches file experts while resident experts compute)
  - RAM rebalancing (swap recently used SSD experts into RAM after each reply)
  - SSD read merging (`STRATA_IO_MERGE_GAP_KIB`) and an interleaved `experts.bin`
  - conversation tail retention and checkpoints
  - speculative-decoding reprobe (`STRATA_LOOKUP_REPROBE_ROUNDS`)
  - memory guards and safe shutdown
- The 12 GB VRAM trick itself is mostly Strata's: experts split across VRAM, RAM (up to 16 GiB) and SSD, Q4 KV,
  32K GPU KV page cache, speculative decoding.
- Only a few `_WIN32` / `_MSC_VER` blocks exist in the engine patch. The Windows-specific part is the installer.

## Setup
- Clean clone: `/repos/Strata` at `296b59d2`, no patches. Strata already has conversation caching
  (`src/core/conversation_*.cpp`, `conv_cache.hpp`), so that idea may be covered.
- Dev container: Debian 13, 16 cores, 62 GB RAM, no GPU, no CUDA, no compiler. Cannot build or benchmark the engine.
- Test host "trinity": Debian 13, i7-7700K (4C/8T, AVX2, no AVX-512), 46.9 GB RAM, RTX 3060 12 GB (CC 8.6).
  - Use IQ2_XS or Q2_0 (48 GB tier). About 80 GB disk.
  - Expect lower speed than Strata's RTX 5070 numbers (3060 has about half the memory bandwidth; old CPU; likely PCIe 3.0).

## Test plan
1. Container (no GPU): run `python tools/test_setup_*.py`; install cmake/g++ and try a CPU-only build of the
   unit tests (`conversation_*_test`, `conv_cache_test`, `expert_multi_test`). Not yet verified that CMake can
   build without CUDA. Write a CPU-only test for each new change.
2. Scripts for the GPU host: A/B wrapper around `bench.sh` (flag off vs on in one session, JSON output) and a
   temperature-0 byte-identical check with the flag off.
3. On trinity: pull the branch, build, run the scripts, send back results. Keep a change only if the gain repeats.

## Order of work
1. Set up the container toolchain and see what CPU-only tests can cover.
2. Check which GyroStrata ideas Strata already has; list the real gaps with rough sizes.
3. For each gap: opt-in change behind a flag (default path byte-identical, per `AGENTS.md`), CPU test, A/B script,
   own branch.
4. Baseline first: stock Strata on trinity with IQ2_XS and Q2_0 via `bench.sh`.

Candidate ideas, likely order for this hardware: CPU kernel speed-ups, SSD/CPU overlap, RAM rebalancing, read
merging, reprobe. Skip trellis/Hadamard work.

## Open questions
- SSH access from the container to trinity, or pass files back and forth?
- Is Strata already installed on trinity (model, version)?
- Can packages be installed in the container?
