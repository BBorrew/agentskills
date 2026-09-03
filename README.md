# Agent Skills

[Agent Skills](https://agentskills.io) are a simple, open format for giving agents new capabilities and expertise.

Skills are folders of instructions, scripts, and resources that agents can discover and use to perform better at specific tasks. Write once, use everywhere.

## Custom Skills

### ChatGPT Pro Conversation Bridge

```text
skills/chatgpt-pro-conversation-bridge/SKILL.md
```

Creates and continues persistent ChatGPT web conversations through the private `chatgpt-github-console` control plane. Use it to create long-lived reviewer/modifier/manager roles, wake an existing role with later tasks, or let a scheduled orchestrator trigger persistent high-capability ChatGPT conversations through GitHub without using the OpenAI API.

Verified control primitives:

```text
/new <first message>
/continue <message>
```

## Getting Started

- [Documentation](https://agentskills.io) - Guides and tutorials
- [Specification](https://agentskills.io/specification) - Format details
- [Example Skills](https://github.com/anthropics/skills) - See what's possible

This repo contains the specification, documentation, reference SDK, and personal reusable skills. Also see a list of example skills [here](https://github.com/anthropics/skills).

## About

Agent Skills is an open format maintained by [Anthropic](https://anthropic.com) and open to contributions from the community.
