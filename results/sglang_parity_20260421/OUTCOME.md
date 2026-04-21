# SGLang Engine Parity — Gate 2-FAIRNESS hero 3-arm replication on A100

**Date:** 2026-04-21
**Hardware:** 1× NVIDIA A100-SXM4 80GB (RunPod SECURE, on-demand $1.49/hr)
**Pod:** `y25od5cajpk039` (deleted post-run)
**Engine:** SGLang 0.5.9 (`sgl_kernel` 0.3.21) via InferGrid `SGLangAdapter`
**Model:** meta-llama/Llama-3.1-8B-Instruct, bfloat16, context_length=4096, `mem_fraction_static=0.88`, tp=1
**InferGrid main tip:** `810e2c2` (PR #89)
**Total cost:** ~$0.94 (37 min; ceiling was $5)

## Verdict: **SGLang-internally-fairer**

Under contended FIFO the SGLang engine by itself delivers a quiet-tenant p99 that is **5.6× better** than vLLM's FIFO quiet p99 on the same workload (283 ms vs vLLM hero 1585 ms). InferGrid's token-bucket layer stacks on top of that and pulls quiet p99 the rest of the way down to near-solo levels (69 ms), within ~12% of the vLLM hero's bucket number. **Both cross-engine claims are validated:** the InferGrid engine-adapter surface is correct, and the rate-limit admission path works identically on SGLang as on vLLM.

Against the task's pre-committed verdict buckets:
- **MATCH** requires FIFO quiet_p99 ≥ 10× solo. On SGLang the ratio is **0.81×** — so formally not MATCH (FIFO quiet is actually marginally *better* than the saturated-solo baseline because one quiet arrival per second barely perturbs a fully-loaded engine). The spirit of the claim — "works on both engines" — holds on the token-bucket arm (0.20× of solo, well under 1.5×).
- **SGLang-internally-fairer** requires FIFO quiet_p99 < 5× solo. **Triggered** (0.81×).
- Worse on SGLang (bucket > 2× solo): **not triggered** (0.20×).

## 3-arm table (post-10s warmup)

| Metric | Arm 0 solo | Arm 1 FIFO | Arm 5b token-bucket |
|---|---:|---:|---:|
| Config | `configs/gate2_fairness_fifo_sglang.yaml` | same | `configs/gate2_fairness_token_bucket_sglang.yaml` |
| Workload | 1 tenant @ 32 RPS, 300 s | 1 flooder @ 32 + 1 quiet @ 1, 300 s | same as Arm 1, `rate_limit_rpm=600, burst=10` |
| flooder count_ok | 9 153 | 9 153 | 2 900 |
| flooder count_err (429) | 0 | 0 | 6 271 (**68.4% throttled**) |
| flooder ttft_p99 | **348.7 ms** | 358.9 ms | 66.0 ms |
| quiet count_ok | — | 311 | 311 |
| quiet ttft_p50 | — | 55.4 ms | 43.0 ms |
| quiet ttft_p95 | — | 119.1 ms | 60.0 ms |
| **quiet ttft_p99** | **—** | **283.2 ms** | **69.1 ms** |
| quiet ttft_max | — | 370.5 ms | 101.8 ms |

Full-bench p99s (no warmup filter) are in `post_warmup.json`; they track the post-10s numbers within 1 %.

## Comparison to vLLM v3 hero

| | Arm 0 solo (task-given) | Arm 1 FIFO quiet_p99 | Arm 5b bucket quiet_p99 |
|---|---:|---:|---:|
| vLLM v3 hero reference | 53.9 ms | 1 585.1 ms | 61.5 ms |
| SGLang this run (post-10s) | 348.7 ms | **283.2 ms** | **69.1 ms** |
| SGLang / vLLM ratio | 6.47× (not apples-to-apples, see note) | **0.18×** (SGLang ~5.6× fairer) | **1.12×** (parity) |

Arm 5b flooder-429 rate: SGLang 68.4% (6271/9171) vs vLLM v3 68.3% (6488/9497) — near-perfect parity on the admission layer.

### Note on Arm 0 solo comparability

The task's Arm 0 spec (`flooder_rps=32 quiet_rps=0 num_quiet=0`) is a single tenant driving the engine to saturation — the p99 reported here (348.7 ms) is the tail of that saturated load. The vLLM v3 hero "solo=53.9" is an *unloaded* quiet-only baseline (`flooder_rps=0 quiet_rps=1`, matches `gate2_preprint_pod1_evidence/arm0_solo`, quiet_p99=62.5 ms; the 53.9 ms reference appears to be a post-warmup number not persisted in `summary.json`). The cross-engine ratio for Arm 0 is therefore not apples-to-apples; the load-bearing cross-engine comparisons are Arm 1 (contended FIFO) and Arm 5b (bucket), which *are* run with matched workloads on both engines.

## Raw engine launch arguments (SGLang)

```
python -m sglang.launch_server \
  --model-path meta-llama/Llama-3.1-8B-Instruct \
  --port 8002 \
  --mem-fraction-static 0.88 \
  --tp 1 \
  --context-length 4096 \
  --dtype bfloat16
```

SGLang selected `flashinfer` attention backend, `xgrammar` grammar backend, `schedule_policy=fcfs`, `chunked_prefill_size=8192`, captured 36 cuda graph batch sizes [1..256], KV cache 444 563 tokens, available GPU mem after warmup 8.08 GB. See `sglang_solo/engine.log` for the full server_args.

## Environment gotcha (for repro)

`runpod/pytorch:2.4.0-py3.11-cuda12.4.1-devel-ubuntu22.04` does **not** ship `libnuma1` / `libnuma-dev`, and SGLang 0.5.9's `sgl_kernel` silently fails to load `common_ops.abi3.so` without it. The import error cascades through `sglang.srt.layers.quantization.fp8_kernel` at launch time and the engine subprocess exits with code 1 before `Application startup complete`. Fix:

```
apt-get update && apt-get install -y libnuma1 libnuma-dev
```

After that, first SGLang boot took ~30 s to `/health=200` (weights loaded in 2.1 s, KV allocated, cuda graph captured in 3.15 s).

## Artefacts in this directory

- `sglang_solo/{summary.json,tenant_flooder.csv,bench.log,server.log,engine.log}` — Arm 0 (solo, 32 RPS flooder alone)
- `sglang_fifo/{summary.json,tenant_flooder.csv,tenant_quiet_0.csv,bench.log,server.log,engine.log}` — Arm 1 (FIFO contended)
- `sglang_tokenbucket/{summary.json,tenant_flooder.csv,tenant_quiet_0.csv,bench.log,server.log,engine.log}` — Arm 5b (token-bucket)
- `post_warmup.json` — full-bench vs post-10s-warmup percentiles for every tenant in every arm
- `run.log` — orchestrator log (engine start / bench / stop for all 3 arms)

## Pre-committed criteria check (Arm 5b, bucket runbook)

| Criterion | Value | Result |
|---|---|---|
| Arm 5b quiet_p99 ≤ 300 ms (5× Arm 0 baseline) — *original runbook, vLLM baseline* | 69.1 ms | **PASS** |
| Arm 5b flooder gets 429'd (rate-limit fires) | 68.4% 429 rate | **PASS** |
| Arm 5b quiet ALSO getting 429s (plumbing bug) | 0 | **PASS** (no plumbing regression) |

All three pre-committed criteria pass on SGLang.
