+++
date = '2026-09-10T11:00:00+08:00'
draft = false
title = 'Securing Non-Human Identities: Zero-Trust Attestation for Autonomous AI Agents'
summary = "When an AI agent calls your internal API, how do you know it's actually your agent — and not something that hijacked its session? The answer requires rethinking identity from the ground up."
tags = ["Cybersecurity", "Agentic Systems"]
[cover]
  image = "https://images.unsplash.com/photo-1550751827-4bd374c3f58b?w=1200&q=80"
  alt = "Digital security concept with glowing padlock and circuit board pattern"
+++

There's a class of problem that shows up when you deploy autonomous agents into production and no one talks about it until something goes wrong: **agent identity**.

Your agent calls an internal payments API. The API sees a valid bearer token. But is that request really from your agent? Did the agent decide to make that call, or did a malicious instruction injected into a retrieved document trick it into calling it? Did the token get stolen by a different process running in the same container?

Traditional identity systems weren't designed for this. A service account is issued to a workload. A workload is run by a human who broadly understands what it does. Agents are different: they make decisions dynamically, they consume untrusted content, and their actions are driven by model outputs that no human explicitly approved.

## The Non-Human Identity Problem

Human identity is hard. Non-human identity — service accounts, API keys, CI/CD tokens — is a known-solved problem with well-understood patterns: short-lived credentials, workload identity federation, least-privilege scopes.

Agents break several assumptions that make those patterns work:

**1. Behaviour is not pre-determined.** A service account for a data pipeline does exactly what the pipeline code does. An agent's actions depend on runtime model outputs. The identity system has no prior knowledge of what the agent will do.

**2. Agents consume untrusted input.** An agent tasked with summarising customer emails processes attacker-controlled content. Prompt injection attacks can trick the agent into making API calls it was never intended to make — but using its legitimate credentials.

**3. Agent sessions are long-lived and stateful.** An agent handling a complex workflow may hold a session open for minutes to hours. Stolen session credentials have a wide exploitation window.

**4. Multi-agent systems multiply the attack surface.** When one agent spawns sub-agents, how does the sub-agent verify the spawning agent is legitimate? How does a downstream service know which root workflow a request traces back to?

## A Zero-Trust Framework for Agent Identity

The core principle: **every action by an agent must be attributable, constrained, and independently verifiable — not just at session start, but at every operation.**

### Layer 1: Workload Attestation

Before an agent gets any credentials, the infrastructure must attest to the integrity of the execution environment. This is the same principle behind TPM-based boot attestation and cloud provider workload identity.

In practice:

```
Agent Process Startup
    ↓
Request attestation token from cloud metadata service
(GCP: service account token with workload identity)
(AWS: instance identity document + STS)
    ↓
Exchange attestation token for short-lived agent credential
(15-minute lifetime, single-audience scope)
    ↓
Agent operational with credential
```

The key constraint: the credential is scoped to a specific **audience** (the set of services this agent is permitted to call) derived from the agent's declared role in your agent registry. An orchestrator agent cannot obtain credentials scoped to payment APIs.

### Layer 2: Fine-Grained Operation Authorization

A credential that authorises an agent to call `POST /transfer` is too broad. The agent should present context about the *reason* for each call.

Implement operation-level authorisation using **structured audit claims** embedded in the request:

```http
POST /internal/payments/transfer
Authorization: Bearer <agent-credential>
X-Agent-Workflow-Id: wf_8f3a2b1c
X-Agent-Step-Id: step_12
X-Agent-Intent: "Refund transaction tx_abc123 per user request ref:ticket_789"
X-Agent-Trace-Parent: <opentelemetry-traceparent>
```

The payments API validates:
1. The bearer credential (standard JWT validation)
2. The workflow ID maps to an active, non-cancelled workflow session
3. The intent field matches the allowed intents for this operation (checked against a policy)
4. The trace parent is a real, active span in your telemetry system

This is "intent-aware authorisation." A compromised or prompt-injected agent that calls `POST /transfer` with the wrong workflow ID or mismatched intent gets rejected.

### Layer 3: Prompt Injection Mitigations

No identity scheme fully compensates for an agent that can be tricked into requesting legitimate operations for illegitimate reasons. Defensive layers:

**Instruction/data separation**: Process untrusted content in a separate context that the model is trained to treat as data, not instructions. Modern models support system/user/tool role distinctions that help with this — be explicit and consistent.

**Constrained output schemas**: If an agent's job is to extract information from a document, force its output into a typed schema. An agent that can only output `ExtractedData` cannot emit tool calls through injection.

**Canary tokens**: Embed unique random strings in sensitive contexts. If an agent's outbound request contains a canary token that was only present in protected context (the system prompt, a secrets file), you know it's leaking information it shouldn't.

**Semantic guardrails at the tool layer**: Before executing a tool call, a fast classifier checks whether the tool call is plausibly consistent with the current task context. A refund-processing agent asking to read `/etc/passwd` is anomalous regardless of credentials.

### Layer 4: Multi-Agent Trust Propagation

When agent A spawns agent B, how should downstream services reason about B's authority?

The answer: **delegation chains**, inspired by SPIFFE/SPIRE and certificate delegation.

```
Root Workflow credential (audience: orchestrator)
    ↓ spawns
Sub-Agent credential (audience: {data-access, summarisation})
    Carries: parent_workflow_id, delegating_agent_id, delegation_depth = 1
    ↓ calls
Data Access Service
    Validates: credential valid, delegation chain valid, depth ≤ max_depth
```

Critically: sub-agent credentials should have **fewer permissions** than the parent, never more. A delegation that escalates privilege is a red flag worth blocking at the infrastructure layer.

## Incident Response Implications

When something does go wrong, your ability to respond depends entirely on the quality of your audit trail. Every agent operation should log:

- `workflow_id`, `agent_id`, `step_id` (correlate across services)
- The exact model response that triggered the action (or a hash of it)
- Input and output summaries (not raw content, but enough to reconstruct intent)
- The credential presented and its validation result

With this, you can answer: *"Which agent, in which workflow, made this call, because the model produced what output, in response to what input?"*

That's the bar. Anything less and you're flying blind when you need to understand a security incident.

---

Agent identity is not a solved problem yet — the tooling is young and the attack patterns are still emerging. But the principles from zero-trust architecture apply directly: never trust implicitly, verify continuously, and assume breach at every layer.
