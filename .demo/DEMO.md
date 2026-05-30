# Depot CI Agentic Validation Demo

Run-through for the Battery product discussion. The goal is a punchy 5-10
minute demo that makes "dynamic workflows" and "intelligence around validation
loops" concrete without overclaiming magic.

## Core Narrative

Historically CI was a human-paced loop:

```text
edit -> push -> wait -> read CI -> fix -> push again
```

Agentic development breaks that model. Agents can generate code faster than
traditional CI can validate it, so CI has to become a callable validation
substrate:

```text
edit -> run real CI on local patch -> read logs -> fix -> rerun -> push once green
```

Depot CI is not just faster runners. The differentiation is the stack working
together:

- Depot CI for orchestration.
- Depot Cache for incremental rebuilds and tests.
- Depot Registry for fast artifact handoff between jobs.
- Depot Container Builds for native, cached image builds.
- CLI/API/logs/status/SSH for agents and engineers to drive the loop directly.
- Per-second economics that make many short validation loops practical.

That combination enables targeted feedback loops for agents, whether they call
static workflows, select a narrow job from a larger workflow, or generate a
dynamic workflow on the fly.

## Demo Thesis

Lead with **static workflow, dynamic targeting**.

The money shot is not "watch an agent write YAML." The money shot is:

> An agent understands the change it made, chooses the smallest meaningful CI
> loop, runs real CI against local uncommitted code, reads logs, fixes the code,
> reruns, and only pushes once green.

Dynamic workflow generation is still worth mentioning, but as the next level:

> Because Depot CI accepts workflow YAML and exposes CLI/API control, this can
> be a checked-in workflow, a narrow job from the main workflow, or a workflow
> an agent generates on the fly. The important thing is Depot gives agents a
> fast, cache-backed, observable execution substrate.

## Pre-Call Prep

Do these before the call.

1. Build or confirm the custom CI base image exists:

   ```bash
   depot ci run --workflow .depot/workflows/snapshot-e2e.yml
   ```

   Workflow: `.depot/workflows/snapshot-e2e.yml`

   The image is referenced by `.depot/workflows/lint.yml`:

   ```yaml
   runs-on:
     size: 4x16
     image: d58mfwccbf.registry.depot.dev/falseflag-ci-base:go1.26-node22-pnpm10.13-pw1.49-spectral6.14-postgres16
   ```

2. Have one successful narrow run ID ready.

   Good options:

   ```bash
   depot ci run --workflow .depot/workflows/lint.yml
   depot ci run --workflow .depot/workflows/ci.yml --job test-js
   ```

3. Have one failed-then-fixed run ID ready.

   The most compelling version is a tiny JS/dashboard or lint failure:

   - Make a small local change that fails lint or tests.
   - Run the targeted Depot CI job.
   - Capture the run ID.
   - Fix the code locally.
   - Rerun the same targeted loop.

4. Keep the full matrix as dashboard evidence, not live execution.

   Live-run the narrow loop. Show the full workflow in the Depot dashboard or
   CLI output from an earlier run.

5. Open tabs/windows:

   - Repo terminal at `/Users/kylegalbraith/projects/depot/false-flag-demo`
   - `.depot/workflows/lint.yml`
   - `.depot/workflows/ci.yml`
   - `.depot/workflows/build.yml`
   - Depot CI dashboard with recent narrow and full runs
   - Optional: local FalseFlag dashboard at `http://localhost:3030/projects`

6. Optional local product pre-flight:

   ```bash
   docker compose down -v
   docker compose up -d --build
   make seed
   make smoke
   ```

   This is not the center of the investor demo, but it proves the repo has real
   product surface area behind the CI story.

## 1. Set The Problem, 45 Seconds

Say:

> Agents do not need a prettier PR page. They need a fast way to prove their
> changes before they push. Traditional CI is push, wait, guess. Agentic CI is:
> run real validation on the local patch, read the result programmatically, fix,
> rerun, and push once green.

Show:

```text
Old loop:   edit -> push -> wait -> read CI -> fix -> push again
Agent loop: edit -> run CI locally -> read logs -> fix -> rerun -> push once green
```

Transition:

> The question Battery is asking as "dynamic workflows" is really: can Depot
> make validation callable, targeted, and fast enough that agents can use it as
> an inner loop?

## 2. Show The Repo Has Real CI Gravity, 60 Seconds

Say:

> This is not a toy repo. FalseFlag is a synthetic feature flag platform with a
> Go API, Remix dashboard, TypeScript SDK, Go SDK, Kubernetes operator, MCP
> server, Hurl tests, Playwright tests, Docker images, Postgres and SQLite
> backends, generated code, and conformance tests across runtimes.

Show the workflows:

```bash
ls .depot/workflows
```

Point at:

- `.depot/workflows/ci.yml`
- `.depot/workflows/lint.yml`
- `.depot/workflows/build.yml`
- `.depot/workflows/snapshot-e2e.yml`

Then show the jobs in the larger workflow:

```bash
rg '^  [a-zA-Z0-9_-]+:' .depot/workflows/ci.yml
```

Say:

> This is the kind of CI surface real customers accumulate: codegen, lint,
> tests, race tests, contract tests, image builds, image scanning, smoke tests,
> browser tests, and backend matrices.

## 3. Show The Optimized Validation Substrate, 90 Seconds

Open:

```bash
sed -n '1,180p' .depot/workflows/lint.yml
```

Show the custom image:

```yaml
runs-on:
  size: 4x16
  image: d58mfwccbf.registry.depot.dev/falseflag-ci-base:go1.26-node22-pnpm10.13-pw1.49-spectral6.14-postgres16
```

Say:

> This image snapshots the expensive setup: Go, Node, pnpm, Playwright,
> Spectral, browser dependencies, and the base Postgres image. The agent does
> not wait for dependency installation from scratch every time it wants
> feedback.

Show parallel setup:

```yaml
parallel:
  - name: go mod download
    run: go mod download
  - name: pnpm install
    working-directory: js
    run: pnpm install --frozen-lockfile
```

Show parallel validation:

```yaml
parallel:
  - name: Check codegen freshness
  - name: Lint go
  - name: Lint JavaScript
  - name: Typecheck
  - name: Lint OpenAPI
```

Say:

> This is what I mean by dynamic validation loops. The workflow is structured so
> independent checks can run together, and the agent can choose this loop
> directly when the change calls for it.

## 4. Live Agentic Loop, 2-3 Minutes

This is the centerpiece. Use a small local change and a narrow validation loop.

Agent prompt to use:

```text
Add a `starts_with` string operator to the FalseFlag targeting engine.

Implement the Go evaluation path first and add a shared conformance fixture that
uses `starts_with` against a user/email or request/path attribute. Keep the
change small and focused: update only the predicate/operator handling and the
fixture needed to prove the behavior.

Before you push or commit anything, validate the change with Depot CI against
the smallest relevant loop. Start with:

depot ci run --workflow .depot/workflows/ci.yml --job conformance

If CI fails, inspect status and logs with `depot ci status` and `depot ci logs`,
fix the failing runtime, and rerun only the conformance job until it is green.
Report what failed, what you changed, and the final green run ID.
```

Expected failure path:

> The Go runtime learns `starts_with` first, but the TypeScript SDK's matching
> evaluator does not know it yet. The shared conformance job should fail with a
> real cross-runtime mismatch. The agent reads the logs, adds the TypeScript
> twin, and reruns only `conformance`.

Option A: run the dedicated fast lint workflow:

```bash
depot ci run --workflow .depot/workflows/lint.yml
```

Option B: target one job from the larger workflow:

```bash
depot ci run --workflow .depot/workflows/ci.yml --job conformance
```

Say:

> No commit. No PR. Depot detects the local diff, uploads a patch, applies it
> after checkout, and runs the real workflow remotely. That is the core unlock
> for agents: they can validate work before polluting the branch or waiting on
> GitHub events.

Then inspect the run:

```bash
depot ci status <run-id>
```

If showing a failure:

```bash
depot ci logs <run-id> --job conformance
```

Say:

> This is the feedback an agent can consume. It does not need a browser. It can
> ask for status, fetch the failed job logs, edit the code, and rerun the same
> loop.

Then rerun after the fix:

```bash
depot ci run --workflow .depot/workflows/ci.yml --job conformance
```

Close the beat:

> The important part is not that a human clicked rerun. The important part is
> that this is all CLI/API-driven, so an agent can own the loop.

## 5. Show The Full Pipeline Without Waiting Live, 90 Seconds

Open `.depot/workflows/ci.yml` and `.depot/workflows/build.yml`, or switch to a
Depot dashboard run.

Show these pieces:

- `build-images` calls `.depot/workflows/build.yml`.
- `build.yml` uses `depot/bake-action`.
- Images are saved and handed to downstream jobs.
- `smoke` pulls `api`, `proxy`, and `mcp` images via `depot/pull-action`.
- `dashboard-e2e` is sharded across 6 shards and runs against the saved images.
- `contract-test`, `smoke`, and `dashboard-e2e` run across Postgres and SQLite.

Say:

> The live loop is narrow because agents need fast feedback. The full confidence
> loop is broader: images, smoke tests, browser tests, matrices. Depot connects
> the pieces underneath: cached container builds, registry handoff, CI
> orchestration, logs, metrics, and status.

Useful snippets to show:

```yaml
uses: depot/bake-action@v1
```

```yaml
uses: depot/pull-action@v1
with:
  build-id: ${{ needs.build-images.outputs.build-id }}
  targets: api,proxy,mcp
```

```yaml
matrix:
  backend: [postgres, sqlite]
  shard: [1, 2, 3, 4, 5, 6]
```

## 6. Answer "How Is This Different?", 60 Seconds

Say:

> GitHub Actions, Circle, and Buildkite are mostly human-paced CI systems:
> push a commit, wait for an event, inspect a UI. Depot CI is programmable CI
> for engineers and agents: run against local patches, scope to jobs, fetch
> logs, SSH into sandboxes, use custom images, parallelize steps, and pay by the
> second.

Then add:

> And it is not one isolated feature. It is Depot CI, Depot Cache, Depot
> Registry, Depot Container Builds, logs, metrics, and CLI/API control all
> connecting under the hood. That is what makes short targeted validation loops
> practical at agent scale.

If they ask specifically about dynamic workflows:

> There are three levels. Level one is a static workflow with dynamic targeting:
> the agent picks `test-js`, `lint`, or `smoke` based on the change. Level two
> is a static workflow with dynamic subsets and matrices. Level three is an
> agent-generated workflow when the repo does not already express the needed
> check. Depot supports the substrate for all three, but the most credible demo
> is level one because it is useful immediately.

## 7. Close With The Series B Story, 45 Seconds

Say:

> The bigger market shift is that validation volume explodes when agents are
> writing code. The winning CI platform is no longer just the one with faster
> runners. It is the one agents can use as infrastructure: fast startup,
> cache-backed, API-driven, observable, debuggable, and cheap enough to call
> constantly.

Then:

> That is the wedge. Depot starts as acceleration for builds and CI, but in an
> agentic world it becomes the validation substrate that lets code-writing
> agents ship safely.

## Optional Product Context: FalseFlag In 90 Seconds

Use this only if they want to see the app before the CI loop.

```bash
docker compose ps
open http://localhost:3030/projects
```

Say:

> This is FalseFlag, a believable feature flag platform built specifically to
> create real CI gravity: API, dashboard, SDKs, proxy, operator, MCP server, and
> multiple config strategies.

Click:

```text
acme-internal -> feature-x -> Edit
```

Say:

> The product is demo-quality, but the CI load is real. That is the point: Depot
> is not being shown against a toy "hello world" repo.

## Commands Cheat Sheet

Narrow local-patch runs:

```bash
depot ci run --workflow .depot/workflows/lint.yml
depot ci run --workflow .depot/workflows/ci.yml --job test-js
depot ci status <run-id>
depot ci logs <attempt-id>
```

Custom image:

```bash
depot ci run --workflow .depot/workflows/snapshot-e2e.yml
```

Local app smoke:

```bash
docker compose up -d --build
make seed
make smoke
```

Repo inspection:

```bash
ls .depot/workflows
rg '^  [a-zA-Z0-9_-]+:' .depot/workflows/ci.yml
sed -n '1,180p' .depot/workflows/lint.yml
sed -n '1,220p' .depot/workflows/build.yml
```

## Talk Track In One Paragraph

Depot's pitch for agentic development is not "we made runners faster." It is:
we made CI callable. An agent can run real CI against local uncommitted code,
target only the validation loop that matters, reuse cached dependencies and
images, read status and logs through the CLI/API, debug with SSH if needed, and
rerun until green before pushing. That is what turns CI from a human-paced gate
into infrastructure for agent-driven development.
