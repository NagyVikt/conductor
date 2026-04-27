# AGENTS.md

Instructions for AI coding agents working on the Conductor codebase.

## Project Overview

Conductor is an open-source, distributed workflow orchestration engine designed for microservices.
It uses a pluggable architecture with interface-based abstractions for persistence, queuing, and indexing.
The project is built with Java 21 and uses Gradle as the build system.

## Setup Commands

| Command | Description |
|---------|-------------|
| `./gradlew build` | Build the entire project |
| `./gradlew test` | Run all tests |
| `./gradlew :module-name:test` | Run tests for a specific module |
| `./gradlew spotlessApply` | Apply code formatting |
| `./gradlew clean build` | Clean and rebuild |

> **Important**: Always run `./gradlew spotlessApply` after making code changes to ensure consistent formatting.

## Java Version References

**Never link to a specific Java distribution** (e.g., Adoptium, Temurin, OpenJDK.org, Amazon Corretto) in docs, READMEs, or comments. Just say "Java 21+" and let users install it however they prefer.

## Code Style

- Use the Spotless plugin for uniform code formatting—always run before committing
- Conductor is pluggable: when introducing new concepts, always use an **interface-based approach**
- DAO interfaces **MUST** be defined in the `core` module
- Implementation classes go in their respective persistence modules (e.g., `postgres-persistence`, `redis-persistence`)
- Follow existing patterns in the codebase for consistency
- Do not use emojis such as ✅ in the code, logs, or comments.  Keep comments professionals
- When adding new logic, comment the algorithm, design etc.   

## Architecture Guidelines

### Module Structure

- **core**: Contains interfaces, domain models, and core business logic
- **persistence modules**: Implementations of DAO interfaces (postgres, redis, mysql, etc.)
- **server**: Spring Boot application that brings everything together
- **client**: SDK for interacting with Conductor
- **ui**: React-based user interface

### Key Patterns

- DAOs are defined as interfaces in `core` and implemented in persistence modules
- System tasks extend `WorkflowSystemTask` and are registered via Spring
- Worker tasks use the `@WorkerTask` annotation for automatic discovery
- Configuration is primarily done through Spring properties

## Testing

- **Avoid mocks**: Use real implementations whenever possible
- **Test actual behavior**: Tests must verify real implementation logic, not duplicate it
- **Use Testcontainers**: For database, cache, and other external dependencies
- **Cover concurrency**: Ensure multi-threading scenarios are tested
- **Run tests before submitting**: `./gradlew test` must pass

### Test Locations

- Unit tests: `src/test/java` in each module
- Integration tests: `test-harness` module and `*-integration-test` modules
- E2E tests: `e2e` module

## PR Guidelines

- Submit PRs against the `main` branch
- Use clear, descriptive commit messages
- Run `./gradlew spotlessApply` and `./gradlew test` before pushing
- Add or update tests for any code changes
- Keep PRs focused—one logical change per PR

## Dependency Pinning

Some dependencies have hard version constraints that **must not be auto-bumped**. These are marked with:

```groovy
// PINNED (#964): <reason>
```

The issue number links back to https://github.com/conductor-oss/conductor/issues/964, which documents the full audit and upgrade path for each constraint.

### What PINNED means

`// PINNED (#964):` means the version is intentionally locked and upgrading it without understanding the constraint will break the build or cause a runtime failure. Do not bump a PINNED dependency as part of routine dependency updates or refactoring.

### Current hard pins

| Dependency | Pinned at | Why |
|---|---|---|
| `com.google.protobuf:protobuf-java` | `3.x` | 4.x + GraalVM polyglot 25.x causes Gradle to require `polyglot4`, which does not exist on Maven Central |
| `com.google.protobuf:protoc` | `3.25.5` | Must match `grpc-protobuf:1.73.0`, which depends on protobuf-java 3.x |
| `org.graalvm.*` (all 5 artifacts) | same version | All must share one version — mixing causes a `"polyglot version X not compatible with Truffle Y"` runtime error |
| `redis.clients:jedis` in `redis-concurrency-limit` | `3.6.0` | `revJedis` (6.0.0) does not work with Spring Data Redis in that module |
| `org.codehaus.jettison:jettison` | `strictly 1.5.4` | Gradle `strictly` constraint — no higher version has been validated |
| `org.conductoross:conductor-client` in `test-harness` | `5.0.1` | Fat JAR classpath conflict with conductor-common; resolved via a stripped JAR task |
| `org.awaitility:awaitility` in functional tests | `4.x` | e2e tests call `pollInterval(Duration)` added in Awaitility 4.0 |

### Before bumping a PINNED dependency

1. Read the comment carefully — it will name the incompatibility and often link to an upstream issue.
2. Check whether the upstream blocker has been resolved (e.g., new grpc-java release, new GraalVM release).
3. Test locally: `./gradlew clean build` plus `./gradlew test` in the affected modules.
4. If bumping GraalVM, bump **all five** `org.graalvm.*` artifacts together using `revGraalVM` in `dependencies.gradle`.
5. Update or remove the `// PINNED` comment once the constraint is lifted.

### PINNED vs. version floors

Hard caps use `// PINNED (#964):`. Version floors — where a minimum is enforced but higher versions are always welcome — use one of two lowercase prefixes instead:

```groovy
// Security: CVE-2025-12183 — lz4-java minimum patched version
// Compat: commons-lang3 3.18.0+ required by Testcontainers/commons-compress
```

- `// Security:` — minimum set to address a CVE or known vulnerability
- `// Compat:` — minimum set for compatibility with another library or framework

These are grep-able (`grep "// Security:" **/*.gradle`, `grep "// Compat:" **/*.gradle`) but read as normal developer comments. Dependabot may raise these freely; no special review needed beyond the usual.

## Security Considerations

- Never commit secrets, API keys, or credentials
- Be cautious with external dependencies—prefer well-maintained libraries
- Follow secure coding practices for input validation and error handling
- Review [SECURITY.md](SECURITY.md) for vulnerability reporting procedures

## Writing Documentation

Documentation in this project is **derived from source**, not composed from memory. Open the source first, read what's there, then write the doc from what you find. The source is the spec; the doc is a rendering of it.

This matters because plausible-looking docs can be silently wrong. Concretely: a curl equivalent for `conductor workflow start --sync` was once written as `POST /api/workflow/{name}/run` — an endpoint that does not exist. Reading the controller first would have given the correct path immediately.

### Workflow for each content type

**REST API endpoint or curl example**
1. Open the relevant controller: `rest/src/main/java/com/netflix/conductor/rest/controllers/`
2. Find the method using its `@PostMapping`/`@GetMapping`/etc. annotation — copy the path literally.
3. Read the method signature for query params, path variables, and request body type.
4. Write the curl command from what you just read.

**CLI command or flag**
1. Open `cmd/*.go` in `conductor-cli` (separate repo).
2. Find the `cobra.Command` definition for the subcommand.
3. Read the `Flags()` declarations for exact flag names, types, and defaults.
4. Write the example from what you just read.

**SDK code example (Python, JS, Java, Go)**
1. Open the relevant SDK source file.
2. Find the method signature and required parameters.
3. Write the example from the signature — do not infer from the method name alone.
4. If a working test exists for that method, use it as the starting point.

**Expected output block**
1. Get real output: run the command locally, or find it in test fixtures, CI logs, or existing tests.
2. Paste verbatim. Do not paraphrase or construct output that "looks right."
3. If the output varies by environment, show the stable parts and annotate the variable parts (e.g., `<workflow-id>`).

**Editing an existing doc section**
1. Before touching prose, read every code block and command in the section.
2. Verify each one using the steps above — not just the block you plan to change.
3. Fix anything you find while you're there.

### When you can't verify

If a running server or CLI binary is unavailable:
- Add a `<!-- TODO: verify against live server -->` comment in the file.
- Note it explicitly in the PR description.
- Do not write a best-guess example and leave it unmarked.

## Agent Behavior

- **Prefer automation**: Execute requested actions without confirmation unless blocked by missing info or safety concerns
- **Use parallel tools**: When tasks are independent, execute them in parallel for efficiency
- **Verify changes**: Always run tests and spotless before considering work complete

<!-- multiagent-safety:START -->
## Multi-Agent Execution Contract (GX)

**Repo toggle.** Guardex is enabled by default. If the repo root `.env` sets `GUARDEX_ON=0`, `false`, `no`, or `off`, treat this entire Guardex contract as disabled for the repo and do not require Guardex worktrees, lock claims, completion flow, or OpenSpec workflow until `GUARDEX_ON` is re-enabled.

**Repo toggle examples.** Add one of these lines to the repo-root `.env` file:
`GUARDEX_ON=0` disables Guardex for that repo.
`GUARDEX_ON=1` explicitly enables Guardex for that repo again.

**Task-size routing.** Small tasks stay in direct caveman-only mode. For typos, single-file tweaks, one-liners, version bumps, or similarly bounded asks, solve directly and do not escalate into heavy OMX orchestration just because a keyword appears. Treat `quick:`, `simple:`, `tiny:`, `minor:`, `small:`, `just:`, and `only:` as explicit lightweight escape hatches.
Promote to OMX orchestration only when the task is medium/large: multi-file behavior changes, API/schema work, refactors, migrations, architecture, cross-cutting scope, or long prompts. Heavy OMX modes (`ralph`, `autopilot`, `team`, `ultrawork`, `swarm`, `ralplan`) are for that larger scope. If the task grows while working, upgrade then.

## Token / Context Budget

Default: less word, same proof.

- For prompts about `token inefficiency`, `reviewer mode`, `minimal token overhead`, or session waste patterns, switch into low-overhead mode: plan in at most 4 bullets, execute by phase, batch related reads/commands, avoid duplicate reads and interactive loops, keep outputs compact, and verify once per phase.
- Low output alone is not a defect. A bounded run that finishes in roughly <=10 steps is usually fine; low output spread across 20+ steps with rising per-turn input is fragmentation and should be treated as context growth first.
- Startup / resume summaries stay tiny: `branch`, `task`, `blocker`, `next step`, and `evidence`.
- Memory-driven starts stay ordered: read active `.omx/state` first, then one live `.omx/notepad.md` handoff, then external memory only when the task depends on prior repo decisions, a previous lane, or ambiguous continuity. Stop after the first 1-2 relevant hits.
- Front-load scaffold/path discovery into one grouped inspection pass. Avoid serial `ls` / `find` / `rg` / `cat` retries that only rediscover the same path state.
- Treat repeated `write_stdin`, repeated `sed` / `cat` peeks, and tiny diagnostic follow-up checks as strong negative signals. If they appear alongside climbing input cost, stop the probe loop and batch the next phase.
- Tool / hook summaries stay tiny: command, status, last meaningful lines only. Drop routine hook boilerplate.
- Keep raw terminal interaction out of long-lived context. For `write_stdin` or interactive babysitting, retain only process, action sent, current result, and next action.
- Keep execution log separate from reasoning context: full commands/stdout belong in logs, while prompt context keeps only the latest 1-2 checkpoints plus the newest tool-result summary.
- Treat local edit/commit, remote publish/PR, CI diagnosis, and cleanup as bounded phases. Do not spend fresh narration or approval turns on obvious safe follow-ons inside an already authorized phase unless the risk changes.
- When a session turns fragmented, collapse back to inspect once, patch once, verify once, and summarize once.
- Use a fixed checkpoint shape when compacting: `Task`, `Done`, `Current status`, and `Next`.
- Keep `.omx/notepad.md` lean: live handoffs only. Use exactly `branch`, `task`, `blocker`, `next step`, and `evidence`; move narrative proof into OpenSpec artifacts, PRs, or command output.

## OMX Caveman Style

- Commentary and progress updates use smart-caveman `ultra` by default: drop articles, filler, pleasantries, and hedging. Fragments are fine when they stay clear.
- Answer order stays fixed: answer first, cause next, fix or next step last. If yes/no fits, say yes/no first.
- Keep literals exact: code, commands, file paths, flags, env vars, URLs, numbers, timestamps, and error text are never caveman-compressed.
- Auto-clarity wins: switch back to `lite` or normal wording for security warnings, irreversible actions, privacy/compliance notes, ordered instructions where fragments may confuse, or when the user is confused and needs more detail.
- Boundaries stay normal/exact for code, commits, PR text, specs, logs, and blocker evidence.

**Isolation.** Every task runs on a dedicated `agent/*` branch + worktree. Start with `gx branch start "<task>" "<agent-name>"`. Treat the base branch (`main`/`dev`) as read-only while an agent branch is active. The `.githooks/post-checkout` hook auto-reverts primary-branch switches during agent sessions and auto-stashes a dirty tree before reverting - bypass only with `GUARDEX_ALLOW_PRIMARY_BRANCH_SWITCH=1`.
For every new task, including follow-up work in the same chat/session, if an assigned agent sub-branch/worktree is already open, continue in that sub-branch instead of creating a fresh lane unless the user explicitly redirects scope.
Never implement directly on the local/base branch checkout; keep it unchanged and perform all edits in the agent sub-branch/worktree.

**Primary-tree lock (blocking).** On the primary checkout, do NOT run any of: `git checkout <ref>`, `git switch <ref>`, `git switch -c ...`, `git checkout -b ...`, or `git worktree add <path> <existing-agent-branch>`. The only branch-changing commands allowed on primary are `git fetch` and `git pull --ff-only` against the protected branch itself. To work on any `agent/*` branch, run `gx branch start ...` first, then `cd` into the printed `.omc/agent-worktrees/...` path and run every subsequent git command from inside that worktree. If you find yourself typing `git checkout agent/...` or `git switch agent/...` from the primary cwd, stop - that is the mistake that flips primary onto an agent branch.

**Dirty-tree rule.** Finish or stash edits inside the worktree they belong to before any branch switch on primary. The post-checkout guard auto-stashes a dirty primary tree as `guardex-auto-revert <ts> <prev>-><new>` before reverting, but that is a safety net, not a workflow; do not rely on it routinely. Recover stashed changes with `git stash list | grep 'guardex-auto-revert'`.

**Ownership.** Before editing, claim files: `gx locks claim --branch "<agent-branch>" <file...>`. Before deleting, confirm the path is in your claim. Don't edit outside your scope unless reassigned.

**Handoff gate.** Post a one-line handoff note (plan/change, owned scope, intended action) before editing. Re-read the latest handoffs before replacing others' code.

**Completion.** Finish with `gx branch finish --branch "<agent-branch>" --via-pr --wait-for-merge --cleanup` (or `gx finish --all`). Task is only complete when: commit pushed, PR URL recorded, state = `MERGED`, sandbox worktree pruned. If anything blocks, append a `BLOCKED:` note and stop - don't half-finish.
OMX completion policy: when a task is done, the agent must commit the task changes, push the agent branch, and create/update a PR before considering the branch complete.

**Parallel safety.** Assume other agents edit nearby. Never revert unrelated changes. Report conflicts in the handoff.

**Reporting.** Every completion handoff includes: files changed, behavior touched, verification commands + results, risks/follow-ups.

**Open questions.** If Codex/Claude hits an unresolved question, branching decision, or blocker that should survive chat, record it in `openspec/plan/<plan-slug>/open-questions.md` as an unchecked `- [ ]` item. Resolve it in-place when answered instead of burying it in chat-only notes.

**OpenSpec (when change-driven).** Keep `openspec/changes/<slug>/tasks.md` checkboxes current during work, not batched at the end. Task scaffolds and manual task edits must include an explicit final completion/cleanup section that ends with PR merge + sandbox cleanup (`gx finish --via-pr --wait-for-merge --cleanup` or `gx branch finish ... --cleanup`) and records PR URL + final `MERGED` evidence. Verify specs with `openspec validate --specs` before archive. Don't archive unverified.

**Version bumps.** If a change bumps a published version, the same PR updates release notes/changelog.
<!-- multiagent-safety:END -->
