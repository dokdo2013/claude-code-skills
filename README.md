# Claude Code Skills

Personal Claude Code skills collection, distributed as a plugin marketplace.

## Plugins

| Plugin | Description |
|--------|-------------|
| **memory-todo** | Memory-based todo management — list, add, complete via natural language (Korean + English) |
| **dylabs-devops** | dylabs DevOps runbook — deploy services, manage Terraform, debug Kubernetes, configure monitoring |
| **todo-review** | Review todo implementation status — parallel sub-agent investigation, completion verification, cleanup recommendations |
| **side-effect-analyzer** | Cross-repo side effect analysis — trace upstream/downstream impact of code changes across all services |
| **dead-code-remover** | Dead code analysis and safe removal — detect unused imports, functions, types, classes and remove with build/test verification |
| **feature-flag** | Feature flag management across meloming platforms — add, toggle, remove flags consistently across Front/iOS/Android with PostHog |
| **sre-daily-check** | SRE daily health check — cluster nodes, pods, Alertmanager, Prometheus, WAF BLOCK logs, Pod right-sizing |
| **posthog-daily-check** | PostHog daily product-health check — DAU, exception spikes, rageclick hotspots, feature flag rollout changes, dead tracking detection |

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

### side-effect-analyzer

| Trigger | Action |
|---------|--------|
| "사이드이펙트 점검" / "side effect check" | Analyzes cross-repo impact of current changes |
| "영향 범위 분석" / "impact analysis" | Greps all repos for changed symbols, dispatches sub-agents |
| Before commit/PR | Auto-suggests side effect check if not done yet |

### dead-code-remover

| Trigger | Action |
|---------|--------|
| "dead code 정리" / "dead code removal" | Full pipeline: detect → confirm → remove → build/test verify |
| "미사용 코드 삭제" / "unused code cleanup" | Same full pipeline |
| "안 쓰는 코드 정리" | Same full pipeline |

### sre-daily-check

| Trigger | Action |
|---------|--------|
| "/sre" / "SRE 점검" / "시스템 점검" | Runs daily infra health check — cluster, alerts, metrics, WAF, pod right-sizing |
| "클러스터 상태" / "daily health check" | Same workflow |

### posthog-daily-check

| Trigger | Action |
|---------|--------|
| "/posthog" / "PostHog 점검" / "제품 지표 점검" | Runs daily product-health check — DAU, exceptions, rageclicks, flag rollouts, dead tracking |
| "지표 이상 탐지" / "posthog daily check" | Same workflow |

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
│   ├── todo-review/
│   │   ├── .claude-plugin/plugin.json
│   │   └── skills/todo-review/SKILL.md
│   ├── side-effect-analyzer/
│   │   ├── .claude-plugin/plugin.json
│   │   └── skills/side-effect-analyzer/SKILL.md
│   └── dead-code-remover/
│       ├── .claude-plugin/plugin.json
│       └── skills/dead-code-remover/SKILL.md
└── skills/                                   # Legacy (pre-plugin format)
    ├── memory-todo/SKILL.md
    └── dylabs-devops/SKILL.md
```
