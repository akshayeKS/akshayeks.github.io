---
layout: default
title: "Airbox: Durable Infrastructure for AI Agents on Your Existing Airflow"
date: 2026-01-29
description: "Building the missing layer between AI agents and production reality—durability, human oversight, and observability on top of Airflow."
tags: [ai, agents, airflow, infrastructure, python]
categories: [engineering, ai]
---

# Airbox: Durable Infrastructure for AI Agents on Your Existing Airflow

We're building Airbox—a platform that makes AI agents production-ready by giving them durability, human oversight, and observability. And we're doing it on top of the Airflow infrastructure you already run.

---

## The Vision

Here's what running AI agents in production actually looks like today:

Your agent processes customer contracts for 6 hours. It crashes at hour 5. You start over from scratch. A procurement workflow needs VP approval mid-execution—you hack together a polling system. Your team launches 200 parallel agents. Three hours later, you've burned through your LLM budget with zero visibility into what happened.

**Agents aren't stateless microservices.** They run for hours. They fail. They need human input. They make decisions that need audit trails. And the infrastructure to support all of this? It doesn't exist.

That's what Airbox is: **the missing layer between your AI agents and production reality.**

---

## Why Airflow?

We made a deliberate choice: build *on* Airflow, not beside it.

Enterprises already run Airflow. It handles their ETL, ML pipelines, and scheduled jobs. Security policies are configured. Compliance is sorted. Platform teams know how to operate it.

The last thing anyone needs is another orchestrator to learn, deploy, and maintain.

Airbox is a Python package. You `pip install` it into your existing Airflow cluster. Your agent workflows compile to standard Airflow DAGs. Zero migration. Zero new infrastructure. Just new capabilities for AI workloads.

```
┌─────────────────────────────────────────────────────────────────┐
│                    YOUR EXISTING AIRFLOW                        │
│       (security, compliance, executors, monitoring)             │
├─────────────────────────────────────────────────────────────────┤
│                      AIRBOX LAYER                               │
│   Workflows → DAGs    |    HITL    |    Observability          │
│   Checkpoints/Resume  |  Swarms    |    Decision Lineage       │
└─────────────────────────────────────────────────────────────────┘
```

---

## What Airbox Does

### 1. Workflows That Survive Failures

Write agent workflows in plain Python. Each step is automatically checkpointed—if your workflow crashes and restarts, it resumes from where it left off.

```python
@workflow(workflow_id="process_order", trigger=on_event("order.created"))
def process_order(event):
    order = step.run("validate", validate_order, event.data)
    inventory = step.run("check_stock", check_inventory, order)
    payment = step.run("charge", process_payment, order)
    return {"status": "completed", "order_id": order.id}
```

No more losing 5 hours of work to a network timeout. Every completed step is cached and replayed on restart.

### 2. Human-in-the-Loop, Built In

Agents shouldn't make every decision alone. When you need human judgment—approvals, reviews, escalations—Airbox pauses the workflow and delivers the request where people actually work: **Slack and Email**.

```python
if order.total > 10000:
    approved = approval(
        subject=f"Large order: ${order.total}",
        assigned_to=["finance@company.com"],
        channel="slack",
        timeout=timedelta(hours=24),
    )
    if not approved:
        return {"status": "needs_review"}
```

The workflow waits without consuming resources. When the human responds, it picks up exactly where it left off.

### 3. Parallel Agents at Scale

Process thousands of items concurrently. Airbox handles fan-out, fan-in, and coordination—including shared budgets so you don't blow through API limits.

```python
results = await step.parallel(
    *[analyze_document(doc) for doc in documents],
    max_concurrency=50,
    token_budget=500_000,
)
```

If some agents fail, you get partial results with clear error reporting. No all-or-nothing failures.

### 4. Token Tracking and Cost Control

Know exactly what you're spending. Airbox tracks every LLM call—tokens used, cost incurred, latency observed—broken down by workflow, by agent, by customer.

Set budgets. Get alerts before you hit them. No more surprise bills.

### 5. Decision Lineage for Compliance

"Why did the AI reject this claim?"

Every decision your agent makes is traced: which tools it called, what data it saw, what the LLM returned, whether a human approved it. A complete audit trail for compliance, debugging, and continuous improvement.

### 6. 500+ Tool Integrations

Agents need to interact with the world—GitHub, Slack, Salesforce, databases, internal APIs. Airbox integrates with [Composio](https://composio.dev) for 500+ external services, plus a simple decorator for your internal tools:

```python
@tool
def lookup_customer(customer_id: str) -> Customer:
    return db.customers.get(customer_id)
```

---

## Under the Hood

Building Airbox means solving real distributed systems problems. A few highlights:

**Deterministic Replay.** When a workflow restarts, completed steps must return their cached results—even if external state has changed. This requires careful step identification and checkpoint storage that handles edge cases like non-deterministic inputs.

**Worker Efficiency.** A human approval might take 24 hours. You can't hold an Airflow worker hostage the entire time. We use Airflow's deferrable operators to pause workflows, release workers, and resume only when the response arrives.

**Event Routing.** When thousands of events flow through the system, efficiently matching them to the right workflows—with wildcards, filters, and exactly-once delivery—requires thoughtful indexing and routing infrastructure.

**Slack Reliability.** Delivering messages to Slack sounds simple. Handling webhook retries, out-of-order delivery, modal timeouts, rate limits, and user ID mismatches? That's where the engineering lives.

We're building on Airflow's battle-tested scheduler while adding the primitives it doesn't have—durability, HITL, observability—specifically designed for AI workloads.

---

## What We're Not Building

Focus matters. Here's what we've deliberately left out:

- **LLM hosting** — Use OpenAI, Anthropic, or your own models
- **Vector databases** — Use Pinecone, Weaviate, or whatever you prefer
- **500 tool integrations** — Composio handles this better than we could
- **Another orchestrator** — Airflow already exists and works

Airbox is the agent layer. Not the full stack.

---

## Current Status

The core SDK is working: `@workflow`, `step.run()`, checkpoints, event triggers, HITL primitives. We have 41 tests passing and example workflows running.

**What's next:**
- Slack and Email delivery for approvals (real notifications, not webhooks)
- Console UI for monitoring and configuration
- Token tracking dashboards
- Production-ready checkpoint storage
- Swarm coordination for large-scale parallel execution

---

## Who This Is For

Airbox is for teams that:

- Already run Airflow and don't want to migrate
- Are deploying AI agents that need human oversight
- Need audit trails for compliance
- Want cost visibility before the bill arrives
- Process high volumes with parallel agents

If you're building agents that run for more than a few seconds, talk to humans, or need to be debugged when things go wrong—we're building the infrastructure layer you need.

---

## Get Involved

We're looking for design partners—teams running AI agents on Airflow who want to help shape what we build.

If that sounds like you, reach out. We'd love to learn what's breaking in your setup and build the right solutions together.

---

*Airbox: Durable agents on the Airflow you already run.*

---

[← Back to Blog](/blog) ・ [Home](/) ・ [Contact](/contact)
