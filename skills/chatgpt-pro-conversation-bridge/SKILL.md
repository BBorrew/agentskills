---
name: chatgpt-pro-conversation-bridge
description: Creates and continues persistent ChatGPT web conversations through a private GitHub control plane and local browser bridge. Use when the user wants to create a new long-lived ChatGPT/Pro role, wake or continue an existing role, let a scheduled orchestrator trigger high-capability ChatGPT conversations, or coordinate multi-agent research workflows through GitHub without using the OpenAI API.
compatibility: Requires GitHub access to the private chatgpt-github-console repository, an online self-hosted runner labeled chatgpt-local, a localhost DrA1ex/chatgpt-bridge instance, and a connected logged-in ChatGPT Chrome extension.
metadata:
  author: BBorrew
  version: "1.0.0"
  control-repo: BBorrew/chatgpt-github-console
---

# ChatGPT Pro Conversation Bridge

Use this skill as a control-plane adapter between an orchestrating agent and persistent ChatGPT web conversations.

The verified primitives are intentionally small:

1. **Create** a real persistent ChatGPT conversation with `/new <first message>`.
2. **Continue** that exact conversation later with `/continue <message>`.
3. Read the target assistant's response from the GitHub Issue after the self-hosted workflow completes.

Do not claim unsupported full-history synchronization or direct OpenAI API access.

## Mental model

```text
orchestrator / scheduled task
        ↓
GitHub Issue comment
        ↓
private GitHub Actions workflow
        ↓
self-hosted runner: chatgpt-local
        ↓
127.0.0.1 ChatGPT browser bridge
        ↓
logged-in ChatGPT web conversation
        ↓
assistant response
        ↓
GitHub Issue comment
        ↓
orchestrator reads result
```

GitHub is the durable **control/message bus**. The ChatGPT conversation is the persistent **role memory/reasoning worker**. A shared store such as Google Drive should remain the canonical project/research state when running long-lived multi-agent workflows.

## Important model behavior

Do **not** change the target model or reasoning effort unless the user explicitly asks.

This skill does not itself select Pro. It sends a turn into the normal logged-in ChatGPT web conversation. If the user's target conversation/browser defaults already provide the desired Pro configuration, preserve it.

## Control repository

Default control repository:

```text
BBorrew/chatgpt-github-console
```

The repository is expected to remain private.

The workflow accepts slash-command Issue comments from the repository owner and executes them on the self-hosted runner.

## When to activate this skill

Activate when the user asks for actions such as:

- “创建一个新的 Pro 对话/评审者/修改者”
- “继续刚才那个 reviewer 对话”
- “唤醒 Reviewer A 做下一轮”
- “让 scheduled task 每小时推动这些 Pro 角色”
- “通过 GitHub Bridge 给那个 ChatGPT 对话发消息”
- “建立一个持久化 manager/reviewer/modifier conversation”
- orchestrating multiple persistent ChatGPT roles where GitHub is the wake-up/control channel

Do not activate merely for ordinary prose drafting or a one-off explanation about ChatGPT.

## Primitive A: create a persistent role/conversation

Preferred abstraction:

```text
one durable GitHub Issue = one persistent ChatGPT role/conversation
```

### Procedure

1. Determine the stable role/topic name.
2. Search the control repository for an existing Issue representing that role if reuse is plausible.
3. If the user explicitly wants a **new** role/conversation, create a new Issue with a stable descriptive title.
4. Add this Issue comment:

```text
/new <role initialization and first task>
```

5. Poll/read Issue comments until `github-actions[bot]` returns either:
   - the assistant response and a created ChatGPT conversation ID, or
   - a concrete failure message.
6. Treat the Issue number as the durable control handle for future turns.
7. Report the created role/Issue/conversation back to the user when useful.

### Example

Issue title:

```text
CHI Reviewer - Methods
```

Comment:

```text
/new You are Reviewer A. Maintain a strict CHI methodology-reviewer role across future turns. Read the canonical shared research state, perform the assigned review, and write the requested output to the agreed shared location.
```

Do not require the user to manually supply a ChatGPT conversation URL when `/new` is appropriate.

## Primitive B: continue an existing role/conversation

### Procedure

1. Resolve the role to its existing GitHub Issue.
   - Prefer an explicit Issue mapping from current context or the project role registry.
   - Otherwise search Issues by stable role/title.
   - Do not guess between ambiguous candidate Issues.
2. Add:

```text
/continue <next task>
```

3. Read Issue comments until the new `github-actions[bot]` response appears.
4. Verify success. The response footer should identify the bound ChatGPT conversation.
5. Return or use the response to decide the next workflow action.

### Example

```text
/continue Manuscript v12 is now current. Review only the changed methodology sections and unresolved validity risks. Use the canonical shared project state rather than relying only on chat history.
```

## Never fork a role accidentally

If `/continue` fails because runtime prerequisites are offline, **do not recover by sending `/new`**.

`/new` creates a different ChatGPT conversation and therefore forks the role's persistent memory.

For an existing role, preserve the Issue and conversation binding and repair the runtime prerequisite instead.

## Runtime preflight and failure handling

A command needs all of these:

- private control repository available;
- self-hosted runner online with `chatgpt-local` label;
- local `DrA1ex/chatgpt-bridge` running;
- bridge bound to loopback, normally `127.0.0.1:8080`;
- Chrome/Chromium logged into the intended ChatGPT account;
- Bridge browser extension connected.

If a command does not complete:

1. inspect the GitHub Issue response and Actions job state;
2. identify whether the runner, bridge, extension, or ChatGPT login is unavailable;
3. tell the user the specific prerequisite that needs restoration;
4. retry only after that prerequisite is restored.

Do not bypass CAPTCHA, MFA, Cloudflare, device verification, login protections, or similar controls.

## Result handling

For `/new`, a successful bot reply should contain:

- the target assistant response;
- a note that a new ChatGPT conversation was created and bound;
- the concrete conversation ID / ChatGPT URL.

For `/continue`, a successful bot reply should contain:

- the target assistant response;
- a note that the same bound conversation was continued.

When waiting for results, distinguish the command comment authored by the repository owner from the later bot reply. Do not mistake the command itself for the result.

## GitHub write restrictions

If the environment blocks the Issue-comment write through a safety or authorization check, do not attempt to bypass it or substitute unrelated GitHub writes. Ask the user to post the exact slash command manually in the intended Issue, then continue by reading the workflow result.

Never request or expose:

- GitHub runner registration tokens;
- ChatGPT cookies or browser profile data;
- local bridge `API_TOKEN`;
- local extension `BRIDGE_TOKEN`;
- authentication/session files.

## Multi-agent research/orchestration pattern

For a workflow with reviewers, modifiers, and a manager, create each persistent role once and retain the Issue mapping.

Example role registry:

```text
Reviewer-Methods      -> Issue #10 -> persistent conversation
Reviewer-Contribution -> Issue #11 -> persistent conversation
Reviewer-Critical-AC  -> Issue #12 -> persistent conversation
Modifier-Methods      -> Issue #20 -> persistent conversation
Modifier-Writing      -> Issue #21 -> persistent conversation
Manager-Integrator    -> Issue #30 -> persistent conversation
```

The orchestrator should normally operate as a state machine:

```text
read canonical shared state
→ decide next role
→ /continue role task
→ read GitHub result
→ update/inspect canonical shared state
→ decide next role
→ repeat
```

Use `/new` only for first-time role creation or an explicitly requested clean-room role.

## Google Drive-centered workflows

When Google Drive is the project's information hub:

- Drive is the canonical versioned state and construction record.
- GitHub carries wake-up commands and target responses.
- Persistent ChatGPT conversations provide role-specific reasoning memory.
- Scheduled tasks provide timing and orchestration.

Prompts to target roles should point them to explicit Drive files/folders/version identifiers instead of assuming the entire current state exists in conversation memory.

Prefer instructions such as:

```text
Read the current manifest/version in the shared Drive workspace, perform your role-specific task, write your output to the designated location, and report completion plus blockers.
```

## Scheduling pattern

A scheduled orchestrator can use the bridge as a wake-up path when GitHub access is available to that scheduled run.

Typical hourly iteration:

```text
Run N:
- inspect Drive/project state and prior GitHub results
- choose next role(s)
- send /continue commands

Run N+1:
- read completed role outputs/results
- decide whether to review, modify, verify, or escalate
- send next /continue command(s)
```

The scheduler does not need to be the strongest reasoning model. Its job is state tracking, routing, and wake-up. Persistent target conversations perform the role-specific reasoning.

## Supported console commands relevant to this skill

Primary:

```text
/new <first message>
/continue <message>
```

Diagnostics/discovery when needed:

```text
/status
/sessions [filter]
/bind <conversation-id-or-URL>
```

`/bind` is for attaching an Issue to an already-existing ChatGPT conversation. Prefer `/new` when the user explicitly wants a new persistent role.

## Security boundary

Treat the bridge as an unofficial browser integration, not an OpenAI-supported API.

Keep the control repository private and the bridge on localhost. Allow only trusted owner-controlled commands to reach the self-hosted runner.

Do not place secrets or authentication state in GitHub, Google Drive instructions, model prompts, or Issue comments.

## Capability boundary

Currently verified:

- creating a real persistent ChatGPT conversation through GitHub;
- binding its ID to the controlling Issue;
- continuing exactly that conversation through later GitHub commands;
- mirroring the assistant answer back to GitHub for the orchestrator to read.

Not currently guaranteed:

- complete synchronization of every message entered directly through other ChatGPT clients;
- indefinite autonomous execution without a scheduler/orchestrator;
- immunity to ChatGPT web UI changes.

Keep the canonical project state outside chat history for workflows that need reliable long-term iteration.
