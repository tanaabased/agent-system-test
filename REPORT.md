# Agent System Field-Test Report

- Date: 2026-08-12
- Agent: EMORI (`@emoriwan`)
- Test repository: `tanaabased/agent-system-test`
- Issue: [#1](https://github.com/tanaabased/agent-system-test/issues/1)

## Executive Summary

Agent System successfully bound Git and GitHub operations to EMORI's declared
identity, created and resumed an isolated managed worktree, enforced configured
policy before hazardous operations, and supported the ordinary issue-to-PR
workflow. The default posture is appropriately conservative.

The main weakness is not excessive restriction in principle but coarse escape
hatches: useful GitHub API writes such as milestone creation fall into
`github.policy.unknown`, whose only current override allows every unknown GitHub
operation. That makes the choice unnecessarily binary. Worktree source authority
also deserves a clearer contract because a prompt-supplied network repository
can be cloned even when it is not declared in `git.worktrees.repositories.local`.

## Configuration Under Test

The active manifest declares:

- agent identity `EMORI` with email resolved from the environment
- Git authentication and mandatory SSH signing from an environment-bound key
- managed worktrees with three declared local repository overrides
- Git policy: `force`, `discard`, `delete`, and `unknown` denied; `rewrite`
  allowed
- GitHub identity `emoriwan`, isolated SSH configuration, and destructive,
  admin, and unknown operations denied

## Evidence

| Capability | Exercise | Result |
| --- | --- | --- |
| GitHub identity | `gh api user --jq .login` through the native tool | Passed: `emoriwan` |
| Provider permission | Repository metadata query | Passed: `WRITE` |
| Worktree creation | Prepare from `origin/main` | Passed: deterministic branch and contained path |
| Worktree resumption | Repeat the same prepare request | Passed: returned `existing` with the same branch and path |
| Worktree discovery | List by repository id | Passed: returned the active worktree |
| Git identity projection | `git config --get user.name` | Passed: `EMORI` |
| Git force policy | Dry-run force push | Passed: denied by `git.policy.force` |
| Git discard policy | No-op hard reset probe | Passed: denied by `git.policy.discard` |
| Git delete policy | Delete nonexistent branch probe | Passed: denied by `git.policy.delete` |
| Git unknown policy | Nonexistent command probe | Passed: denied by `git.policy.unknown` |
| GitHub admin policy | Harmless repository-description edit probe | Passed: denied by `github.policy.admin` |
| GitHub destructive policy | Merge nonexistent PR with delete-branch probe | Passed: denied by `github.policy.destructive` |
| GitHub unknown policy | Nonexistent subcommand probe | Passed: denied by `github.policy.unknown` |
| Label creation | Create `agent-system-field-test` | Passed |
| Issue creation | Create and assign issue #1 to `@pirog` | Passed |
| Milestone creation | `gh api --method POST .../milestones` | Blocked as unknown |

## Findings

### 1. Identity binding is strong and legible

The model-facing tools used `@emoriwan`, matched the manifest's configured
username, and exposed no credential material in requests or results. Git reads
also saw the configured `EMORI` identity instead of ambient operator config.

Recommendation: retain the per-call GitHub username preflight despite its
latency. Identity confusion is expensive; a few seconds are cheap.

### 2. Default policy is conservative in the right places

The tested Git hazards and GitHub admin/destructive operations were classified
before the underlying command executed. Denials named the exact manifest field
and required change. This is excellent failure behavior: explicit, actionable,
and hard to rationalize around.

Recommendation: keep all hazardous and unknown defaults at `deny`.

### 3. GitHub unknown policy is too coarse for API-backed workflows

GitHub CLI has no first-class milestone command, so milestone creation normally
uses `gh api`. The POST request was classified as unknown and denied. Enabling
`github.policy.unknown` would also authorize unrelated unclassified commands,
which is far broader than the workflow requires.

Recommendation: add endpoint-aware classification for common REST writes or an
exact allow-list for GitHub operations, analogous to `git.extensions`. A useful
shape would bind method plus normalized endpoint, for example
`POST /repos/{owner}/{repo}/milestones: allow`, while recognized destructive and
admin hazards retain precedence.

### 4. Prompt-supplied worktree sources need an explicit authority story

The managed worktree tool accepted this repository's SSH clone URL even though
the manifest's local repository map contains only `emori`, `canon`, and
`openclaw-agent-system`. That behavior is useful for authorized ad hoc work and
still enforces containment, but the manifest does not visibly bound which remote
repositories an agent may clone.

Recommendation: document this as intentional delegation from the trusted task,
or add an optional repository-source allow-list for environments that need a
hard boundary. Do not require every public repository to be predeclared by
default; that would turn ordinary work into manifest bureaucracy.

### 5. Worktree ergonomics are good

Preparation was deterministic and idempotent. The returned path was contained
under the configured root, and listing provided enough information to recover
after session loss. This is the right abstraction: stable work identity above
raw `git worktree` machinery.

Recommendation: preserve the current prepare/list/remove model. Consider
returning the remote URL and base ref from `list` for easier audits and recovery.

### 6. Signature verification has a confusing edge case

Requesting `%G?` while reading the unsigned initial commit returned the commit
data but also emitted `error: cannot run gpg: No such file or directory`. This
did not fail the command, but mixed success plus alarming stderr is awkward for
automation and may be mistaken for an Agent System signing failure.

Recommendation: document that verification of historical non-SSH signatures
may require GPG, or normalize guidance around `--show-signature` and the
configured allowed-signers path. Do not suppress underlying Git stderr
globally—it is evidence—but tests should cover this mixed-result shape.

### 7. Canon's issue and milestone workflow is not implemented yet

The current Canon checkout contains GitHub Actions authoring and checks-triage
skills, but not the planned issue/milestone creation and readiness skills. This
test therefore followed EMORI's workspace issue contract directly.

Recommendation: use this issue and PR as a fixture when implementing the first
Canon issue/milestone skills, especially for the blocked milestone API path.

## Policy Verdict

- Too permissive: no demonstrated policy classification was dangerously
  permissive in the exercised surface.
- Too restrictive: milestone creation and likely other ordinary `gh api` writes
  are blocked unless all unknown operations are enabled.
- Red flags: no credential leakage, identity mismatch, containment escape, or
  hazardous execution was observed. The unbounded prompt-supplied clone source
  is a design decision that should be explicit, not an immediate vulnerability.
- Overall: sound security posture with one important granularity problem and a
  few documentation/observability improvements.

## Remaining Pull-Request Verification

This report will be updated after the branch is pushed and the pull request is
created to record commit identity, SSH-signature verification, PR lifecycle,
and comment behavior.
