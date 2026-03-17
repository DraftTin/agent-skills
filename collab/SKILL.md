# Skill: Inter-Agent Collaboration (collab)

Enable multiple AI agents (Claude, GPT, Gemini, or any CLI agent) in a tmux session to collaborate through a structured protocol. Agents exchange messages via a shared channel file and use tmux for wakeup notifications.

This skill is model-agnostic. Any agent that can read files, write files, and run bash commands can participate.

## When to use

- When the user asks two or more agents to work together
- When a task benefits from architect + reviewer roles
- When agents need to propose, review, and iterate before implementing

## Architecture

```
.collab/                         ← project-local runtime state (gitignored)
├── agents.json                  ← agent registry (user-triggered writes only)
├── channel.md                   ← ONE active topic
└── archive/                     ← completed topics
    ├── 001_topic_name.md
    └── ...
```

**Transport model:**
- **Files** carry all content (durable, inspectable)
- **Tmux** is only used to wake agents ("go read the channel")
- **Registry** is a lightweight address book — written on registration, read-only during normal operation

## Agent registry

### agents.json

A simple address book. Agents write to it only when explicitly registering. During normal collaboration, agents only read it to look up target pane_ids.

```json
{
  "agents": [
    {
      "name": "Archie",
      "pane_id": "%18",
      "role": "architect",
      "model": "claude-opus-4",
      "description": "Lead architect for Umple LSP. Designs features, writes production code, thinks long-term."
    },
    {
      "name": "Rex",
      "pane_id": "%45",
      "role": "reviewer",
      "model": "gpt-5.4",
      "description": "Code reviewer. Focuses on correctness, edge cases, and design critique."
    }
  ]
}
```

### Agent name

Each agent has a unique name used for addressing. The name can be:
- **User-provided:** "Register yourself as Archie"
- **Auto-generated:** If no name given, use `<role>-<4 random hex chars>`, e.g., `opus-a3f2`

The name is saved to the agent's own config file (CLAUDE.md, CODEX.md, etc.) and memory so it remembers its name across sessions.

### Description

Derived from the agent's config/memory files, not invented fresh each time. No ephemeral state, no pane details. It should be able to summarize the role, capability, and other characterstics of the agent accurately.

## Registration

Registration is **user-triggered only**. No auto-detection on every task.

### Register (user says "register yourself" or "register as <name>")

1. Detect own pane_id: `tmux display-message -p '#{pane_id}'`
2. Determine name:
   - If user provided a name → use it
   - If agent already has a name in its config/memory → use it
   - Otherwise → generate `<role>-<4 random hex chars>`
3. **Check for name conflict:** if the name already exists in `agents.json`:
   - Ask the user: "An agent named X already exists. Replace it?"
   - If yes → overwrite
   - If no → ask user for a different name
4. Read own config/memory files to generate role and description
5. Write entry to `.collab/agents.json`
6. Save name to own config file (CLAUDE.md, CODEX.md, etc.) and memory

### Replace on restore

If a session is closed and restored, the agent registers with the same name (read from its config). The old entry is overwritten with the new pane_id. No conflict prompt needed — same name = same agent returning.

### Update pane (user says "update your pane")

Agent re-detects pane_id and updates its entry in `agents.json`. Only when user explicitly asks — not automatic.

### Remove agent (user says "remove agent X")

Agent deletes the entry with that name from `agents.json`.

### List agents

**MUST read `.collab/agents.json` from disk every time.** Never rely on memory or cached state — another agent may have registered or been removed since you last checked. Always read the actual file and display its current contents.

### Unknown agent

If a target name is not in `agents.json`, do NOT guess. Ask the user to register that agent or provide its pane_id.

## Permissions setup

This skill uses tmux commands for pane detection (during registration only) and message delivery. To avoid permission prompts, configure your agent's CLI to auto-allow tmux commands:

- **Claude Code:** Add `"Bash(tmux *)"` to `allowedTools` in `.claude/settings.json`
- **Codex CLI (OpenAI):** Use `--full-auto` flag or equivalent
- **Gemini CLI:** Configure to allow shell commands without prompting
- **Other agents:** Allow `tmux send-keys`, `tmux list-panes`, `tmux display-message`, `tmux copy-mode`

## Collaboration workflow

### 1. Initialize

```bash
mkdir -p .collab/archive
```

### 2. Start a topic

Write to `.collab/channel.md`:

```markdown
# Topic: [descriptive name]

## Status: ACTIVE

## [From: your-id] [To: target-id]

Your message here...
```

### 3. Send a message

Look up target pane_id from `agents.json`, deliver:

```bash
# Exit copy/scroll mode (prevents keys going to wrong place)
tmux copy-mode -t TARGET_PANE -q 2>/dev/null
sleep 0.1
# Send wakeup with summary
tmux send-keys -t TARGET_PANE -l "[COLLAB] From YOUR_ID: Short summary. See .collab/channel.md" && sleep 0.1 && tmux send-keys -t TARGET_PANE Enter
```

**Wakeup rules:**
- Always include `[COLLAB]` prefix
- Include sender name and 1-line action summary (≤120 chars)
- For unregistered agents, include `Protocol: ~/.agent-skills/collab/SKILL.md`
- Always exit copy mode before sending

### 4. Respond to a wakeup

When you receive a `[COLLAB]` message:
1. You are now in collaborative mode (auto-activation)
2. Read `.collab/channel.md`
3. Append your response (never rewrite prior messages)
4. Wake the sender back with your summary

### 5. Resolve and archive

When both agents agree:
1. Set `## Status: RESOLVED` in `channel.md`
2. **The agent who writes RESOLVED owns the cleanup:**
   - Move content to `.collab/archive/NNN_topic_name.md`
   - Reset `channel.md` to idle
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

### Delivery rules

- **Deliver by `pane_id`** (`%XX`) — stable across tmux layout changes
- **Exit copy mode first** — `tmux copy-mode -t PANE -q 2>/dev/null` before every send
- **Use `-l` flag** — `tmux send-keys -t PANE -l "text"` for literal mode
- **Delayed Enter** — `&& sleep 0.1 && tmux send-keys -t PANE Enter`
- **If delivery fails** — ask the user for the correct pane, don't guess

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

- Each agent registers in `agents.json` with a unique `name`
- Messages addressed by `[To: agent-id]` or `[To: *]` for broadcast
- Wakeup sent only to addressed agent(s)
- User says "list agents" → agent reads and displays `agents.json`

## Project config

Each agent's project config file should include one line:

```markdown
For multi-agent collaboration, follow ~/.agent-skills/collab/SKILL.md. Your name is "<name>".
```

This is the only per-project setup needed. The name is saved after first registration.
