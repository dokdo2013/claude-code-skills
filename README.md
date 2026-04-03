# Claude Code Skills

Personal Claude Code skills collection.

## Skills

| Skill | Description |
|-------|-------------|
| [memory-todo](skills/memory-todo/SKILL.md) | Memory-based todo management — list, add, complete via natural language |

## Installation

Copy skill directories to `~/.claude/skills/`:

```bash
cp -r skills/memory-todo ~/.claude/skills/
```

## Usage

Once installed, skills are automatically triggered by Claude Code based on conversation context:

- "투두 있어?" / "what's on my todo?" → Lists todos
- "이거 적어둬" / "add this to my todo" → Adds new todo
- "이거 끝났어" / "mark this done" → Completes todo
