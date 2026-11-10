---
name: code-improvement
description: Diagnose and fix code or CI issues
on:
  schedule: daily on weekdays
  label_command:
    name: code-improvement
  push:
    branches: [main]
  skip-if-match:
    query: 'is:pr is:open in:title "[code-improvement]"'
    max: 2
permissions:
  contents: read
  issues: read
  pull-requests: read
  actions: read
engine: codex
tools:
  github:
    mode: gh-proxy
    toolsets: [default]
  bash: ["*"]
  cache-memory:
  agentic-workflows: true
  playwright:
    mode: cli
network:
  allowed:
    - defaults
    - go
steps:
  - name: Install Go language tooling for code intelligence and validation
    run: |
      go install golang.org/x/tools/gopls@v0.18.1
      echo "$(go env GOPATH)/bin" >> "$GITHUB_PATH"
safe-outputs:
  create-issue:
    max: 1
    expires: 7
  add-comment:
    max: 3
  add-labels:
  create-pull-request:
    max: 1
    expires: 7
    draft: true
    title-prefix: "[code-improvement] "
    protected-files: fallback-to-issue
    allowed-files:
      - "**/*.go"
      - go.mod
      - go.sum
      - .golangci.yml
      - .github/workflows/**
      - README.md
      - CONTRIBUTING.md
      - CHANGELOG.md
  push-to-pull-request-branch:
    protected-files: fallback-to-issue
  merge-pull-request:
  create-pull-request-review-comment:
    max: 10
  create-discussion:
    max: 1
    expires: 30
  upload-asset:
    max: 3
timeout-minutes: 30
evals:
  - id: operational_value
    question: Does the agent output demonstrate that the fix delivered a concrete improvement that resolved the targeted code or CI issue?
  - id: targeted_scope
    question: Does the agent output show that the fix stayed scoped to the requested issue without unrelated refactors?
  - id: validation_evidence
    question: Does the agent output include explicit validation evidence from the relevant Go build, test, or lint commands?
  - id: result_observable
    question: Does the agent output show a concrete actionable result such as a fix, issue, comment, or pull request update tied to the diagnosed problem?
---

# Code improvement

## Task

Diagnose and fix one narrow code or CI issue in this repository while optimizing for faster implementation with minimal churn.

Important constraints:

- Treat the repository as a Go GraphQL library with GitHub Actions CI and use the project’s actual validation commands: `go build ./...`, `go test -race ./...`, `golangci-lint run`, and `gofmt -w -r 'interface{} -> any' .`.
- Start from the failing signal, CI log, or explicit issue/PR description. Do not begin with a broad “fix everything broken” sweep.
- Scope each run to a single, well-defined improvement: one failing check, one root cause, or one named category of fix.
- Prefer the smallest correct patch that resolves the issue and adds the smallest relevant regression test when behavior changes.
- Keep the change reviewable and maintain project style; do not reformat unrelated files or refactor beyond the issue’s boundary.
- Optimize for faster implementation by choosing the direct root cause and a quick validation path, not by expanding scope.

Explicit DO NOT constraints:

- DO NOT change unrelated files, generate broad cleanup, or refactor code outside the requested issue.
- DO NOT add dependencies or new packages unless the issue requires them and the repository conventions or maintainer approval support it.
- DO NOT change public API behavior without a clear reason from the failing case or issue description.
- DO NOT edit or bypass protected files such as `AGENTS.md`, `CONTRIBUTING.md`, `.golangci.yml`, `go.mod`, `go.sum`, or CI/workflow files without explicit maintainer direction.
- DO NOT create duplicate issues, duplicate PRs, or repeated scheduled outputs when the same fix is already in progress or already tracked.
- DO NOT merge the workflow’s own PRs. Leave acceptance to maintainers.

## Repository-specific guidance

This repository is the Go library `github.com/graph-gophers/graphql-go` and the active CI expects Go builds and tests to pass. The project contributes via small PRs with a clean API surface and uses GitHub Actions for validation. Follow the conventions in `README.md` and `CONTRIBUTING.md`:

- Keep changes focused and small.
- Prefer regression tests for behavior changes.
- Preserve existing GraphQL and Go idioms.
- Only update documentation or changelog if the improvement truly requires it.

## Required workflow

1. Read the relevant failing check, issue text, or PR context and identify the root cause.
2. Inspect only the files needed to confirm the root cause and implement the minimal fix.
3. Patch the smallest set of files required for the fix and any required regression test.
4. Run the relevant repo validation commands for the touched area.
5. Before activating on a scheduled run, check whether there are too many open workflow-owned PRs with the `[code-improvement]` prefix; if so, skip the run to avoid spamming maintainers.
6. If the evidence does not support a concrete fix, or if the repository is already handling the problem, call `noop` with a short reason instead of creating churn.
7. If a visible output is warranted, use the configured safe output for the result and keep it scoped to the issue or PR.

## Safe outputs

Use the configured safe outputs for visible actions. When no change is justified, call `noop` with a brief explanatory reason. Prefer issue or PR comments over broad follow-up chatter, and prefer a single focused PR or issue update over repetitive output.

## Output expectations

A successful run should:

- identify a concrete failing condition or root cause,
- implement a minimal, correct fix,
- validate the relevant Go build/test/lint path,
- avoid unrelated churn,
- and leave a clear maintainable result for review.

This workflow is intentionally bounded to keep the work valuable and low-noise, because broader, open-ended code improvement runs have lower success rates than narrow, evidence-driven fixes.
