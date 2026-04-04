# Claude Code Skills

Personal Claude Code skills collection, distributed as a plugin marketplace.

## Plugins

| Plugin | Description |
|--------|-------------|
| **memory-todo** | Memory-based todo management — list, add, complete via natural language (Korean + English) |
| **dylabs-devops** | dylabs DevOps runbook — deploy services, manage Terraform, debug Kubernetes, configure monitoring |
| **todo-review** | Review todo implementation status — parallel sub-agent investigation, completion verification, cleanup recommendations |

## Installation

### Add marketplace

```bash
/plugin marketplace add dokdo2013/claude-code-skills
```

### Install a plugin

```bash
# Todo manager
/plugin install memory-todo@claude-code-skills

# DevOps runbook (dylabs team only)
/plugin install dylabs-devops@claude-code-skills
```

### Verify

```bash
/plugins
```

## Usage

### memory-todo

| Trigger | Action |
|---------|--------|
| "투두 있어?" / "what's on my todo?" | Lists todos |
| "이거 적어둬" / "add this to my todo" | Adds new todo |
| "이거 끝났어" / "mark this done" | Completes todo |

Todos are stored in `MEMORY-TODO.md` + `project-*.md` files in your project's memory directory.

### dylabs-devops

| Trigger | Action |
|---------|--------|
| "배포 설정" / "deploy new service" | Walks through the deployment checklist |
| "모니터링 추가" / "add monitoring" | PrometheusRule + Grafana dashboard setup |
| "인프라 변경" / "terraform plan" | Terraform workflow with approval gate |
| "장애 대응" / "debug k8s" | Diagnostic commands for Kubernetes troubleshooting |

### todo-review

| Trigger | Action |
|---------|--------|
| "투두 리뷰" / "todo review" | Reviews today+yesterday todos with parallel sub-agents |
| "어디까지 됐어" / "what's done?" | Checks implementation status of todo items |
| "상태 확인" / "check status" | Verifies completion, recommends cleanup |

## Development

### Local testing

```bash
claude --plugin-dir ./plugins/memory-todo
```

### Validate

```bash
/plugin validate .
```

## Structure

```
claude-code-skills/
├── .claude-plugin/marketplace.json           # Marketplace catalog
├── README.md
├── plugins/
│   ├── memory-todo/
│   │   ├── .claude-plugin/plugin.json        # Plugin metadata
│   │   └── skills/memory-todo/SKILL.md       # Skill definition
│   ├── dylabs-devops/
│   │   ├── .claude-plugin/plugin.json
│   │   └── skills/dylabs-devops/SKILL.md
│   └── todo-review/
│       ├── .claude-plugin/plugin.json
│       └── skills/todo-review/SKILL.md
└── skills/                                   # Legacy (pre-plugin format)
    ├── memory-todo/SKILL.md
    └── dylabs-devops/SKILL.md
```
