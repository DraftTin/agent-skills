# Skill: Inter-Agent Collaboration (collab)

Enable multiple AI agents (Claude, GPT, Gemini, or any CLI agent) in a tmux session to collaborate on tasks through a structured protocol. Agents discover each other via a registry, exchange messages through a shared channel file, and use tmux only for wakeup notifications.

This skill is model-agnostic. Any agent that can read files, write files, and run bash commands can participate.

## When to use

- When the user asks two or more agents to work together ("solve this with Codex", "talk to the other agent")
- When a task benefits from architect + reviewer roles
- When agents need to propose, review, and iterate before implementing

## Quick start

### 1. Initialize collaboration

Create the `.collab/` directory in the project root:

```bash
mkdir -p .collab/archive
```

### 2. Register yourself

Auto-detect your tmux pane and write to the registry:

```bash
# Get your pane address
MY_PANE=$(tmux display-message -p '#{session_name}:#{window_index}.#{pane_index}')

# Write/update your entry in agents.json
# (Use your agent tool to read/write JSON, or use jq)
```

Write to `.collab/agents.json`:
```json
{
  "agents": [
    {
      "id": "your-agent-id",
      "role": "architect",
      "model": "your-model",
      "pane": "session:window.pane",
      "status": "active",
      "last_seen": "ISO-timestamp"
    }
  ],
  "channel": ".collab/channel.md",
  "created": "ISO-timestamp"
}
```

### 3. Start a topic

Clear `channel.md` and write your message:

```markdown
# Topic: [descriptive name]

## Status: ACTIVE

## [From: your-id] [To: target-id]

Your message here...
```

### 4. Wake the other agent

Look up target pane from registry, send a short summary:

```bash
PANE=$(jq -r '.agents[] | select(.id=="TARGET_ID") | .pane' .collab/agents.json)
tmux send-keys -t "$PANE" -l "[COLLAB] From YOUR_ID: One-line summary. See .collab/channel.md" && sleep 0.1 && tmux send-keys -t "$PANE" Enter
```

**Wakeup rules:**
- Always include `[COLLAB]` prefix
- Include sender name and 1-line action summary
- Keep under ~120 characters
- Reference `.collab/channel.md` for details

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
2. Move the content to `.collab/archive/NNN_topic_name.md`
3. Reset `channel.md` to empty/idle
4. The agent who writes `RESOLVED` owns the cleanup

## Protocol rules

### Channel rules
- **One active topic at a time** in `channel.md`
- **Append-only** during active discussion — never rewrite prior messages
- **Archive on resolve** — keeps context small for future topics

### Topic statuses
- `ACTIVE` — agents are exchanging messages
- `ADDRESSED` — one agent has responded, waiting for review
- `BLOCKED` — waiting on user input, permissions, or external action
- `RESOLVED` — both agents agree, ready to archive

### Agent statuses (in agents.json)
- `active` — agent is running and responsive
- `idle` — agent is running but not in collaborative mode
- `stale` — pane no longer exists (detected by validation)

### Safety rules
- **Validate panes before sending** — check `tmux list-panes` to confirm target pane exists; mark `stale` if missing
- **Receiver must acknowledge** — append an ack or response in `channel.md` and update `last_seen`
- **Delivery failure** — if pane is stale, mark topic `BLOCKED` and leave message in channel for user
- **No silent partial work** — if collaborative mode is active and you need user attention, play a notification sound

### Collaborative mode lifecycle
- **Activation:** User explicitly requests it, OR you receive a `[COLLAB]` message
- **First message:** Must tell the other agent that collab mode is active and remind them of the protocol
- **During:** Exchange messages via channel + wakeup until resolved
- **Exit:** One agent writes `RESOLVED`, archives, resets channel, notifies other agent that collab mode is deactivated

### Sound notifications (when user attention needed)
- Play `afplay /System/Library/Sounds/Glass.aiff` (macOS) when:
  - Collaborative session is DONE (both agents agree)
  - A permission prompt blocks progress
  - Something needs user intervention
- Do NOT play after every exchange between agents

## Multi-agent support

The protocol supports N agents:
- Each registers in `agents.json`
- Messages are addressed by `[To: agent-id]` or `[To: *]` for broadcast
- Wakeup is sent only to the addressed agent(s)
- Registry lookup by `id` or `role`

## File reference

```
.collab/                     ← project-local, gitignored
├── agents.json              ← live agent registry
├── channel.md               ← ONE active topic
└── archive/                 ← completed topics
    ├── 001_topic_name.md
    └── ...
```

## Example exchange

```
Agent A writes to channel.md:
  # Topic: Fix rename timeout
  ## Status: ACTIVE
  ## [From: opus] [To: codex]
  The rename handler times out...

Agent A wakes Agent B:
  [COLLAB] From opus: Analyze rename timeout root cause. See .collab/channel.md

Agent B reads, responds in channel.md:
  ## [From: codex] [To: opus]
  Root cause is indexWorkspace()...

Agent B wakes Agent A:
  [COLLAB] From codex: Root cause identified, proposal in channel. See .collab/channel.md

...iterate until resolved...

Agent A writes:
  ## Status: RESOLVED

Agent A moves to archive/001_rename_timeout.md, resets channel.md
Agent A notifies:
  [COLLAB] From opus: Topic resolved. Collaborative mode deactivated.
```
