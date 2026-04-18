# Gate 0 Launch Post (draft, unshipped)

Target surfaces: HN Show, /r/LocalLLaMA, X thread, Shrey's LinkedIn.

Ship decision after: Gate 0.5 (bench harness fix) + Jay signs off on the scheduling-cliff framing.

---

## HN Show title candidates

1. **"Show HN: InferGrid – bare-metal LLM orchestration that stays below the scheduling cliff"**
2. "Show HN: InferGrid – run two LLMs on one A100 without Kubernetes (and our benchmark broke first)"
3. "Show HN: We profiled vLLM and SGLang – they've converged, and the real waste is in scheduling"

Pick #1 for the default; fall back to #2 if we want the contrarian pull.

---

## One-line elevator pitch

InferGrid is a pip-installable middleware that keeps vLLM (and SGLang) below the concurrency level where throughput saturates but latency explodes — so two models can share one GPU without a Kubernetes cluster.

---

## Body (post-ready draft, ~450 words)

**Two weeks ago we thought vLLM lagged SGLang by 29% on Llama-3.1-8B. The data said otherwise.**

I've been building InferGrid — a middleware that sits on top of vLLM and SGLang to do three things the big-league orchestrators (Dynamo, llm-d, AIBrix) don't do well: lightweight multi-model serving on 1–4 GPUs, admission control under overload, and bare-metal deployment with zero Kubernetes. The v0 of this thesis was: vLLM leaves 29% throughput on the table vs SGLang, and we can recover it.

I ran the profiling on RunPod A100 SXM and H100 SXM ($18 total). Three concurrency sweeps per engine, 200 requests per level, 2 repeats. Here's what the data actually said:

- **vLLM and SGLang have converged.** At c=128, SGLang hits 5,276 tok/s, vLLM hits 5,334. That's a <2% gap, not 29%.
- **GPUs aren't 81% idle.** They're 95–99% busy. The waste isn't hardware — it's scheduling.
- **There's a clear scheduling cliff.** Going from c=128 → c=256 gains 2% throughput and costs 1,434% TTFT (vLLM A100). Same shape on H100. Same shape on SGLang. **Hardware-independent.**

That last point reshaped the whole pitch. The product is no longer "beat SGLang"; it's "stay below the cliff while multiplexing models on one box."

**So this weekend I ran Gate 0 — the first live GPU bring-up of `infergrid serve`.** Two models, Llama-3.1-8B-Instruct and Qwen2.5-7B-Instruct, co-resident on a single A100-SXM4-80GB. Budget: $4.50. Actual: $5.76, because of five distinct dependency and infrastructure regressions that each bit once (transformers 5.x dropped an attribute vLLM 0.8.5 relies on; numpy 2.4 broke numba; vLLM v1 engine OOMs under co-load; pod lacked SSH port mapping; HF_TOKEN didn't propagate into SSH shells — all fixed in the fix branch, all in the repo).

**Result: system passed. Our benchmark harness broke first.**

- 3h52m server uptime, both engines `healthy: true` the whole way
- 181 requests admitted, 0 rejected, 0 timed out by admission
- 55.7 GB / 80 GB VRAM, matching the 0.35+0.35 config exactly
- 10.2 ms router overhead per model
- No OOM, no crash, no need to restart
- The multi-model bench harness then hung on `alternating | concurrency=1` after the first wave, hit aiohttp's 300s timeout, and never recovered. Deferred to Gate 0.5.

Repo: **github.com/coconut-labs/infergrid**
Full Gate 0 post-mortem (including the 6-run recovery log): `results/gate0_20260418/GATE0_OUTCOME.md`

Next up: Gate 0.5 is a local bench-harness fix (no GPU), then Gate 1 is admission-control TTFT on H100 ($15). arXiv preprint around week 4.

Happy to discuss anywhere — the "engines have converged, the scheduling is the product" framing feels like the least-crowded corner of this market. Curious what you're seeing.

— Shrey

---

## Response playbook (top-5 expected questions)

**Q: Why not just use Dynamo / llm-d?**
A: Both require Kubernetes. For 1–4 GPUs on a single box, K8s is pure overhead. Lightweight orchestration is a real gap in the landscape (see `docs/inference_orchestration_gaps_report.md`).

**Q: Is this just Ollama with extra steps?**
A: Ollama is LRU model-swap + llama.cpp. We're frequency+recency multi-model + vLLM/SGLang + admission control + KV tiering (stub). Different problem: Ollama optimizes for hobbyists on a single prompt; InferGrid for teams sharing a box under concurrent load.

**Q: Why should I care about the scheduling cliff when I can just run at lower concurrency?**
A: Because the engines' default concurrency is at or above the cliff. Most people don't know they're there. Our admission controller holds the line for you.

**Q: When will this be usable?**
A: `infergrid serve --config ...` works today on any A100 (see PR #19 for reproduction). The bench harness and Gate 1 (admission TTFT validation) are in progress.

**Q: vLLM 0.8.5 is ancient. Why?**
A: That's what the Phase 1 profiling pinned. Upgrade is on the roadmap once Gate 1 data lands. Newer vLLM avoids the compat issues we hit; we have pins in `requirements-gpu.txt` that cover 0.8.5 specifically.

---

## X thread outline (4 tweets)

1. "Hook tweet: 181 multi-model requests on one A100 for 3h52m. Zero OOM. Our own benchmark broke first. <link>"
2. "The data that killed our original thesis: vLLM–SGLang gap is <2%, not 29%. Engines converged. The real waste is in scheduling quality. Chart."
3. "Scheduling cliff at c=128→256 on both A100 and H100. Hardware-independent. This is the knob InferGrid holds for you. Chart."
4. "6-run recovery log → fix PR → system passed → bench hung. Deferred to Gate 0.5 (local repro, no GPU). Gate 1 at ~$15. arXiv drafting. Repo: <link>"

---

## Timing

Per Phase B roadmap: HN post target **2026-05-13 09:00 PT**. That's ~25 days out. Hold this draft until Gate 0.5 fix lands and at least one partial admission-control datapoint is in hand — otherwise the "system works" claim is weaker than the tweets imply.

**Do not ship until:**
- [ ] Gate 0.5 bench fix merged
- [ ] At least one clean concurrency-1 benchmark run showing p50/p99 TTFT
- [ ] Jay reviews the framing (especially the "we were wrong" opening)
- [ ] Landing page refresh is live (per Phase B §2)
