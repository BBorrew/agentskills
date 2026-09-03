---
name: chatgpt-pro-conversation-bridge
description: Creates fresh or persistent ChatGPT web conversations through a private GitHub control plane, routes scheduled work to the logged-in Pro web experience, tracks terminal state, and operates the automatic Windows Bridge/browser lifecycle maintainer. Use for fresh per-job Pro workers, persistent reviewer roles, scheduled multi-agent research, Bridge health, capacity backpressure, safe tab garbage collection, or local runner/Bridge recovery without the OpenAI API.
compatibility: Requires GitHub access to BBorrew/chatgpt-github-console, a Windows self-hosted runner labeled chatgpt-local, localhost DrA1ex/chatgpt-bridge, and a logged-in connected ChatGPT Chrome extension. Routine maintenance can run every five minutes; login/CAPTCHA/MFA/device verification remains human-only.
metadata:
  author: BBorrew
  version: "2.0.0"
  control-repo: BBorrew/chatgpt-github-console
---

# ChatGPT Pro Conversation Bridge

Use this skill as a control-plane adapter between an orchestrating agent and the logged-in ChatGPT web product.

```text
orchestrator / scheduled task
        ↓
private GitHub Issue command
        ↓
self-hosted runner: chatgpt-local
        ↓
localhost browser bridge
        ↓
logged-in ChatGPT conversation
        ↓
terminal response and conversation ID
        ↓
GitHub + external canonical state
```

GitHub is the durable command/result bus. Google Drive or another shared store should remain the canonical project state. ChatGPT conversations are reasoning workers, not the sole durable ledger.

## Primary primitives

```text
/new <first message>
/continue <message>
/status
/roles
```

`/new` creates a real ChatGPT conversation, captures its exact conversation ID, binds it to the Issue, and mirrors the answer. `/continue` targets the exact bound conversation.

Never use `/new` as a hidden fallback for a failed `/continue`, because that forks the role into a different conversation.

## Choose the operating mode

### Fresh conversation per concrete Job

This is the recommended mode for reproducible automated research work.

1. Compute a deterministic `logical_job_key` from project, iteration, phase, role, and input version.
2. Create one GitHub Job Issue for one concrete attempt.
3. Send one `/new` containing the complete task package and external file identifiers.
4. Keep the returned conversation ID for audit.
5. Verify OUTPUT and RECEIPT in the canonical shared store.
6. Start the next Job in a new conversation instead of relying on old chat context.

This reduces context drift and makes attempts independently auditable.

### Persistent conversation per role

Use one Issue per role, create it once with `/new`, and send later tasks with `/continue`. Use this only when accumulated conversation memory is intentionally part of the workflow.

## Idempotent scheduling protocol

A scheduled orchestrator must not depend on its own prior-memory state. Recompute the required logical Jobs and reconcile GitHub plus the canonical store every run.

| Observed state | Required action |
|---|---|
| No record | Dispatch one `/new` |
| `PLANNED`, `DISPATCHED`, `QUEUED`, `RUNNING`, `BUSY` | Wait; send no second prompt |
| `COMPLETED` | Verify Receipt and artifacts |
| `VERIFIED` | Allow dependent work |
| `ERROR` | Retry only after proving the old attempt cannot still complete |
| `NEEDS_INSPECTION` / ambiguous | Do not duplicate; investigate |
| `STALE` | Do not consume the output |

Never interrupt a Pro worker that is still generating.

## Model and connector preflight

Do not switch the target model or effort unless explicitly requested. Use policy:

```text
PRO_DEFAULT_DO_NOT_CHANGE
```

When a worker needs GitHub or Google Drive, instruct it to perform harmless real read calls before substantive work. It must record observable evidence and must not pretend that tools exist because the prompt says so.

Recommended attestation fields:

```text
ui_model_label
bridge_model_slug
self_reported_model
model_attestation_state
drive_preflight + evidence
github_preflight + evidence
```

Self-report alone is not definitive proof. `MISMATCH`, `UNAVAILABLE`, or failed connector preflight blocks canonical writes.

## Automatic lifecycle maintainer

The control repository includes a conservative Windows watchdog and lifecycle manager. Install or upgrade it once by commenting in a trusted owner-controlled Issue:

```text
/admin maintenance-install
```

It registers the Windows task:

```text
ChatGPT Bridge Automatic Maintainer
```

The task runs at logon and every five minutes. It can:

- restart the local GitHub runner listener/service when absent;
- start `npm.cmd start` when Bridge port 8080 is not listening;
- maintain a dedicated Bridge-owned control tab;
- classify Bridge-created worker tabs from live Bridge observations and local durable run state;
- apply capacity backpressure;
- safely close only terminal Bridge-owned worker tabs after a grace period;
- preserve bounded local logs and a current health snapshot.

Diagnostics and control:

```text
/admin maintenance-status
/admin tab-status
/admin tab-gc dry-run
/admin tab-gc execute
/admin repair control-tab
/admin maintenance-run
```

## Safe tab collection invariants

The maintainer may only manage a tab that carries a Bridge launch-token ownership proof. Ordinary user-opened ChatGPT tabs are external and must never be closed automatically.

Never close, refresh, navigate, reuse, cancel, or restart a page associated with any of:

```text
QUEUED
SUBMITTING
RUNNING
BUSY
activeRequest != null
streaming
thinking
active tool use
```

A worker page becomes `REAPABLE` only when:

1. its exact conversation ID has been durably recorded;
2. its local run state is terminal (`completed` or `error`);
3. live browser observation shows no active request and final/stopped generation;
4. the terminal grace period has elapsed;
5. the expected URL and launch token match during close.

Conflicting or incomplete evidence becomes `QUARANTINED/NEEDS_INSPECTION`; never guess that it is safe.

Closing a tab does not delete its ChatGPT conversation. The conversation ID, GitHub record, shared outputs, and local SQLite state remain.

## Capacity rules

Default conservative settings:

```text
active generation soft cap: 6
managed tab soft cap: 10
managed tab hard cap: 12
terminal close grace: 15 minutes
orphan grace: 2 hours
max close operations per tick: 6
```

When lifecycle status says `pauseDispatch=true`, send no new `/new` commands. Wait for safe GC or resolve quarantine first.

## Failure and recovery

For failed commands, inspect in this order:

1. GitHub command status and workflow;
2. runner listener;
3. Node Bridge reachability;
4. extension clients and tab observations;
5. ChatGPT login and protection state.

The watchdog handles routine process loss. It deliberately does not kill the user's shared Chrome process.

Human action is still required for:

- login expiry;
- CAPTCHA, MFA, Cloudflare, or device verification;
- account restriction/security warning;
- Windows user session not signed in;
- power, network, or hardware outage;
- breaking ChatGPT UI or extension protocol changes.

Never bypass account protections.

## Google Drive-centered pattern

```text
Google Drive           = canonical versions, inputs, outputs, evidence, construction record
GitHub                 = idempotent command/result/status bus
Scheduled orchestrator = clock and deterministic state machine
Fresh Pro conversation = clean worker for one Job
Manager/Integrator     = authorized synthesis and version gate
Automatic maintainer   = runner/Bridge supervision and browser lifecycle control
```

Each worker task should specify project ID, logical key, Job ID, attempt, role, input version, exact read IDs/paths, allowed writes, prohibited writes, artifact contract, and Receipt schema.

A model's final answer is not sufficient completion. Verify the promised artifacts before advancing.

## Security boundary

Keep the control repository private and Bridge bound to loopback. Never reveal or store:

- runner registration tokens;
- ChatGPT cookies or browser profile files;
- local Bridge `API_TOKEN`;
- extension `BRIDGE_TOKEN`;
- authentication/session material.

This is an unofficial browser integration and may break when the web product changes.

## Capability boundary

Implemented and tested:

- create a real new ChatGPT conversation and record its exact ID;
- continue the exact bound conversation;
- mirror answers and terminal state to GitHub;
- serialize same-conversation prompts and preserve concurrent work;
- supervise the runner and Bridge;
- enforce capacity without interrupting active Pro output;
- ownership-guarded, terminal-only tab garbage collection.

Not guaranteed:

- full synchronization of messages entered directly through every ChatGPT client;
- recovery from login/security challenges without a human;
- immunity to future ChatGPT web changes;
- operation while the Windows host is powered off or signed out.
