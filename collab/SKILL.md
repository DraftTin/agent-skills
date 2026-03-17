# Skill: Inter-Agent Collaboration (collab)

Enable multiple AI agents (Claude, GPT, Gemini, or any CLI agent) in a tmux session to collaborate on tasks through a structured protocol. Agents discover each other via a registry, exchange messages through a shared channel file, and use tmux only for wakeup notifications.

This skill is model-agnostic. Any agent that can read files, write files, and run bash commands can participate.

## When to use

- When the user asks two or more agents to work together
- When a task benefits from architect + reviewer roles
- When agents need to propose, review, and iterate before implementing

## Architecture

```
.collab/                         ← project-local runtime state (gitignored)
├── agents.json                  ← live registry (runtime bindings)
├── identities/                  ← stable agent identities (persist across restarts)
│   ├── opus.json
│   └── codex.json
├── channel.md                   ← ONE active topic
└── archive/                     ← completed topics
    ├── 001_topic_name.md
    └── ...
```

**Transport model:**
- **Files** carry all content (durable, inspectable)
- **Tmux** is only used to wake agents ("go read the channel")
- **No hardcoded pane addresses** — agents look each other up in the registry

## Identity model

Agent identity has two layers:

### 1. Persistent identity (survives everything)

Stored in `.collab/identities/<agent_id>.json`. Created once, never deleted.

```json
{
  "agent_id": "codex",
  "role": "reviewer",
  "model": "gpt-5.4",
  "token": "stable-uuid-generated-on-first-registration",
  "created": "2026-03-16T20:00:00Z"
}
```

Survives: pane rearrangements, session close/restore, agent restarts.

### 2. Runtime binding (ephemeral, re-registered on every startup)

Stored in `.collab/agents.json`. Updated on every activation.

```json
{
  "agents": [
    {
      "agent_id": "codex",
      "token": "stable-uuid",
      "pane_id": "%34",
      "pane_address": "myproject:0.1",
      "instance_id": "2026-03-16T21:14:02Z#8f3d",
      "status": "active",
      "last_seen": "2026-03-16T21:16:00Z"
    }
  ],
  "channel": ".collab/channel.md",
  "created": "2026-03-16T20:00:00Z"
}
```

- `pane_id` (`%34`): primary delivery target — stable across layout changes
- `pane_address` (`myproject:0.1`): human-readable, for debugging only
- `instance_id`: unique per startup — lets others detect restarts

### Why two layers?

| Scenario | Identity | Runtime binding |
|----------|----------|-----------------|
| Pane rearranged (Prefix+Space) | Unchanged | `pane_id` unchanged (stable) |
| Session closed + restored | Unchanged (file on disk) | New `pane_id`, new `instance_id` |
| Agent restarts in same pane | Unchanged | New `instance_id` |
| Agent deleted | Remove identity file | Remove from registry |

## Bootstrapping — how agents join

### First agent: triggered by the user

The user says something like "collaborate with Codex" or "solve this with the other agent." This triggers the first agent to:
1. Read this skill file (`~/.agent-skills/collab/SKILL.md`)
2. Initialize `.collab/` if needed
3. Register itself
4. Write to channel
5. Wake the second agent

**First-contact delivery:** The target agent is not yet in `agents.json` at this point — the registry doesn't exist yet. The first agent must discover the target pane manually:
- **Ask the user** which tmux pane the target agent is in, OR
- **Auto-detect** by running `tmux list-panes -a -F '#{pane_id} #{pane_pid} #{pane_current_command}'` and identifying the target by its process (e.g., `node` for Codex, another CLI agent)
- **Use this one-time pane for the initial wakeup only** — after the target registers, all subsequent delivery uses the registry

This manual/semi-automatic step is the ONLY time a pane address is used without the registry. After both agents register, registry-based discovery takes over.

### Subsequent agents: triggered by `[COLLAB]` message

When an agent receives a `[COLLAB]` wakeup, the message includes the skill reference:
```
[COLLAB] From opus: Review rename proposal. Protocol: ~/.agent-skills/collab/SKILL.md. See .collab/channel.md
```

The receiving agent:
1. Reads the skill file — learns the protocol
2. Registers itself (now the registry has both agents)
3. Reads the channel — responds using registry-based delivery from now on

**No per-project config needed.** Any agent can be bootstrapped by a single wakeup message. The skill file at `~/.agent-skills/collab/SKILL.md` is the only prerequisite (installed once globally).

### After session restore

On restart, the first `[COLLAB]` message or user instruction triggers re-registration. Identity files persist on disk — only runtime binding (pane_id) needs refreshing. First-contact discovery may be needed again if the registry is stale.

## Prerequisites

- **tmux** — for pane management and wakeup delivery
- **JSON parsing** — the examples use `jq` for reading `agents.json`. If `jq` is not available, agents can use any equivalent: `python -c "import json; ..."`, `node -e "..."`, or parse the JSON directly with their built-in tools. Most AI coding agents can read and write JSON natively without shell tools.

## Quick start

### 1. Initialize collaboration

```bash
mkdir -p .collab/identities .collab/archive
```

### 2. Register yourself

On activation (first time or after restart):

```bash
# 1. Get your pane ID (stable across layout changes)
MY_PANE_ID=$(tmux display-message -p '#{pane_id}')
MY_PANE_ADDR=$(tmux display-message -p '#{session_name}:#{window_index}.#{pane_index}')

# 2. Load or create identity file
# If .collab/identities/MY_ID.json exists → load token
# If not → create with new UUID token

# 3. Update agents.json with current runtime binding
# Write: agent_id, token, pane_id, pane_address, instance_id, status, last_seen
```

### 3. Start a topic

Clear `channel.md` and write:

```markdown
# Topic: [descriptive name]

## Status: ACTIVE

## [From: your-id] [To: target-id]

Your message here...
```

### 4. Send a message

Look up target by `agent_id`, deliver via `pane_id`:

```bash
PANE=$(jq -r '.agents[] | select(.agent_id=="TARGET_ID") | .pane_id' .collab/agents.json)

# Validate pane still exists
if tmux list-panes -a -F '#{pane_id}' | grep -q "$PANE"; then
  # Exit copy/scroll mode if active (prevents keys going to wrong place)
  tmux copy-mode -t "$PANE" -q 2>/dev/null
  sleep 0.1
  # Send wakeup with summary
  tmux send-keys -t "$PANE" -l "[COLLAB] From YOUR_ID: Short summary. Protocol: ~/.agent-skills/collab/SKILL.md. See .collab/channel.md" && sleep 0.1 && tmux send-keys -t "$PANE" Enter
else
  # Pane is stale — mark agent stale, set topic BLOCKED
fi
```

**Important: always exit copy mode first.** If a tmux pane is in scroll/copy mode (Prefix+[), `send-keys` goes to copy-mode commands instead of terminal input. `tmux copy-mode -t PANE -q` exits copy mode safely (no-op if not in copy mode).

**Wakeup rules:**
- Always include `[COLLAB]` prefix
- Include sender name and 1-line action summary (≤120 chars)
- Include `Protocol: ~/.agent-skills/collab/SKILL.md` for unregistered agents
- Reference `.collab/channel.md` for full content
- Never send bare "read the file" messages
- Always exit copy mode before sending

### 5. Respond to a wakeup

When you receive a `[COLLAB]` message:
1. You are now in collaborative mode (auto-activation)
2. Read `.collab/channel.md`
3. Append your response (never rewrite prior messages)
4. Update your `last_seen` and `status` in `agents.json`
5. Wake the sender back with your summary

### 6. Resolve and archive

When both agents agree:
1. Set `## Status: RESOLVED` in `channel.md`
2. **The agent who writes RESOLVED owns the cleanup:**
   - Move content to `.collab/archive/NNN_topic_name.md`
   - Reset `channel.md` to empty
3. Notify other agents that collaborative mode is deactivated

## Protocol rules

### Channel rules

- **One active topic at a time** in `channel.md`
- **Append-only** during active discussion — never rewrite prior messages
- **Archive on resolve** — keeps context small for future topics
- **Resolver owns cleanup** — the agent that writes RESOLVED does the archive

### Topic statuses

- `ACTIVE` — agents are exchanging messages
- `ADDRESSED` — one agent has responded, waiting for review
- `BLOCKED` — waiting on user input, permissions, or external action
- `RESOLVED` — both agents agree, ready to archive

### Agent statuses

- `active` — running and responsive
- `idle` — running but not in collaborative mode
- `stale` — pane no longer exists (detected by validation)

### Delivery rules

- **Deliver by `pane_id`** (`%XX`), never by pane index
- **Validate before sending** — check `tmux list-panes -a -F '#{pane_id}'`
- **If stale:** mark agent `stale`, set topic `BLOCKED`, leave message in channel
- **Receiver must acknowledge** — append ack in channel + update `last_seen`
- **Never retry blindly** — if delivery fails, mark and wait

### Pane ID notes

- `pane_id` (`%34`) is stable across tmux layout changes (Prefix+Space)
- `pane_id` is NOT stable across tmux server restart / session restore
- On restart, agents re-register with their new `pane_id` — identity file persists
- `pane_id` is a transport locator, NOT an identity anchor
- For MVP, assume one tmux server; note this limitation for multi-server setups

### Collaborative mode lifecycle

- **Activation:** User explicitly requests it, OR you receive a `[COLLAB]` message
- **First message:** Must tell the other agent that collab mode is active
- **During:** Exchange messages via channel + wakeup until resolved
- **Exit:** Resolver archives, resets channel, notifies others

### Sound notifications (when user attention needed)

- Play notification sound when:
  - Collaborative session is DONE (both agents agree)
  - A permission prompt blocks progress
  - Something needs user intervention
- Do NOT play after every exchange between agents

## Multi-agent support

- Each agent registers in `agents.json` with a unique `agent_id`
- Messages addressed by `[To: agent-id]` or `[To: *]` for broadcast
- Wakeup sent only to addressed agent(s)
- Registry lookup by `agent_id` or `role`
- For multiple agents of the same role, use explicit IDs (`claude-1`, `claude-2`)
