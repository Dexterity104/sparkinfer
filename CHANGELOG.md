# Changelog

Notable changes to sparkinfer. Format loosely follows [Keep a Changelog](https://keepachangelog.com);
versions track the GitHub [releases](https://github.com/gittensor-ai-lab/sparkinfer/releases).

## [0.2.0] — 2026-06-25

Evaluation-pipeline hardening, anti-gaming controls, and the live frontier dashboard.

### Added
- **Opt-in RTX 5090 evaluation** — the PR auto-eval bot runs the on-device eval only after the
  PR template's *Tested on RTX 5090* box is ticked (auto-applies `test-on-5090`) or a maintainer
  greenlights it; otherwise the PR is labeled `not-tested` and skipped (no GPU). Falsely ticking
  the box is treated as gaming.
- **Live optimization-journey chart** on the [dashboard](https://gittensor-ai-lab.github.io/sparkinfer/dashboard/)
  — recorded passes (history) plus optimizations that have **landed** on the frontier; the bot
  appends each frontier-advancing merge automatically. Accuracy (token-match / KL) now tracks the
  frontier instead of a stale manual value.
- **Community safety hardening** (merged PRs) — input/scratch bounds guards across the MoE expert
  FFN, decode runner, and router kernel; GGUF load-time validation (reject unsupported GGML types,
  clamp invalid `general.alignment`, bounds-check tensor regions vs file size).

### Security (anti-gaming)
- **Sensitive-path merge gate** — `CODEOWNERS` + a `sensitive-paths-guard` status check + branch
  protection block any non-maintainer PR touching the eval/scoring/governance paths (`eval/`,
  `bench/scripts/`, `.gittensor/`, `dashboard/data.json`, `.github/`). The bot also grades with
  `bench/scripts` pinned to `origin/main`, so a PR cannot grade itself.
- **Contributor denylist + auto-block** — `.github/blocked-contributors.txt` (+ `FLAGGED.md`
  evidence log); the bot flags, comments, closes, and skips eval for any PR whose opener or commit
  author/committer is blocked. First entry: a 2-account sybil pair sharing one git identity.
- **Copycat detection** — diff-fingerprint each PR against earlier ones; ≥80% containment of a
  *different* author's earlier diff → `copycat` label, skipped eval, logged to `.github/copycats.json`;
  2 strikes auto-blocks the author.

### Changed
- PRs are evaluated **oldest-first**, so the original of any duplicate is graded before its copy.
- Dashboard: removed the obsolete **emission-weights** panel (scoring is speedup-only — there is no
  per-subsystem budget).

### Fixed (evaluation pipeline)
- Provisioning self-heals: abandon phantom-`running` hosts in ~2 min, retry across hosts, blacklist
  repeat offenders, and survive SSH drops during the 17 GB model download (nohup + resumable fetch).
- Build: pin `g++-12` as the CUDA host compiler (nvcc vs Ubuntu 24.04 GCC 13.3 `cstdio` break);
  cap `-j2` to avoid OOM on 64 GB eval boxes.
- A submission that does not compile now yields a clean `eval:REJECT` instead of an infra error.
- **Force-clean per-PR checkout** — each PR builds its own commit (a stale-checkout bug had graded
  several PRs against the wrong code).
- Labels/comments applied via the GitHub REST API (the GraphQL path silently failed on a
  deprecation warning).

### Verified
- **RTX 5090** frontier ratcheted to **187.61 tok/s** (PDL decode; #8, `eval:L`), **top-1 98%**
  token agreement vs llama.cpp (KL ≈ 0.14 nats).

### Contributors
First community contributors — thank you! 🎉
[@galuis116](https://github.com/galuis116), [@jaso0n0818](https://github.com/jaso0n0818),
[@kiannidev](https://github.com/kiannidev), [@philluiz2323](https://github.com/philluiz2323).

> A fifth early account was removed for sybil / eval-gaming (one git identity across two logins,
> farming merged-PR emissions) — see **Security** above and `.github/FLAGGED.md`.

[0.2.0]: https://github.com/gittensor-ai-lab/sparkinfer/releases/tag/v0.2.0

## [0.1.0] — 2026-06-22

First release of the consolidated **sparkinfer** monorepo (kernels + MoE engine + runtime + benchmarks).

### Added
- **Native GGUF loading** — mmap parser + on-GPU **byte-exact Q4_K / Q6_K dequant**;
  expert weights kept quantized resident (Q4_K_M-sized footprint, not bf16).
- **Qwen3-MoE runtime** — embed → RMSNorm → QKV → per-head QK-norm → RoPE → paged GQA
  flash-decode → routed top-k MoE (+ optional shared expert) → LM head → greedy decode.
- **Kernels** — flash-decode (hd128/256/512), **flash-decoding (KV-split)** attention,
  **fused quantized MoE expert FFN** (dequant only the routed experts on-read), decode
  GEMV (coalesced `[out,in]`), GEMM, fused RMSNorm, RoPE.
- **CUDA-graph decode** — the per-token compute is captured once and replayed.
- **Turnkey harness** — `bench/scripts/bench.sh` (decode tok/s, `--compare` vs llama.cpp)
  and `accuracy.sh` (token-match / KL / perplexity); auto-detect arch, fetch model.
- **Accuracy gate** — `qwen3_gguf_score` teacher-forced scorer (per-position argmax +
  top-k logprobs + perplexity), for regression-checking optimizations.
- **Prebuilt binaries** attached to this release (sm_120 / CUDA 13 / glibc 2.39), with
  automatic **source-build fallback** when incompatible.

### Verified
- **RTX 5090** (sm_120, CUDA 13): `ctest` 5/5, compute-sanitizer 0 errors,
  **163.88 tok/s** decode, **100% top-1 token agreement** with llama.cpp (KL ≈ 0.14 nats),
  21.4 GB resident.
- **RTX PRO 6000** (sm_120, CUDA 12.8): **0.60 → 134 tok/s** decode across 6 source-verifiable
  optimization passes.

### Fixed (during RTX 5090 / CUDA 13 bring-up)
- CUDA 13 removed `cudaDeviceProp::memoryClockRate` / `memoryBusWidth` → query via
  `cudaDeviceGetAttribute` (portable across CUDA 12.x / 13).
- Flash-decode scratch (`fa_*`) was NULL on the non-GGUF path (allocated only in
  `load_gguf`) → moved to the constructor (caught by compute-sanitizer).
- Top-level superbuild was missing `enable_testing()` → `ctest` found no tests.

[0.1.0]: https://github.com/gittensor-ai-lab/sparkinfer/releases/tag/v0.1.0
