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

An agent understands the change it made, chooses the smallest meaningful CI
loop, runs real CI against local uncommitted code, reads logs, fixes the code,
reruns, and only pushes once green.

Dynamic workflow generation is still worth mentioning, but as the next level:

Because Depot CI accepts workflow YAML and exposes CLI/API control, this can be
a checked-in workflow, a narrow job from the main workflow, or a workflow an
agent generates on the fly. The important thing is Depot gives agents a fast,
cache-backed, observable execution substrate.

## Pre-Call Prep

Do these before the call.

1. Confirm the repo instructions include the Depot CI defaults:
   - Depot org: `d58mfwccbf`
   - GitHub repo: `Zagrit-HQ/false-flag-demo`
   - Main workflow: `.depot/workflows/ci.yml`
   - Fast validation workflow: `.depot/workflows/lint.yml`

2. Build or confirm the custom CI base image exists:

   ```bash
   depot ci run --workflow .depot/workflows/snapshot-e2e.yml
   ```

   The image is referenced by `.depot/workflows/lint.yml`:

   ```yaml
   runs-on:
     size: 4x16
     image: d58mfwccbf.registry.depot.dev/falseflag-ci-base:go1.26-node22-pnpm10.13-pw1.49-spectral6.14-postgres16
   ```

3. Have one successful narrow run ID ready.

   Good options:

   ```bash
   depot ci run --org d58mfwccbf --repo Zagrit-HQ/false-flag-demo --workflow .depot/workflows/lint.yml
   depot ci run --org d58mfwccbf --repo Zagrit-HQ/false-flag-demo --workflow .depot/workflows/ci.yml --job conformance
   ```

4. Have one failed-then-fixed run ID ready, or rehearse the deterministic
   `starts_with` flow below once before the call.

5. Keep the full matrix as dashboard evidence, not live execution.

   Live-run the narrow loop. Show the full workflow in the Depot dashboard or
   CLI output from an earlier run.

6. Open tabs/windows:
   - Repo terminal at `/Users/kylegalbraith/projects/depot/false-flag-demo`
   - `.depot/workflows/lint.yml`
   - `.depot/workflows/ci.yml`
   - `.depot/workflows/build.yml`
   - Depot CI dashboard with recent narrow and full runs
   - Optional: local FalseFlag dashboard at `http://localhost:3030/projects`

7. Optional local product pre-flight:

   ```bash
   docker compose down -v
   docker compose up -d --build
   make seed
   make smoke
   ```

## 1. Set The Problem, 45 Seconds

**Talk track:**

The core shift is that agents make code generation cheap, but they make
validation demand explode. If every agent-produced change has to go through the
old loop of push, wait, read logs, copy context back, and push again, the team
does not actually get faster.

What agents need is a way to call the real delivery pipeline while they are
working: run targeted CI against the local patch, inspect the result, fix what
broke, rerun the same loop, and only push once the change is already green.

Show:

```text
Old loop:
edit -> push -> wait for CI -> move logs back into the agent context
     -> agent fixes code -> push again -> repeat until green

New loop:
edit -> agent triggers targeted CI on the local patch -> agent reads status/logs
     -> agent fixes code -> reruns the same loop -> push once green
```

**Transition:**

The question Battery is asking as "dynamic workflows" is really: can Depot make
validation callable, targeted, and fast enough that agents can use it as an
inner loop?

## 2. Show The Repo Has Real CI Gravity, 60 Seconds

**Talk track:**

This is not a toy repo. FalseFlag is a synthetic feature flag platform with a
Go API, Remix dashboard, TypeScript SDK, Go SDK, Kubernetes operator, MCP
server, Hurl tests, Playwright tests, Docker images, Postgres and SQLite
backends, generated code, and conformance tests across runtimes.

Show the workflows:

```bash
ls .depot/workflows
```

Then show the jobs in the larger workflow:

```bash
rg '^  [a-zA-Z0-9_-]+:' .depot/workflows/ci.yml
```

**Talk track:**

This is the kind of CI surface real customers accumulate: codegen, lint, tests,
race tests, contract tests, image builds, image scanning, smoke tests, browser
tests, and backend matrices.

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

**Talk track:**

This how we leverage Depot Cache for customers inside of Depot CI.
This snapshots the expensive setup: Go, Node, pnpm, Playwright, Spectral,
browser dependencies, and the base Postgres image. The agent does not wait for
dependency installation from scratch every time it wants feedback.

The runner is automatically populated with this cache for anything the agent wants to validate.
It just has to specify that it wants to run on this snapshot.

Show parallel setup and validation in `.depot/workflows/lint.yml`:

```yaml
parallel:
  - name: go mod download
  - name: pnpm install
```

```yaml
parallel:
  - name: Check codegen freshness
  - name: Lint go
  - name: Lint JavaScript
  - name: Typecheck
  - name: Lint OpenAPI
```

**Talk track:**

This is what I mean by dynamic validation loops. The workflow is structured so
independent checks can run together, and the agent can choose this loop directly
when the change calls for it.

The same substrate also lets an agent learn from the checked-in validation loops,
construct a bespoke workflow for a specific change, and have Depot CI execute it
against the local patch.

## 4. Live Agentic Loop, 2-3 Minutes

This is the centerpiece. Use one prompt that forces a real CI-guided loop.

Agent prompt to paste:

```text
Use the depot-ci skill for continuous verification in this repo.

Add a new `starts_with` string predicate to the FalseFlag targeting engine.
Focus only on the Go implementation for now and only address other runtimes if
Depot CI proves they fail.

Make sure you add a shared fixture under `tests/eval-corpus/**`.
```

Expected failure path:

The Go runtime learns `starts_with` first, but the TypeScript SDK evaluator does
not know it yet. The shared conformance job fails with a real cross-runtime
mismatch. The agent reads the logs, adds the TypeScript twin, and reruns only
`conformance`.

**Talk track while it runs:**

No commit. No PR. Depot detects the local diff, uploads a patch, applies it
after checkout, and runs the real workflow remotely. That is the core unlock for
agents: they can validate work before polluting the branch or waiting on GitHub
events.

Commands to narrate if needed:

```bash
depot ci status <run-id>
depot ci diagnose --org d58mfwccbf --run <run-id>
depot ci logs <run-id> --job conformance
```

**Close the beat:**

The important part is not that a human clicked rerun. The important part is that
this is all CLI/API-driven, so an agent can own the loop.

All of the products of Depot are just running in the background making this pipeline
orders of magnitude faster. So the agent can write the code, get validation, turn CI green,
and then push the code.

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

**Talk track:**

The live loop is narrow because agents need fast feedback. The full confidence
loop is broader: images, smoke tests, browser tests, matrices. Depot connects
the pieces underneath: faster container builds baked in, registry integration out of the box,
CI orchestration, logs, metrics, and status.

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

## DO THIS SECTION AS CLOSING

## 6. Answer "How Is This Different?", 60 Seconds

**Talk track:**

The old delivery infrastructure model assumes humans are the scarce resource. A human writes code,
pushes a branch, waits for CI, reads the logs, and decides what to do next. That
model is already painful for teams, but it breaks completely when agents start
producing many more candidate changes.

In an agentic world, the bottleneck moves from writing code to validating code.
The winning delivery infrastructure platform is the one agents can call continuously while they work.

That is the difference with Depot. Depot CI is not just a faster place to run a
push-triggered workflow. It is a programmable validation substrate.

**Then add:**

The advantage compounds across the product surface. Depot CI handles the
orchestration, runner performance, and workflow plumbing with very little
dependency on GitHub. Depot Cache is plugged directly into the execution
environment, so cached results are available to the next runner automatically.
Depot Registry gives each validation loop fast access to the images and
environment snapshots produced by earlier runs.

That means the entire validation loop can be driven independently by agents
inside their existing workflow. They do not have to test something locally,
discover CI behaves differently, have an engineer copy logs between tools, or wait for a
push-triggered system to tell them what broke. The agent can run the same CI
environment it will be judged by, inspect the result, fix the code, and rerun
the targeted loop immediately.

**If they ask specifically about dynamic workflows:**

There are three levels. Level one is a static workflow with dynamic targeting:
the agent picks `conformance`, `lint`, or `smoke` based on the change. Level two
is a static workflow with dynamic subsets and matrices. Level three is an
agent-generated workflow when the repo does not already express the needed
check. Depot supports all three of these.

## Future looking: agent written validation expansion

We intend to expand the functionality across Depot CI to make things even more
generic and accessible to agents so that they can write & execute their own
validation workflows, however they want to express them, via this same interface.

We've already solved the hardest part of that problem by having this validation
engine being directly integrated with all of the other Depot components in our
specialized delivery execution layer.

## 7. Close With The Series B Story, 45 Seconds

**Talk track:**

The bigger market shift is that validation volume explodes when agents are
writing code. The winning CI platform is no longer just the one with faster
runners. It is the one agents can use as infrastructure: fast startup,
cache-backed, API-driven, observable, debuggable, and cheap enough to call
constantly.

**Then:**

That is the wedge. Depot starts as acceleration for builds and CI, but in an
agentic world it becomes the validation substrate that lets code-writing agents
ship safely.

## Commands Cheat Sheet

Narrow local-patch runs:

```bash
depot ci run --org d58mfwccbf --repo Zagrit-HQ/false-flag-demo --workflow .depot/workflows/lint.yml
depot ci run --org d58mfwccbf --repo Zagrit-HQ/false-flag-demo --workflow .depot/workflows/ci.yml --job conformance
depot ci status <run-id>
depot ci diagnose --org d58mfwccbf --run <run-id>
depot ci logs <run-id> --job conformance
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
