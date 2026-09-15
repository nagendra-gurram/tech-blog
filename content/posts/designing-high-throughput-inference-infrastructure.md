+++
date = '2026-09-12T09:00:00+08:00'
draft = false
title = 'Designing High-Throughput Inference Infrastructure for Multi-Agent Workflows'
summary = "Serving tens of thousands of concurrent model calls for agent pipelines requires more than spinning up more GPUs. Here's the architecture that actually scales."
tags = ["AI", "Infrastructure", "System Design"]
[cover]
  image = "https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80"
  alt = "Server rack data center with blue LED lights illuminating dense compute hardware"
+++

When you have a single LLM behind a product, inference infrastructure is straightforward: throw requests at an API endpoint, add rate-limit handling, move on. When you move to **multi-agent workflows** — orchestrators spinning up sub-agents, agents calling other models for specialised tasks, retrieval pipelines doing embedding lookups on every turn — the traffic pattern transforms entirely.

Your infrastructure assumptions will break. Here's what I've learned designing systems that handle this gracefully.

## Why Multi-Agent Inference Is Different

In a simple chat product, you have roughly one model call per user turn. In an agent workflow:

- The orchestrator calls a planner model
- The planner spawns three sub-agents
- Each sub-agent makes 2-8 tool calls, each of which may call an embedding model
- Some sub-agents call a vision model for document parsing
- Results flow back to a synthesis model
- The orchestrator may decide to retry one sub-agent with different context

A single user request can trigger **30-100 model invocations**, with complex dependency graphs and variable latency at each node. The fan-out is non-trivial, the dependency graph is a DAG not a chain, and any single node's latency affects overall completion time.

## Model Serving Tiers

Not all inference calls deserve the same treatment. A practical tiering model:

```
Tier 0 — Orchestrator / Planning      (large model, low concurrency, <5 RPS per workflow)
Tier 1 — Task Execution               (medium model, high concurrency, ~50 RPS)
Tier 2 — Embedding / Classification   (small/fast model, very high concurrency, ~500 RPS)
```

Each tier has different requirements:

| Tier | Latency target | Batch-friendly | KV-cache benefit | Priority |
|---|---|---|---|---|
| 0 — Orchestrator | p99 < 8s | No | High (long system prompts) | High |
| 1 — Execution | p99 < 4s | Moderate | Medium | Medium |
| 2 — Embedding | p99 < 300ms | Yes | N/A | Low |

Mixing these on the same serving cluster is a mistake I see constantly. Long orchestrator calls starve embedding requests, tanking retrieval latency for all active agents.

## The Dispatcher Architecture

```
                     ┌──────────────────────────────┐
                     │         API Gateway           │
                     │   (auth, rate limit, routing) │
                     └────────────┬─────────────────-┘
                                  │
                     ┌────────────▼─────────────────-┐
                     │         Dispatcher             │
                     │  - Priority queue per tier     │
                     │  - Adaptive batching           │
                     │  - Timeout / deadline aware    │
                     └──────┬──────────┬─────────────┘
                            │          │
               ┌────────────▼──┐  ┌────▼───────────────┐
               │  Tier 0 Pool  │  │   Tier 1/2 Pool     │
               │  (GPU A100x4) │  │  (GPU H100x8 batch) │
               └───────────────┘  └─────────────────────┘
```

The dispatcher is the critical component. Responsibilities:

**1. Priority-aware queuing**
Every request carries a deadline derived from the parent workflow's SLA. Requests with earlier deadlines get dispatched first within a tier. This prevents a slow background job from blocking a latency-sensitive interactive workflow.

**2. Adaptive continuous batching**
For Tier 1 and 2, requests with similar prompt lengths get co-batched to maximise GPU utilisation. We use a 50ms batching window — beyond that, we dispatch whatever we have.

**3. Speculative execution**
For workflows where sub-agent results are needed jointly, dispatch all sub-agents concurrently and cancel stragglers once enough results have arrived. For some tasks, 4 out of 5 sub-agents completing is sufficient.

## KV-Cache Strategy

For orchestrator-tier calls, system prompts are often thousands of tokens long and identical across thousands of agent invocations. Prefix caching — where the model server reuses the computed KV cache for the shared prefix — can cut time-to-first-token by 60-70% for these calls.

Practical requirements:
- System prompts and few-shot examples must be **identical bytes** — even a trailing space busts the cache
- Store canonical system prompts in a versioned registry; all agents load by ID, not inline text
- Cache warm-up: on deploy, send a batch of requests with the new system prompt before routing production traffic

vLLM's prefix caching and Google's TPU-backed serving both support this. The win is substantial enough to justify the operational complexity.

## Handling Back-pressure

When the model cluster is saturated, naive behaviour is to queue everything until requests time out. Better approach: **propagate back-pressure upstream and make smart shedding decisions early**.

```python
class InferenceClient:
    async def generate(self, request: InferenceRequest) -> str:
        if self.queue_depth > HIGH_WATERMARK:
            if request.priority < Priority.HIGH:
                raise BackpressureError("Queue saturated, shedding low-priority request")

        deadline = request.created_at + timedelta(seconds=request.timeout_s)
        async with asyncio.timeout_at(deadline.timestamp()):
            return await self._dispatch(request)
```

Workflows should handle `BackpressureError` by retrying with exponential backoff or degrading gracefully (e.g., skipping optional enrichment steps and continuing with partial results).

## Cost Governance

Multi-agent workflows can silently balloon token costs. A single misconfigured retry policy in a sub-agent can multiply your bill by 10x overnight.

Essential guardrails:
- **Per-workflow token budget**: set hard limits on total input+output tokens for any workflow instance
- **Anomaly alerting**: alert on any workflow instance consuming >3σ above the rolling mean
- **Daily cost attribution**: tag every inference call with `workflow_id`, `agent_type`, `tenant_id` — push this into your cost reporting pipeline

We use OpenTelemetry attributes for this; they flow naturally into whatever observability backend you're using.

---

Getting inference infrastructure right for multi-agent systems is genuinely hard. The patterns above took several failed iterations to arrive at. The TL;DR: tier your models, build a smart dispatcher, aggressively use KV cache, and instrument everything.

Next up: memory systems for long-running agents.
