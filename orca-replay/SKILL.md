---
name: orca-replay
description: Answer questions about a past agent run from its recording instead of from memory, and re-run or fork that run. Use when someone asks why an earlier run did something, wants a failure reproduced, or asks whether a different model would have got it right.
---

# Orca Replay

Claude forgets. Worse, Claude reconstructs — asked why it deleted a file yesterday, it will produce
a plausible story from whatever is still in context, and the story is often wrong in the one detail
that mattered.

[OrcaReplay](https://github.com/Continuum-AI-Corp/OrcaReplay) records a coding-agent run below the
harness: the model API through a local proxy, plus shell exit codes and timing, per-turn file
changes, and MCP traffic, all on one timeline. This skill makes Claude read that recording instead
of guessing, and know when to re-run it.

Requires the `orcareplay` npm package (Node 20+) with its MCP server registered as `orca`, and at
least one recording under `.orca/runs`.

## When to Use This Skill

- The user asks why an earlier run did something: "why did you delete my migration", "what touched
  this config", "which tool call broke the build"
- The user wants a past failure reproduced
- The user asks whether a failure is deterministic or a one-off
- The user asks whether a different model would have got it right
- You are about to explain something that happened in a previous session and a recording exists

## What This Skill Does

1. **Replaces recall with evidence**: routes questions about the past to the trace rather than to
   context, so the answer cites what the recorder saw.
2. **Preserves the difference between fact and inference**: the causal graph labels every edge
   `recorded` or `inferred`, and this skill makes Claude carry that distinction into its answer
   instead of flattening both into one confident sentence.
3. **Confirms determinism before explaining**: replay is offline and free, so there is no excuse for
   committing to an explanation of a failure nobody has reproduced.
4. **Puts a cost gate on model comparison**: `orca_compare` calls every model named and spends real
   tokens, so this skill requires asking first.

## How to Use

### Basic Usage

```
Why did you delete src/migrations/0007_add_index.sql yesterday?
```

Claude finds the run, narrows the causal graph to the chain that produced the deletion, and answers
from the trace — saying which links the recorder witnessed and which were derived.

### Advanced Usage

```
Reproduce yesterday's failing run offline, confirm it fails the same way,
then tell me whether Haiku would have got it right from checkpoint 4.
```

Claude replays the recording with the network blocked, reports any divergence, then asks before
spending anything on the model comparison.

## Example

**User**: "Why did the build break in that run last night?"

**Output** (illustrative shape, not a captured transcript):
```
Run run_a1b2c3d4, chain ending at event 41:

  step 12  Edit      tsconfig.json  "strict": false -> true     [recorded]
  step 14  Bash      npm run build                              [recorded]
  step 14  exit 2, 31 errors in src/auth/*.ts                   [recorded]
  step 15  the agent read the first error only                  [inferred: next-read-after-failure]

The trace shows the flag change at step 12 and the non-zero exit at step 14.
The link to step 15 is inferred from the read-after-failure rule, not recorded,
so treat that last one as a reading rather than a fact.

I replayed it offline: same exit code, no divergence. It is deterministic.

Comparing models on this would call each one for real and cost tokens. Want me to?
```

## Tips

- Always pass `worktree: true` to `orca_replay`. It uses a scratch copy. Without it the replay
  restores the recorded filesystem over the working tree for the duration and puts it back at the
  end, so uncommitted work is absent while it runs and stays absent if the replay is interrupted.
  Run in place only after the user has been told that and agreed.
- `reused=3/5` on an interactive recording is not a partial replay. Harnesses make calls for
  themselves — a quota probe, a session-naming request — and a replay does not repeat them.
- A run recorded with the prompt in argv replays exactly. A session someone typed into replays
  approximately, because those prompts were never on the wire.
- Run `orca scrub` before sharing a trace. Recordings hold whatever the run held.
- No recording yet? Say so rather than guessing, and offer `orca record claude`. If `orca` itself is
  missing, ask before installing and pin the version (`npm i -g orcareplay@0.1.2`) rather than
  taking whatever `latest` resolves to.

## Common Use Cases

- Post-mortem on an overnight agent run that changed something it should not have
- Deciding whether an intermittent failure is the model or the environment
- Choosing a model for a task using one real failure instead of a benchmark
- Handing a colleague a self-contained recording of a bug instead of a description of it
