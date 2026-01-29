---
layout: default
title: "Building Production-Ready Migration Agents with Claude Code"
date: 2026-01-29
description: "A deep dive into designing agentic systems for complex, multi-phase infrastructure migrations - with lessons from orchestrating 66 Airflow clusters across 4 waves."
tags: [ai, agents, claude, migrations, infrastructure, airflow]
categories: [engineering, ai]
---

# Building Production-Ready Migration Agents with Claude Code

*A deep dive into designing agentic systems for complex, multi-phase infrastructure migrations - with lessons from orchestrating 66 Airflow clusters across 4 waves.*

---

## Introduction

Infrastructure migrations are a paradox: they're simultaneously repetitive (run the same steps for each resource) and complex (each resource has unique edge cases). This makes them ideal candidates for AI-assisted automation - but also reveals the limits of simple "chat with AI" approaches.

This post explores how to build **migration agents** - AI systems that can orchestrate complex, multi-phase migrations with:

- State persistence across sessions
- Human-in-the-loop checkpoints for critical decisions
- Resume capability when things go wrong
- Audit trails for compliance and debugging

We'll use a real-world example - migrating 66 Airflow clusters from v2 to v3 - to illustrate the patterns. But these principles apply to any migration: database upgrades, cloud provider switches, framework version bumps, or monolith-to-microservice decompositions.

---

## Why Migrations Need Agents, Not Scripts

### The Traditional Approach

Most migrations start with a script:

```bash
#!/bin/bash
for cluster in $(cat clusters.txt); do
  create_pr $cluster
  wait_for_review
  merge_pr
  deploy $cluster
  verify $cluster
done
```

This works until:

- The script crashes at cluster 47 - where do you resume?
- A PR needs changes - how do you track that?
- A deploy fails in prod - do you rollback or skip?
- You need to pause for a week - where were you?

### The Agent Approach

An agent maintains state and context:

```yaml
cluster: email.notifications
status: deploy_pending
current_step: 3
history:
  - timestamp: 2024-01-15T10:00:00Z
    action: pr_created
    pr_number: 12345
  - timestamp: 2024-01-15T14:30:00Z
    action: pr_merged
awaiting: manual_helm_deploy
```

When you return after a week, the agent knows exactly where you left off and what needs to happen next.

---

## Architecture: The Orchestrator Pattern

The key insight is separating **orchestration** from **implementation**:

```
┌─────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR AGENT                        │
│  - Tracks state across all resources                         │
│  - Determines next action for each resource                  │
│  - Manages human checkpoints                                 │
│  - NEVER modifies files directly                             │
└──────────────────────┬──────────────────────────────────────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
    ┌─────────┐   ┌─────────┐   ┌─────────┐
    │  Skill  │   │  Skill  │   │  Skill  │
    │ Infra   │   │  DAG    │   │ Deploy  │
    │ Setup   │   │ Migrate │   │ Verify  │
    └─────────┘   └─────────┘   └─────────┘
```

The orchestrator is a **coordinator**, not an implementer. It knows:

- What needs to happen
- In what order
- What's blocking progress

But it delegates the actual work to specialized sub-skills.

### Why This Separation Matters

1. **Testability**: Each skill can be tested independently
2. **Reusability**: The infra setup skill works for any cluster
3. **Debuggability**: When something fails, you know which skill failed
4. **Evolvability**: Change the migration process without rewriting skills

---

## State Management: The Heart of Migration Agents

### Choosing Your State Store

| Option | Pros | Cons |
|--------|------|------|
| Git branch | Versioned, auditable, collaborative | Merge conflicts, requires commits |
| Database | Fast queries, transactions | Another dependency, no versioning |
| File system | Simple, fast | No collaboration, lost on restart |
| External service | Managed, scalable | Latency, cost, another dependency |

For migrations, **git-based state** is often ideal:

- Natural audit trail (commit history)
- Collaboration (multiple engineers can work on different clusters)
- Review process (PRs for state changes)
- Recovery (revert to previous state)

### State Schema Design

Design your state schema around these questions:

1. What's the unit of work? (cluster, service, table, etc.)
2. What phases does each unit go through?
3. What data do you need at each phase?
4. What are the human checkpoints?

Example schema for a migration with multiple phases and environments:

```yaml
# Per-resource state file
resource:
  id: email.notifications
  wave: 2
  environments:
    - test
    - prod-us
    - prod-eu
  metadata:
    complexity: high
    owner: platform-team

# Shared across environments (code changes)
code_migration:
  status: completed  # pending | in_progress | completed
  pr_number: 12345
  pr_url: https://github.com/org/repo/pull/12345

# Per-environment tracking
environment_status:
  test:
    status: migrate_pending
    is_primary: true  # Run here first
    phases:
      infra:
        status: completed
        completed_at: 2024-01-15T10:00:00Z
      deploy:
        status: completed
        workflow_run_id: 12345678
      migrate:
        status: in_progress
        items_completed: 5
        items_total: 12
      verify:
        status: pending

  prod-us:
    status: blocked  # Blocked until test completes
    is_primary: false
    # ... same structure

# Audit trail
history:
  - timestamp: 2024-01-15T10:00:00Z
    action: infra_pr_created
    actor: migration-agent
    details: "PR #12345 created"
```

### State Transition Rules

Define explicit rules for state transitions:

```
pending → in_progress → completed
                     ↘ failed → blocked

Key rules:
- test environment must complete before prod environments unblock
- each phase must complete before next phase starts
- failed states require human intervention to unblock
```

---

## Human-in-the-Loop: Where Automation Meets Reality

### Identifying Checkpoint Types

Not all human checkpoints are equal:

| Type | Example | Automation Level |
|------|---------|------------------|
| Approval | "PR needs review" | Agent creates, human approves |
| Confirmation | "Did you run pacman apply?" | Human does, tells agent |
| Decision | "Which DAGs to migrate?" | Human chooses, agent executes |
| Intervention | "Deploy failed, please fix" | Human fixes, agent resumes |

### Designing Resume Commands

Each checkpoint needs a corresponding resume command:

```
Checkpoint: PR merged
Resume: /migrate pr-merged <resource>
Agent action: Update state, proceed to next phase

Checkpoint: Manual deploy done
Resume: /migrate deploy-confirmed <resource> <environment>
Agent action: Verify deployment, proceed to migration

Checkpoint: Test environment verified
Resume: /migrate test-done <resource>
Agent action: Unblock prod environments
```

### The Checkpoint Prompt Pattern

When reaching a checkpoint, the agent should:

1. Clearly state what happened: "Infrastructure PR #12345 created"
2. Explain what's needed: "Please review and merge the PR"
3. Provide the resume command: "Run `/migrate pr-merged email.notifications` when done"
4. Update state: Mark as `awaiting_human`

```markdown
## Checkpoint Reached

**Cluster**: email.notifications
**Status**: PR created, awaiting review

### What happened
Created infrastructure PR #12345 that adds the airflow3 component.

### What's needed
1. Review the PR for correctness
2. Merge when approved
3. Run `pacman apply` to provision resources

### Resume
When complete, run:
```
/migrate pr-merged email.notifications
```
```

---

## Skill Design: Building Reusable Components

### Anatomy of a Good Skill

A skill should be:

1. **Single-purpose**: Does one thing well
2. **Idempotent**: Safe to run multiple times
3. **Stateless**: Doesn't maintain its own state (orchestrator does that)
4. **Documented**: Clear inputs, outputs, and failure modes

```markdown
# Skill: infrastructure-setup

## Purpose
Creates infrastructure PR for migrating a resource to the new platform.

## Inputs
- resource_id: The resource to migrate (e.g., "email.notifications")
- target_version: Version to migrate to (e.g., "3.0")

## Outputs
- pr_number: The created PR number
- pr_url: URL to the PR

## Idempotency
If a PR already exists for this resource, returns existing PR info.

## Failure Modes
- Resource not found → Error with helpful message
- Manifest invalid → Error with validation details
- Git conflict → Error with conflict resolution steps
```

### Skill Composition

Complex operations should compose multiple skills:

```
/migrate start email.notifications

Orchestrator:
  1. Check state → pending
  2. Invoke skill: infrastructure-setup
  3. Update state → pr_created, awaiting_human
  4. Present checkpoint to user

User: /migrate pr-merged email.notifications

Orchestrator:
  1. Check state → pr_created
  2. Update state → pr_merged
  3. Invoke skill: deploy-to-test
  4. Update state → deploying
  5. Check deployment status
  6. Update state → deployed, awaiting_human
  7. Present checkpoint to user
```

---

## Batch Operations: Scaling to Many Resources

### The Temptation of Parallelism

When you have 66 clusters to migrate, parallel processing is tempting:

```python
# DON'T DO THIS
async def migrate_all():
    await asyncio.gather(*[migrate(c) for c in clusters])
```

This creates problems:

- **State corruption**: Concurrent git operations conflict
- **Debugging nightmares**: Which cluster failed?
- **Resource exhaustion**: API rate limits, memory pressure
- **Lost context**: Hard to track where each cluster is

### The Sequential-with-State Approach

Instead, process one at a time but track state for all:

```python
def migrate_next():
    # Find clusters ready for next step
    ready = [c for c in clusters if c.can_proceed()]

    if not ready:
        return "All clusters at checkpoints or completed"

    # Process first ready cluster
    cluster = ready[0]
    result = process_step(cluster)

    # Update state
    cluster.update_state(result)

    return f"Processed {cluster.id}, {len(ready)-1} more ready"
```

### Batch Status Queries

While processing is sequential, status queries can aggregate:

```markdown
## Wave 2 Status

| Cluster | Current Step | Last Done | Next Action |
|---------|-------------|-----------|-------------|
| email.notifications | Deploy | PR Merged | Run helm deploy |
| email.scale | PR Review | PR Created | Awaiting review |
| data.pipeline | Blocked | - | Waiting for test realm |

**Summary**: 2 actionable, 1 blocked, 12 completed
```

---

## Error Handling and Recovery

### Failure Categories

Design your error handling around failure categories:

| Category | Example | Recovery |
|----------|---------|----------|
| Transient | API timeout | Auto-retry with backoff |
| Retriable | PR merge conflict | Update and retry |
| Blocking | Deploy crash | Human intervention |
| Fatal | Resource deleted | Abort and alert |

### State-Aware Recovery

When recovering from failures, the agent should:

1. Read current state from persistent storage
2. Validate state against reality (PRs exist? Deployments running?)
3. Reconcile differences (update state to match reality)
4. Present options to the user

```markdown
## Recovery Analysis

**Cluster**: email.notifications
**Recorded State**: deploying
**Actual State**: Deploy workflow failed

### Discrepancy Found
State says "deploying" but workflow #12345678 shows "failed".

### Options
1. **Retry deploy**: Re-run the workflow
2. **Mark as blocked**: Investigate manually
3. **Skip to next step**: If you deployed manually

Which would you like to do?
```

---

## Cloud Infrastructure Considerations

### Running Agents in Production

Migration agents can run in several configurations:

**1. Local CLI (Development/Small Scale)**

```
Developer laptop → Claude Code → Git → GitHub
                              ↘ Cloud APIs
```

Pros: Simple, interactive, full control
Cons: Requires human present, laptop must stay on

**2. Scheduled Jobs (Batch Processing)**

```
Cron/Airflow → Agent Container → Git → GitHub
                              ↘ Cloud APIs
```

Pros: Automated, runs overnight, handles batches
Cons: Less interactive, checkpoint handling harder

**3. Event-Driven (Real-Time)**

```
GitHub Webhook → Lambda/Cloud Run → Agent → Git
PR Merged      →                          → Next Step
```

Pros: Immediate response, minimal latency
Cons: Complex setup, cold start issues

**4. Persistent Service (Enterprise)**

```
Agent Service ←→ State Database
      ↕
API/Slack/UI ←→ Users
```

Pros: Always available, multiple interfaces
Cons: Infrastructure overhead, session management

### Infrastructure Requirements

| Component | Purpose | Options |
|-----------|---------|---------|
| State Store | Persist migration progress | Git, PostgreSQL, DynamoDB |
| Secret Management | API keys, credentials | Vault, AWS Secrets Manager |
| Compute | Run agent | Lambda, ECS, GKE, local |
| Observability | Track progress, debug | CloudWatch, Datadog, Prometheus |
| Notifications | Alert on checkpoints | Slack, PagerDuty, email |

### Multi-Environment Patterns

Migrations often span multiple environments (test, staging, prod):

```yaml
environments:
  test:
    aws_profile: company-test
    cluster_suffix: -test
    approval_required: false

  prod-us:
    aws_profile: company-prod-us
    cluster_suffix: -prod
    approval_required: true
    blocked_by: test  # Can't proceed until test completes

  prod-eu:
    aws_profile: company-prod-eu
    cluster_suffix: -prod-eu
    approval_required: true
    blocked_by: test
```

---

## Permission Delegation and Security

### The Permission Challenge

Migration agents need powerful permissions:

- Create/merge PRs
- Deploy to production
- Modify infrastructure

But giving an AI unrestricted access is risky.

### Layered Permission Model

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: Static Permissions                              │
│ - Read access to all repositories                        │
│ - Write access to migration state files only             │
│ - Create PRs (but not merge)                             │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 2: Human-Gated Permissions                         │
│ - Merge PRs → Requires human approval                    │
│ - Deploy to prod → Requires human confirmation           │
│ - Delete resources → Requires explicit command           │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 3: Audit Trail                                     │
│ - All actions logged with timestamp and actor            │
│ - State changes tracked in git history                   │
│ - Alerts on unexpected operations                        │
└─────────────────────────────────────────────────────────┘
```

### Implementing Permission Boundaries

```python
class PermissionBoundary:
    """Enforces permission boundaries for agent actions."""

    ALLOWED_WITHOUT_APPROVAL = [
        "read_state",
        "update_state",
        "create_pr",
        "check_status",
    ]

    REQUIRES_HUMAN_CONFIRMATION = [
        "deploy_to_test",
        "run_migration_dry_run",
    ]

    REQUIRES_EXPLICIT_COMMAND = [
        "merge_pr",
        "deploy_to_prod",
        "run_migration",
        "delete_resource",
    ]

    def can_proceed(self, action: str, context: dict) -> tuple[bool, str]:
        if action in self.ALLOWED_WITHOUT_APPROVAL:
            return True, "Allowed"

        if action in self.REQUIRES_HUMAN_CONFIRMATION:
            if context.get("human_confirmed"):
                return True, "Human confirmed"
            return False, f"Requires confirmation: /migrate confirm {action}"

        if action in self.REQUIRES_EXPLICIT_COMMAND:
            if context.get("explicit_command"):
                return True, "Explicit command received"
            return False, f"Requires explicit command: /migrate {action}"

        return False, "Unknown action"
```

### Credential Management

Never embed credentials in agent code or state:

```yaml
# BAD - credentials in state
environment_status:
  prod:
    aws_access_key: AKIA...
    aws_secret_key: ...

# GOOD - reference to credential source
environment_status:
  prod:
    credential_source: aws_profile:company-prod
    # Agent uses: AWS_PROFILE=company-prod
```

For CI/CD agents, use:

- **AWS**: IAM roles with OIDC federation
- **GitHub**: Fine-grained PATs or GitHub Apps
- **Kubernetes**: Service accounts with RBAC

---

## Sandbox Environments for Agent Development

### Why Sandboxes Matter

When developing migration agents, you need to:

1. Test the full workflow without affecting production
2. Iterate quickly on edge cases
3. Validate state transitions
4. Train the agent on your specific patterns

### Sandbox Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    SANDBOX ENVIRONMENT                   │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Mock GitHub │  │ Mock Cloud  │  │ Mock Deploy │     │
│  │   API       │  │   APIs      │  │   Target    │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│         ↑               ↑               ↑               │
│         └───────────────┼───────────────┘               │
│                         │                               │
│              ┌──────────┴──────────┐                    │
│              │   Agent Under Test   │                    │
│              └──────────┬──────────┘                    │
│                         │                               │
│              ┌──────────┴──────────┐                    │
│              │   Test State Store   │                    │
│              │   (isolated branch)  │                    │
│              └─────────────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

### Implementing Sandbox Mode

```python
class MigrationAgent:
    def __init__(self, sandbox_mode: bool = False):
        self.sandbox_mode = sandbox_mode

        if sandbox_mode:
            self.github = MockGitHubClient()
            self.cloud = MockCloudClient()
            self.state_branch = "sandbox/migration-test"
        else:
            self.github = GitHubClient()
            self.cloud = CloudClient()
            self.state_branch = "migration/v3"

    def create_pr(self, resource: str) -> dict:
        if self.sandbox_mode:
            # Create real PR but to a test repo
            return self.github.create_pr(
                repo="org/migration-sandbox",
                title=f"[SANDBOX] {resource}",
                ...
            )
        else:
            return self.github.create_pr(
                repo="org/infrastructure",
                ...
            )
```

### Test Scenarios

Build test scenarios that exercise edge cases:

```python
SANDBOX_SCENARIOS = {
    "happy_path": {
        "resources": ["test.simple"],
        "pr_review_delay": 0,
        "deploy_success": True,
        "expected_final_state": "completed",
    },
    "pr_needs_changes": {
        "resources": ["test.needs-changes"],
        "pr_review_result": "changes_requested",
        "expected_checkpoint": "pr_changes_requested",
    },
    "deploy_failure": {
        "resources": ["test.deploy-fails"],
        "deploy_success": False,
        "deploy_error": "ImagePullBackOff",
        "expected_checkpoint": "deploy_failed",
    },
    "partial_migration": {
        "resources": ["test.partial"],
        "migration_items": 10,
        "fail_at_item": 5,
        "expected_state": "migration_partial",
    },
}
```

---

## Making Internal Tools Agentic

### The Evolution of Internal Tools

Traditional internal tools follow a command-driven model:

```
User → Command → Tool → Output
"deploy email.notifications" → Deploy script → "Deployed successfully"
```

Agentic tools add context and intelligence:

```
User → Intent → Agent → [Multiple tools] → Outcome + Next steps
"deploy email.notifications" → Agent checks state →
  "Can't deploy: PR not merged. Would you like me to check PR status?"
```

### Patterns for Agentic Tools

**1. Context-Aware Commands**

```python
# Traditional
def deploy(resource: str):
    run_deploy_script(resource)

# Agentic
def deploy(resource: str, agent_context: AgentContext):
    state = agent_context.get_state(resource)

    if state.pr_status != "merged":
        return AgentResponse(
            status="blocked",
            message=f"Cannot deploy: PR #{state.pr_number} not merged",
            suggestions=[
                f"Check PR status: /status {resource}",
                f"After merge: /deploy {resource}",
            ]
        )

    if state.last_deploy_failed:
        return AgentResponse(
            status="confirm_required",
            message=f"Last deploy failed. Retry?",
            actions=[
                Action("retry", f"/deploy {resource} --force"),
                Action("investigate", f"/logs {resource}"),
            ]
        )

    result = run_deploy_script(resource)
    agent_context.update_state(resource, deploy_status=result)

    return AgentResponse(
        status="success",
        message=f"Deployed {resource}",
        next_steps=[f"Verify: /verify {resource}"]
    )
```

**2. Proactive Suggestions**

```python
class AgentResponse:
    status: str
    message: str
    suggestions: list[str] = []
    warnings: list[str] = []
    next_steps: list[str] = []
    related_resources: list[str] = []

    def format(self) -> str:
        output = [f"**Status**: {self.status}", self.message]

        if self.warnings:
            output.append("\n**Warnings**:")
            output.extend(f"- {w}" for w in self.warnings)

        if self.next_steps:
            output.append("\n**Next Steps**:")
            output.extend(f"1. {s}" for s in self.next_steps)

        if self.related_resources:
            output.append("\n**Related**:")
            output.extend(f"- {r}" for r in self.related_resources)

        return "\n".join(output)
```

**3. Learning from Patterns**

Track common workflows and suggest them:

```python
class WorkflowLearner:
    def record_action(self, user: str, action: str, context: dict):
        self.history.append({
            "user": user,
            "action": action,
            "context": context,
            "timestamp": now(),
        })

    def suggest_next_action(self, current_action: str) -> list[str]:
        # Find what users typically do after this action
        next_actions = Counter()
        for i, entry in enumerate(self.history[:-1]):
            if entry["action"] == current_action:
                next_actions[self.history[i+1]["action"]] += 1

        return [action for action, _ in next_actions.most_common(3)]
```

---

## Extending to Other Migration Types

The patterns in this post apply beyond infrastructure migrations:

### Database Migrations

```yaml
resource:
  type: database_table
  id: users.profiles

phases:
  - name: add_new_column
    type: schema_change
    requires_downtime: false

  - name: backfill_data
    type: data_migration
    batch_size: 10000
    checkpoint: row_id

  - name: validate_data
    type: verification
    query: "SELECT COUNT(*) WHERE new_column IS NULL"
    expected: 0

  - name: drop_old_column
    type: schema_change
    requires_approval: true
```

### API Version Migrations

```yaml
resource:
  type: api_endpoint
  id: /v1/users

phases:
  - name: deploy_v2
    parallel_with: v1

  - name: shadow_traffic
    percentage: 10
    duration: 7d

  - name: compare_responses
    diff_threshold: 0.01

  - name: gradual_rollout
    stages: [25, 50, 75, 100]

  - name: deprecate_v1
    notice_period: 90d
```

### Framework Upgrades

```yaml
resource:
  type: service
  id: payment-processor

phases:
  - name: dependency_audit
    check: compatibility_matrix

  - name: update_dependencies
    pr_type: automated

  - name: run_tests
    required_coverage: 80

  - name: canary_deploy
    traffic: 5%
    duration: 24h
    rollback_on: error_rate > 0.1%

  - name: full_deploy
    requires: canary_success
```

---

## Lessons Learned

After orchestrating 66 Airflow migrations, here are the key takeaways:

### 1. State is Everything

Without persistent state, you're just running scripts with extra steps. Invest in your state schema early - it's the foundation everything else builds on.

### 2. Human Checkpoints Aren't Failures

It's tempting to automate everything. Resist. Strategic human checkpoints:

- Catch edge cases automation misses
- Build trust in the system
- Provide natural pause points
- Enable course correction

### 3. Sequential Beats Parallel

For migrations, reliability beats speed. Process one resource at a time, but track state for all. You'll sleep better.

### 4. The Orchestrator Shouldn't Implement

Keep your orchestrator thin. It tracks state and delegates work. The moment it starts editing files directly, you've created a maintenance nightmare.

### 5. Design for Resume

Assume every operation will be interrupted. Design your state transitions so resuming is always safe and obvious.

### 6. Audit Everything

You will need to explain what happened. Log every state change, every decision, every human confirmation. Your future self will thank you.

---

## Conclusion

Migration agents represent a shift from "automation scripts" to "intelligent assistants." They maintain context across sessions, adapt to failures, and collaborate with humans at critical decision points.

The key patterns:

- **Orchestrator + Skills**: Separate coordination from implementation
- **Persistent State**: Track progress in durable storage
- **Human Checkpoints**: Design explicit pause points with clear resume paths
- **Sequential Processing**: Reliability over speed
- **Permission Boundaries**: Layer security with audit trails

Whether you're migrating Airflow clusters, database schemas, or API versions, these patterns provide a foundation for building agents that can handle the complexity of real-world infrastructure changes.

The future of internal tooling is agentic. Tools that understand context, suggest next steps, and gracefully handle the unexpected will replace the brittle scripts of today. Start building that future now.

---

## Further Reading

- [Temporal.io](https://temporal.io) - Durable execution platform
- [Apache Airflow 3.0 HITL](https://airflow.apache.org/) - Human-in-the-loop patterns
- Event-Driven Architecture - Scheduling patterns
- Claude Code Skills - Building custom skills

---

*This post was written based on real-world experience migrating 66 Airflow clusters from v2 to v3. The patterns and code examples are derived from production systems.*

---

[← Back to Blog](/blog) ・ [Home](/) ・ [Contact](/contact)
