# agentic-team-work

A team of six GitHub Copilot custom agents that turns a work request into a spec, an implementation, verification evidence, an independent review, and an honest status report. The agents share state through files in `.copilot-work/` in the target project, so every handoff is auditable.

## The agents

| Agent | Role | Writes | Never does |
|---|---|---|---|
| [`coordinator`](coordinator.agent.md) | Orchestrator. Delegates each phase in order, loops fixes until checks and review are clean, and returns the reporter's summary. | `coordinator-log.md` (and `.gitignore`, with permission) | Implement, fix, test, or review anything itself |
| [`planner`](planner.agent.md) | Architect. Turns a rough idea into a testable SPEC and a 2-5 milestone PLAN. | `spec.md`, `plan.md` | Write code, pseudo-code, or function signatures |
| [`coder`](coder.agent.md) | Implementer. Builds one milestone at a time, adds tests including an edge case, runs the project's checks. | Source and tests, `progress.md` | Question the spec or skip ahead to later milestones |
| [`tester`](tester.agent.md) | CI runner. Finds the project's test, lint, format, and type-check commands, runs them, and reports PASS / FAIL / NOT RUN with pasted output. | `test-report.md` | Edit code or suggest fixes |
| [`critic`](critic.agent.md) | Skeptical senior reviewer. Compares the code to the spec item by item and rates defects as blocker, warning, or nit. | `review-report.md` | Modify code or praise it |
| [`reporter`](reporter.agent.md) | Status reporter. Reads all artifacts and writes a COMPLETE / PARTIAL / BLOCKED summary. | `summary.md` | Run commands, change code, or claim success over failing checks |

The separation is deliberate. The agent that writes code does not verify it, the agent that verifies it does not fix it, and the agent that reports on it did not do any of the work. Each one has a narrow tool set and hard limits, so no single agent can grade its own homework.

## How the work flows

```
request ─► planner ─► coder ─► tester ─┐
                        ▲              ├─► clean? ─► next milestone
                        └─ fix round ◄─┤                │
                                critic ┘                ▼
                                          final tester + critic ─► reporter ─► summary
```

1. **Plan.** The planner writes `spec.md` (goal, inputs, acceptance criteria, failure modes) and `plan.md` (milestones with definitions of done).
2. **Build, per milestone.** The coder implements one milestone. The tester and critic then run, in parallel, against the result.
3. **Fix rounds.** If any check fails or the critic raises a blocker or warning, the coder gets the defects quoted verbatim and tries again. Nits are carried to the final report. After two failed fix rounds the coordinator stops and reports rather than thrashing.
4. **Final verification.** Tester and critic cover the whole change, not just the last milestone.
5. **Report.** The reporter writes `summary.md` with an honest status, what was verified, and what remains.

## Shared workspace

```
.copilot-work/
  spec.md             planner
  plan.md             planner
  progress.md         coder (appended per milestone)
  test-report.md      tester
  review-report.md    critic
  summary.md          reporter
  coordinator-log.md  coordinator
```

The coordinator offers to add `.copilot-work/` to `.gitignore` and archives artifacts from earlier runs so stale reports are never mistaken for current ones.

## Using them together

### Full pipeline (the usual case)

Give the coordinator a request. It runs everything and returns one summary.

```
@coordinator Add a --json flag to the export command that emits the same data as structured JSON. Errors should still go to stderr.
```

For a bug fix, state the symptom and the expected behavior:

```
@coordinator Fix: importing a CSV with a BOM header drops the first column. Add regression coverage.
```

For eliminating a whole class of problem, say so. The coordinator tells the planner to include a mechanical check that finds zero instances, which gives the rest of the team an objective finish line:

```
@coordinator Eliminate all unwrap() calls on external input in src/parser/.
```

### Individual agents

Each agent works on its own when you only need one phase.

- **Plan before committing to work.** `@planner Draft a spec and plan for rate limiting on the public API.` Review `.copilot-work/spec.md` and edit it yourself before any code is written.
- **Implement a reviewed plan, one step at a time.** `@coder Implement milestone 2 from .copilot-work/plan.md.`
- **Get an independent check on code you wrote by hand.** `@tester Run the project's checks and report.` then `@critic Review my changes against .copilot-work/spec.md.`
- **Summarize a finished effort for a teammate.** `@reporter Summarize the current state of .copilot-work/.`

### Mixed workflows

- **Human-gated planning.** Run `@planner`, read and tighten the spec, then hand the same request to `@coordinator`. Start from a spec you trust and the downstream agents inherit that quality. (The coordinator archives existing artifacts on a new run, so re-state the tightened requirements in your request.)
- **Review-only on existing work.** Write a short `.copilot-work/spec.md` describing what the change should do, then run `@tester` and `@critic` for evidence-based verification and a skeptical review without touching the code.
- **Resume after a blocker.** If the coordinator stops after two failed fix rounds, read `coordinator-log.md` and `review-report.md`. Fix the underlying issue (often an ambiguous spec), then re-run with a corrected request.

## Practices that produce good results

- **Write acceptance criteria that can fail.** The coder and critic treat the spec as a contract. Vague wording gives vague code. The planner is told to avoid words like "handle" and "manage" without defining them.
- **Small milestones.** Each one covers one behavior or one module, so a failed review has a small blast radius.
- **Evidence over assertion.** Statuses come from pasted command output and the critic's explicit verdicts. Nobody says "done" on their own authority.
- **Independent verification.** Tests and review are done by agents that did not write the code, and the critic is told to assume at least one defect exists.
- **Honest reporting.** "Approved with nits" is never rewritten as "all good", and PARTIAL or BLOCKED is a perfectly acceptable outcome when it is the truth.
- **Keep the team out of your way.** Agents run non-interactively. If the request is too vague to plan, the team stops and says what it needs instead of guessing.

## Setup

Place the `*.agent.md` files in `~/.copilot/agents/` (user-level) or `.github/agents/` (per-repo). Project conventions in `.github/copilot-instructions.md` or `AGENTS.md` are read by the coordinator and passed along to the team.
