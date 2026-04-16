# Admission Control Implementation Progress

## Completed

### 1. AdmissionController (`src/infergrid/router/admission.py`)
- Implemented concurrency limiting using `asyncio.Lock` + in-flight counter (not raw Semaphore, which cannot support priority ordering)
- Priority queue with `asyncio.PriorityQueue` using `(priority, sequence_number)` tuples for priority + FIFO ordering
- Cancelled/timed-out entries are skipped during `release()` drain
- Prometheus metrics: `queue_depth` gauge, `in_flight` gauge, `admission_wait_seconds` histogram, `rejected_total` counter (by reason), `admitted_total` counter
- Defined `AdmissionTimeoutError` for clean error propagation

### 2. Configuration (`src/infergrid/common/config.py`)
- Added `max_concurrent: int = 128` and `admission_queue_size: int = 1024` to `InferGridConfig`
- Wired through `from_yaml()` and `from_cli_args()`

### 3. Router Integration (`src/infergrid/router/router.py`)
- `WorkloadRouter.__init__` creates an `AdmissionController` from config
- `route_request()` calls `acquire()` before engine forwarding and `release()` in `finally` block
- Priority derived from request length bucket (short=0, medium=1, long=2, xlarge=3)
- `handle_request()` catches `AdmissionTimeoutError` and returns 503 with queue depth info
- `snapshot()` includes admission stats

### 4. CLI (`src/infergrid/cli.py`)
- Added `--max-concurrent` flag to `serve` command (default: 128)
- Help text explains the scheduling cliff context

### 5. Tests (`tests/unit/test_admission.py`)
- Below-threshold pass-through
- Above-threshold queuing
- Priority ordering (short before xlarge)
- FIFO within same priority
- Timeout behavior (returns False, timed-out entries skipped)
- Stats accuracy
- Concurrent stress test (100 simultaneous acquires)
- Fast-path overhead test (<1ms per request)
- Constructor validation

### 6. Benchmark (`benchmarks/scripts/benchmark_admission.py`)
- Sends N concurrent requests through direct engine and through InferGrid
- Compares TTFT distributions
- Uses existing `AsyncBenchmarkClient` from profiling_utils
- Saves CSV results, comparison JSON, and prints summary table

## Design Decisions

1. **Lock + counter instead of Semaphore**: `asyncio.Semaphore` cannot support priority ordering. Using a lock-protected counter with a PriorityQueue gives us both concurrency control and priority-based admission.

2. **Slot transfer in release()**: When a queued waiter is admitted, the in-flight count stays the same (slot transfers directly). This prevents a race where the count momentarily drops and another fast-path request sneaks in.

3. **Cancelled entry cleanup**: Timed-out waiters set `cancelled=True` on their queue entry. `release()` skips cancelled entries when draining the queue, so stale entries don't block live waiters.

4. **Admission in route_request, not handle_request**: All paths through `route_request` (including from queue workers) get admission-controlled. The 503 response is handled in `handle_request` which is the HTTP boundary.

5. **Default max_concurrent=128**: Based on profiling data showing the cliff at c=128->c=256 across both vLLM and SGLang on A100 and H100 GPUs.
