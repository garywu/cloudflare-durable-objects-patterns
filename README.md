# Durable Object Patterns on Cloudflare Workers

```typescript
export class PipelineController extends DurableObject {
  private running = false

  async alarm() {
    if (!this.running) return
    const items = await this.env.DB.prepare(
      'SELECT * FROM queue WHERE status = ? LIMIT 10'
    ).bind('pending').all()

    for (const item of items.results) {
      try {
        await this.env.PROCESSOR.fetch('https://proc/run', {
          method: 'POST',
          body: JSON.stringify(item),
        })
        await this.env.DB.prepare(
          'UPDATE queue SET status = ? WHERE id = ?'
        ).bind('done', item.id).run()
      } catch {
        await this.env.DB.prepare(
          'UPDATE queue SET status = ? WHERE id = ?'
        ).bind('failed', item.id).run()
      }
    }

    // Self-schedule: run again in 30 seconds
    await this.ctx.storage.setAlarm(Date.now() + 30_000)
  }

  async fetch(req: Request) {
    this.running = true
    await this.ctx.storage.setAlarm(Date.now() + 1_000)
    return new Response('started')
  }
}
```

That's a self-driving pipeline. No cron. No queue. No external trigger.

The Durable Object schedules itself, processes work from the database, handles errors, and loops. It runs until you tell it to stop. It survives restarts, redeploys, and hibernation. It's 30 lines.

This article covers four production-tested patterns for building stateful coordination on Cloudflare Workers using Durable Objects.

---

## Table of Contents

- [The Problem](#the-problem)
- [Pattern 1: The Adaptive Controller](#pattern-1-the-adaptive-controller)
- [Pattern 2: The Storage Sidecar](#pattern-2-the-storage-sidecar)
- [Pattern 3: The Budget Governor](#pattern-3-the-budget-governor)
- [Pattern 4: The Event Reactor](#pattern-4-the-event-reactor)
- [The Meta-Pattern: Control Plane / Data Plane](#the-meta-pattern-control-plane--data-plane)
- [Design Principles](#design-principles)
- [Layer 0 vs Layer 2: Raw DO vs Agents SDK](#layer-0-vs-layer-2-raw-do-vs-agents-sdk)
- [Wrangler Configuration](#wrangler-configuration)
- [Small Patterns That Add Up](#small-patterns-that-add-up)
- [Anti-Patterns](#anti-patterns)
- [Comparisons](#comparisons)
- [References](#references)

---

## The Problem

Workers are stateless. Every request starts fresh. No memory of the last request. No running counter. No timer ticking in the background.

For most HTTP handlers, that's fine. For coordination — scheduling, rate limiting, budget management, adaptive processing — you need state that survives requests.

The typical answer is D1 polling + cron:

```typescript
// Cron trigger: runs every minute whether there's work or not
export default {
  async scheduled(event, env) {
    const pending = await env.DB.prepare(
      'SELECT * FROM jobs WHERE status = ? AND next_run_at < ?'
    ).bind('pending', Date.now()).all()

    for (const job of pending.results) {
      await processJob(env, job)  // What if two cron invocations overlap?
    }
  }
}
```

Problems:

- **Wasteful.** Cron fires every minute. Most ticks find nothing to do.
- **Race-prone.** Two cron invocations can overlap and process the same job.
- **Fixed cadence.** Can't speed up when there's lots of work or slow down when things are failing.
- **No state.** Success rates, batch sizes, cooldowns — all require state that cron doesn't have.
- **No control API.** Can't pause, resume, or inspect the scheduler at runtime.

Durable Objects solve all of these. A DO is a single-threaded, addressable actor with co-located storage and a built-in alarm. It processes one request at a time. No races. It self-schedules via `alarm()`. No cron waste. It holds state across requests. Adaptive behavior becomes trivial.

```
┌──────────────────────────────────────────────────┐
│              Durable Object                       │
│                                                   │
│  ┌─────────┐  ┌──────────┐  ┌─────────────────┐  │
│  │ alarm() │  │  state   │  │  fetch() API    │  │
│  │ (timer) │  │ (memory) │  │  /start /stop   │  │
│  └────┬────┘  └────┬─────┘  └────────┬────────┘  │
│       │            │                 │            │
│       └────────────┼─────────────────┘            │
│                    │                              │
│            single-threaded                        │
│            serialized access                      │
│            no races                               │
└──────────────────────────────────────────────────┘
         │                          ▲
         │ read/write               │ HTTP
         ▼                          │
    ┌─────────┐              ┌──────────┐
    │   D1    │              │  Worker  │
    │  (data) │              │ (router) │
    └─────────┘              └──────────┘
```

---

## Pattern 1: The Adaptive Controller

A self-scheduling alarm loop that adjusts batch size and interval based on success rate. Production code from an app store scraper that processes thousands of listings per hour.

The core idea: measure how well things are going, then tune the knobs.

```
success rate > 95%  →  ramp up batch size, shorten interval
success rate > 80%  →  gentle ramp
success rate < 50%  →  halve batch, slow down
success rate < 20%  →  emergency cooldown (5 min pause)
```

### State Shape

```typescript
interface ControllerState {
  total_dispatched: number
  total_completed: number
  total_failed: number
  last_tick_at: number
  running: boolean
  current_batch_size: number
  current_interval_ms: number
  cooldown_until: number
  metrics: Record<number, { dispatched: number; completed: number; failed: number }>
}
```

The `metrics` map is keyed by timestamp (floored to the minute). Entries older than 5 minutes are pruned on each tick. This creates a rolling window for rate calculations.

### The Adapt Method

This is the brain. After each tick, it looks at the rolling success rate and adjusts:

```typescript
private adapt() {
  const sr = this.successRate()
  if (sr < 0.2) {
    // CRITICAL → cooldown + min batch + max interval
    this.state.cooldown_until = Date.now() + 300_000
    this.state.current_batch_size = 2
    this.state.current_interval_ms = 120_000
  } else if (sr < 0.5) {
    // LOW → halve batch, slow interval
    this.state.current_batch_size = Math.max(2, Math.floor(this.state.current_batch_size * 0.5))
    this.state.current_interval_ms = Math.min(120_000, Math.floor(this.state.current_interval_ms * 1.5))
  } else if (sr > 0.95) {
    // GREAT → ramp up aggressively
    this.state.current_batch_size = Math.min(50, Math.ceil(this.state.current_batch_size * 1.25))
    this.state.current_interval_ms = Math.max(10_000, Math.floor(this.state.current_interval_ms * 0.8))
  } else if (sr > 0.8) {
    // GOOD → gentle ramp
    this.state.current_batch_size = Math.min(50, Math.ceil(this.state.current_batch_size * 1.1))
    this.state.current_interval_ms = Math.max(10_000, Math.floor(this.state.current_interval_ms * 0.95))
  }
}
```

| Threshold | Name | Batch Size | Interval | Action |
|-----------|------|-----------|----------|--------|
| < 0.2 | CRITICAL | Set to 2 | Set to 120s | 5 min cooldown |
| < 0.5 | LOW | ×0.5 (min 2) | ×1.5 (max 120s) | Back off |
| > 0.8 | GOOD | ×1.1 (max 50) | ×0.95 (min 10s) | Gentle ramp |
| > 0.95 | GREAT | ×1.25 (max 50) | ×0.8 (min 10s) | Aggressive ramp |

### The Alarm Loop

```typescript
async alarm() {
  // Reconstruct state from storage (hibernation-safe)
  await this.loadState()

  if (!this.state.running) return

  // Respect cooldown
  if (Date.now() < this.state.cooldown_until) {
    await this.ctx.storage.setAlarm(this.state.cooldown_until)
    return
  }

  // Get work from DB
  const batch = await this.env.DB.prepare(
    'SELECT id, url FROM listings WHERE status = ? LIMIT ?'
  ).bind('pending', this.state.current_batch_size).all()

  if (batch.results.length === 0) {
    // Nothing to do — check again later (backoff)
    await this.ctx.storage.setAlarm(Date.now() + 60_000)
    return
  }

  // Process batch
  const now = Math.floor(Date.now() / 60_000) * 60_000 // floor to minute
  if (!this.state.metrics[now]) {
    this.state.metrics[now] = { dispatched: 0, completed: 0, failed: 0 }
  }

  for (const item of batch.results) {
    this.state.metrics[now].dispatched++
    this.state.total_dispatched++
    try {
      await this.env.SCRAPER.fetch('https://scraper/process', {
        method: 'POST',
        body: JSON.stringify(item),
      })
      this.state.metrics[now].completed++
      this.state.total_completed++
    } catch {
      this.state.metrics[now].failed++
      this.state.total_failed++
    }
  }

  // Prune old metrics (keep 5 min window)
  const cutoff = Date.now() - 300_000
  for (const key in this.state.metrics) {
    if (Number(key) < cutoff) delete this.state.metrics[key]
  }

  // Adapt and reschedule
  this.adapt()
  this.state.last_tick_at = Date.now()
  await this.saveState()
  await this.ctx.storage.setAlarm(Date.now() + this.state.current_interval_ms)
}
```

### The HTTP API

```typescript
async fetch(request: Request): Promise<Response> {
  const url = new URL(request.url)

  switch (url.pathname) {
    case '/start':
      this.state.running = true
      await this.saveState()
      await this.ctx.storage.setAlarm(Date.now() + 1_000)
      return Response.json({ status: 'started' })

    case '/stop':
      this.state.running = false
      await this.saveState()
      return Response.json({ status: 'stopped' })

    case '/reset':
      this.state = this.defaultState()
      await this.saveState()
      return Response.json({ status: 'reset' })

    case '/status':
      return Response.json({
        running: this.state.running,
        batch_size: this.state.current_batch_size,
        interval_ms: this.state.current_interval_ms,
        success_rate: this.successRate(),
        total_dispatched: this.state.total_dispatched,
        total_completed: this.state.total_completed,
        total_failed: this.state.total_failed,
        in_cooldown: Date.now() < this.state.cooldown_until,
      })

    default:
      return new Response('not found', { status: 404 })
  }
}
```

The controller starts cold (batch=5, interval=30s), ramps up as things succeed, and backs off hard when they don't. No human tuning required after deployment.

---

## Pattern 2: The Storage Sidecar

Cloudflare Workflows have a 1 MiB state limit. Pass 500 items between steps and you hit it. The Storage Sidecar solves this: a Durable Object that stores heavy data alongside the Workflow, keyed by pipeline ID.

```
Without sidecar:
  Step 1 → [500 items in state] → Step 2    💥 1 MiB limit

With sidecar:
  Step 1 → store in DO → [batch_id string] → Step 2 → load from DO
                                                        ✅ 22 bytes in state
```

### The DO as a Mini Database

One DO per pipeline run. The DO uses SQLite (via `this.ctx.storage.sql`) to manage multiple tables:

```typescript
export class PipelineSidecar extends DurableObject {
  private initialized = false

  private ensureTables() {
    if (this.initialized) return
    this.ctx.storage.sql.exec(`
      CREATE TABLE IF NOT EXISTS items (
        id TEXT PRIMARY KEY,
        step TEXT NOT NULL,
        data TEXT NOT NULL,
        created_at INTEGER DEFAULT (unixepoch())
      );
      CREATE TABLE IF NOT EXISTS seen_hashes (
        hash TEXT PRIMARY KEY,
        first_seen INTEGER DEFAULT (unixepoch())
      );
      CREATE TABLE IF NOT EXISTS runs (
        run_id TEXT PRIMARY KEY,
        status TEXT DEFAULT 'running',
        started_at INTEGER DEFAULT (unixepoch()),
        finished_at INTEGER
      );
      CREATE TABLE IF NOT EXISTS batches (
        batch_id TEXT PRIMARY KEY,
        step TEXT NOT NULL,
        payload TEXT NOT NULL,
        created_at INTEGER DEFAULT (unixepoch()),
        expires_at INTEGER
      );
      CREATE TABLE IF NOT EXISTS events (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        event_type TEXT NOT NULL,
        data TEXT,
        created_at INTEGER DEFAULT (unixepoch())
      );
    `)
    this.initialized = true
  }

  async fetch(request: Request): Promise<Response> {
    this.ensureTables()
    const url = new URL(request.url)
    const method = request.method

    // Mini-router
    if (method === 'POST' && url.pathname === '/batch/store') {
      const { batch_id, step, payload, ttl_seconds } = await request.json() as any
      const expires = ttl_seconds ? Math.floor(Date.now() / 1000) + ttl_seconds : null
      this.ctx.storage.sql.exec(
        'INSERT OR REPLACE INTO batches (batch_id, step, payload, expires_at) VALUES (?, ?, ?, ?)',
        batch_id, step, JSON.stringify(payload), expires
      )
      return Response.json({ stored: batch_id })
    }

    if (method === 'GET' && url.pathname.startsWith('/batch/')) {
      const batch_id = url.pathname.split('/')[2]
      const row = this.ctx.storage.sql.exec(
        'SELECT payload FROM batches WHERE batch_id = ?', batch_id
      ).one()
      if (!row) return new Response('not found', { status: 404 })
      return Response.json(JSON.parse(row.payload as string))
    }

    if (method === 'POST' && url.pathname === '/dedup') {
      const { hashes } = await request.json() as any
      const novel: string[] = []
      for (const hash of hashes) {
        const exists = this.ctx.storage.sql.exec(
          'SELECT 1 FROM seen_hashes WHERE hash = ?', hash
        ).one()
        if (!exists) {
          this.ctx.storage.sql.exec('INSERT INTO seen_hashes (hash) VALUES (?)', hash)
          novel.push(hash)
        }
      }
      return Response.json({ novel, deduped: hashes.length - novel.length })
    }

    return new Response('not found', { status: 404 })
  }

  // Daily cleanup via alarm
  async alarm() {
    this.ensureTables()
    const now = Math.floor(Date.now() / 1000)
    this.ctx.storage.sql.exec('DELETE FROM batches WHERE expires_at IS NOT NULL AND expires_at < ?', now)
    this.ctx.storage.sql.exec('DELETE FROM events WHERE created_at < ?', now - 86400)
    // Re-schedule for tomorrow
    await this.ctx.storage.setAlarm(Date.now() + 86_400_000)
  }
}
```

### Using the Sidecar from a Workflow

```typescript
import { WorkflowEntrypoint, WorkflowStep, WorkflowEvent } from 'cloudflare:workers'

export class ContentPipeline extends WorkflowEntrypoint<Env, PipelineParams> {
  async run(event: WorkflowEvent<PipelineParams>, step: WorkflowStep) {
    const pipelineId = event.payload.pipeline_id

    // Step 1: Fetch data (might return hundreds of items)
    const items = await step.do('fetch-items', async () => {
      const res = await this.env.DATA_SOURCE.fetch('https://source/items')
      return await res.json() as Item[]
    })

    // Step 2: Store in sidecar (NOT in workflow state)
    const batchId = await step.do('store-batch', async () => {
      const id = this.env.PIPELINE_SIDECAR.idFromName(pipelineId)
      const sidecar = this.env.PIPELINE_SIDECAR.get(id)
      const batch_id = crypto.randomUUID()
      await sidecar.fetch('https://sidecar/batch/store', {
        method: 'POST',
        body: JSON.stringify({
          batch_id,
          step: 'fetched',
          payload: items,  // Heavy data goes to DO, not workflow state
          ttl_seconds: 3600,
        }),
      })
      return batch_id  // Only this tiny string goes into workflow state
    })

    // Step 3: Transform (loads from sidecar, stores result back)
    const transformedBatchId = await step.do('transform', async () => {
      const id = this.env.PIPELINE_SIDECAR.idFromName(pipelineId)
      const sidecar = this.env.PIPELINE_SIDECAR.get(id)
      const res = await sidecar.fetch(`https://sidecar/batch/${batchId}`)
      const items = await res.json() as Item[]

      const transformed = items.map(transformItem)

      const new_batch_id = crypto.randomUUID()
      await sidecar.fetch('https://sidecar/batch/store', {
        method: 'POST',
        body: JSON.stringify({
          batch_id: new_batch_id,
          step: 'transformed',
          payload: transformed,
          ttl_seconds: 3600,
        }),
      })
      return new_batch_id
    })

    // Step 4: Publish (loads final data from sidecar)
    await step.do('publish', async () => {
      const id = this.env.PIPELINE_SIDECAR.idFromName(pipelineId)
      const sidecar = this.env.PIPELINE_SIDECAR.get(id)
      const res = await sidecar.fetch(`https://sidecar/batch/${transformedBatchId}`)
      const items = await res.json()
      // ... publish items
    })
  }
}
```

The workflow state between steps is always a single string. The heavy data lives in the sidecar's SQLite. The sidecar also handles deduplication via `seen_hashes` — hash each item, check before processing, skip duplicates across runs.

---

## Pattern 3: The Budget Governor

A control plane DO that manages AI model costs across multiple processors. Production code from an opportunity analysis system that runs multiple LLM-based processors on a budget.

The problem: you have 5 processors that call AI models. Each has a different cost profile. You have a daily budget of $10. You need to run them all, stay under budget, and degrade gracefully when money gets tight.

```
┌──────────────────────────────────────────────────────┐
│                  Budget Governor (DO)                  │
│                                                       │
│  config ← D1 (live reload)     budget: $10/cycle      │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │  Phase: gather → analyze → evaluate → critique  │  │
│  │                                                 │  │
│  │  Each tick: pick next processor, check budget,  │  │
│  │  select model tier, execute, log telemetry      │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  Processor registry:                                  │
│  ┌──────────┬──────────┬──────────┬──────────┐       │
│  │ keyword  │ compet.  │ market   │ critique │       │
│  │ gather   │ analyze  │ evaluate │ review   │       │
│  │ w=0.3    │ w=0.25   │ w=0.25   │ w=0.2    │       │
│  └──────────┴──────────┴──────────┴──────────┘       │
└──────────────────────────────────────────────────────┘
```

### Processor Configuration

```typescript
interface ProcessorConfig {
  name: string
  type: 'deterministic' | 'llm'
  enabled: boolean
  phase: string  // 'gather' | 'analyze' | 'evaluate' | 'critique'
  model_tier?: 'default' | 'expensive' | 'premium'
  weight: number  // budget share (0.0-1.0)
  max_cost_per_call: number
  batch_size: number
}
```

Weights sum to 1.0 across all processors. A processor with `weight: 0.3` gets 30% of the cycle budget. If it underspends, the surplus redistributes to later processors.

### Model Selection with Budget Awareness

```typescript
private selectModel(proc: ProcessorConfig, cycle: CycleState): string {
  const remaining = this.config.budget_max_per_cycle - cycle.cost_so_far
  // Budget tight? Downgrade to default
  if (remaining < this.config.budget_max_per_cycle * 0.2 && proc.model_tier !== 'default') {
    return this.config.model_default
  }
  switch (proc.model_tier) {
    case 'premium': return this.config.model_premium
    case 'expensive': return this.config.model_expensive
    default: return this.config.model_default
  }
}
```

When the remaining budget drops below 20% of the cycle cap, every processor gets downgraded to the cheapest model. This prevents a single expensive processor from starving the rest.

### The Governor Loop

```typescript
interface CycleState {
  cycle_id: string
  started_at: number
  cost_so_far: number
  tokens_so_far: number
  processors_run: string[]
  current_phase: string
}

async alarm() {
  await this.loadConfig()  // Fresh config from D1 every tick

  if (!this.config.enabled) {
    await this.ctx.storage.setAlarm(Date.now() + 60_000)
    return
  }

  const cycle = await this.getCurrentCycle()

  // Budget check: hard mode = stop, soft mode = skip expensive
  if (cycle.cost_so_far >= this.config.budget_max_per_cycle) {
    if (this.config.budget_mode === 'hard') {
      // Hard stop — wait for next cycle
      await this.startNewCycle()
      return
    }
    // Soft mode — only run deterministic processors
  }

  // Pick next processor (phase-ordered, round-robin within phase)
  const next = this.nextProcessor(cycle)
  if (!next) {
    // All processors run this cycle — start fresh
    await this.startNewCycle()
    return
  }

  // Execute one processor per tick (stay under CPU limit)
  const model = next.type === 'llm' ? this.selectModel(next, cycle) : undefined
  const result = await this.executeProcessor(next, model, cycle)

  // Log telemetry to D1
  await this.env.DB.prepare(`
    INSERT INTO processor_telemetry (cycle_id, processor, model, cost, tokens, duration_ms, success)
    VALUES (?, ?, ?, ?, ?, ?, ?)
  `).bind(
    cycle.cycle_id, next.name, model ?? 'n/a',
    result.cost, result.tokens, result.duration_ms, result.success ? 1 : 0
  ).run()

  // Update cycle state
  cycle.cost_so_far += result.cost
  cycle.tokens_so_far += result.tokens
  cycle.processors_run.push(next.name)
  await this.saveCycle(cycle)

  // Reschedule — interval from config (dashboard-adjustable)
  await this.ctx.storage.setAlarm(Date.now() + this.config.tick_interval_ms)
}
```

Key design decisions:

- **One processor per tick.** Each alarm invocation runs one processor. This keeps CPU time well under the 30-second limit and makes the system observable — you can see exactly what ran when.
- **Config from D1.** Every tick reloads config. Change a processor's weight or disable it from a dashboard — takes effect on the next tick. No redeployment.
- **Phase ordering.** Processors run in phase order: gather, analyze, evaluate, critique. Within a phase, round-robin. This ensures upstream data exists before downstream processors need it.
- **Budget modes.** Hard mode: stop everything when cap hit. Soft mode: skip LLM processors, keep running deterministic ones. Choose based on whether incomplete analysis is worse than no analysis.

### Processor Registry

Processors are pure functions. They don't know about budgets, scheduling, or each other.

```typescript
type ProcessorFn = (ctx: ProcessorContext, items: any[]) => Promise<ProcessorResult>

interface ProcessorContext {
  db: D1Database
  r2: R2Bucket
  config: Record<string, any>
  model: string
  remainingBudget: number
}

const registry = new Map<string, ProcessorFn>()

function registerProcessor(name: string, fn: ProcessorFn) {
  registry.set(name, fn)
}

// Example: keyword gathering processor
registerProcessor('keyword-gather', async (ctx, items) => {
  const keywords = await ctx.db.prepare(
    'SELECT * FROM opportunities WHERE status = ? LIMIT ?'
  ).bind('new', 20).all()

  // ... gather data for each keyword

  return {
    success: true,
    cost: 0,  // deterministic, no model cost
    tokens: 0,
    duration_ms: Date.now() - start,
    items_processed: keywords.results.length,
  }
})

// Example: competitive analysis processor (LLM)
registerProcessor('competitive-analyze', async (ctx, items) => {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${ctx.config.openai_key}` },
    body: JSON.stringify({
      model: ctx.model,  // Governor chose this based on budget
      messages: [{ role: 'user', content: buildPrompt(items) }],
    }),
  })
  const data = await response.json()
  const cost = calculateCost(ctx.model, data.usage)
  return { success: true, cost, tokens: data.usage.total_tokens, duration_ms: /* ... */ }
})
```

The Governor doesn't know what processors do. Processors don't know about budgets. The registry is the only coupling point.

---

## Pattern 4: The Event Reactor

Uses the [Cloudflare Agents SDK](https://developers.cloudflare.com/agents/) (Layer 2) to build persistent per-entity agents that react to domain events, maintain strategy state, and emit commands.

Unlike raw Durable Objects, the Agent class gives you:

- Built-in SQLite (`this.sql`) — no manual table management
- Synced state (`this.setState()`) — WebSocket clients get updates automatically
- Scheduling (`this.schedule()`) — cron expressions or one-time delays
- Workflow integration (`this.runWorkflow()`) — spawn durable multi-step processes

### One Agent Per Entity

```typescript
import { Agent, AgentWorkflow } from 'agents'

interface BrandState {
  brand_id: string
  strategy: 'growth' | 'maintain' | 'harvest'
  content_count: number
  last_research_at: number | null
  last_publish_at: number | null
  pending_commands: Command[]
}

export class BrandAgent extends Agent<Env, BrandState> {
  // Called once when the agent is first created
  async onStart() {
    // Schedule recurring strategy review
    this.schedule('strategy-review', '0 */6 * * *')  // Every 6 hours
    // Schedule content check
    this.schedule('content-check', '*/30 * * * *')    // Every 30 minutes
  }

  // React to incoming events
  async onMessage(message: { type: string; payload: any }) {
    switch (message.type) {
      case 'research.completed': {
        // Research finished — decide what to do with the results
        const { keyword_count, brand_id } = message.payload
        this.setState({
          ...this.state,
          last_research_at: Date.now(),
        })

        // If growth strategy, immediately trigger content generation
        if (this.state.strategy === 'growth' && keyword_count > 10) {
          await this.emitCommand({
            type: 'GENERATE_CONTENT',
            target_queue: 'CONTENT_QUEUE',
            payload: { brand_id, keyword_count, priority: 'high' },
          })
        }
        break
      }

      case 'content.published': {
        this.setState({
          ...this.state,
          content_count: this.state.content_count + 1,
          last_publish_at: Date.now(),
        })

        // Log to agent's SQLite for historical analysis
        this.sql.exec(
          'INSERT INTO events (type, data, ts) VALUES (?, ?, ?)',
          'content.published', JSON.stringify(message.payload), Date.now()
        )
        break
      }

      case 'budget.warning': {
        // Budget getting tight — switch strategy
        if (this.state.strategy === 'growth') {
          this.setState({ ...this.state, strategy: 'maintain' })
        }
        break
      }
    }
  }

  // Scheduled tasks
  async onSchedule(name: string) {
    switch (name) {
      case 'strategy-review': {
        // Load historical data from agent's SQLite
        const recentEvents = this.sql.exec(
          'SELECT * FROM events WHERE ts > ? ORDER BY ts DESC',
          Date.now() - 86_400_000
        ).toArray()

        // Analyze performance and adjust strategy
        const publishes = recentEvents.filter(e => e.type === 'content.published')
        if (publishes.length < 3 && this.state.strategy === 'growth') {
          // Not enough output — trigger research
          await this.emitCommand({
            type: 'RESEARCH_KEYWORDS',
            target_queue: 'RESEARCH_QUEUE',
            payload: { brand_id: this.state.brand_id, depth: 'deep' },
          })
        }
        break
      }

      case 'content-check': {
        // Check if content pipeline is stalled
        const hoursSincePublish = this.state.last_publish_at
          ? (Date.now() - this.state.last_publish_at) / 3_600_000
          : Infinity

        if (hoursSincePublish > 24) {
          // Pipeline stalled — spawn diagnostic workflow
          await this.runWorkflow('PipelineDiagnostic', {
            brand_id: this.state.brand_id,
            last_publish_at: this.state.last_publish_at,
          })
        }
        break
      }
    }
  }

  // Emit commands to Queues — agent never does the work itself
  private async emitCommand(command: Command) {
    await this.env[command.target_queue].send({
      source: `brand-agent:${this.state.brand_id}`,
      type: command.type,
      payload: command.payload,
      timestamp: Date.now(),
    })
  }
}
```

### Agent Workflow for Multi-Step Processes

```typescript
export class PipelineDiagnostic extends AgentWorkflow<Env, DiagnosticParams> {
  async run(event, step) {
    const status = await step.do('check-services', async () => {
      const results = await Promise.allSettled([
        this.env.RESEARCH_SERVICE.fetch('https://research/health'),
        this.env.CONTENT_SERVICE.fetch('https://content/health'),
        this.env.PUBLISH_SERVICE.fetch('https://publish/health'),
      ])
      return results.map((r, i) => ({
        service: ['research', 'content', 'publish'][i],
        healthy: r.status === 'fulfilled' && (r.value as Response).ok,
      }))
    })

    // Update the agent that spawned this workflow
    await step.do('report-back', async () => {
      const agentId = this.env.BRAND_AGENT.idFromName(event.payload.brand_id)
      const agent = this.env.BRAND_AGENT.get(agentId)
      await agent.fetch('https://agent/diagnostic-result', {
        method: 'POST',
        body: JSON.stringify({ status, checked_at: Date.now() }),
      })
    })
  }
}
```

The pattern: **schedule -> check state -> emit commands -> react to events -> update state**. The agent is a decision-maker. It never fetches URLs, transforms data, or writes to databases directly. It decides what should happen and tells Queues to make it happen.

---

## The Meta-Pattern: Control Plane / Data Plane

All four patterns share the same architecture. Once you see it, you can't unsee it.

```
┌─────────────────────────────────────────────────────┐
│              CONTROL PLANE (Durable Object)          │
│                                                      │
│  Self-scheduling via alarm()                         │
│  Budget management (hard/soft caps)                  │
│  Model routing (tier selection by budget headroom)   │
│  Adaptive scheduling (success rate → batch + interval)│
│  Config from DB (dashboard changes = live, no redeploy)│
│  Telemetry: cost, tokens, duration, success/fail     │
│  HTTP API: /status, /start, /stop                    │
│                                                      │
└────────────────────┬────────────────────────────────┘
                     │
                     │ picks processor, provides context
                     ▼
┌─────────────────────────────────────────────────────┐
│              DATA PLANE (Processor Registry)          │
│                                                      │
│  Pure functions: registerProcessor(name, fn)          │
│  ctx = { db, r2, config, selectModel, budget }       │
│  No awareness of other processors                    │
│  Communicate only through the artifact store         │
│                                                      │
└────────────────────┬────────────────────────────────┘
                     │
                     │ reads and writes
                     ▼
┌─────────────────────────────────────────────────────┐
│              ARTIFACT STORE (D1 + R2)                │
│                                                      │
│  The only shared state                               │
│  Every operation writes back to the store            │
│  Store compounds over time — system gets smarter     │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**Control Plane** (Durable Object) decides *what* to run and *when*. It manages scheduling, budgets, model selection, and observability. It holds ephemeral coordination state.

**Data Plane** (Processor Registry) does the *work*. Pure functions that receive a context object and return results. No coordination logic. No awareness of budgets or scheduling.

**Artifact Store** (D1 + R2) is the *memory*. Every processor reads from it and writes back to it. The store compounds — each cycle adds data that makes the next cycle smarter. D1 for structured data, R2 for blobs.

This separation means:
- Swap processors without touching coordination logic
- Change scheduling without touching processors
- Add observability at the control plane without modifying anything else
- Test processors in isolation with mock contexts

---

## Design Principles

**1. DO owns coordination, not data.**
Heavy data lives in D1 and R2. The DO holds scheduling state, metrics, and config. If a DO's storage were wiped, it should be able to reconstruct from D1.

**2. Alarm replaces cron.**
DOs self-schedule via `alarm()`. Cron only exists as a bootstrap safety net — if a DO somehow stops scheduling itself, the cron trigger restarts it. The DO is the primary scheduler.

**3. Single-threaded = no races.**
All mutations in a DO are serialized. Two requests never execute concurrently. This eliminates an entire class of bugs — no locks, no optimistic concurrency, no retry-on-conflict loops.

**4. Hibernation-aware.**
A DO can be evicted from memory at any time. On wake, reconstruct ephemeral state from storage. Never assume in-memory variables persist across alarm invocations. Always `loadState()` at the top of `alarm()` and `fetch()`.

**5. Small state surface.**
Keep DO storage small. Metrics windows, config caches, cycle state. Not item lists, not content blobs, not user data. If it grows unbounded, it belongs in D1.

**6. Config from DB.**
Every tick reloads config from D1. A dashboard writes to D1. The DO picks it up on the next tick. No redeployment required to change batch sizes, intervals, model selections, or enabled/disabled flags.

**7. Fallback gracefully.**
If a DO is unavailable (rare, but possible during migrations), fall back to direct DB operations. The system degrades to "no adaptive scheduling" rather than "no processing."

---

## Layer 0 vs Layer 2: Raw DO vs Agents SDK

| | Layer 0 (Raw DurableObject) | Layer 2 (Agent class) |
|---|---|---|
| **Storage** | `this.ctx.storage` (KV + SQL) | `this.sql` (SQL) + `this.state` (synced KV) |
| **Scheduling** | `this.ctx.storage.setAlarm()` | `this.schedule()` (cron + one-time) |
| **WebSocket** | Manual `webSocketMessage()` | Built-in, auto state sync |
| **Workflows** | Separate `WorkflowEntrypoint` | `this.runWorkflow()` integration |
| **Real-time UI** | Build it yourself | State changes push to connected clients |
| **Boilerplate** | More (table init, routing) | Less (built-in SQL, scheduling) |
| **Control** | Maximum | Opinionated (good defaults) |

**Use Layer 0 when:**
- No UI consumers (internal coordination only)
- You need maximum control over storage layout
- You want minimal dependencies
- Simple alarm loop with HTTP API (Patterns 1-3)

**Use Layer 2 when:**
- Real-time dashboard needs to show state
- Multiple scheduling patterns (cron + delays)
- Workflow integration is core to the design
- Per-entity agents with lifecycle management (Pattern 4)

Both layers run on the same infrastructure. An Agent *is* a Durable Object with extra features. You can start with Layer 0 and migrate to Layer 2 when you need its capabilities.

---

## Wrangler Configuration

### Pattern 1: Adaptive Controller

```jsonc
// wrangler.jsonc
{
  "name": "adaptive-controller",
  "main": "src/index.ts",
  "compatibility_date": "2025-12-01",
  "d1_databases": [
    { "binding": "DB", "database_name": "scraper-db", "database_id": "..." }
  ],
  "durable_objects": {
    "bindings": [
      { "name": "CONTROLLER", "class_name": "AdaptiveController" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["AdaptiveController"] }
  ],
  // Safety net: restart controller if it stops self-scheduling
  "triggers": {
    "crons": ["*/5 * * * *"]
  }
}
```

### Pattern 2: Storage Sidecar

```jsonc
// wrangler.jsonc
{
  "name": "pipeline-engine",
  "main": "src/index.ts",
  "compatibility_date": "2025-12-01",
  "durable_objects": {
    "bindings": [
      { "name": "PIPELINE_SIDECAR", "class_name": "PipelineSidecar" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["PipelineSidecar"] }
  ],
  "workflows": [
    { "name": "content-pipeline", "binding": "CONTENT_PIPELINE", "class_name": "ContentPipeline" }
  ]
}
```

### Pattern 3: Budget Governor

```jsonc
// wrangler.jsonc
{
  "name": "budget-governor",
  "main": "src/index.ts",
  "compatibility_date": "2025-12-01",
  "d1_databases": [
    { "binding": "DB", "database_name": "governor-db", "database_id": "..." }
  ],
  "r2_buckets": [
    { "binding": "ARTIFACTS", "bucket_name": "artifacts" }
  ],
  "durable_objects": {
    "bindings": [
      { "name": "GOVERNOR", "class_name": "BudgetGovernor" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["BudgetGovernor"] }
  ],
  "triggers": {
    "crons": ["*/5 * * * *"]
  }
}
```

### Pattern 4: Event Reactor (Agents SDK)

```jsonc
// wrangler.jsonc
{
  "name": "brand-agents",
  "main": "src/index.ts",
  "compatibility_date": "2025-12-01",
  "durable_objects": {
    "bindings": [
      { "name": "BRAND_AGENT", "class_name": "BrandAgent" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["BrandAgent"] }
  ],
  "queues": {
    "producers": [
      { "queue": "research-queue", "binding": "RESEARCH_QUEUE" },
      { "queue": "content-queue", "binding": "CONTENT_QUEUE" }
    ],
    "consumers": [
      { "queue": "events-queue", "max_batch_size": 10 }
    ]
  },
  "workflows": [
    { "name": "pipeline-diagnostic", "binding": "PIPELINE_DIAGNOSTIC", "class_name": "PipelineDiagnostic" }
  ]
}
```

---

## Small Patterns That Add Up

### Singleton DO Addressing

One controller for the whole system. Use a fixed name:

```typescript
const id = env.CONTROLLER.idFromName('main')
const controller = env.CONTROLLER.get(id)
await controller.fetch('https://controller/start')
```

### Per-Entity DO Addressing

One DO per entity. Use the entity's natural key:

```typescript
const id = env.BRAND_AGENT.idFromName(brandId)
const agent = env.BRAND_AGENT.get(id)
await agent.fetch('https://agent/event', {
  method: 'POST',
  body: JSON.stringify(event),
})
```

### Alarm Bootstrap from Cron

The cron trigger acts as a safety net. If the DO stops self-scheduling for any reason (bug, exception, redeploy), cron restarts it:

```typescript
export default {
  async scheduled(event: ScheduledEvent, env: Env) {
    const id = env.CONTROLLER.idFromName('main')
    const controller = env.CONTROLLER.get(id)
    // GET /status — if not running, POST /start
    const res = await controller.fetch('https://controller/status')
    const status = await res.json() as { running: boolean }
    if (!status.running) {
      await controller.fetch('https://controller/start', { method: 'POST' })
    }
  }
}
```

### Rolling Metrics Window

Track success/failure rates over a sliding window:

```typescript
private successRate(): number {
  const cutoff = Date.now() - 300_000  // 5 min window
  let completed = 0, total = 0

  for (const [ts, m] of Object.entries(this.state.metrics)) {
    if (Number(ts) < cutoff) continue
    completed += m.completed
    total += m.dispatched
  }

  return total === 0 ? 1.0 : completed / total
}
```

### HTTP Mini-Router Inside fetch()

DOs don't have a router framework. Pattern-match on `pathname`:

```typescript
async fetch(request: Request): Promise<Response> {
  const url = new URL(request.url)
  const { pathname } = url
  const method = request.method

  if (method === 'GET' && pathname === '/status') return this.handleStatus()
  if (method === 'POST' && pathname === '/start') return this.handleStart()
  if (method === 'POST' && pathname === '/stop') return this.handleStop()
  if (method === 'POST' && pathname === '/reset') return this.handleReset()
  if (method === 'POST' && pathname === '/event') return this.handleEvent(request)

  return new Response('not found', { status: 404 })
}
```

### blockConcurrencyWhile for Safe Initialization

Load state exactly once when the DO wakes up:

```typescript
export class Controller extends DurableObject {
  private state!: ControllerState

  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env)
    ctx.blockConcurrencyWhile(async () => {
      const stored = await ctx.storage.get<ControllerState>('state')
      this.state = stored ?? this.defaultState()
    })
  }
}
```

`blockConcurrencyWhile` guarantees no `fetch()` or `alarm()` runs until initialization completes. All subsequent requests see initialized state.

### Dynamic Import Inside DO

Keep the DO class lightweight. Load heavy dependencies only when needed:

```typescript
async alarm() {
  // Only import the processor when we actually need it
  const { processKeywords } = await import('./processors/keywords')
  await processKeywords(this.env, this.state)
}
```

This keeps the DO's initial load fast and memory footprint small during idle periods.

### Location Hints for Regional Pinning

Pin a DO near its data source:

```typescript
const id = env.CONTROLLER.idFromName('main')
const controller = env.CONTROLLER.get(id, {
  locationHint: 'wnam',  // Western North America
})
```

Location hints are suggestions, not guarantees. Useful when your DO primarily talks to a D1 database in a specific region.

---

## Anti-Patterns

| Don't | Do | Why |
|-------|-----|-----|
| Store large data in DO storage | Use D1 for structured data, R2 for blobs | DO storage is for coordination state, not data warehousing |
| Use cron as primary scheduler | Use `alarm()` self-scheduling, cron as safety net | Cron is fixed-interval and can overlap. Alarms are precise and self-adjusting |
| Assume memory persists across requests | Reconstruct from storage on every `alarm()` / `fetch()` | DOs hibernate. In-memory variables vanish |
| Put business logic in the DO | DO coordinates, processors do the work | Keeps DOs testable and processors swappable |
| Use multiple alarm patterns | Single alarm loop, process all due events in one tick | A DO has one alarm. `setAlarm()` overwrites the previous one |
| Ignore CPU limits | One processor per tick, bounded batch sizes | DOs have a 30-second CPU limit per request. Exceed it and the request dies |
| Fan out from inside a DO | Emit to a Queue, let consumers fan out | A DO doing 100 fetches in a loop is fragile and slow |
| Hard-code configuration | Load config from D1 on each tick | Dashboard-driven changes without redeployment |

---

## Comparisons

### vs AWS Lambda + DynamoDB + EventBridge

AWS has no actor model. Lambda is stateless. Coordination requires external state (DynamoDB) with conditional writes for concurrency control. EventBridge provides scheduling but no co-located storage. Every "adaptive controller" pattern requires DynamoDB transactions, conditional updates, and careful handling of concurrent Lambda invocations.

DOs give you single-threaded execution with co-located storage. The concurrency problem doesn't exist.

| Aspect | AWS | Cloudflare DO |
|--------|-----|---------------|
| Concurrency control | DynamoDB conditional writes | Single-threaded (automatic) |
| Scheduling | EventBridge rules | `alarm()` (self-scheduling) |
| State access latency | Network hop to DynamoDB | In-process (co-located) |
| Cold start | Lambda cold start + DynamoDB connection | DO activation (~5ms) |
| Cost model | Per-invocation + DynamoDB RCU/WCU | Per-request + duration |

### vs Azure Durable Entities

Azure Durable Entities are the closest analog. They're virtual actors with single-threaded execution and built-in persistence. The key difference: storage is not co-located. Entity state is stored in Azure Storage Tables, accessed over the network. DOs have SQLite in-process.

Azure also requires the Durable Functions runtime, which adds orchestration overhead. DOs are lighter — just a class with `fetch()` and `alarm()`.

### vs Microsoft Orleans

Orleans pioneered virtual actors. It's the conceptual ancestor of Durable Objects. Differences:

- .NET only (DOs are TypeScript/JavaScript)
- Self-managed clustering (DOs are fully managed)
- Grain persistence is pluggable but always networked (DO SQLite is co-located)
- Orleans has grain timers and reminders (similar to `alarm()`)
- Orleans runs on VMs you manage (DOs run on Cloudflare's edge)

If you're building in .NET and manage your own infrastructure, Orleans is battle-tested. If you want serverless with zero infrastructure, DOs.

### vs Temporal Activities

Temporal is a workflow engine, not an actor framework. It orchestrates activities (functions) with durable execution guarantees. Temporal Workers poll for tasks; they don't hold state between invocations.

For the patterns in this article, Temporal would model each as a workflow with activities. The "adaptive controller" would be a long-running workflow with a `ContinueAsNew` loop. It works, but it's heavier infrastructure — you run Temporal Server (or use Temporal Cloud) plus worker processes.

DOs are lighter for actor-style coordination. Temporal is better for complex multi-step orchestration with human approvals and long waits. Cloudflare Workflows (built on DOs) cover simpler workflow needs without external infrastructure.

### vs Akka Actors

Akka (now Apache Pekko) is the JVM actor framework. It provides:

- Actor hierarchies with supervision
- Cluster sharding for distributed actors
- Event sourcing and CQRS built in
- Persistence via plugins (Cassandra, JDBC, etc.)

Akka is powerful but complex. Cluster management, split-brain resolution, serialization — all your problem. DOs abstract all of this away. You write a class, deploy it, and Cloudflare handles placement, routing, and persistence.

Akka wins on: maturity, ecosystem, event sourcing patterns, supervision trees.
DOs win on: simplicity, zero ops, global distribution, serverless pricing.

### vs @cloudflare/actors

The [`@cloudflare/actors`](https://github.com/nichochar/actors) library builds on raw DOs to provide a typed RPC interface, lifecycle hooks, and state management. It's a convenience layer — still Durable Objects underneath.

Use `@cloudflare/actors` when you want typed actor methods instead of HTTP routing. Use the Agents SDK when you need WebSocket state sync, scheduling, and workflow integration. Use raw DOs when you want maximum control and minimal abstraction.

---

## References

### Cloudflare Platform
- [Cloudflare Durable Objects Documentation](https://developers.cloudflare.com/durable-objects/)
- [Rules of Durable Objects](https://developers.cloudflare.com/durable-objects/best-practices/rules-of-dos/) — Best practices guide (Dec 2025)
- [Durable Objects Alarms](https://developers.cloudflare.com/durable-objects/best-practices/alarms/)
- [Durable Objects Storage API](https://developers.cloudflare.com/durable-objects/api/storage-api/) — Transactional KV and SQLite storage
- [Cloudflare Agents SDK](https://developers.cloudflare.com/agents/) — Stateful agent framework built on Durable Objects
- [Cloudflare Agents SDK GitHub](https://github.com/cloudflare/agents) — Source code and examples
- [Cloudflare Workflows](https://developers.cloudflare.com/workflows/) — Durable multi-step execution engine
- [Cloudflare Queues](https://developers.cloudflare.com/queues/) — Message passing between Workers
- [`@cloudflare/actors` Library](https://github.com/nichochar/actors) — Actor model abstraction over DOs
- [Building Real-Time Games with Workers and Durable Objects](https://blog.cloudflare.com/building-real-time-games-using-workers-durable-objects-and-unity/) — Game state management with DOs
- [Durable Objects: Easy, Fast, Correct — Choose Three](https://blog.cloudflare.com/durable-objects-easy-fast-correct-choose-three/) — Architecture deep dive
- [Introducing Workers Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/) — Original announcement and design rationale

### Companion Articles
- [Event-Driven Architecture on Cloudflare Workers](https://github.com/garywu/cloudflare-event-driven-architecture) — Queues, fan-out, idempotency, outbox pattern
- [Composable Processor Architecture](https://github.com/garywu/composable-processor-architecture) — 7 processor types, artifact store, budget management

### Equivalent Systems (Comparisons)
- [Microsoft Orleans — Virtual Actors](https://learn.microsoft.com/en-us/dotnet/orleans/overview) — .NET virtual actor framework, closest equivalent to DO model
- [Azure Durable Entities](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-entities) — Stateful entities in Azure Durable Functions
- [Temporal](https://temporal.io/) — Durable execution platform with workflow-as-code
- [Akka Actors](https://doc.akka.io/concepts/index.html) — JVM actor model (original inspiration for many stateful compute patterns)

### Architecture Patterns
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) — Foundational messaging patterns (Hohpe & Woolf)
- [Microservices Patterns](https://microservices.io/patterns/) — Chris Richardson's pattern catalog

---

*Production patterns extracted from systems processing thousands of items per hour on Cloudflare's developer platform.*
