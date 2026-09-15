+++
date = '2026-09-08T10:30:00+08:00'
draft = false
title = 'Event-Driven Distributed Architectures for Real-Time AI Telemetry'
summary = "As AI systems generate massive volumes of structured events, traditional telemetry pipelines collapse under the load. Here's how to design event-driven infrastructure that scales with your AI workloads."
tags = ["System Design", "Infrastructure"]
[cover]
  image = "https://images.unsplash.com/photo-1451187580459-43490279c0fa?w=1200&q=80"
  alt = "Earth at night from space showing illuminated city networks, representing data connectivity"
+++

Production AI systems emit a flood of telemetry: model latency per step, token counts, tool call sequences, embedding retrieval scores, agent decisions, safety filter outcomes. If you're running multi-agent workflows at any meaningful scale, you're looking at millions of structured events per hour.

The challenge isn't storage — cheap. It's **real-time processing**: you need to detect anomalies while the workflow is still running, compute quality signals that feed back into routing decisions, and maintain audit logs with sub-second write durability.

Traditional logging pipelines — fire-and-forget structured logs to a batch-ingestion SIEM — aren't built for this. Here's the architecture that is.

## Why Standard Telemetry Pipelines Break

Consider what happens when you take a standard logging stack (agent → log → Fluentd → Elasticsearch → Kibana) and throw 2M events per hour at it:

- **Batch ingestion lag**: 30-60 seconds before events are queryable. Useless for in-flight anomaly detection.
- **Schema brittleness**: Elasticsearch dynamic mapping breaks when agent tool calls suddenly include a new field.
- **Fan-out costs**: When 10,000 concurrent agent workflows each write 200 events/minute, you need a pipeline that handles 2M writes/min with single-digit ms write paths.
- **No streaming semantics**: Detecting "agent X has made >20 calls in the last 5 seconds" requires a stream processor, not a log search.

The answer is a **streaming-first, schema-registry-backed event backbone** with multiple consumer tiers consuming from the same stream.

## The Core Architecture

```
                   Agents / Model Servers / Tools
                              │
                    ┌─────────▼──────────┐
                    │   Event Collector   │
                    │  (SDK / Sidecar)    │
                    │  - Schema validate  │
                    │  - Batch + compress │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────────────────────┐
                    │         Event Backbone              │
                    │    (Kafka / Pub/Sub / Kinesis)      │
                    │                                     │
                    │  Topics:                            │
                    │  - ai.agent.steps        (high vol) │
                    │  - ai.model.calls        (high vol) │
                    │  - ai.safety.events      (med vol)  │
                    │  - ai.workflow.lifecycle (low vol)  │
                    └──────┬────────────┬────────────────┘
                           │            │
             ┌─────────────▼──┐    ┌────▼──────────────────┐
             │ Stream Processor│    │   Archival Consumer    │
             │ (Flink / Dataflow│   │ (Batch: BigQuery/S3)   │
             │  - Anomaly detect│   │  - Long-term storage   │
             │  - Quality scores│   │  - Cost attribution    │
             │  - Alerts        │   │  - Replay capability   │
             └─────────────────┘    └────────────────────────┘
```

### Event Schema & Registry

Every event type is registered with a schema registry (Confluent Schema Registry, AWS Glue, or Google's Pub/Sub schema library). Schemas use Avro or Protobuf for compact binary serialisation.

```protobuf
message AgentStepEvent {
  string workflow_id    = 1;
  string agent_id       = 2;
  int32  step_number    = 3;
  int64  timestamp_us   = 4;  // microseconds since epoch
  string action_type    = 5;  // tool_call | model_call | decision
  string action_name    = 6;
  bytes  input_hash     = 7;  // sha256 of input, not raw content
  bytes  output_hash    = 8;
  int32  duration_ms    = 9;
  int32  tokens_in      = 10;
  int32  tokens_out     = 11;
  StepStatus status     = 12;
  string error_code     = 13; // empty if success
  map<string, string> labels = 14; // tenant, environment, region
}
```

Schema evolution is handled through backward-compatible additions. The schema registry enforces compatibility before any producer can deploy a new event version.

### The Event Collector: Client-Side SDK

Rather than having every service directly produce to Kafka, implement a thin collector SDK:

```python
class TelemetryCollector:
    """Thread-safe, async-friendly event collector with local buffering."""

    def __init__(self, config: CollectorConfig):
        self._buffer: asyncio.Queue[Event] = asyncio.Queue(maxsize=50_000)
        self._producer = KafkaProducer(config.kafka)

    async def emit(self, event: Event) -> None:
        try:
            self._buffer.put_nowait(event)
        except asyncio.QueueFull:
            # Back-pressure: shed lowest-priority events, never block the agent
            metrics.increment("telemetry.events_shed")

    async def _flush_loop(self) -> None:
        batch = []
        async for event in self._drain_buffer(max_size=500, timeout_ms=100):
            batch.append(event.SerializeToString())
        if batch:
            await self._producer.send_batch("ai.agent.steps", batch)
```

The critical design choices:
- **Never block the agent**: if the buffer is full, shed the event and record the drop
- **Batch micro-aggregation**: 100ms batching window cuts Kafka RPS by 100x
- **Local durability**: for critical events (safety violations, workflow lifecycle), use a write-ahead log before the Kafka send

### Stream Processing: Real-Time Anomaly Detection

This is where the architecture pays off. Apache Flink or Google Dataflow consumes from the event backbone and runs continuous computations:

```sql
-- Detect agents exceeding step budgets in real time
SELECT
  workflow_id,
  agent_id,
  COUNT(*) as step_count,
  SUM(tokens_in + tokens_out) as total_tokens,
  TUMBLE_END(ts, INTERVAL '1' MINUTE) as window_end
FROM agent_steps
GROUP BY
  workflow_id,
  agent_id,
  TUMBLE(ts, INTERVAL '1' MINUTE)
HAVING
  step_count > 30 OR total_tokens > 500000
```

When this fires, the output goes to a `workflow.anomalies` topic. A consumer of that topic can issue a graceful cancellation signal to the offending workflow via the workflow orchestration system — all within seconds of the anomaly starting.

Other real-time computations:
- **Quality score rolling average**: track tool call success rate per agent type, alert if it drops below threshold
- **Safety event correlation**: if safety filter triggers spike for a specific user's workflows, flag for review
- **Cost pacing**: if a tenant is consuming tokens 3x faster than their daily budget allows, throttle their agent creation rate

## Replay and Forensics

One of the strongest arguments for an event-backbone architecture: **every workflow is fully replayable**.

Because every event is durably stored (Kafka with 7-day retention feeding into BigQuery for long-term), you can:

1. Take any `workflow_id` from production
2. Filter all events for that workflow from the archive
3. Replay them through a test environment with a pinned model version
4. Observe the agent's behaviour deterministically

This is invaluable for post-incident analysis and for building evals. The production event stream becomes your ground truth for "what did the system actually do."

## Operational Considerations

**Partition strategy for high-volume topics**: Partition by `workflow_id` — this ensures all events for a workflow land on the same partition, preserving ordering for stream joins and simplifying replay.

**Compaction for lifecycle events**: The `ai.workflow.lifecycle` topic uses log compaction — the current state of each workflow is always queryable without scanning history.

**Dead-letter handling**: Events that fail schema validation go to a `ai.dlq.*` topic rather than being dropped. Alert on DLQ depth — schema validation failures often indicate a producer-side bug.

---

Real-time telemetry for AI workloads is a genuinely interesting infrastructure problem. The event-driven model I've described here has held up well at scale — the key insight is treating observability as a first-class streaming system rather than an afterthought bolted onto logs.
