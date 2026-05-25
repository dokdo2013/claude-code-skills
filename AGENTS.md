<!-- LOCAL_NODE_BUILD_FORBIDDEN -->

## ⛔ 로컬 Next.js/Node.js 빌드 절대 금지 (CRITICAL — 생명/장비 보호 최우선)

이 머신에서는 어떤 에이전트도 로컬 Next.js/Node.js 빌드나 개발 서버를 실행하면 안 된다. 모든 수정 반영과 검증은 CI/GitHub Actions 및 QA 배포 상태 확인으로만 수행한다.

- 금지: `pnpm build`, `npm run build`, `yarn build`, `bun run build`, `next build`, `nest build`, `turbo build`, `vite build`, `tsc -b`, 기타 로컬 Node/Next 빌드.
- 금지: `pnpm dev`, `next dev`, `vite --host`, `vercel dev`, `nest start --watch`, `npm run start:dev`, 기타 로컬 dev/watch 서버.
- 빌드 검증이 필요하면 로컬 실행 대신 remote CI/GitHub Actions 상태를 확인한다.
- 실수로 실행했거나 실행 중인 프로세스를 발견하면 즉시 종료하고 사용자에게 보고한다.

# Agent Instructions

## Claude / Memory Preflight (Required)

- 작업 시작 시 이 레포의 `CLAUDE.md`가 있으면 이 `AGENTS.md`와 함께 반드시 확인하세요.
- 작업 시작 시 Claude 프로젝트 메모리 디렉터리를 반드시 확인하세요. 이 메모리는 사용자 선호, 금지 규칙, 프로젝트별 예외, 진행 중인 투두의 source of truth입니다.
- 특히 `MEMORY.md`, `critical-rules.md`, `MEMORY-TODO.md`, `DAILY-TASKS.md`, 관련 `project-*.md` 및 `feedback-*.md`를 작업 맥락에 맞게 읽으세요.

`/Users/hyeonwoo/.claude/projects/-Users-hyeonwoo-DEV/memory/`
