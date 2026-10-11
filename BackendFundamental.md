*Q1. A Node API becomes slow under load, but CPU and DB look fine. How do you diagnose it?*

If CPU and DB are healthy but latency climbs under load, the time is going to waiting, not working. Something is queueing: the event loop, the libuv threadpool, a connection pool, an outbound socket limit, or GC. I diagnose by finding where requests wait, not by guessing at optimizations.

# Step 1: Characterize the problem

- Percentiles, not averages. Check p50/p95/p99 per endpoint. If only p99 is bad, suspect GC pauses or head-of-line blocking. If everything degrades, suspect saturation.
- Latency vs. concurrency. If throughput plateaus while latency keeps rising, requests are queueing somewhere. By Little’s Law, concurrency = throughput × latency, so a flat throughput ceiling means a fixed-size resource is exhausted.
- Scope. One endpoint or all? Since a deploy? Only on one instance?

# Step 2: Check the event loop first

Node runs your JS on one thread, so one blocked tick delays every in-flight request.

Per-core CPU hides this. A single Node process pinned at 100% shows as 12.5% on an 8-core machine, so “CPU looks fine” often isn’t true. Check per-process, per-core usage.

Measure event loop health directly:

const { monitorEventLoopDelay, performance } = require('perf_hooks');

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

let last = performance.eventLoopUtilization();
setInterval(() => {
  const cur = performance.eventLoopUtilization();
  const elu = performance.eventLoopUtilization(cur, last).utilization;
  last = cur;
  console.log({
    p50_ms: h.percentile(50) / 1e6,
    p99_ms: h.percentile(99) / 1e6,
    elu, // >0.9 = saturated
  });
  h.reset();
}, 5000);

If loop delay p99 is in the tens or hundreds of ms, something is blocking it. Typical culprits are JSON.parse/stringify on large payloads, sync fs/crypto, catastrophic regex backtracking, big array sorts, or heavy template rendering. Confirm with a CPU profile (node --cpu-prof, or clinic flame / 0x for flame graphs).

# Step 3: Trace one slow request end to end

Add OpenTelemetry (or manual timing spans) and split request time into:

1. Queue time: time between arrival and handler start (event loop lag, backlog)
2. Handler time: your code
3. Wait on pool: time to acquire a DB or HTTP connection
4. Downstream time: DB query, external API, Redis

Whichever bucket grows with load is your bottleneck. This is the step that turns guessing into evidence.

# Step 4: Reproduce and verify

Use autocannon or k6 to ramp concurrency while watching event loop delay, pool queue depth, and p99 together. Change one variable at a time (pool size, threadpool size, keep-alive) and re-measure. A fix isn’t a fix until the load test shows it.

*Q2. Two concurrent requests try to deduct stock for the last item. How do you prevent overselling?*

I would prevent overselling using three things: atomic updates, proper concurrency control, and database transactions.

1. Atomic Updates: An atomic update means checking the stock quantity and deducting it in a single database operation. For example, I would use a conditional UPDATE query that deducts stock only when the available quantity is greater than zero. This prevents the application from checking and deducting stock in two separate steps.

2. Concurrency Control: When two requests arrive simultaneously, the database manages concurrent updates using mechanisms such as row-level locks and transaction isolation. If only one item is available, one request will successfully deduct it, while the other will find insufficient stock. This prevents both requests from purchasing the same item.

3. Database Transactions: I would wrap the stock deduction and order creation in a transaction. If both operations succeed, I commit the transaction. If order creation or another required operation fails, I roll back the transaction so the stock deduction is not left incomplete.

In summary, atomic updates prevent invalid stock deductions, concurrency control handles simultaneous requests, and transactions maintain consistency between inventory and orders. Together, these ensure that stock never becomes negative and overselling is prevented.