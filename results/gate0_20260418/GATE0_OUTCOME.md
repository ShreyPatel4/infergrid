# Gate 0 — First Live GPU Bring-up of `infergrid serve`

**Date:** 2026-04-18
**Hardware:** 1 × NVIDIA A100-SXM4-80GB on RunPod (SECURE tier)
**Pod ID:** au17ivg77skuzw (terminated)
**Duration:** 3h 52m wall clock (16:33 → 20:26 UTC)
**Cost:** ~$5.76 (budget was $4.50 — $1.26 over, primarily due to debugging rounds 1-5)

## Verdict: PASS (system) + TODO (bench harness)

**Core Gate 0 objective was met:** `infergrid serve` holds two open-weight LLMs co-resident on one A100 and serves OpenAI-compatible traffic without OOM, crash, or admission-controller rejection for >3h.

**Bench harness objective deferred:** the multi-model benchmark client (`benchmarks/scripts/benchmark_multi_model.py`) stalled on `alternating | Concurrency: 1` after the first stretch of requests; requires follow-up investigation before Gate 1.

## Proven

| Check | Result |
|---|---|
| Both models registered on `/v1/models` | PASS — `meta-llama/Llama-3.1-8B-Instruct`, `Qwen/Qwen2.5-7B-Instruct` |
| Llama-3.1-8B returns valid `/v1/chat/completions` | PASS (see `smoke.jsonl`) |
| Qwen2.5-7B returns valid `/v1/chat/completions` | PASS (see `smoke.jsonl`) |
| Both engines co-resident in VRAM | PASS — 55.7 GB / 80 GB (aggregate 0.70, matches 0.35+0.35 config) |
| No CUDA OOM during 3h+ runtime | PASS |
| No admission-controller rejections | PASS — 181 admitted, 0 rejected, 0 timed_out |
| Both engines `healthy: true` at shutdown | PASS |
| Per-model request count | Llama 103, Qwen 78 |
| Router-internal avg latency | 10.2 ms (Llama), 10.2 ms (Qwen) |

## Config used

```yaml
# configs/gate0_multi_model.yaml
port: 8000
max_concurrent: 128
models:
  - model_id: meta-llama/Llama-3.1-8B-Instruct
    engine: vllm
    dtype: bfloat16
    gpu_memory_utilization: 0.35   # lowered from 0.42 → 0.40 → 0.35
    max_model_len: 4096
  - model_id: Qwen/Qwen2.5-7B-Instruct
    engine: vllm
    dtype: bfloat16
    gpu_memory_utilization: 0.35
    max_model_len: 4096
```

Pod env: `VLLM_USE_V1=0` (v0 engine gave better error surface and co-load stability).

Dep versions (from `pip_freeze.txt`): `vllm==0.8.5`, `transformers==4.57.6`, `numpy==2.2.6`, `numba==0.61.2`, `torch==2.6.0+cu124`.

## Known issue: benchmark harness stall

The `benchmark_multi_model.py` harness was invoked with `--concurrency 1,8,32,128 --workload all --num-requests 100` after smoke tests passed. It entered `alternating | Concurrency: 1` at 17:13:42 UTC and never advanced past the "Starting multi-model benchmark" log line. Three hours later (20:20:10 UTC) an aiohttp 300-second timeout fired and wrote a single `500` to the access log; the benchmark client did not recover.

**What was NOT the cause:**
- Server was healthy the entire time (continuous 200s on `/infergrid/status` and `/metrics` through 20:26).
- 181 requests were successfully admitted and served (smoke + partial bench).
- No OOM, no engine crash, no admission-controller rejection.

**Likely causes (to investigate in follow-up):**
1. Harness deadlocks if its internal `asyncio.Semaphore` and the router's admission queue interact badly when requests alternate between two models at concurrency=1.
2. A specific prompt in the alternating schedule triggered a long vLLM generation that exceeded the client-side aiohttp 300s timeout.
3. Client retries without releasing its slot, blocking further sends.

## Six-run recovery log (what bit us and how we fixed it)

| run | killed by | fix |
|---|---|---|
| 1 | Pod created without SSH port mapping | pass `ports="22/tcp,8000/http"` to `runpod.create_pod` |
| 2 | `HF_TOKEN` in container env not inherited by SSH sessions | scp `/root/.gate0_env` to pod, bootstrap sources it |
| 3 | `transformers==5.5.4` removed `all_special_tokens_extended` | pin `transformers>=4.51.1,<5.0` (PR #16) |
| 4 | `numpy==2.4.4` broke `numba` (vLLM imports numba on load) | pin `numpy>=1.24,<2.3` (PR #16) |
| 5 | Qwen co-load OOM under vLLM v1 engine | `VLLM_USE_V1=0` + `gpu_memory_utilization` 0.40 → 0.35 |
| 6 | (bench harness stall — see above) | SUCCESS on system objective; harness TODO |

All five pre-existing regressions are fixed in `fix/gate0-compat-hardening` (PR #16). Cost of the recovery journey: ~$3 of pod time over budget.

## Artifacts in this directory

- `bootstrap.log` — full shell narrative of the successful run-6
- `server.log` (99 KB) — infergrid serve log, includes every request access entry
- `smoke.jsonl` — both model smoke responses verbatim
- `status_before.json` / `status_after.json` — infergrid router snapshots
- `prometheus_dump.txt` — Prometheus metrics at shutdown
- `nvidia_smi_final.txt` — GPU state at shutdown (55.7 GB used, healthy)
- `gpu_trace.csv` (3.4 MB) — 1 Hz GPU samples for the full 3h+ runtime
- `pip_freeze.txt` — complete Python environment on the pod
- `gate0_multi_model.yaml` — the config that served the two models
- `benchmarks/run_config.json` + `switch_latency.json` — partial bench output before the hang

## Next actions

1. **Ship this PR (#18)** — Gate 0 outcome on the record.
2. **Follow-up: investigate bench harness stall** — reproduce locally with mocked endpoints before re-spending on GPU time. Add asyncio timeout fallbacks and request-level logging. Add a smaller smoke-bench harness (50 requests, concurrency 1-8) as a Gate 0.5 acceptance test.
3. **Gate 1 can proceed** in parallel once Gates 0+0.5 land. Per Phase B roadmap (PR #17), Gate 1 is the admission-control TTFT benchmark on H100 at ~$15.
