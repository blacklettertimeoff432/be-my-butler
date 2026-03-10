# Changelog

All notable changes to BMB will be documented in this file.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), versioned per [Semantic Versioning](https://semver.org/).

## [0.1.0] - 2026-03-10

### Added
- 11-step multi-agent orchestration pipeline (`/BMB` skill)
- 8 specialized agents: Consultant, Architect, Executor, Frontend, Tester, Verifier, Simplifier, Writer
- Cross-model blind verification (Codex, Gemini support)
- Council debate with blind divergent framing
- Worktree isolation for safe parallel execution
- 6 recipe types: feature, bugfix, refactor, research, review, infra
- FTS5 knowledge base with `knowledge-index.sh` and `knowledge-search.sh`
- 3-tier auto-learning system (project → global → CLAUDE.md promotion)
- Conversation logger (Python FIFO-based async logging)
- Cross-model runner with `--profile` permission control
- `/BMB-brainstorm`, `/BMB-refactoring`, `/BMB-setup` skills
- Install / Doctor / Uninstall lifecycle scripts
- Interactive architecture docs page (GitHub Pages)
- i18n: English, Korean, Japanese, Traditional Chinese
- CI workflows: ShellCheck, markdownlint, install smoke test
- Demo todo-app example project

### Fixed
- CI workflow failures (ShellCheck, markdownlint, install test)
- Repository URL references updated to `project820/be-my-butler`
- GitHub metadata: topics, homepage URL, Discussions enabled

### Known Issues
- `bmb-learn.sh`: missing `mkdir -p` for global learnings directory
- Worktree cleanup does not delete branches after `git worktree remove`

[0.1.0]: https://github.com/project820/be-my-butler/releases/tag/v0.1.0
