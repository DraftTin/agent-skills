# Agent Skills

A collection of reusable skills for AI coding agents. Model-agnostic — works with Claude, GPT, Gemini, or any CLI-based agent.

## Available Skills

| Skill | Description |
|-------|-------------|
| [collab](collab/) | Inter-agent collaboration protocol for tmux sessions |

## Installation

```bash
git clone https://github.com/yourname/agent-skills ~/.agent-skills
```

Then reference skills from your project's agent config:

```markdown
# In CLAUDE.md, CODEX.md, or any agent instruction file:
For collaboration protocol, read ~/.agent-skills/collab/SKILL.md
```

## How it works

Each skill is a self-contained markdown file (`SKILL.md`) that teaches an agent a workflow. Skills are model-agnostic — any agent that can read a file and follow instructions can use them.

Skills may include:
- `SKILL.md` — the skill definition (when/how to use)
- `templates/` — starter files for the skill's runtime state

Runtime state (e.g., `.collab/`) lives in the project directory and is gitignored.
