# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Repo Rules — Read First

**Never push directly to `main`.** Branch protection requires a pull request and a passing CI build. Bypassing this rule is grounds for revoking Claude Code access.

Before any commit:
1. Create a feature branch (`git checkout -b <type>/<short-description>`)
2. Update `grafana_alloy/CHANGELOG.md` with the version entry
3. Update `grafana_alloy/README.md` if any options or behavior changed
4. Open a PR via `gh pr create` and let CI run

---

## Project Structure

```
grafana_alloy/
  config.yaml          # Addon manifest — options, schema, version
  Dockerfile           # Builds the addon image
  README.md            # User-facing documentation
  CHANGELOG.md         # Version history
  build.yaml           # Multi-arch build config
  rootfs/
    etc/alloy/
      config.alloy     # Default Alloy config (non-override mode)
    etc/services.d/
      alloy/run        # s6 service entrypoint — sources options, execs alloy
.github/
  workflows/           # CI: build and publish to GHCR on push to main
  PULL_REQUEST_TEMPLATE.md
.claude/
  agents/              # 14-agent Claude Code pipeline
```

---

## Scope Constraints

Changes should stay within `grafana_alloy/` and `.github/workflows/` unless there is an explicit reason to touch something else. Do not modify the repo root unless adding repo-level tooling (like this file).

---

## Development Conventions

- **Version format:** `major.minor.patch` in `grafana_alloy/config.yaml`. Bump `minor` for new features, `patch` for bug fixes.
- **Options:** Declare in both `options` (with default) and `schema` (with type) in `config.yaml`. Use `password?` for secrets/tokens (masked in UI, optional).
- **Secret handling:** Addon options support `!secret <key>` in the HA UI — the supervisor resolves secrets.yaml references before writing `/data/options.json`. Read secrets via `bashio::config 'key'`, then export as env vars for Alloy. Never read or parse `secrets.yaml` directly.
- **Alloy env vars:** Export as `UPPER_SNAKE_CASE` env vars in `rootfs/etc/services.d/alloy/run` before the `exec` call. Alloy reads them via `env("VAR_NAME")` in config files.
- **No secrets in files:** Tokens, passwords, and credentials must never appear in config files, documentation, dashboards, or committed code. Use env vars, HA secrets, or `!secret` references.

---

## Agent Pipeline

A 14-agent pipeline lives in `.claude/agents/`. Use it for any non-trivial task.

### Pipeline Flow

```
Git (branch) → Memory → Router → Research → Plan
  → [per subtask] Test → Execute → Debug? → Review → Critic?
→ E2E → Critic (final) → Docs → Git (PR) → Summarize
```

### Agent Reference

| Agent | When to use directly |
|---|---|
| **Orchestrator** | Hand it a task and let the full pipeline run |
| **Plan** | Research and plan a task before any implementation |
| **Research** | Investigate a codebase area, error, or dependency |
| **Execute** | Implement a specific, well-defined subtask |
| **Test** | Write failing tests before implementation (TDD) |
| **Review** | Check a piece of work against acceptance criteria |
| **Critic** | Adversarial review — security, edge cases, failure modes |
| **Debug** | Diagnose an unknown error or flaky test |
| **E2E** | Write integration and end-to-end tests for a completed feature |
| **Docs** | Write inline comments, READMEs, changelogs, or ADRs |
| **Git** | Create a branch, write a commit message, or open a PR |
| **Summarize** | Produce a clean summary of completed work |

---

## Common Workflows

### New addon option
```
Plan → Execute → Docs (README + CHANGELOG) → Git (PR)
```

### Bug fix
```
Research → Execute → Review → Git (PR)
```

### Version bump / release prep
```
Docs (CHANGELOG) → Execute (config.yaml version) → Git (PR)
```

---

## CI

On push to `main`, `.github/workflows/` builds and publishes a multi-arch Docker image to:
```
ghcr.io/just-broome/hassos-addons/grafana_alloy:<version>
```

The image is referenced directly in `config.yaml` (`image:` field). HA pulls the image tagged with the addon version on install/update.
