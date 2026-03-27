---
type: research
topic: Claude Code Channels
sources: 14
date: 2026-03-27
description: Comprehensive research on Claude Code Channels -- what they are, how they work, architecture, supported platforms, security model, agent teams messaging, and third-party extensions
squirrel: research-agent
tags: [claude-code, channels, mcp, messaging, agent-teams, anthropic]
---

# Claude Code Channels -- Research Report

## Executive Summary

Claude Code Channels is a feature shipped by Anthropic on March 20, 2026 in Claude Code v2.1.80. It is currently in **research preview**. A channel is an MCP server that pushes events into a running Claude Code session, enabling two-way communication between Claude Code and external messaging platforms (Telegram, Discord, iMessage) as well as arbitrary webhook sources. Channels are **not** a user-to-user collaboration feature. They are a developer-to-session bridge: you message your own Claude Code instance from your phone or pipe CI/monitoring events into it.

Separately, Claude Code's **Agent Teams** feature (experimental, shipped February 6, 2026) includes an inter-agent messaging system via `SendMessage` tool that enables direct peer-to-peer communication between Claude Code instances on the same machine. This is distinct from Channels but the two are beginning to converge in community feature requests.

---

## Key Findings

- Channels are MCP servers that declare a `claude/channel` capability and push `notifications/claude/channel` events into a running session
- Three official channel plugins ship in the research preview: Telegram, Discord, and iMessage, plus a localhost demo (fakechat)
- Channels are two-way: Claude reads inbound events and can reply back through the same platform via an exposed MCP tool
- Events only arrive while the session is open -- there is no message queue for offline sessions
- Security is enforced via sender allowlists; each plugin requires pairing before messages are accepted
- Enterprise/Team organizations must explicitly enable channels via managed settings
- Channels require `claude.ai` login; API key / Console auth is not supported
- Permission relay (v2.1.81+) allows remote approval/denial of tool-use prompts from your phone
- Agent Teams use a separate messaging primitive (`SendMessage` tool with mailbox system) for inter-agent coordination, not Channels
- Third-party projects like Agent HQ are already building on the channel infrastructure to enable inter-agent communication across persistent sessions

---

## Detailed Analysis

### 1. What Channels Are

A channel is an MCP server that:
1. Declares the `claude/channel` capability in its constructor
2. Emits `notifications/claude/channel` events when something happens externally
3. Connects to Claude Code over stdio (spawned as a subprocess)

Events arrive in the session wrapped in XML-like tags:

```
<channel source="telegram" chat_id="12345" sender="Patrick">
hey, how's the build going?
</channel>
```

Claude processes the event in the context of whatever session is running -- it has access to your files, terminal, MCP tools, and full conversation history. It can then reply back through the channel using an MCP tool exposed by the channel server.

Channels are **push-based**, not poll-based. This distinguishes them from standard MCP servers (which Claude queries on demand) and from Remote Control (which gives you a UI to drive your session from claude.ai).

### 2. Supported Platforms

| Platform | Type | Auth Method | Source |
|----------|------|-------------|--------|
| Telegram | Two-way chat bridge | BotFather token + pairing code | Official plugin |
| Discord | Two-way chat bridge | Discord Developer Portal token + pairing code | Official plugin |
| iMessage | Two-way chat bridge | macOS Full Disk Access (reads ~/Library/Messages/chat.db) | Official plugin |
| Fakechat | Two-way localhost demo | None (localhost:8787) | Official plugin |
| Custom webhook | One-way or two-way | Developer-defined | Build your own |

All official plugins are in the `anthropics/claude-plugins-official` GitHub repository under `external_plugins/`.

### 3. Architecture

```
External System (Telegram, CI, etc.)
        |
        v
Channel Server (MCP, runs locally, spawned by Claude Code)
        |  stdio
        v
Claude Code Session (your terminal, your files, your context)
        |
        v  (optional: reply tool)
Channel Server
        |
        v
External System
```

Key architectural details:
- The channel server runs as a local subprocess of Claude Code, communicating over stdio
- For chat platforms (Telegram, Discord): the plugin polls the platform API for new messages
- For webhooks: the server listens on a local HTTP port
- No cloud relay is involved -- everything runs on your machine
- The `--channels` flag specifies which plugins are active for a given session
- Multiple channels can run simultaneously (space-separated)

### 4. Setup and Usage

Install a channel plugin:
```
/plugin install telegram@claude-plugins-official
```

Configure credentials:
```
/telegram:configure <bot-token>
```

Launch with channels enabled:
```bash
claude --channels plugin:telegram@claude-plugins-official
```

Pair your account (from Telegram, DM your bot, get a code, then in Claude Code):
```
/telegram:access pair <code>
/telegram:access policy allowlist
```

### 5. Security Model

- **Sender allowlists**: every approved channel plugin maintains a list of allowed sender IDs; everyone else is silently dropped
- **Pairing flow**: Telegram and Discord use a code-based pairing system; iMessage uses Apple ID self-detection
- **Permission relay** (v2.1.81+): channels that declare `claude/channel/permission` can forward tool-approval prompts to your phone; you reply `yes <id>` or `no <id>` to approve/deny remotely
- **Enterprise controls**: `channelsEnabled` master switch + `allowedChannelPlugins` allowlist in managed settings
- **Development bypass**: `--dangerously-load-development-channels` for testing custom channels not on the approved list
- **Gate on sender, not room**: in group chats, the allowlist checks `message.from.id`, not `message.chat.id`

### 6. Building Custom Channels

The channel contract requires:
1. Declaring `capabilities: { experimental: { 'claude/channel': {} } }` in the MCP Server constructor
2. Emitting `notifications/claude/channel` with `content` (string) and optional `meta` (key-value attributes)
3. Optionally exposing a reply tool via standard MCP tool registration
4. Optionally declaring `claude/channel/permission` for remote permission relay

The `instructions` field in the Server constructor is injected into Claude's system prompt, telling it how to interpret and respond to channel events.

Full webhook receiver example is documented at: https://code.claude.com/docs/en/channels-reference

### 7. Agent Teams Messaging (Separate System)

Agent Teams (experimental, v2.1.32+) have their own messaging infrastructure that is distinct from Channels:

- **SendMessage tool**: any teammate can message any other teammate directly or broadcast to all
- **Mailbox system**: messages are delivered automatically to recipients; no polling needed
- **Message types**: `message` (direct), `broadcast` (all), `shutdown_request/response`, `plan_approval_response`
- **Shared task list**: coordination backbone at `~/.claude/tasks/{team-name}/`
- **Team config**: stored at `~/.claude/teams/{team-name}/config.json`

Agent Teams messaging is intra-machine, intra-team only. It does not use the Channel infrastructure. There is an active feature request (GitHub issue #30140, closed as duplicate of #4993) to add a shared channel primitive to agent teams -- a persistent, ordered log that all team members can read and write to.

### 8. What Channels Are NOT

To be precise about what this feature does and doesn't do:

- **Not user-to-user communication**: Channels connect YOU to YOUR Claude Code session. They don't connect two developers or two Claude instances.
- **Not inter-session sharing**: Channels don't pass context between separate Claude Code sessions.
- **Not a collaboration platform**: There's no shared workspace, shared conversation, or multi-user chat through Channels.
- **Not always-on by default**: Events only arrive while the session is open. For persistent availability, you must run Claude in a background process (tmux, etc.).
- **Not available with API keys**: Requires claude.ai login specifically.

### 9. Third-Party Extensions

**Agent HQ** (by aj-dev-smith): An MCP server that uses the channel infrastructure to enable inter-agent communication between multiple persistent Claude Code instances. Each agent runs its own Agent HQ server on a different port. Agent A's server POSTs to Agent B's HTTP listener, which pushes the message into Agent B's session as a channel notification. Includes a Telegram integration for human monitoring of agent exchanges.

**Claude Code Remote** (by JessyTsui): Controls Claude Code remotely via email, Discord, and Telegram. Different from official Channels in that it focuses on starting tasks and receiving completion notifications rather than live bidirectional chat.

### 10. Competitive Context

Channels were widely covered as Anthropic's response to **OpenClaw**, a competing tool that let developers message AI coding assistants from third-party apps. Anthropic's advantage: native integration with Claude Code's permission system, security model, and MCP infrastructure, plus the brand trust factor.

### 11. Relationship to Other Claude Code Features

| Feature | How it differs from Channels |
|---------|------------------------------|
| Remote Control | You drive your session from claude.ai or mobile app; Channels push events FROM external sources INTO your session |
| Claude in Slack | Spawns a web session from @Claude mention; Channels work with your local session |
| Claude Code on the Web | Fresh cloud sandbox from GitHub; Channels are local-first |
| Standard MCP servers | Claude queries them on demand; Channels push events proactively |
| Agent Teams | Inter-agent messaging within a team; Channels bridge external platforms |
| Subagents | Report back to parent only; no peer messaging; no external bridge |

---

## Open Questions and Future Direction

1. Will Channels eventually support inter-user or inter-session communication?
2. Will the Agent Teams messaging system and Channels converge into a unified primitive?
3. Will offline message queuing be added so events aren't lost when sessions are closed?
4. Will API key / Console authentication be supported?
5. When will Channels exit research preview?
6. Slack channel plugin is conspicuously absent from the launch -- likely coming given Claude in Slack already exists

---

## Confidence Assessment

**High confidence** on: what Channels are, how they work technically, supported platforms, security model, setup process, agent teams messaging. These are all documented in official Anthropic docs and confirmed across multiple sources.

**Medium confidence** on: competitive context with OpenClaw, future roadmap, timeline for GA. Based on press coverage and community discussion, not official roadmap.

**Low confidence** on: whether Channels will expand to support user-to-user or session-to-session communication. No official signals in this direction; the architecture is currently single-user, single-session by design.

---

## Sources

1. [Official Channels documentation](https://code.claude.com/docs/en/channels)
2. [Channels reference (build your own)](https://code.claude.com/docs/en/channels-reference)
3. [Agent Teams documentation](https://code.claude.com/docs/en/agent-teams)
4. [VentureBeat: Anthropic ships Claude Code Channels](https://venturebeat.com/orchestration/anthropic-just-shipped-an-openclaw-killer-called-claude-code-channels)
5. [Techzine: Claude Code Channels](https://www.techzine.eu/news/devops/139777/anthropic-builds-openclaw-rival-claude-code-channels/)
6. [LowCode Agency: What Is Claude Code Channels](https://www.lowcode.agency/blog/claude-code-channels)
7. [Cyrus: What are Claude Code Channels](https://www.atcyrus.com/stories/what-are-claude-code-channels)
8. [MacStories: Hands-On with Claude Code's Telegram and Discord Integrations](https://www.macstories.net/stories/first-look-hands-on-with-claude-codes-new-telegram-and-discord-integrations/)
9. [GitHub: claude-plugins-official (Telegram)](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/telegram)
10. [GitHub: claude-plugins-official (Discord)](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/discord)
11. [GitHub: claude-plugins-official (iMessage)](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/imessage)
12. [GitHub Issue #30140: Shared channel for agent teams](https://github.com/anthropics/claude-code/issues/30140)
13. [GitHub Issue #28300: Multi-agent collaboration across machines](https://github.com/anthropics/claude-code/issues/28300)
14. [Agent HQ MCP server](https://glama.ai/mcp/servers/aj-dev-smith/claude-channel-agenthq)
