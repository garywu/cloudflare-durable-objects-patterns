# Durable Object Patterns on Cloudflare Workers

> **Stateful compute primitives for the edge — four production patterns, design principles, and practical examples.**

Cloudflare Durable Objects (DOs) are one of the most underappreciated primitives in cloud computing. They give you something no other serverless platform offers: **a single-threaded, globally addressable, stateful compute unit** that lives as long as you need it to.

This is not a getting-started tutorial. This is a pattern catalog for engineers building production systems on Durable Objects. Every pattern here comes from real deployed code.

## Table of Contents

- [Why Durable Objects Matter](#why-durable-objects-matter)
- [The Meta-Pattern: Control Plane / Data Plane](#the-meta-pattern-control-plane--data-plane)
- [Pattern 1: Adaptive Controller](#pattern-1-adaptive-controller)
- [Pattern 2: Storage Sidecar](#pattern-2-storage-sidecar)
- [Pattern 3: Budget Governor](#pattern-3-budget-governor)
- [Pattern 4: Event Reactor](#pattern-4-event-reactor)
- [Layer Guide: Raw DO to Server to Agent](#layer-guide-raw-do-to-server-to-agent)
- [Design Principles](#design-principles)
- [Small Examples](#small-examples)
- [Comparisons](#comparisons)
- [Anti-Patterns](#anti-patterns)
- [Wrangler Configuration](#wrangler-configuration)
- [References](#references)

---

## Why Durable Objects Matter

Every serverless platform gives you stateless compute. Most bolt on state through external databases — DynamoDB, Redis, Postgres. This works, but it creates a fundamental tension: your compute is ephemeral but your state is not, and the glue between them (connection pooling, caching, consistency) becomes the hardest part of your system.

Durable Objects resolve this tension by fusing state and compute into a single primitive:

- **Persistent state**: Each DO has its own transactional storage (key-value or SQLite). State survives restarts, deployments, and migrations.
- **Single-threaded execution**: Only one request executes at a time within a DO. No locks, no races, no distributed consensus. If your code is running, you have exclusive access to your state.
- **Global addressing**: Every DO has a unique ID. Any Worker, anywhere in the world, can send a request to any DO by its ID. Cloudflare routes to the right machine automatically.
- **Location intelligence**: DOs migrate to be near their callers. A DO serving mostly European users will run in Europe.
- **Alarm scheduling**: A DO can set an alarm to wake itself up in the future — even if no external requests arrive.

These properties combine to make DOs the natural home for **control logic**: the code that decides what to do, when to do it, and how to adapt when things go wrong.

```
+------------------------------------------------------+
|                    The Key Insight                    |
|                                                      |
|  A Durable Object is not a database row.             |
|  It's not a cache entry.                             |
|  It's not a microservice.                            |
|                                                      |
|  It's a tiny, autonomous agent with perfect memory   |
|  and a guaranteed single thread of execution.        |
|                                                      |
|  Think of it as an actor that never forgets.         |
+------------------------------------------------------+
```

### What DOs Are Good At

| Use Case | Why DOs Excel |
|----------|---------------|
| Coordination | Single-writer eliminates distributed locking |
| Scheduling | Alarms are durable — survive restarts |
| Rate limiting | Atomic counters with zero contention |
| Session state | Per-user state without a session store |
| Leader election | The DO *is* the leader — no election needed |
| Workflow state | Transactional storage for checkpoint/resume |
| Real-time collaboration | WebSocket + state in one place |

### What DOs Are Not Good At

DOs are not general-purpose databases. They are not designed for fan-out queries across thousands of entities. Each DO is independent — there is no cross-DO transaction or query. If you need to search across all your DOs, you need a separate index (D1, KV, or a dedicated indexing DO).

---

## The Meta-Pattern: Control Plane / Data Plane

Before diving into specific patterns, understand the architectural meta-pattern that all four share:

```
+----------------------------------------------------------+
|                    CONTROL PLANE (DO)                     |
|                                                          |
|  * Scheduling: when should work happen?                  |
|  * Routing: which worker/model/service handles it?       |
|  * Budget: how much resource to allocate?                |
|  * Decisions: what to do based on observed state?        |
|  * Adaptation: how to adjust based on results?           |
|                                                          |
|  State: metrics, config, decision history, budgets       |
|  Execution: single-threaded, alarm-driven                |
+--------------------+---------------------+---------------+
                     |  commands/config     |  results/telemetry
                     v                      ^
+----------------------------------------------------------+
|                   DATA PLANE (Workers)                    |
|                                                          |
|  * Processing: do the actual work                        |
|  * I/O: call APIs, read databases, transform data        |
|  * Parallelism: fan out across many Workers              |
|  * Stateless: get config from DO, return results         |
|                                                          |
|  Also: Workflows, Queues, R2, D1, external APIs          |
+----------------------------------------------------------+
```

**Why this separation matters:**

1. **Reasoning**: All decision logic is in one place with one thread. You can read a DO's code and understand every possible state transition. No distributed state to reason about.

2. **Scaling**: The data plane scales horizontally (more Workers, more Workflow instances). The control plane does not need to scale — it just coordinates.

3. **Resilience**: If the data plane fails (API timeout, bad response), the control plane observes the failure, adapts, and retries. The controller never crashes because a downstream call failed.

4. **Observability**: Every decision is made in the DO, so every decision can be logged with full context. You do not need distributed tracing to understand why something happened — the DO knows.

Every pattern in this article is a variation of this meta-pattern. The Adaptive Controller adapts batch sizes. The Budget Governor allocates resources. The Event Reactor makes decisions based on domain events. The Storage Sidecar provides state to a Workflow control plane. They all separate "deciding what to do" from "doing it."

---

## Pattern 1: Adaptive Controller

**A self-driving batch processor that learns from its own performance.**

This is the most common DO pattern in production. You have a recurring task — enriching records, syncing data, processing a queue — and you need it to run reliably without manual tuning. The DO manages the batch loop: it processes a batch, measures success, and adjusts its own parameters.

### The Problem

Fixed batch processing breaks in predictable ways:
- Batch too large -> downstream rate limits -> cascade failure
- Batch too small -> underutilized capacity -> slow throughput
- Fixed schedule -> no adaptation to load or API health
- External cron -> no state between runs -> can't learn

### The Solution

A Durable Object that:
1. Wakes itself on a schedule (alarm)
2. Processes a batch of work
3. Measures success rate and latency
4. Adjusts batch size based on observed performance
5. Repeats

```typescript
// enrichment-controller.ts — Adaptive batch processor DO

interface ControllerState {
  // Adaptive parameters
  batchSize: number;
  intervalMs: number;

  // Rolling metrics (last 100 results)
  metrics: {
    successRate: number;
    avgLatencyMs: number;
    consecutiveErrors: number;
    results: Array<{ success: boolean; latencyMs: number; timestamp: number }>;
  };

  // Operational state
  status: "running" | "paused" | "backoff";
  lastRunAt: number;
  totalProcessed: number;
  totalErrors: number;
}

const DEFAULT_STATE: ControllerState = {
  batchSize: 10,
  intervalMs: 60_000, // 1 minute
  metrics: {
    successRate: 1.0,
    avgLatencyMs: 0,
    consecutiveErrors: 0,
    results: [],
  },
  status: "running",
  lastRunAt: 0,
  totalProcessed: 0,
  totalErrors: 0,
};

export class EnrichmentController implements DurableObject {
  private state: DurableObjectState;
  private env: Env;
  private controller: ControllerState = DEFAULT_STATE;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
    this.env = env;

    // Restore state from storage on construction
    this.state.blockConcurrencyWhile(async () => {
      const stored = await this.state.storage.get<ControllerState>("controller");
      if (stored) {
        this.controller = stored;
      }
    });
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/start":
        return this.start();
      case "/pause":
        return this.pause();
      case "/status":
        return Response.json(this.controller);
      case "/reset":
        return this.reset();
      default:
        return new Response("Not found", { status: 404 });
    }
  }

  // -- Lifecycle -----------------------------------------------

  private async start(): Promise<Response> {
    this.controller.status = "running";
    await this.scheduleNext();
    await this.save();
    return Response.json({ status: "started", batchSize: this.controller.batchSize });
  }

  private async pause(): Promise<Response> {
    this.controller.status = "paused";
    await this.state.storage.deleteAlarm();
    await this.save();
    return Response.json({ status: "paused" });
  }

  private async reset(): Promise<Response> {
    this.controller = { ...DEFAULT_STATE };
    await this.state.storage.deleteAlarm();
    await this.save();
    return Response.json({ status: "reset" });
  }

  // -- The Heartbeat -------------------------------------------

  async alarm(): Promise<void> {
    if (this.controller.status !== "running") return;

    try {
      // 1. Fetch a batch of work
      const items = await this.fetchPendingItems(this.controller.batchSize);

      if (items.length === 0) {
        // Nothing to do — schedule next check and return
        await this.scheduleNext();
        return;
      }

      // 2. Process the batch
      const results = await this.enrichBatch(items);

      // 3. Record metrics
      this.recordResults(results);

      // 4. Adapt based on performance
      this.adapt();

      // 5. Update operational counters
      this.controller.lastRunAt = Date.now();
      this.controller.totalProcessed += results.filter((r) => r.success).length;
      this.controller.totalErrors += results.filter((r) => !r.success).length;
    } catch (err) {
      // Entire batch failed — record as consecutive error
      this.controller.metrics.consecutiveErrors++;
      this.adapt();
      console.error("Controller alarm error:", err);
    }

    // Always schedule next run (even after errors)
    await this.scheduleNext();
    await this.save();
  }

  // -- Batch Processing ----------------------------------------

  private async fetchPendingItems(limit: number): Promise<WorkItem[]> {
    // Fetch from your data source — D1, KV, external API, etc.
    const response = await fetch(
      `${this.env.API_BASE}/items/pending?limit=${limit}`,
      { headers: { Authorization: `Bearer ${this.env.API_KEY}` } }
    );
    if (!response.ok) throw new Error(`Fetch failed: ${response.status}`);
    return response.json();
  }

  private async enrichBatch(
    items: WorkItem[]
  ): Promise<Array<{ success: boolean; latencyMs: number }>> {
    // Process items in parallel with individual error handling
    return Promise.all(
      items.map(async (item) => {
        const start = Date.now();
        try {
          await this.enrichItem(item);
          return { success: true, latencyMs: Date.now() - start };
        } catch {
          return { success: false, latencyMs: Date.now() - start };
        }
      })
    );
  }

  private async enrichItem(item: WorkItem): Promise<void> {
    // Your enrichment logic here — call an API, transform data, etc.
    const response = await fetch(`${this.env.ENRICHMENT_API}/enrich`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(item),
    });
    if (!response.ok) throw new Error(`Enrichment failed: ${response.status}`);

    // Write result back
    await fetch(`${this.env.API_BASE}/items/${item.id}`, {
      method: "PATCH",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${this.env.API_KEY}`,
      },
      body: JSON.stringify({ enriched: true, data: await response.json() }),
    });
  }

  // -- Metrics -------------------------------------------------

  private recordResults(
    results: Array<{ success: boolean; latencyMs: number }>
  ): void {
    const now = Date.now();

    // Add to rolling window
    for (const result of results) {
      this.controller.metrics.results.push({
        ...result,
        timestamp: now,
      });
    }

    // Keep only last 100 results
    if (this.controller.metrics.results.length > 100) {
      this.controller.metrics.results =
        this.controller.metrics.results.slice(-100);
    }

    // Recalculate aggregates
    const recent = this.controller.metrics.results;
    const successes = recent.filter((r) => r.success).length;
    this.controller.metrics.successRate = successes / recent.length;
    this.controller.metrics.avgLatencyMs =
      recent.reduce((sum, r) => sum + r.latencyMs, 0) / recent.length;

    // Track consecutive errors
    const allFailed = results.every((r) => !r.success);
    if (allFailed) {
      this.controller.metrics.consecutiveErrors++;
    } else {
      this.controller.metrics.consecutiveErrors = 0;
    }
  }

  // -- Adaptation ----------------------------------------------

  private adapt(): void {
    const { successRate, consecutiveErrors } = this.controller.metrics;

    if (consecutiveErrors >= 5) {
      // Circuit breaker: back off hard
      this.controller.batchSize = 5;
      this.controller.intervalMs = Math.min(
        this.controller.intervalMs * 2,
        300_000 // cap at 5 minutes
      );
      this.controller.status = "backoff";
      console.log(
        `[adapt] Circuit break: ${consecutiveErrors} consecutive errors. ` +
        `Batch=5, interval=${this.controller.intervalMs}ms`
      );
      return;
    }

    // Resume from backoff if things are improving
    if (this.controller.status === "backoff" && successRate > 0.8) {
      this.controller.status = "running";
      this.controller.intervalMs = 60_000;
      console.log("[adapt] Recovered from backoff");
    }

    if (successRate > 0.95) {
      // Excellent — increase batch size
      this.controller.batchSize = Math.min(
        Math.ceil(this.controller.batchSize * 1.5),
        50 // cap
      );
      console.log(
        `[adapt] Success rate ${(successRate * 100).toFixed(1)}% ` +
        `-> batch size ${this.controller.batchSize}`
      );
    } else if (successRate >= 0.8) {
      // Good — maintain current settings
      console.log(
        `[adapt] Success rate ${(successRate * 100).toFixed(1)}% ` +
        `-> maintaining batch size ${this.controller.batchSize}`
      );
    } else if (successRate >= 0.5) {
      // Degraded — reduce batch size
      this.controller.batchSize = Math.max(
        Math.floor(this.controller.batchSize * 0.5),
        5 // floor
      );
      console.log(
        `[adapt] Success rate ${(successRate * 100).toFixed(1)}% ` +
        `-> reduced batch to ${this.controller.batchSize}`
      );
    } else {
      // Critical — minimum batch + backoff
      this.controller.batchSize = 5;
      this.controller.intervalMs = Math.min(
        this.controller.intervalMs * 1.5,
        300_000
      );
      console.log(
        `[adapt] Success rate ${(successRate * 100).toFixed(1)}% ` +
        `-> minimum batch, interval ${this.controller.intervalMs}ms`
      );
    }
  }

  // -- Scheduling ----------------------------------------------

  private async scheduleNext(): Promise<void> {
    const next = Date.now() + this.controller.intervalMs;
    await this.state.storage.setAlarm(next);
  }

  // -- Persistence ---------------------------------------------

  private async save(): Promise<void> {
    await this.state.storage.put("controller", this.controller);
  }
}

interface WorkItem {
  id: string;
  [key: string]: unknown;
}

interface Env {
  API_BASE: string;
  API_KEY: string;
  ENRICHMENT_API: string;
}
```

### How It Works

```
                    +---------------+
                    |   alarm()     | <-- DO wakes itself
                    +-------+-------+
                            |
                    +-------v-------+
                    | fetchPending  | -- get batch of work
                    +-------+-------+
                            |
                    +-------v-------+
                    | enrichBatch   | -- parallel processing
                    +-------+-------+
                            |
                    +-------v-------+
                    | recordResults | -- rolling window metrics
                    +-------+-------+
                            |
                    +-------v-------+
                    |   adapt()     | -- adjust batch size
                    +-------+-------+
                            |
                    +-------v-------+
                    | scheduleNext  | -- set next alarm
                    +---------------+
```

### Key Insight

The DO **learns from its own performance**. After every batch, it observes the success rate and adjusts. If the downstream API is healthy, batches grow. If it is struggling, batches shrink. If it is failing, the controller backs off. No external monitoring needed — the controller is self-healing.

This is fundamentally different from a cron job with fixed parameters. A cron job does the same thing regardless of conditions. An Adaptive Controller responds to reality.

### Adaptation Thresholds

| Success Rate | Action | Reasoning |
|-------------|--------|-----------|
| > 95% | Increase batch (x1.5, cap 50) | Downstream is healthy, maximize throughput |
| 80-95% | Maintain | Good enough, do not rock the boat |
| 50-80% | Decrease batch (x0.5, floor 5) | Downstream struggling, reduce pressure |
| < 50% | Minimum batch + backoff | Near-failure, protect downstream |
| 5 consecutive failures | Circuit break | Something is fundamentally wrong |

---

## Pattern 2: Storage Sidecar

**Stateful storage companion for Cloudflare Workflows.**

### The Problem

Cloudflare Workflows are excellent for durable execution — steps retry, state persists, and the runtime handles failures. But they have a hard constraint: **1 MiB total state limit**. If your Workflow accumulates data across steps (API responses, intermediate results, large payloads), you hit this wall fast.

You also cannot share state between Workflow instances without an external store. Two Workflows processing the same entity have no way to coordinate through Workflow state alone.

### The Solution

Pair each Workflow (or each entity) with a Durable Object that acts as a **storage sidecar**. The Workflow orchestrates steps. The DO holds state. The Workflow reads and writes to the DO instead of accumulating step state.

```typescript
// storage-sidecar.ts — Stateful companion for Workflows

export class StorageSidecar implements DurableObject {
  private state: DurableObjectState;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const key = url.searchParams.get("key");

    switch (request.method) {
      case "GET": {
        if (key) {
          // Get single key
          const value = await this.state.storage.get(key);
          if (value === undefined) {
            return new Response(null, { status: 404 });
          }
          return Response.json({ key, value });
        }
        // Get all state
        const all = await this.state.storage.list();
        const entries = Object.fromEntries(all);
        return Response.json(entries);
      }

      case "PUT": {
        if (!key) {
          return new Response("Missing key parameter", { status: 400 });
        }
        const value = await request.json();
        await this.state.storage.put(key, value);
        return Response.json({ key, written: true });
      }

      case "POST": {
        // Append to an array stored at `key`
        if (!key) {
          return new Response("Missing key parameter", { status: 400 });
        }
        const item = await request.json();
        const existing =
          (await this.state.storage.get<unknown[]>(key)) || [];
        existing.push(item);
        await this.state.storage.put(key, existing);
        return Response.json({ key, length: existing.length });
      }

      case "DELETE": {
        if (key) {
          await this.state.storage.delete(key);
          return Response.json({ key, deleted: true });
        }
        // Delete all
        await this.state.storage.deleteAll();
        return Response.json({ cleared: true });
      }

      default:
        return new Response("Method not allowed", { status: 405 });
    }
  }
}

// -- Using the sidecar from a Workflow -----------------------

interface WorkflowParams {
  entityId: string;
  sources: Array<{ id: string; url: string }>;
}

interface Env {
  STORAGE_SIDECAR: DurableObjectNamespace;
  PUBLISH_API: string;
}

export class ProcessingWorkflow extends WorkflowEntrypoint<Env, WorkflowParams> {
  async run(event: WorkflowEvent<WorkflowParams>, step: WorkflowStep) {
    const { entityId, sources } = event.payload;

    // Get a handle to this entity's sidecar DO
    const sidecarId = this.env.STORAGE_SIDECAR.idFromName(entityId);
    const sidecar = this.env.STORAGE_SIDECAR.get(sidecarId);

    // Step 1: Initialize sidecar state
    await step.do("init-sidecar", async () => {
      await sidecar.fetch(
        new Request("https://sidecar/?key=status", {
          method: "PUT",
          body: JSON.stringify({
            phase: "started",
            startedAt: Date.now(),
          }),
        })
      );
    });

    // Step 2: Fetch data from multiple sources
    for (const source of sources) {
      await step.do(`fetch-${source.id}`, async () => {
        const response = await fetch(source.url);
        const data = await response.json();

        // Append to sidecar instead of accumulating in step state
        await sidecar.fetch(
          new Request("https://sidecar/?key=rawData", {
            method: "POST",
            body: JSON.stringify({
              source: source.id,
              data,
              fetchedAt: Date.now(),
            }),
          })
        );

        // Return minimal data to Workflow state (stays under 1 MiB)
        return {
          source: source.id,
          status: "fetched",
          records: (data as unknown[]).length,
        };
      });
    }

    // Step 3: Process — read all data from sidecar
    const result = await step.do("process", async () => {
      const response = await sidecar.fetch(
        new Request("https://sidecar/?key=rawData")
      );
      const { value: allData } = (await response.json()) as {
        value: unknown[];
      };

      // Process the full dataset (which could be many MiB)
      const processed = transform(allData);

      // Store result in sidecar
      await sidecar.fetch(
        new Request("https://sidecar/?key=result", {
          method: "PUT",
          body: JSON.stringify(processed),
        })
      );

      return { recordsProcessed: processed.length };
    });

    // Step 4: Publish result
    await step.do("publish", async () => {
      const response = await sidecar.fetch(
        new Request("https://sidecar/?key=result")
      );
      const { value: finalResult } = (await response.json()) as {
        value: unknown;
      };

      await fetch(`${this.env.PUBLISH_API}/publish`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(finalResult),
      });

      // Clean up sidecar
      await sidecar.fetch(
        new Request("https://sidecar/", { method: "DELETE" })
      );
    });

    return { entityId, status: "completed" };
  }
}

function transform(data: unknown[]): unknown[] {
  // Your processing logic here
  return data;
}
```

### Why Not Just Use D1 or KV?

You could — but the sidecar pattern has specific advantages:

| Feature | Storage Sidecar (DO) | D1 | KV |
|---------|----------------------|----|----|
| Consistency | Strong (single-writer) | Strong | Eventually consistent |
| Latency | Microseconds (co-located) | ~1-5ms | ~10-50ms |
| Schema | Schemaless | SQL schema | Key-value |
| Lifetime | Tied to entity | Global | Global |
| Cleanup | `deleteAll()` removes everything | Manual DELETE | Manual DELETE |
| Concurrency | Serialized automatically | Row-level locks | Last-writer-wins |

The sidecar is ideal when state is **temporary** (lives for the duration of a workflow), **entity-scoped** (belongs to one thing), and **consistency-sensitive** (multiple steps must see the same state).

### Key Insight

The Workflow handles **orchestration** (what steps to run, in what order, with what retry policy). The DO handles **state** (accumulating data across steps, providing a consistent view). This separation lets each primitive do what it is best at.

### Typed RPC Variant

If you prefer type safety over HTTP, use Durable Object RPC (available since 2024):

```typescript
export class StorageSidecarRPC extends DurableObject {
  async get(key: string): Promise<unknown> {
    return this.ctx.storage.get(key);
  }

  async set(key: string, value: unknown): Promise<void> {
    await this.ctx.storage.put(key, value);
  }

  async append(key: string, item: unknown): Promise<number> {
    const existing =
      (await this.ctx.storage.get<unknown[]>(key)) || [];
    existing.push(item);
    await this.ctx.storage.put(key, existing);
    return existing.length;
  }

  async clear(): Promise<void> {
    await this.ctx.storage.deleteAll();
  }
}

// Usage from a Worker or Workflow — fully typed, no HTTP serialization:
//
// const sidecar = env.STORAGE_SIDECAR.get(id);
// await sidecar.set("key", { foo: "bar" });
// const value = await sidecar.get("key");
```

---

## Pattern 3: Budget Governor

**A resource allocation controller that distributes budget across competing processors.**

### The Problem

You have multiple processing tasks that share a limited budget — API calls with rate limits, dollars allocated to LLM inference, time windows for batch operations. Each processor type has different costs, different priorities, and different performance characteristics. You need something to allocate resources intelligently.

### The Solution

A Durable Object that acts as a **budget governor**: it receives a total budget per cycle, allocates it across processor types by weight, routes to appropriate model tiers, and tracks spend per processor.

```typescript
// budget-governor.ts — Resource allocation controller

interface ProcessorConfig {
  id: string;
  name: string;
  weight: number;        // Budget weight (0-1, all weights sum to 1)
  phase: number;         // Execution order (lower = earlier)
  modelTier: "premium" | "standard" | "economy";
  enabled: boolean;
}

interface BudgetAllocation {
  allocated: number;
  spent: number;
  items: number;
  errors: number;
  avgLatencyMs: number;
}

interface BudgetCycle {
  cycleId: string;
  totalBudget: number;   // Total budget units for this cycle
  startedAt: number;
  allocations: Record<string, BudgetAllocation>;
}

interface GovernorState {
  processors: ProcessorConfig[];
  currentCycle: BudgetCycle | null;
  cycleHistory: Array<{
    cycleId: string;
    totalBudget: number;
    totalSpent: number;
    completedAt: number;
  }>;
  modelRouting: Record<
    string,
    {
      model: string;
      costPerCall: number;
      maxConcurrency: number;
    }
  >;
}

export class BudgetGovernor implements DurableObject {
  private state: DurableObjectState;
  private env: Env;
  private governor: GovernorState;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
    this.env = env;
    this.governor = {
      processors: [],
      currentCycle: null,
      cycleHistory: [],
      modelRouting: {
        premium: {
          model: "gpt-4o",
          costPerCall: 0.03,
          maxConcurrency: 5,
        },
        standard: {
          model: "gpt-4o-mini",
          costPerCall: 0.005,
          maxConcurrency: 20,
        },
        economy: {
          model: "gemini-2.0-flash",
          costPerCall: 0.001,
          maxConcurrency: 50,
        },
      },
    };

    this.state.blockConcurrencyWhile(async () => {
      const stored =
        await this.state.storage.get<GovernorState>("governor");
      if (stored) this.governor = stored;
    });
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/configure":
        return this.configure(await request.json());
      case "/start-cycle":
        return this.startCycle(await request.json());
      case "/execute":
        return this.executeCycle();
      case "/report-usage":
        return this.reportUsage(await request.json());
      case "/status":
        return Response.json(this.governor);
      default:
        return new Response("Not found", { status: 404 });
    }
  }

  // -- Configuration -------------------------------------------

  private async configure(
    config: { processors: ProcessorConfig[] }
  ): Promise<Response> {
    // Validate weights sum to 1
    const totalWeight = config.processors
      .filter((p) => p.enabled)
      .reduce((sum, p) => sum + p.weight, 0);

    if (Math.abs(totalWeight - 1.0) > 0.01) {
      return Response.json(
        {
          error: `Weights sum to ${totalWeight}, must equal 1.0`,
        },
        { status: 400 }
      );
    }

    this.governor.processors = config.processors;
    await this.save();
    return Response.json({
      configured: true,
      processors: config.processors.length,
    });
  }

  // -- Cycle Management ----------------------------------------

  private async startCycle(
    params: { totalBudget: number }
  ): Promise<Response> {
    const cycleId = `cycle_${Date.now()}`;

    // Allocate budget by weight
    const allocations: BudgetCycle["allocations"] = {};
    for (const proc of this.governor.processors.filter(
      (p) => p.enabled
    )) {
      allocations[proc.id] = {
        allocated: params.totalBudget * proc.weight,
        spent: 0,
        items: 0,
        errors: 0,
        avgLatencyMs: 0,
      };
    }

    this.governor.currentCycle = {
      cycleId,
      totalBudget: params.totalBudget,
      startedAt: Date.now(),
      allocations,
    };

    await this.save();

    return Response.json({
      cycleId,
      allocations: Object.entries(allocations).map(
        ([id, a]) => ({
          processor: id,
          allocated: a.allocated,
        })
      ),
    });
  }

  // -- Phase-Ordered Execution ---------------------------------

  private async executeCycle(): Promise<Response> {
    const cycle = this.governor.currentCycle;
    if (!cycle) {
      return Response.json(
        { error: "No active cycle" },
        { status: 400 }
      );
    }

    // Group processors by phase, execute in order
    const phases = new Map<number, ProcessorConfig[]>();
    for (const proc of this.governor.processors.filter(
      (p) => p.enabled
    )) {
      const group = phases.get(proc.phase) || [];
      group.push(proc);
      phases.set(proc.phase, group);
    }

    const results: Record<string, unknown> = {};

    // Execute phases in order (processors within a phase run in parallel)
    const sortedPhases = [...phases.entries()].sort(
      ([a], [b]) => a - b
    );

    for (const [phase, processors] of sortedPhases) {
      const phaseResults = await Promise.all(
        processors.map(async (proc) => {
          const allocation = cycle.allocations[proc.id];
          if (
            !allocation ||
            allocation.spent >= allocation.allocated
          ) {
            return {
              processor: proc.id,
              skipped: true,
              reason: "budget exhausted",
            };
          }

          const routing =
            this.governor.modelRouting[proc.modelTier];
          const remainingBudget =
            allocation.allocated - allocation.spent;
          const maxItems = Math.floor(
            remainingBudget / routing.costPerCall
          );

          // Dispatch to data plane Worker
          const response = await fetch(
            `${this.env.PROCESSOR_API}/process`,
            {
              method: "POST",
              headers: {
                "Content-Type": "application/json",
              },
              body: JSON.stringify({
                processorId: proc.id,
                model: routing.model,
                maxItems: Math.min(
                  maxItems,
                  routing.maxConcurrency
                ),
                cycleId: cycle.cycleId,
              }),
            }
          );

          return {
            processor: proc.id,
            phase,
            ...((await response.json()) as object),
          };
        })
      );

      for (const result of phaseResults) {
        results[result.processor] = result;
      }
    }

    // Archive cycle
    this.governor.cycleHistory.push({
      cycleId: cycle.cycleId,
      totalBudget: cycle.totalBudget,
      totalSpent: Object.values(cycle.allocations).reduce(
        (sum, a) => sum + a.spent,
        0
      ),
      completedAt: Date.now(),
    });

    // Keep last 50 cycles
    if (this.governor.cycleHistory.length > 50) {
      this.governor.cycleHistory =
        this.governor.cycleHistory.slice(-50);
    }

    this.governor.currentCycle = null;
    await this.save();

    return Response.json({ cycleId: cycle.cycleId, results });
  }

  // -- Usage Reporting -----------------------------------------

  private async reportUsage(report: {
    processorId: string;
    cost: number;
    items: number;
    errors: number;
    avgLatencyMs: number;
  }): Promise<Response> {
    const cycle = this.governor.currentCycle;
    if (!cycle) {
      return Response.json(
        { error: "No active cycle" },
        { status: 400 }
      );
    }

    const allocation = cycle.allocations[report.processorId];
    if (!allocation) {
      return Response.json(
        { error: "Unknown processor" },
        { status: 400 }
      );
    }

    allocation.spent += report.cost;
    allocation.items += report.items;
    allocation.errors += report.errors;
    // Running average
    const prevItems = allocation.items - report.items;
    allocation.avgLatencyMs =
      (allocation.avgLatencyMs * prevItems +
        report.avgLatencyMs * report.items) /
      allocation.items;

    const overBudget = allocation.spent > allocation.allocated;

    await this.save();

    return Response.json({
      processorId: report.processorId,
      remaining: allocation.allocated - allocation.spent,
      overBudget,
    });
  }

  private async save(): Promise<void> {
    await this.state.storage.put("governor", this.governor);
  }
}

interface Env {
  PROCESSOR_API: string;
}
```

### Budget Allocation Flow

```
+------------------------------------------------------------------+
|                     Budget Governor DO                            |
|                                                                  |
|  Total Budget: $1.00/cycle                                       |
|                                                                  |
|  +---------------+  +---------------+  +---------------+         |
|  | Keyword       |  | Competitor    |  | Backlink      |         |
|  | Enrichment    |  | Analysis      |  | Discovery     |         |
|  |               |  |               |  |               |         |
|  | Weight: 0.5   |  | Weight: 0.3   |  | Weight: 0.2   |         |
|  | Phase: 1      |  | Phase: 1      |  | Phase: 2      |         |
|  | Tier: std     |  | Tier: prem    |  | Tier: econ    |         |
|  | Budget: $0.50 |  | Budget: $0.30 |  | Budget: $0.20 |         |
|  +-------+-------+  +-------+-------+  +-------+-------+         |
|          |                  |                   |                 |
+----------+------------------+-------------------+-----------------+
           |                  |                   |
           v                  v                   v
    +-----------+      +-----------+       +-----------+
    |gpt-4o-mini|      |   gpt-4o  |       | gemini    |
    | 100 items |      |  10 items |       | 200 items |
    |$0.005/call|      |$0.03/call |       |$0.001/call|
    +-----------+      +-----------+       +-----------+
```

### Key Insight

The DO is a **resource allocator**, not a task executor. It never calls an LLM or processes a record directly. It decides *how much* each processor gets, *which model tier* to use, and *in what order* to run them. The actual work happens in stateless Workers that report back their usage.

This means you can change allocation strategies (more budget to keyword enrichment, less to backlinks) without touching any processing code. The governor's configuration is the single source of truth for resource distribution.

---

## Pattern 4: Event Reactor

**An autonomous agent that responds to domain events and emits commands.**

This pattern uses the [Cloudflare Agents SDK](https://agents.cloudflare.com) to build a DO that acts as a **domain-level decision maker**. It receives events (research completed, content published, metric threshold crossed), maintains decision context, and emits commands to other services.

### The Problem

Distributed systems generate events. Something has to watch those events and decide what to do next. Traditional approaches:

- **Event-driven microservice**: Stateless, so it re-fetches context on every event. Expensive and slow.
- **State machine in a database**: Works, but the logic is spread across application code and database queries. Hard to reason about.
- **Workflow engine**: Good for linear flows, but awkward for reactive, event-driven logic.

### The Solution

A Durable Object (via the Agents SDK) that maintains a persistent view of the world and reacts to events by emitting commands.

```typescript
// brand-agent.ts — Event-driven decision maker using Agents SDK

import { Agent } from "agents";

interface BrandState {
  // Identity
  brandId: string;
  brandName: string;

  // Decision context — what we know about the world
  observations: {
    lastResearchAt: number | null;
    lastPublishAt: number | null;
    contentCount: number;
    avgEngagement: number;
    topKeywords: string[];
    competitorMoves: Array<{
      competitor: string;
      action: string;
      observedAt: number;
    }>;
  };

  // Pending decisions
  pending: Array<{
    id: string;
    type: string;
    payload: unknown;
    createdAt: number;
  }>;

  // Decision log — why we did what we did
  decisions: Array<{
    id: string;
    type: string;
    reason: string;
    result: "pending" | "success" | "failed";
    timestamp: number;
  }>;

  // Cycle tracking
  lastCycleAt: number;
  cycleIntervalMs: number;
}

const DEFAULT_BRAND_STATE: BrandState = {
  brandId: "",
  brandName: "",
  observations: {
    lastResearchAt: null,
    lastPublishAt: null,
    contentCount: 0,
    avgEngagement: 0,
    topKeywords: [],
    competitorMoves: [],
  },
  pending: [],
  decisions: [],
  lastCycleAt: 0,
  cycleIntervalMs: 3600_000, // 1 hour
};

interface DomainEvent {
  type: string;
  payload: unknown;
  timestamp: number;
}

interface Command {
  type: string;
  payload: unknown;
}

interface Env {
  COMMAND_QUEUE: Queue;
}

export class BrandAgent extends Agent<Env, BrandState> {
  // -- Initialization ------------------------------------------

  initialState: BrandState = DEFAULT_BRAND_STATE;

  async onStart(): Promise<void> {
    // Schedule periodic decision cycle
    this.schedule("decision-cycle", "*/60 * * * *"); // every hour
  }

  // -- Event Handlers ------------------------------------------

  async onRequest(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/event":
        return this.handleEvent(await request.json());
      case "/state":
        return Response.json(this.state);
      case "/decisions":
        return Response.json(this.state.decisions.slice(-20));
      default:
        return new Response("Not found", { status: 404 });
    }
  }

  private async handleEvent(
    event: DomainEvent
  ): Promise<Response> {
    console.log(`[${this.state.brandId}] Event: ${event.type}`);

    switch (event.type) {
      case "research.completed":
        return this.onResearchCompleted(event);
      case "content.published":
        return this.onContentPublished(event);
      case "engagement.report":
        return this.onEngagementReport(event);
      case "competitor.detected":
        return this.onCompetitorDetected(event);
      default:
        return Response.json({
          accepted: true,
          action: "ignored",
        });
    }
  }

  // -- Event: Research Completed -------------------------------

  private async onResearchCompleted(
    event: DomainEvent
  ): Promise<Response> {
    const { keywords, opportunities } = event.payload as {
      keywords: string[];
      gaps: string[];
      opportunities: string[];
    };

    // Update observations
    this.setState({
      ...this.state,
      observations: {
        ...this.state.observations,
        lastResearchAt: Date.now(),
        topKeywords: keywords.slice(0, 20),
      },
    });

    // Decision: should we create content for these opportunities?
    const commands: Command[] = [];

    for (const opportunity of opportunities) {
      if (this.shouldCreateContent(opportunity)) {
        const command: Command = {
          type: "content.create",
          payload: {
            brandId: this.state.brandId,
            keyword: opportunity,
            priority: this.calculatePriority(opportunity),
          },
        };
        commands.push(command);

        this.logDecision(
          "create-content",
          `Research found opportunity "${opportunity}" — creating content`,
          "pending"
        );
      }
    }

    // Emit commands to queue
    if (commands.length > 0) {
      await this.emitCommands(commands);
    }

    return Response.json({
      accepted: true,
      commands: commands.length,
    });
  }

  // -- Event: Content Published --------------------------------

  private async onContentPublished(
    event: DomainEvent
  ): Promise<Response> {
    const { contentId, keyword } = event.payload as {
      contentId: string;
      keyword: string;
    };

    this.setState({
      ...this.state,
      observations: {
        ...this.state.observations,
        lastPublishAt: Date.now(),
        contentCount: this.state.observations.contentCount + 1,
      },
    });

    // Decision: should we promote this content on social?
    if (this.shouldPromote(keyword)) {
      await this.emitCommands([
        {
          type: "social.schedule",
          payload: {
            brandId: this.state.brandId,
            contentId,
            platforms: ["twitter", "linkedin"],
            timing: "optimal",
          },
        },
      ]);

      this.logDecision(
        "promote-content",
        `Content "${keyword}" published — scheduling social promotion`,
        "pending"
      );
    }

    return Response.json({ accepted: true });
  }

  // -- Event: Engagement Report --------------------------------

  private async onEngagementReport(
    event: DomainEvent
  ): Promise<Response> {
    const { avgEngagement, underPerforming } = event.payload as {
      avgEngagement: number;
      topPerforming: string[];
      underPerforming: string[];
    };

    this.setState({
      ...this.state,
      observations: {
        ...this.state.observations,
        avgEngagement,
      },
    });

    // Decision: adjust strategy based on engagement
    if (avgEngagement < 0.02) {
      // Low engagement — trigger research to find better topics
      await this.emitCommands([
        {
          type: "research.request",
          payload: {
            brandId: this.state.brandId,
            focus: "engagement-optimization",
            context: { underPerforming },
          },
        },
      ]);

      this.logDecision(
        "engagement-pivot",
        `Engagement at ${(avgEngagement * 100).toFixed(1)}% — triggering research pivot`,
        "pending"
      );
    }

    return Response.json({ accepted: true });
  }

  // -- Event: Competitor Detected ------------------------------

  private async onCompetitorDetected(
    event: DomainEvent
  ): Promise<Response> {
    const { competitor, action, details } = event.payload as {
      competitor: string;
      action: string;
      details: unknown;
    };

    // Record observation
    const moves = [
      ...this.state.observations.competitorMoves,
      { competitor, action, observedAt: Date.now() },
    ].slice(-50); // Keep last 50

    this.setState({
      ...this.state,
      observations: {
        ...this.state.observations,
        competitorMoves: moves,
      },
    });

    // Decision: react to competitor move?
    if (
      action === "new-content" &&
      this.isHighPriorityCompetitor(competitor)
    ) {
      await this.emitCommands([
        {
          type: "research.request",
          payload: {
            brandId: this.state.brandId,
            focus: "competitive-response",
            context: { competitor, action, details },
          },
        },
      ]);

      this.logDecision(
        "competitive-response",
        `Competitor "${competitor}" published new content — investigating`,
        "pending"
      );
    }

    return Response.json({ accepted: true });
  }

  // -- Scheduled: Decision Cycle -------------------------------

  async onSchedule(scheduleName: string): Promise<void> {
    if (scheduleName !== "decision-cycle") return;

    const now = Date.now();
    const hoursSinceResearch =
      this.state.observations.lastResearchAt
        ? (now - this.state.observations.lastResearchAt) / 3600_000
        : Infinity;
    const hoursSincePublish =
      this.state.observations.lastPublishAt
        ? (now - this.state.observations.lastPublishAt) / 3600_000
        : Infinity;

    // If no research in 24 hours, request it
    if (hoursSinceResearch > 24) {
      await this.emitCommands([
        {
          type: "research.request",
          payload: {
            brandId: this.state.brandId,
            focus: "routine-discovery",
          },
        },
      ]);
      this.logDecision(
        "routine-research",
        `No research in ${hoursSinceResearch.toFixed(0)}h — requesting routine discovery`,
        "pending"
      );
    }

    // If no publishing in 48 hours, something is stuck
    if (hoursSincePublish > 48) {
      this.logDecision(
        "pipeline-stall",
        `No publishing in ${hoursSincePublish.toFixed(0)}h — pipeline may be stalled`,
        "pending"
      );
    }

    this.setState({
      ...this.state,
      lastCycleAt: now,
    });
  }

  // -- Decision Helpers ----------------------------------------

  private shouldCreateContent(_opportunity: string): boolean {
    // Simple heuristic — in production, this could consult
    // historical performance, budget, content calendar, etc.
    const recentContent = this.state.decisions
      .filter((d) => d.type === "create-content")
      .filter((d) => Date.now() - d.timestamp < 86400_000);
    return recentContent.length < 5; // Max 5 pieces per day
  }

  private shouldPromote(keyword: string): boolean {
    return this.state.observations.topKeywords.includes(keyword);
  }

  private calculatePriority(opportunity: string): number {
    const isTopKeyword =
      this.state.observations.topKeywords.includes(opportunity);
    return isTopKeyword ? 1 : 2;
  }

  private isHighPriorityCompetitor(competitor: string): boolean {
    const recentMoves = this.state.observations.competitorMoves
      .filter((m) => m.competitor === competitor)
      .filter(
        (m) => Date.now() - m.observedAt < 7 * 86400_000
      );
    return recentMoves.length >= 3; // Active if 3+ moves in a week
  }

  // -- Command Emission ----------------------------------------

  private async emitCommands(commands: Command[]): Promise<void> {
    for (const command of commands) {
      await this.env.COMMAND_QUEUE.send({
        ...command,
        brandId: this.state.brandId,
        emittedAt: Date.now(),
      });
    }
  }

  private logDecision(
    type: string,
    reason: string,
    result: "pending" | "success" | "failed"
  ): void {
    const decisions = [
      ...this.state.decisions,
      {
        id: `dec_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`,
        type,
        reason,
        result,
        timestamp: Date.now(),
      },
    ].slice(-100); // Keep last 100

    this.setState({
      ...this.state,
      decisions,
    });
  }
}
```

### Event Flow

```
External Events              Brand Agent DO             Command Queue
                             +-----------------+
research.completed ------->  | Update state    |
                             | Evaluate opps   |-------> content.create
                             | Log decision    |-------> content.create
                             +-----------------+

content.published  ------->  +-----------------+
                             | Update counters |
                             | Check promotion |-------> social.schedule
                             | Log decision    |
                             +-----------------+

engagement.report  ------->  +-----------------+
                             | Update metrics  |
                             | Check thresholds|-------> research.request
                             | Log decision    |
                             +-----------------+

          alarm()  ------->  +-----------------+
   (decision-cycle)          | Check staleness |
                             | Detect stalls   |-------> research.request
                             | Log decisions   |
                             +-----------------+
```

### Key Insight

The DO **makes decisions**; queues **do the work**. The BrandAgent never calls an API, never generates content, never posts to social media. It observes events, updates its internal model of the world, and emits commands. This makes the decision logic easy to test (feed events, check commands) and easy to reason about (all decisions are in one file with full state context).

The decision log is particularly powerful. Every command the agent emits has a recorded reason. When you debug why something happened, you read the decision log — not distributed traces across five services.

---

## Layer Guide: Raw DO to Server to Agent

Durable Objects can be used at three levels of abstraction. Each layer adds capabilities and opinions.

### Layer 0: Raw Durable Object

The fundamental primitive. You implement `fetch()` and optionally `alarm()`. You manage storage directly.

```typescript
export class RawCounter implements DurableObject {
  private state: DurableObjectState;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === "/increment") {
      const current =
        (await this.state.storage.get<number>("count")) || 0;
      const next = current + 1;
      await this.state.storage.put("count", next);
      return Response.json({ count: next });
    }

    if (url.pathname === "/get") {
      const count =
        (await this.state.storage.get<number>("count")) || 0;
      return Response.json({ count });
    }

    return new Response("Not found", { status: 404 });
  }
}
```

**Use when:**
- Simple state + simple logic
- You need maximum control
- No framework dependencies desired
- Learning DOs for the first time

### Layer 1: Server Framework (Hono on DO)

Add a router framework for cleaner HTTP handling. Hono works well because it is lightweight and designed for edge runtimes.

```typescript
import { Hono } from "hono";

export class ServerCounter implements DurableObject {
  private state: DurableObjectState;
  private app: Hono;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;

    this.app = new Hono();

    this.app.post("/increment", async (c) => {
      const current =
        (await this.state.storage.get<number>("count")) || 0;
      const next = current + 1;
      await this.state.storage.put("count", next);
      return c.json({ count: next });
    });

    this.app.get("/count", async (c) => {
      const count =
        (await this.state.storage.get<number>("count")) || 0;
      return c.json({ count });
    });

    this.app.post("/reset", async (c) => {
      await this.state.storage.put("count", 0);
      return c.json({ count: 0 });
    });
  }

  fetch(request: Request): Promise<Response> {
    return this.app.fetch(request);
  }
}
```

Alternatively, use **Durable Object RPC** (no HTTP overhead):

```typescript
export class RPCCounter extends DurableObject {
  async increment(): Promise<number> {
    const current =
      (await this.ctx.storage.get<number>("count")) || 0;
    const next = current + 1;
    await this.ctx.storage.put("count", next);
    return next;
  }

  async getCount(): Promise<number> {
    return (await this.ctx.storage.get<number>("count")) || 0;
  }

  async reset(): Promise<void> {
    await this.ctx.storage.put("count", 0);
  }
}

// Caller (from a Worker):
// const id = env.COUNTER.idFromName("my-counter");
// const counter = env.COUNTER.get(id);
// const value = await counter.increment(); // Typed, no HTTP
```

**Use when:**
- Multiple endpoints with different methods
- You want middleware (auth, logging, CORS)
- Team is familiar with Express/Hono patterns
- Need typed RPC between Workers and DOs

### Layer 2: Agents SDK

The [Cloudflare Agents SDK](https://agents.cloudflare.com) wraps DOs with higher-level primitives: managed state (`setState`/`this.state`), scheduled tasks (`schedule`), WebSocket management, and SQLite storage.

```typescript
import { Agent } from "agents";

interface CounterState {
  count: number;
  lastIncrementAt: number;
  history: Array<{ value: number; timestamp: number }>;
}

export class AgentCounter extends Agent<Env, CounterState> {
  initialState: CounterState = {
    count: 0,
    lastIncrementAt: 0,
    history: [],
  };

  async onRequest(request: Request): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === "/increment") {
      const next = this.state.count + 1;
      this.setState({
        count: next,
        lastIncrementAt: Date.now(),
        history: [
          ...this.state.history.slice(-99),
          {
            value: next,
            timestamp: Date.now(),
          },
        ],
      });
      return Response.json({ count: next });
    }

    return Response.json(this.state);
  }

  // Built-in scheduled task support
  async onSchedule(name: string): Promise<void> {
    if (name === "reset-daily") {
      this.setState({
        ...this.state,
        count: 0,
        history: [],
      });
    }
  }
}
```

**Use when:**
- Building agent-style systems (event-driven, stateful, autonomous)
- Need WebSocket support for real-time UI
- Want managed state with `setState` (React-like pattern)
- Building AI agents that maintain conversation state
- Need scheduled tasks with cron expressions

### Comparison Table

| Feature | Raw DO | Hono/RPC | Agents SDK |
|---------|--------|----------|------------|
| State management | Manual `storage.get/put` | Manual `storage.get/put` | `setState` / `this.state` |
| HTTP routing | Manual URL parsing | Framework router | `onRequest` |
| Scheduling | Manual `setAlarm` | Manual `setAlarm` | `schedule()` with cron |
| WebSockets | Manual upgrade | Manual upgrade | Built-in `onConnect`/`onMessage` |
| SQL storage | Manual SQLite setup | Manual SQLite setup | Built-in `this.sql` |
| Dependencies | None | Hono (~14kb) | `agents` package |
| Learning curve | Low | Low | Medium |
| Best for | Simple DOs | HTTP APIs on DOs | AI agents, real-time apps |

---

## Design Principles

These principles apply across all four patterns and any DO you build.

### 1. Single-Writer Guarantee

The most important property of a DO: **only one request executes at a time**. This eliminates an entire class of bugs:

- No race conditions on state updates
- No lost updates from concurrent writes
- No need for locks, mutexes, or compare-and-swap
- No distributed consensus protocols

```typescript
// This is ALWAYS safe in a DO — no concurrent access possible
async increment(): Promise<number> {
  const current =
    (await this.ctx.storage.get<number>("count")) || 0;
  const next = current + 1;
  await this.ctx.storage.put("count", next);
  return next;
}
// In any other system, this would be a race condition.
// In a DO, it's guaranteed correct.
```

Design implication: **put coordination logic in DOs**. Anything that requires "read current state, make a decision, write new state" belongs in a DO. The single-writer guarantee makes this pattern trivially correct.

### 2. Alarm as Heartbeat

DOs can set alarms — a timer that calls `alarm()` at a specified time. This is the mechanism for **self-healing scheduling**:

```typescript
async alarm(): Promise<void> {
  // Do periodic work
  await this.doWork();

  // Schedule next run — the DO keeps itself alive
  await this.state.storage.setAlarm(Date.now() + 60_000);
}
```

Key properties of alarms:
- **Durable**: Survives DO restarts, deployments, and migrations
- **At-most-once**: The alarm fires at most once per scheduled time
- **Self-scheduling**: A DO can schedule its own next alarm
- **Cancellable**: `deleteAlarm()` stops it

This creates a heartbeat pattern: the DO wakes up, does work, schedules the next wakeup. No external cron needed. If the DO crashes mid-work, the alarm is already set — it will wake up and retry.

### 3. State as Context

DOs should store **decision context**, not raw data. The difference:

- **Raw data**: 10,000 user records, full API responses, file contents
- **Decision context**: success rate over last 100 calls, budget remaining, which processors ran, what was decided and why

```typescript
// Bad: storing raw data
interface BadState {
  allRecords: Record<string, unknown>[];  // Grows unbounded
  fullApiResponses: unknown[];            // Huge payloads
}

// Good: storing decision context
interface GoodState {
  metrics: {
    successRate: number;     // Computed from last 100 results
    avgLatencyMs: number;    // Rolling average
  };
  budget: {
    allocated: number;
    spent: number;
  };
  decisions: Array<{         // Why we did what we did
    type: string;
    reason: string;
    timestamp: number;
  }>;
}
```

Raw data belongs in D1, R2, or KV. The DO keeps only what it needs to make decisions.

### 4. Graceful Degradation

Production DOs must handle failure gracefully. The pattern: **observe, classify, adapt, continue**.

```typescript
private adapt(): void {
  const { successRate, consecutiveErrors } = this.metrics;

  // Level 1: Reduce load
  if (successRate < 0.8) {
    this.batchSize = Math.max(this.batchSize * 0.5, 5);
  }

  // Level 2: Back off
  if (consecutiveErrors >= 3) {
    this.intervalMs = Math.min(this.intervalMs * 2, 300_000);
  }

  // Level 3: Circuit break
  if (consecutiveErrors >= 10) {
    this.status = "circuit-broken";
    // Alert via queue/webhook
  }

  // Recovery: gradually return to normal
  if (successRate > 0.95 && this.status === "degraded") {
    this.batchSize = Math.min(
      this.batchSize * 1.2,
      this.maxBatchSize
    );
    this.intervalMs = Math.max(
      this.intervalMs * 0.8,
      this.minIntervalMs
    );
  }
}
```

Never let a DO enter an unrecoverable state. Always have a path back to normal operation.

### 5. Observability

Every decision a DO makes should be logged with context. In production, this is how you debug issues days later.

```typescript
private logDecision(
  type: string,
  reason: string,
  context: unknown
): void {
  const entry = {
    id: `dec_${Date.now()}`,
    type,
    reason,
    context,
    stateSnapshot: {
      batchSize: this.controller.batchSize,
      successRate: this.controller.metrics.successRate,
      status: this.controller.status,
    },
    timestamp: Date.now(),
  };

  // Store in DO for immediate access
  this.decisions.push(entry);
  if (this.decisions.length > 200) {
    this.decisions = this.decisions.slice(-200);
  }

  // Also emit to analytics/logging
  console.log(JSON.stringify(entry));
}
```

### 6. Identity = Addressing

A DO's ID is its address. Choose IDs that are meaningful:

```typescript
// User-scoped DO — one per user
const id = env.USER_STATE.idFromName(`user:${userId}`);

// Entity-scoped DO — one per business entity
const id = env.BRAND_AGENT.idFromName(`brand:${brandId}`);

// Singleton DO — one per purpose
const id = env.ENRICHMENT.idFromName("enrichment-controller");

// Time-scoped DO — one per time window
const id = env.RATE_LIMITER.idFromName(
  `ratelimit:${userId}:${hourBucket}`
);

// Composite scope — one per unique combination
const id = env.COUNTER.idFromName(
  `counter:${tenantId}:${feature}`
);
```

The `idFromName` method gives you a deterministic ID from a string. Any Worker can compute the same ID from the same string, without coordination. This is how DOs achieve global addressing without a registry.

### 7. Composition Over Inheritance

Combine patterns rather than building monolithic DOs:

```typescript
// A controller that is also a budget governor
export class ManagedProcessor implements DurableObject {
  private controller: AdaptiveControllerLogic;
  private budget: BudgetLogic;

  async alarm(): Promise<void> {
    // Check budget before processing
    const allocation = this.budget.getAllocation("enrichment");
    if (allocation.spent >= allocation.allocated) {
      console.log("Budget exhausted — skipping cycle");
      return;
    }

    // Run adaptive controller with budget constraint
    const maxItems = this.budget.remainingItems("enrichment");
    const batchSize = Math.min(
      this.controller.getBatchSize(),
      maxItems
    );

    const results = await this.processBatch(batchSize);
    this.controller.recordResults(results);
    this.budget.recordSpend("enrichment", results.cost);
    this.controller.adapt();

    await this.scheduleNext();
  }
}
```

---

## Small Examples

Practical, self-contained DOs for common use cases.

### Rate Limiter

A sliding-window rate limiter. Each unique key (user, IP, API key) gets its own DO.

```typescript
export class RateLimiter implements DurableObject {
  private state: DurableObjectState;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === "/check") {
      const limit = parseInt(
        url.searchParams.get("limit") || "100"
      );
      const windowMs = parseInt(
        url.searchParams.get("window") || "60000"
      );

      return this.check(limit, windowMs);
    }

    return new Response("Not found", { status: 404 });
  }

  private async check(
    limit: number,
    windowMs: number
  ): Promise<Response> {
    const now = Date.now();
    const windowStart = now - windowMs;

    // Get timestamps of recent requests
    const timestamps =
      (await this.state.storage.get<number[]>("timestamps")) ||
      [];

    // Remove expired entries
    const active = timestamps.filter((t) => t > windowStart);

    if (active.length >= limit) {
      const oldestActive = active[0];
      const retryAfterMs = oldestActive + windowMs - now;

      return Response.json(
        {
          allowed: false,
          current: active.length,
          limit,
          retryAfterMs,
        },
        {
          status: 429,
          headers: {
            "Retry-After": Math.ceil(
              retryAfterMs / 1000
            ).toString(),
          },
        }
      );
    }

    // Allow and record
    active.push(now);
    await this.state.storage.put("timestamps", active);

    return Response.json({
      allowed: true,
      current: active.length,
      limit,
      remaining: limit - active.length,
    });
  }
}

// Worker usage:
// const id = env.RATE_LIMITER.idFromName(`ratelimit:${apiKey}`);
// const limiter = env.RATE_LIMITER.get(id);
// const res = await limiter.fetch(
//   "https://limiter/check?limit=100&window=60000"
// );
// const { allowed } = await res.json();
```

### Session Manager

Per-user session state with automatic expiry.

```typescript
interface SessionData {
  userId: string;
  createdAt: number;
  lastAccessedAt: number;
  data: Record<string, unknown>;
  ttlMs: number;
}

export class SessionManager implements DurableObject {
  private state: DurableObjectState;
  private session: SessionData | null = null;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
    this.state.blockConcurrencyWhile(async () => {
      this.session =
        (await this.state.storage.get<SessionData>("session")) ||
        null;
    });
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/create": {
        const body = (await request.json()) as {
          userId: string;
          ttlMs?: number;
          data?: Record<string, unknown>;
        };
        this.session = {
          userId: body.userId,
          createdAt: Date.now(),
          lastAccessedAt: Date.now(),
          data: body.data || {},
          ttlMs: body.ttlMs || 3600_000, // 1 hour default
        };
        // Set alarm for expiry
        await this.state.storage.setAlarm(
          Date.now() + this.session.ttlMs
        );
        await this.save();
        return Response.json({
          created: true,
          expiresAt: Date.now() + this.session.ttlMs,
        });
      }

      case "/get": {
        if (!this.session) {
          return Response.json(
            { error: "No session" },
            { status: 404 }
          );
        }
        // Touch — extend expiry
        this.session.lastAccessedAt = Date.now();
        await this.state.storage.setAlarm(
          Date.now() + this.session.ttlMs
        );
        await this.save();
        return Response.json(this.session);
      }

      case "/set": {
        if (!this.session) {
          return Response.json(
            { error: "No session" },
            { status: 404 }
          );
        }
        const updates = (await request.json()) as Record<
          string,
          unknown
        >;
        this.session.data = {
          ...this.session.data,
          ...updates,
        };
        this.session.lastAccessedAt = Date.now();
        await this.state.storage.setAlarm(
          Date.now() + this.session.ttlMs
        );
        await this.save();
        return Response.json({ updated: true });
      }

      case "/destroy": {
        this.session = null;
        await this.state.storage.deleteAll();
        return Response.json({ destroyed: true });
      }

      default:
        return new Response("Not found", { status: 404 });
    }
  }

  async alarm(): Promise<void> {
    // Session expired — clean up
    this.session = null;
    await this.state.storage.deleteAll();
  }

  private async save(): Promise<void> {
    if (this.session) {
      await this.state.storage.put("session", this.session);
    }
  }
}
```

### Feature Flag Controller

A centralized feature flag system with gradual rollout support.

```typescript
interface FeatureFlag {
  name: string;
  enabled: boolean;
  rolloutPercent: number; // 0-100
  allowlist: string[];    // Always enabled for these IDs
  blocklist: string[];    // Always disabled for these IDs
  updatedAt: number;
}

export class FeatureFlagController implements DurableObject {
  private state: DurableObjectState;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/evaluate": {
        const params = (await request.json()) as {
          flag: string;
          entityId: string;
        };
        return this.evaluate(params.flag, params.entityId);
      }

      case "/set": {
        const flag = (await request.json()) as FeatureFlag;
        await this.state.storage.put(`flag:${flag.name}`, {
          ...flag,
          updatedAt: Date.now(),
        });
        return Response.json({ set: true });
      }

      case "/list": {
        const flags = await this.state.storage.list({
          prefix: "flag:",
        });
        return Response.json(Object.fromEntries(flags));
      }

      case "/delete": {
        const { name } = (await request.json()) as {
          name: string;
        };
        await this.state.storage.delete(`flag:${name}`);
        return Response.json({ deleted: true });
      }

      default:
        return new Response("Not found", { status: 404 });
    }
  }

  private async evaluate(
    flagName: string,
    entityId: string
  ): Promise<Response> {
    const flag = await this.state.storage.get<FeatureFlag>(
      `flag:${flagName}`
    );

    if (!flag) {
      return Response.json({
        flag: flagName,
        enabled: false,
        reason: "not-found",
      });
    }

    if (!flag.enabled) {
      return Response.json({
        flag: flagName,
        enabled: false,
        reason: "disabled",
      });
    }

    if (flag.blocklist.includes(entityId)) {
      return Response.json({
        flag: flagName,
        enabled: false,
        reason: "blocklisted",
      });
    }

    if (flag.allowlist.includes(entityId)) {
      return Response.json({
        flag: flagName,
        enabled: true,
        reason: "allowlisted",
      });
    }

    // Deterministic percentage rollout based on entity ID
    const hash = await this.hashId(entityId + flagName);
    const bucket = hash % 100;
    const enabled = bucket < flag.rolloutPercent;

    return Response.json({
      flag: flagName,
      enabled,
      reason: enabled
        ? "rollout-included"
        : "rollout-excluded",
      rolloutPercent: flag.rolloutPercent,
    });
  }

  private async hashId(input: string): Promise<number> {
    const encoder = new TextEncoder();
    const data = encoder.encode(input);
    const hashBuffer = await crypto.subtle.digest(
      "SHA-256",
      data
    );
    const hashArray = new Uint8Array(hashBuffer);
    // Use first 4 bytes as a 32-bit integer
    return (
      ((hashArray[0] << 24) |
        (hashArray[1] << 16) |
        (hashArray[2] << 8) |
        hashArray[3]) >>>
      0
    );
  }
}
```

### Aggregator / Counter

A high-throughput counter that buffers writes in memory and flushes periodically.

```typescript
export class AggregatorCounter implements DurableObject {
  private state: DurableObjectState;
  private buffer: Record<string, number> = {};
  private flushIntervalMs = 10_000; // Flush every 10 seconds

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/increment": {
        const { key, amount } = (await request.json()) as {
          key: string;
          amount?: number;
        };
        // Buffer in memory — fast, no I/O
        this.buffer[key] =
          (this.buffer[key] || 0) + (amount || 1);

        // Ensure flush is scheduled
        await this.ensureFlushScheduled();

        return Response.json({ buffered: true });
      }

      case "/get": {
        const key = url.searchParams.get("key");
        if (!key) {
          // Return all counts (stored + buffered)
          const stored =
            (await this.state.storage.get<
              Record<string, number>
            >("counts")) || {};
          const merged = { ...stored };
          for (const [k, v] of Object.entries(this.buffer)) {
            merged[k] = (merged[k] || 0) + v;
          }
          return Response.json(merged);
        }

        const storedCounts =
          (await this.state.storage.get<
            Record<string, number>
          >("counts")) || {};
        const count =
          (storedCounts[key] || 0) + (this.buffer[key] || 0);
        return Response.json({ key, count });
      }

      case "/flush": {
        await this.flush();
        return Response.json({ flushed: true });
      }

      default:
        return new Response("Not found", { status: 404 });
    }
  }

  async alarm(): Promise<void> {
    await this.flush();
  }

  private async ensureFlushScheduled(): Promise<void> {
    const existingAlarm = await this.state.storage.getAlarm();
    if (!existingAlarm) {
      await this.state.storage.setAlarm(
        Date.now() + this.flushIntervalMs
      );
    }
  }

  private async flush(): Promise<void> {
    if (Object.keys(this.buffer).length === 0) return;

    const stored =
      (await this.state.storage.get<Record<string, number>>(
        "counts"
      )) || {};

    for (const [key, amount] of Object.entries(this.buffer)) {
      stored[key] = (stored[key] || 0) + amount;
    }

    await this.state.storage.put("counts", stored);
    this.buffer = {};
  }
}
```

### Leader Election

A coordination DO that ensures exactly one active leader among a group of workers.

```typescript
interface LeaderRecord {
  leaderId: string;
  acquiredAt: number;
  lastHeartbeatAt: number;
  ttlMs: number;
}

export class LeaderElection implements DurableObject {
  private state: DurableObjectState;
  private leader: LeaderRecord | null = null;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
    this.state.blockConcurrencyWhile(async () => {
      this.leader =
        (await this.state.storage.get<LeaderRecord>(
          "leader"
        )) || null;
    });
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/acquire": {
        const { candidateId, ttlMs } =
          (await request.json()) as {
            candidateId: string;
            ttlMs?: number;
          };
        return this.acquire(candidateId, ttlMs || 30_000);
      }

      case "/heartbeat": {
        const { leaderId } = (await request.json()) as {
          leaderId: string;
        };
        return this.heartbeat(leaderId);
      }

      case "/release": {
        const { leaderId } = (await request.json()) as {
          leaderId: string;
        };
        return this.release(leaderId);
      }

      case "/leader": {
        return Response.json({
          leader: this.leader?.leaderId || null,
          acquiredAt: this.leader?.acquiredAt || null,
        });
      }

      default:
        return new Response("Not found", { status: 404 });
    }
  }

  private async acquire(
    candidateId: string,
    ttlMs: number
  ): Promise<Response> {
    // Check if current leader is still valid
    if (this.leader) {
      const elapsed =
        Date.now() - this.leader.lastHeartbeatAt;
      if (elapsed < this.leader.ttlMs) {
        // Leader is still active
        if (this.leader.leaderId === candidateId) {
          return Response.json({
            acquired: true,
            renewed: true,
          });
        }
        return Response.json({
          acquired: false,
          currentLeader: this.leader.leaderId,
          retryAfterMs: this.leader.ttlMs - elapsed,
        });
      }
      // Leader expired — allow takeover
    }

    this.leader = {
      leaderId: candidateId,
      acquiredAt: Date.now(),
      lastHeartbeatAt: Date.now(),
      ttlMs,
    };

    // Set alarm to check for leader expiry
    await this.state.storage.setAlarm(Date.now() + ttlMs);
    await this.state.storage.put("leader", this.leader);

    return Response.json({ acquired: true, renewed: false });
  }

  private async heartbeat(
    leaderId: string
  ): Promise<Response> {
    if (
      !this.leader ||
      this.leader.leaderId !== leaderId
    ) {
      return Response.json(
        { valid: false, error: "Not the leader" },
        { status: 403 }
      );
    }

    this.leader.lastHeartbeatAt = Date.now();
    await this.state.storage.setAlarm(
      Date.now() + this.leader.ttlMs
    );
    await this.state.storage.put("leader", this.leader);

    return Response.json({
      valid: true,
      nextHeartbeatBefore:
        Date.now() + this.leader.ttlMs,
    });
  }

  private async release(
    leaderId: string
  ): Promise<Response> {
    if (
      !this.leader ||
      this.leader.leaderId !== leaderId
    ) {
      return Response.json(
        { released: false, error: "Not the leader" },
        { status: 403 }
      );
    }

    this.leader = null;
    await this.state.storage.delete("leader");
    await this.state.storage.deleteAlarm();

    return Response.json({ released: true });
  }

  async alarm(): Promise<void> {
    // Check if leader has expired
    if (this.leader) {
      const elapsed =
        Date.now() - this.leader.lastHeartbeatAt;
      if (elapsed >= this.leader.ttlMs) {
        // Leader expired — remove
        this.leader = null;
        await this.state.storage.delete("leader");
      } else {
        // Still valid — reschedule check
        await this.state.storage.setAlarm(
          this.leader.lastHeartbeatAt + this.leader.ttlMs
        );
      }
    }
  }
}
```

### Cache Coordinator

A DO that coordinates cache invalidation across multiple KV namespaces or cache zones. Useful when a single update needs to invalidate cache entries across several stores.

```typescript
interface CacheEntry {
  key: string;
  stores: string[];     // Which stores have this cached
  invalidatedAt?: number;
  tags: string[];        // For tag-based invalidation
}

interface Env {
  CACHE_PURGE_BASE: string;
}

export class CacheCoordinator implements DurableObject {
  private state: DurableObjectState;
  private env: Env;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
    this.env = env;
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/register": {
        const { key, store, tags } =
          (await request.json()) as {
            key: string;
            store: string;
            tags: string[];
          };
        const entry =
          (await this.state.storage.get<CacheEntry>(
            `entry:${key}`
          )) || { key, stores: [], tags: [] };
        if (!entry.stores.includes(store)) {
          entry.stores.push(store);
        }
        entry.tags = [
          ...new Set([...entry.tags, ...tags]),
        ];
        await this.state.storage.put(
          `entry:${key}`,
          entry
        );

        // Track tag -> key mapping
        for (const tag of tags) {
          const tagKeys =
            (await this.state.storage.get<string[]>(
              `tag:${tag}`
            )) || [];
          if (!tagKeys.includes(key)) {
            tagKeys.push(key);
            await this.state.storage.put(
              `tag:${tag}`,
              tagKeys
            );
          }
        }

        return Response.json({ registered: true });
      }

      case "/invalidate": {
        const { key } = (await request.json()) as {
          key: string;
        };
        const results = await this.invalidateKey(key);
        return Response.json({
          invalidated: true,
          stores: results,
        });
      }

      case "/invalidate-tag": {
        const { tag } = (await request.json()) as {
          tag: string;
        };
        const keys =
          (await this.state.storage.get<string[]>(
            `tag:${tag}`
          )) || [];

        const allResults: Record<string, string[]> = {};
        for (const key of keys) {
          allResults[key] = await this.invalidateKey(key);
        }

        // Clean up tag mapping
        await this.state.storage.delete(`tag:${tag}`);

        return Response.json({
          invalidated: true,
          tag,
          keysInvalidated: keys.length,
          details: allResults,
        });
      }

      default:
        return new Response("Not found", { status: 404 });
    }
  }

  private async invalidateKey(
    key: string
  ): Promise<string[]> {
    const entry = await this.state.storage.get<CacheEntry>(
      `entry:${key}`
    );
    if (!entry) return [];

    const purgedStores: string[] = [];

    for (const store of entry.stores) {
      try {
        await fetch(
          `${this.env.CACHE_PURGE_BASE}/${store}/purge`,
          {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
            },
            body: JSON.stringify({ keys: [key] }),
          }
        );
        purgedStores.push(store);
      } catch (err) {
        console.error(
          `Failed to purge ${key} from ${store}:`,
          err
        );
      }
    }

    // Mark as invalidated
    entry.invalidatedAt = Date.now();
    entry.stores = [];
    await this.state.storage.put(`entry:${key}`, entry);

    return purgedStores;
  }
}
```

### Workflow Checkpoint

A DO that provides checkpoint/resume capability for long-running processes that are not using Cloudflare Workflows.

```typescript
interface Checkpoint {
  stepId: string;
  status: "pending" | "running" | "completed" | "failed";
  input: unknown;
  output?: unknown;
  error?: string;
  startedAt?: number;
  completedAt?: number;
}

interface WorkflowState {
  workflowId: string;
  status: "running" | "completed" | "failed" | "paused";
  steps: Checkpoint[];
  currentStep: number;
  createdAt: number;
  updatedAt: number;
  metadata: Record<string, unknown>;
}

export class WorkflowCheckpoint implements DurableObject {
  private state: DurableObjectState;
  private workflow: WorkflowState | null = null;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
    this.state.blockConcurrencyWhile(async () => {
      this.workflow =
        (await this.state.storage.get<WorkflowState>(
          "workflow"
        )) || null;
    });
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/create": {
        const body = (await request.json()) as {
          workflowId: string;
          steps: Array<{ stepId: string; input: unknown }>;
          metadata?: Record<string, unknown>;
        };
        this.workflow = {
          workflowId: body.workflowId,
          status: "running",
          steps: body.steps.map((s) => ({
            stepId: s.stepId,
            status: "pending" as const,
            input: s.input,
          })),
          currentStep: 0,
          createdAt: Date.now(),
          updatedAt: Date.now(),
          metadata: body.metadata || {},
        };
        await this.save();
        return Response.json({
          created: true,
          totalSteps: body.steps.length,
        });
      }

      case "/next": {
        if (
          !this.workflow ||
          this.workflow.status !== "running"
        ) {
          return Response.json(
            { error: "No active workflow" },
            { status: 400 }
          );
        }

        const step =
          this.workflow.steps[this.workflow.currentStep];
        if (!step) {
          return Response.json({ done: true });
        }

        step.status = "running";
        step.startedAt = Date.now();
        await this.save();

        return Response.json({
          stepId: step.stepId,
          stepIndex: this.workflow.currentStep,
          totalSteps: this.workflow.steps.length,
          input: step.input,
        });
      }

      case "/complete-step": {
        if (!this.workflow) {
          return Response.json(
            { error: "No workflow" },
            { status: 400 }
          );
        }

        const { output, error } =
          (await request.json()) as {
            output?: unknown;
            error?: string;
          };

        const step =
          this.workflow.steps[this.workflow.currentStep];
        step.completedAt = Date.now();

        if (error) {
          step.status = "failed";
          step.error = error;
          this.workflow.status = "failed";
        } else {
          step.status = "completed";
          step.output = output;
          this.workflow.currentStep++;

          if (
            this.workflow.currentStep >=
            this.workflow.steps.length
          ) {
            this.workflow.status = "completed";
          }
        }

        this.workflow.updatedAt = Date.now();
        await this.save();

        return Response.json({
          status: this.workflow.status,
          completedSteps: this.workflow.currentStep,
          totalSteps: this.workflow.steps.length,
        });
      }

      case "/status": {
        return Response.json(this.workflow);
      }

      case "/resume": {
        if (!this.workflow) {
          return Response.json(
            { error: "No workflow" },
            { status: 400 }
          );
        }
        if (this.workflow.status === "failed") {
          // Reset the failed step to pending
          const failedStep =
            this.workflow.steps[
              this.workflow.currentStep
            ];
          failedStep.status = "pending";
          failedStep.error = undefined;
          failedStep.startedAt = undefined;
          failedStep.completedAt = undefined;
          this.workflow.status = "running";
          await this.save();
        }
        return Response.json({
          resumed: true,
          fromStep: this.workflow.currentStep,
        });
      }

      default:
        return new Response("Not found", { status: 404 });
    }
  }

  private async save(): Promise<void> {
    if (this.workflow) {
      await this.state.storage.put(
        "workflow",
        this.workflow
      );
    }
  }
}
```

---

## Comparisons

How do Durable Objects compare to equivalent systems on other platforms?

### vs AWS Step Functions + DynamoDB

| Aspect | Durable Objects | Step Functions + DynamoDB |
|--------|----------------|--------------------------|
| **State model** | Co-located with compute | Separate database |
| **Latency** | Microseconds (same machine) | 5-50ms (network hop to DDB) |
| **Consistency** | Single-writer (automatic) | Conditional writes (manual) |
| **Scheduling** | Built-in alarms | EventBridge rules (separate service) |
| **Addressing** | Global by ID | By table + primary key |
| **Pricing** | Per-request + storage | Per transition + read/write capacity |
| **Cold start** | ~5-20ms | Step Functions ~50-200ms |
| **WebSocket** | Native support | Requires API Gateway |
| **Max state size** | 10 GiB per DO (SQLite) | 400 KB per DDB item |
| **Cross-entity query** | Not supported (need external index) | Full DDB query capabilities |

**Verdict**: DOs win on latency, consistency model, and developer experience. Step Functions win on cross-entity querying and mature ecosystem. Choose DOs when you need per-entity stateful logic with low latency. Choose Step Functions when you need complex multi-step orchestration with rich querying.

### vs Azure Durable Functions + Durable Entities

| Aspect | Durable Objects | Azure Durable Entities |
|--------|----------------|----------------------|
| **Programming model** | Class with `fetch()`/`alarm()` | Class with operation dispatch |
| **Concurrency** | Single-threaded by default | Single-threaded (entity-level) |
| **State persistence** | Automatic (storage API) | Automatic (entity state) |
| **Scheduling** | Alarms | Durable timers |
| **Signaling** | HTTP/RPC | Entity signals |
| **Runtime** | Cloudflare edge (global) | Azure regions |
| **Language** | JavaScript/TypeScript | C#, JavaScript, Python, Java |
| **Cold start** | ~5-20ms | ~100-500ms |
| **Entity addressing** | `idFromName(string)` | Entity ID (string) |

**Verdict**: The two are architecturally similar — both are virtual actor systems with single-threaded execution and persistent state. Azure Durable Entities have a more mature orchestration layer (Durable Functions). DOs have better latency and global distribution. If you are already on Azure, Durable Entities are excellent. If you want edge-native performance, DOs.

### vs Temporal.io

| Aspect | Durable Objects | Temporal |
|--------|----------------|----------|
| **Execution model** | Event-driven (fetch/alarm) | Workflow replay |
| **State** | Explicit storage API | Implicit (event sourcing) |
| **Retry** | Manual (you implement) | Built-in with policies |
| **Versioning** | Manual (you manage migrations) | Built-in workflow versioning |
| **Observability** | Console.log + custom | Rich UI + history |
| **Hosting** | Cloudflare (managed) | Self-hosted or Temporal Cloud |
| **Complexity** | Low (just a class) | High (SDK, workers, server) |
| **Cross-entity** | Manual coordination | Workflow-to-workflow signals |
| **Best for** | Stateful edge logic | Complex business processes |

**Verdict**: Temporal is a full workflow orchestration platform — it handles retries, versioning, visibility, and complex coordination out of the box. DOs are a lower-level primitive — more flexible but more work. Use Temporal when you have complex, long-running business processes with many steps. Use DOs when you need stateful compute at the edge with low latency and simple coordination.

### vs Akka / Microsoft Orleans (Virtual Actors)

| Aspect | Durable Objects | Akka / Orleans |
|--------|----------------|----------------|
| **Actor model** | Virtual actors | Virtual actors |
| **Location transparency** | Built-in (Cloudflare routes) | Built-in (cluster manages) |
| **Persistence** | Built-in (storage API) | Plugin-based |
| **Messaging** | HTTP/RPC | Typed messages |
| **Clustering** | Managed (Cloudflare) | Self-managed or Azure (Orleans) |
| **Scaling** | Automatic (one DO per ID) | Automatic (one grain per ID) |
| **Language** | TypeScript/JavaScript | Scala/Java (Akka), C# (Orleans) |
| **Infrastructure** | Zero (serverless) | Cluster management required |

**Verdict**: DOs are essentially **serverless virtual actors**. If you know Akka or Orleans, DOs will feel familiar. The main difference: no cluster to manage. Cloudflare handles placement, migration, and failover. The tradeoff: less control over placement and no custom message routing. DOs are Orleans for people who do not want to run infrastructure.

### vs Redis + Custom Scheduling

| Aspect | Durable Objects | Redis + Custom |
|--------|----------------|----------------|
| **State** | Persistent (survives restarts) | Volatile (unless AOF/RDB) |
| **Compute** | Built-in | Separate application server |
| **Scheduling** | Alarms (built-in, durable) | Sorted sets + polling (fragile) |
| **Consistency** | Single-writer per DO | Single-threaded per Redis instance |
| **Scaling** | Per-entity (millions of DOs) | Per-instance (cluster mode) |
| **Operations** | Zero (managed) | Redis cluster management |
| **Cost** | Pay per request | Pay per instance (always-on) |

**Verdict**: Redis is faster for simple read/write operations but requires infrastructure management and custom scheduling logic. DOs are slower per-operation but provide durable compute alongside durable state. If you are building a rate limiter or cache, either works. If you are building a controller that makes decisions and schedules work, DOs are a better fit.

### vs Plain Workers + KV/D1

| Aspect | Durable Objects | Workers + KV/D1 |
|--------|----------------|-----------------|
| **Consistency** | Strong (single-writer) | KV: eventual / D1: strong |
| **Coordination** | Built-in (one thread per ID) | Manual (optimistic locking) |
| **Scheduling** | Alarms | Cron Triggers (minute granularity) |
| **State lifetime** | Tied to DO | Global |
| **Latency** | Microseconds (co-located) | 5-50ms (network hop) |
| **Querying** | Per-entity only | D1: full SQL / KV: key lookup |
| **Complexity** | Medium (DO class) | Low (just a Worker) |

**Verdict**: Start with plain Workers + KV/D1 for simple CRUD applications. Move to DOs when you need per-entity coordination, sub-millisecond state access, or durable scheduling. The migration path is clean: extract the stateful logic from your Worker into a DO, and have the Worker call the DO.

---

## Anti-Patterns

Common mistakes when building with Durable Objects, and what to do instead.

### 1. Using a DO as a Database

**Wrong**: Storing thousands of records in a single DO and querying them.

```typescript
// Bad — the DO becomes a bottleneck
export class UserDatabase implements DurableObject {
  async fetch(request: Request): Promise<Response> {
    // All users in one DO — single-threaded = slow
    const users = await this.state.storage.list({
      prefix: "user:",
    });
    // Filter, sort, paginate... this does not scale
    return Response.json([...users.values()]);
  }
}
```

**Right**: Use D1 or KV for data storage. Use a DO per entity for coordination.

```typescript
// Good — one DO per user, D1 for queries
export class UserController implements DurableObject {
  // Manages ONE user's state and coordination
  // D1 handles queries across all users
}
```

### 2. Unbounded State Growth

**Wrong**: Appending to arrays without bounds.

```typescript
// Bad — grows forever
this.history.push(newEvent);
await this.state.storage.put("history", this.history);
```

**Right**: Use rolling windows or periodic compaction.

```typescript
// Good — bounded
this.history.push(newEvent);
if (this.history.length > 100) {
  this.history = this.history.slice(-100);
}
await this.state.storage.put("history", this.history);
```

### 3. Long-Running Fetch Handlers

**Wrong**: Doing minutes of work inside `fetch()`.

```typescript
// Bad — blocks all other requests to this DO
async fetch(request: Request): Promise<Response> {
  for (const item of allItems) {
    await processItem(item); // 100ms each = 16 min total
  }
  return new Response("Done");
}
```

**Right**: Accept the request quickly, schedule work via alarm.

```typescript
// Good — accept quickly, work in alarm
async fetch(request: Request): Promise<Response> {
  const items = await request.json();
  await this.state.storage.put("pendingWork", items);
  await this.state.storage.setAlarm(Date.now());
  return Response.json({
    accepted: true,
    itemCount: (items as unknown[]).length,
  });
}

async alarm(): Promise<void> {
  const items =
    (await this.state.storage.get<unknown[]>(
      "pendingWork"
    )) || [];
  const batch = items.slice(0, 50);
  const remaining = items.slice(50);

  await this.processBatch(batch);

  if (remaining.length > 0) {
    await this.state.storage.put(
      "pendingWork",
      remaining
    );
    await this.state.storage.setAlarm(Date.now());
  } else {
    await this.state.storage.delete("pendingWork");
  }
}
```

### 4. Cross-DO Transactions

**Wrong**: Trying to update two DOs atomically.

```typescript
// Bad — no cross-DO transactions exist
await from.fetch("/debit", {
  method: "POST",
  body: JSON.stringify({ amount }),
});
// If the next line fails, money is debited but not credited!
await to.fetch("/credit", {
  method: "POST",
  body: JSON.stringify({ amount }),
});
```

**Right**: Use the saga pattern — each step is idempotent and reversible.

```typescript
// Good — saga with compensation
const txId = crypto.randomUUID();

// Step 1: Reserve (idempotent — same txId = same result)
const reserved = await from.fetch("/reserve", {
  method: "POST",
  body: JSON.stringify({ txId, amount }),
});
if (!reserved.ok) return { error: "insufficient funds" };

// Step 2: Credit (idempotent)
const credited = await to.fetch("/credit", {
  method: "POST",
  body: JSON.stringify({ txId, amount }),
});
if (!credited.ok) {
  // Compensate — release the reservation
  await from.fetch("/release", {
    method: "POST",
    body: JSON.stringify({ txId }),
  });
  return { error: "credit failed" };
}

// Step 3: Confirm the debit
await from.fetch("/confirm", {
  method: "POST",
  body: JSON.stringify({ txId }),
});
```

### 5. Ignoring Alarm Failures

**Wrong**: Not rescheduling the alarm after an error.

```typescript
// Bad — if doWork() throws, the alarm is lost forever
async alarm(): Promise<void> {
  await this.doWork(); // Throws -> DO goes silent
}
```

**Right**: Always reschedule, even on failure.

```typescript
// Good — alarm always reschedules
async alarm(): Promise<void> {
  try {
    await this.doWork();
  } catch (err) {
    console.error("Alarm work failed:", err);
  }

  // Always schedule the next alarm
  await this.state.storage.setAlarm(
    Date.now() + this.intervalMs
  );
}
```

### 6. Storing Secrets in DO Storage

**Wrong**: Putting API keys or credentials in DO storage.

```typescript
// Bad — DO storage is not designed for secrets
await this.state.storage.put(
  "apiKey",
  "sk-live-abc123..."
);
```

**Right**: Use environment variables (Wrangler secrets) and pass them at construction time.

```typescript
// Good — secrets come from env
export class MyDO implements DurableObject {
  private apiKey: string;

  constructor(state: DurableObjectState, env: Env) {
    this.apiKey = env.API_KEY;
    // Set via: wrangler secret put API_KEY
  }
}
```

### 7. God Object DO

**Wrong**: A single DO class that handles everything.

```typescript
// Bad — one DO with 50 endpoints and 20 state fields
export class EverythingDO implements DurableObject {
  // User management, rate limiting, caching, scheduling,
  // feature flags, analytics, notifications...
}
```

**Right**: One DO class per concern. Compose by having DOs call each other.

```typescript
// Good — separate concerns
export class UserSession implements DurableObject {
  /* session state */
}
export class RateLimiter implements DurableObject {
  /* rate limits */
}
export class FeatureFlags implements DurableObject {
  /* flags */
}

// Worker composes them:
async function handleRequest(
  request: Request,
  env: Env
): Promise<Response> {
  const userId = getUserId(request);

  const limiter = env.RATE_LIMITER.get(
    env.RATE_LIMITER.idFromName(userId)
  );
  const { allowed } = await (
    await limiter.fetch(
      "https://do/check?limit=100&window=60000"
    )
  ).json();
  if (!allowed)
    return new Response("Too many requests", {
      status: 429,
    });

  const session = env.USER_SESSION.get(
    env.USER_SESSION.idFromName(userId)
  );
  // ... use session
}
```

### 8. Not Handling DO Eviction

**Wrong**: Assuming in-memory state is always available.

```typescript
// Bad — in-memory cache disappears on eviction
export class BadDO implements DurableObject {
  private cache = new Map<string, unknown>();

  async fetch(request: Request): Promise<Response> {
    // cache might be empty after eviction!
    const value = this.cache.get("key");
  }
}
```

**Right**: Always fall through to storage. Use `blockConcurrencyWhile` for initialization.

```typescript
// Good — storage is source of truth, memory is a cache
export class GoodDO implements DurableObject {
  private cache: Map<string, unknown> | null = null;

  constructor(state: DurableObjectState, env: Env) {
    state.blockConcurrencyWhile(async () => {
      const stored = await state.storage.list();
      this.cache = new Map(stored);
    });
  }
}
```

---

## Wrangler Configuration

All examples use `wrangler.jsonc` for Durable Object bindings:

```jsonc
// wrangler.jsonc
{
  "name": "my-do-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-12-01",

  "durable_objects": {
    "bindings": [
      {
        "name": "ENRICHMENT_CONTROLLER",
        "class_name": "EnrichmentController"
      },
      {
        "name": "STORAGE_SIDECAR",
        "class_name": "StorageSidecar"
      },
      {
        "name": "BUDGET_GOVERNOR",
        "class_name": "BudgetGovernor"
      },
      {
        "name": "BRAND_AGENT",
        "class_name": "BrandAgent"
      },
      {
        "name": "RATE_LIMITER",
        "class_name": "RateLimiter"
      },
      {
        "name": "SESSION_MANAGER",
        "class_name": "SessionManager"
      },
      {
        "name": "FEATURE_FLAGS",
        "class_name": "FeatureFlagController"
      },
      {
        "name": "COUNTER",
        "class_name": "AggregatorCounter"
      },
      {
        "name": "LEADER_ELECTION",
        "class_name": "LeaderElection"
      },
      {
        "name": "CACHE_COORDINATOR",
        "class_name": "CacheCoordinator"
      },
      {
        "name": "WORKFLOW_CHECKPOINT",
        "class_name": "WorkflowCheckpoint"
      }
    ]
  },

  "migrations": [
    {
      "tag": "v1",
      "new_classes": [
        "EnrichmentController",
        "StorageSidecar",
        "BudgetGovernor",
        "BrandAgent",
        "RateLimiter",
        "SessionManager",
        "FeatureFlagController",
        "AggregatorCounter",
        "LeaderElection",
        "CacheCoordinator",
        "WorkflowCheckpoint"
      ]
    }
  ]
}
```

---

## References

### Cloudflare Documentation

- **[Durable Objects Overview](https://developers.cloudflare.com/durable-objects/)** — Official documentation for Durable Objects, including concepts, APIs, and getting started guides.

- **[Alarm API](https://developers.cloudflare.com/durable-objects/api/alarms/)** — Documentation for the Durable Object alarm API. Alarms are the mechanism for self-scheduling — a DO can set a timer to wake itself up in the future.

- **[Storage API](https://developers.cloudflare.com/durable-objects/api/storage-api/)** — The transactional key-value and SQLite storage API available to each DO. Covers `get`, `put`, `delete`, `list`, `transaction`, and the newer SQLite-backed storage.

- **[Best Practices](https://developers.cloudflare.com/durable-objects/best-practices/)** — Cloudflare's official best practices for Durable Objects, including guidance on state management, error handling, and performance.

- **[Cloudflare Workflows](https://developers.cloudflare.com/workflows/)** — Durable execution engine for multi-step tasks. Pairs well with the Storage Sidecar pattern for state that exceeds the 1 MiB Workflow state limit.

- **[Cloudflare Queues](https://developers.cloudflare.com/queues/)** — Message queue service. Used by the Event Reactor pattern for emitting commands to downstream services without blocking the DO.

### Cloudflare Agents SDK

- **[Agents SDK Documentation](https://agents.cloudflare.com)** — The official Agents SDK for building AI agents and stateful applications on Cloudflare Workers. Provides higher-level abstractions over Durable Objects including managed state, scheduling, and WebSocket support.

- **[Agents SDK GitHub](https://github.com/cloudflare/agents)** — Source code and examples for the Cloudflare Agents SDK. Includes reference implementations of agent patterns.

### Cloudflare Blog Posts

- **[Building Real-Time Games Using Workers, Durable Objects, and Unity](https://blog.cloudflare.com/building-real-time-games-using-workers-durable-objects-and-unity/)** — A deep dive into using Durable Objects for real-time game state management, demonstrating WebSocket integration and low-latency state synchronization.

- **[Durable Objects: Easy, Fast, Correct — Choose Three](https://blog.cloudflare.com/durable-objects-easy-fast-correct-choose-three/)** — The announcement post for Durable Objects GA, explaining the design philosophy and performance characteristics.

- **[Introducing Workers Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)** — The original introduction of Durable Objects, explaining the motivation (consistent state at the edge) and the fundamental model (single-threaded, globally addressable, persistent).

### Comparison References

- **[Microsoft Orleans](https://microsoft.github.io/orleans/)** — The virtual actor framework for .NET that inspired many of the patterns in this article. Orleans grains are the closest equivalent to Durable Objects in the .NET ecosystem — both provide single-threaded, addressable, persistent compute units.

- **[Temporal.io](https://temporal.io)** — A durable execution platform for workflow orchestration. Temporal provides stronger guarantees around workflow versioning, replay, and visibility than raw Durable Objects, at the cost of more infrastructure and complexity.

- **[Akka Actors](https://doc.akka.io/concepts/index.html)** — The actor model framework for the JVM. Akka's actor model (message-driven, location-transparent, supervised) is the theoretical foundation that Durable Objects implement in a serverless context.

- **[Azure Durable Entities](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-entities)** — Microsoft's virtual actor implementation within Azure Durable Functions. The closest Azure equivalent to Cloudflare Durable Objects, with similar single-threaded execution and persistent state guarantees.

### Tools

- **[Claude Code](https://jsr.io/@anthropic-ai/claude-code)** — The AI coding agent used to develop the Agents SDK patterns documented here. Relevant for understanding the AI agent architecture that the Event Reactor pattern supports.

---

## License

MIT

---

*Built with production patterns from real Cloudflare Workers deployments.*
