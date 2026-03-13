## Inheritance Model (Folder-First)

This repository inherits operational policy from:
- Global baseline: `/Users/jack.reis/Documents/=notes/CLAUDE.md`
- Domain contract: `/Users/jack.reis/Documents/=notes/claude/contracts/DOMAIN-PLUGINS-AGENTS.md`

### Default complex-task behavior
- Plan first using `superpowers:writing-plans`
- Use `openclaw-sync` when orchestration/state is involved
- Execute with `superpowers:executing-plans`

### Local overrides
Keep only repo-specific build/test/deploy details below.

---

# Superpowers

- Fork of the superpowers skills library for Claude Code
- Provides skills: TDD, debugging, brainstorming, git worktrees, code review, and more
- Skills are loaded automatically by Claude Code sessions across all repos

## Portfolio Context

This repo is part of an 18-repo portfolio coordinated from the `=notes` vault.

- **Role**: Skills framework providing development workflows to all Claude Code sessions
- **Coordination**: See `=notes/REPOSITORY-MAP.md` for full dependency graph
- **Relationships**:
  - Skills used by → all repos in the portfolio via Claude Code plugin system
  - Complements → `claude-code-plugins-plus` plugin marketplace (`/Users/jack.reis/Documents/claude-code-plugins-plus/`)
  - Coordinated from → `=notes` vault (`/Users/jack.reis/Documents/=notes/`)

## Repository Remote Policy
- Repo policy: GitHub is primary `origin` and CI/CD source; GitLab is backup remote (`gitlab`) with GitLab CI disabled.

## Local Overrides
- Any capability change must include usage docs/examples.
- Keep plugin/skill APIs stable where possible; note breaking changes explicitly.
- Preserve provider-agnostic interfaces and adapter boundaries.
