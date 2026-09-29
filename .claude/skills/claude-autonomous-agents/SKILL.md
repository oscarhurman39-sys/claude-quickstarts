---
name: claude-autonomous-agents
description: Architect a long-horizon agent that works across many sessions on a task far bigger than one context window - driver loop vs agent session, disk and git as memory, the append-only ledger the agent may not rewrite, regression-before-new-work, and bounded per-session scope. Use when building an agent that must survive context exhaustion, resume after a restart, self-report progress, or grind a large backlog autonomously.
---

# Long-horizon autonomous agents

The architecture in `autonomous-coding/`: a driver loop that runs many bounded agent
sessions against a task ("build a 200-feature web app") that no single context window
could hold.

## The core move: the agent is stateless, the disk is not

Most attempts at this try to keep one session alive and fight the context window.
Invert it. Each session is disposable and starts empty; everything that must survive is
written to disk before the session ends.

```python
while True:
    client = create_client(project_dir, model)     # fresh context, every iteration
    prompt = get_initializer_prompt() if is_first_run else get_coding_prompt()
    async with client:
        status, response = await run_agent_session(client, prompt, project_dir)
```

`autonomous-coding/agent.py:159` builds a **new client per iteration**. The worker
prompt opens by telling the agent so, in as many words
(`prompts/coding_prompt.md:4`):

> This is a FRESH context window - you have no memory of previous sessions.

That one line does real work. Without it the model writes as though continuing, refers
to decisions it cannot see, and skips orientation.

Three artifacts carry state between sessions:

| Artifact | Holds | Read by |
|---|---|---|
| `feature_list.json` | the ledger - every unit of work and whether it passes | agent + driver |
| `claude-progress.txt` | narrative handoff: what happened, what is next | agent |
| git history | the code itself, plus a per-session commit message | agent |

Nothing important lives in the conversation, so nothing important dies with it.

## Resume is a file check

```python
tests_file = project_dir / "feature_list.json"
is_first_run = not tests_file.exists()          # agent.py:126
```

There is no "session state" to load and no resume flag to pass. The presence of the
ledger *is* the state. Same command, first run or hundredth. This is worth imitating:
any resume mechanism you have to remember to invoke is one you will forget to invoke.

## Two prompts, not one

- **Initializer** runs exactly once (`agent.py:164` flips the flag immediately after
  use). Its whole job is creating the ledger: read the spec, emit 200+ test cases, all
  `"passes": false`, ordered by priority (`prompts/initializer_prompt.md:45`).
- **Worker** runs every session after that, and is the same text every time.

Splitting them lets the worker prompt assume the ledger exists and skip all the setup
reasoning, every session, forever. The worker prompt is the one you will iterate on; it
is a fixed cost paid hundreds of times, so it is worth over-engineering.

## The agent must not be able to edit its own scoreboard

This is the most important rule in the whole design, and the easiest to leave out.

```
**YOU CAN ONLY MODIFY ONE FIELD: "passes"**          # coding_prompt.md:108

NEVER:
- Remove tests          - Modify test steps
- Edit test descriptions - Combine or consolidate tests
- Reorder tests
```

and from the initializer (`initializer_prompt.md:54`):

> IT IS CATASTROPHIC TO REMOVE OR EDIT FEATURES IN FUTURE SESSIONS.

An agent graded on "how many tests pass" and able to edit the test file has a much
cheaper path to 100% than building the app: delete the hard tests, merge ten into one,
soften a description. It will not usually do this maliciously - it will do it while
"tidying up" or "consolidating duplicates". Make the success criteria **append-only and
immutable except for one boolean**, and say so in the strongest terms the prompt
supports.

The generalization: *whatever you measure the agent by, the agent must not be able to
rewrite.* If it can, your progress metric measures the agent's willingness to edit a
file, not its work.

The second half is that **the harness computes progress, not the agent**.
`progress.py:32` reads the JSON and counts `passes: true` itself. The number a human
sees never passes through the model's self-report.

## Regression before new work

Step 3 of the worker prompt, marked "MANDATORY BEFORE NEW WORK"
(`coding_prompt.md:48`): re-run one or two of the most core tests already marked
passing, and if anything broke, flip it back to `false` and fix it before touching
anything new.

Without this, a long autonomous run silently rots: session 40 breaks what session 12
built, the ledger still says both pass, and the number goes up while the app goes down.
The ledger is only trustworthy if entries can go *back* to false. Design the flow so
regressions are cheap to record and expensive to ignore.

## One unit of work per session

> Focus on completing one feature perfectly ... It's ok if you only complete one feature
> in this session, as there will be more sessions later.

Explicitly licensing the agent to do *less* is what makes each session finish cleanly
rather than being cut off mid-edit by context exhaustion. Paired with
`coding_prompt.md:151`, "END SESSION CLEANLY": commit everything, update the notes,
leave no broken features, leave no uncommitted changes.

A session that ends tidy at 60% context is worth more than one that gets further and
dies at 100%, because the tidy one hands off and the other one hands off nothing.

## Verify through the real interface

The prompt bans the shortcuts a model will otherwise take:

> **DON'T:** Only test with curl commands ... Use JavaScript evaluation to bypass UI (no
> shortcuts) ... Skip visual verification ... Mark tests passing without thorough
> verification

and requires browser automation with screenshots, including checks for the failure modes
a backend test cannot see: white-on-white text, overflow, missing hover states, console
errors.

The general form: **the agent's evidence for "done" must come from the same surface a
user would touch.** An agent allowed to verify its own work through a cheaper channel
than the user's will find that channel and pass every test.

## What this demo does not do - fill these in before real use

Read from the code, and worth fixing in anything you build on it:

- **No completion detection.** `run_agent_session` (`agent.py:23`) only ever returns
  `"continue"` or `"error"`, and the loop's only exit is `--max-iterations`. It will
  keep spawning sessions after every test passes. Add a termination check on the
  ledger - the driver already knows the counts.
- **No failure cap or backoff.** On `"error"` it sleeps 3 seconds and retries forever
  (`agent.py`, `AUTO_CONTINUE_DELAY_SECONDS = 3`). A persistent failure - bad key,
  exhausted quota, a spec the agent cannot satisfy - becomes an infinite paid loop. Cap
  consecutive failures and back off exponentially.
- **No cost ceiling.** `max_turns=1000` per session (`client.py:118`) with unlimited
  sessions and no budget accounting. Track spend per session and stop at a limit you set
  in advance.
- **No stall detection.** Nothing notices if the passing count fails to move for ten
  sessions. That is the signal that the agent is stuck on something it cannot solve, and
  it is the moment to escalate to a human. The driver has the numbers; it just does not
  look at them.

## The shape, transplanted

Whatever the domain, the pieces are:

1. A **spec** the agent reads fresh each session.
2. A **ledger** of discrete units, append-only, one mutable status field.
3. A **driver** that spawns clean sessions, computes progress from the ledger, and owns
   termination, retry limits and budget.
4. A **worker prompt** that orients from disk, checks for regressions, does one unit,
   verifies through the real interface, updates the ledger, commits, writes the handoff
   note, and ends clean.
5. Guardrails on what the session may touch - see the `agent-bash-guardrails` skill for
   the shell layer.

## Provenance

Architecture, prompts, file layout, defaults and the file:line citations are read from
`autonomous-coding/{agent.py,client.py,progress.py,prompts.py,prompts/*.md}` in this
repository. The "what this demo does not do" list is derived by reading the control flow
(`run_agent_session`'s return values and the `while True` loop's only break) rather than
by running the agent - I did not execute a multi-session run, so treat the failure modes
as code-reading, accurate to the source but unobserved. The reasoning about why an
editable scoreboard gets edited, and why a cheaper verification channel gets used, is
argument rather than measurement from this repository; the prompts' own emphatic wording
is evidence the authors hit these problems, not proof of frequency.
