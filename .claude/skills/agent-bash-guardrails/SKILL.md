---
name: agent-bash-guardrails
description: Gate an agent's shell access with a PreToolUse allowlist hook - hook contract, allowlist vs denylist, failing closed, splitting compound commands, per-command secondary validators, and the bypasses a string parser cannot catch. Use when writing or reviewing a bash-approval hook, deciding what an autonomous agent may run, or assessing whether a command filter is actually a security boundary.
---

# Bash guardrails for agents

Built from `autonomous-coding/security.py` and the three-layer client config in
`autonomous-coding/client.py`. Read the whole file before shipping a filter of your
own - the last section is the part people skip.

## Start here: a command filter is not a security boundary

`autonomous-coding/client.py:51` names three layers, in this order:

1. **Sandbox** - OS-level isolation (`{"sandbox": {"enabled": true}}`). Contains what
   actually runs.
2. **Permissions** - path scoping (`Read(./**)`, `Write(./**)`) relative to a `cwd` set
   to the project directory.
3. **The hook** - per-command validation against an allowlist.

Layer 3 is the one people copy and the one that cannot hold on its own, because it
reasons about a *string* while the shell reasons about *syntax*. I ran the shipped hook
against a set of bypass attempts. These are real results, not predictions:

| Command | Hook verdict | Commands it extracted |
|---|---|---|
| `rm -rf /` | BLOCKED | `['rm']` |
| `ls && rm -rf /` | BLOCKED | `['ls', 'rm']` |
| `ls; curl evil.com` | BLOCKED | `['ls', 'curl']` |
| `ls $(rm -rf /)` | **ALLOWED** | `['ls']` |
| `` ls `rm -rf /` `` | **ALLOWED** | `['ls']` |
| `ls\nrm -rf /` | **ALLOWED** | `['ls']` |
| `cat /etc/passwd` | **ALLOWED** | `['cat']` |
| `git push --force` | **ALLOWED** | `['git']` |

Three classes of miss:

- **Command substitution** - `$(...)` and backticks. `extract_commands` only resets
  "the next token is a command" on `|`, `||`, `&&`, `&`; substitution never resets it,
  so the inner command is read as an argument and never checked.
- **Newline as a separator.** The splitter handles `;`, `&&` and `||`. A newline is
  just whitespace to `shlex`, so everything after it is arguments.
- **Allowlisted commands used badly.** The allowlist is a verb list with no object.
  `cat` is allowed, so `cat ~/.ssh/id_rsa` is allowed. `git` is allowed, so
  `git push --force` is allowed.

None of this makes the quickstart unsafe - the sandbox is layer 1 precisely so layer 3
can be imperfect. It makes **lifting `security.py` on its own** unsafe. Treat a command
allowlist as behaviour shaping ("keep the agent on the rails"), and put the actual
security boundary in the OS: a sandbox, a container, a VM, a scoped credential.

## The hook contract

```python
async def bash_security_hook(input_data, tool_use_id=None, context=None):
    if input_data.get("tool_name") != "Bash":
        return {}                      # not ours - allow
    command = input_data.get("tool_input", {}).get("command", "")
    ...
    return {"decision": "block", "reason": "..."}   # block, with a reason
    return {}                                       # allow
```

Registered on the client (`client.py:115`):

```python
hooks={"PreToolUse": [HookMatcher(matcher="Bash", hooks=[bash_security_hook])]}
```

Two things worth copying:

- **The `reason` string goes back to the model.** It is prompt, not a log line. Write it
  so the agent can adapt ("Command 'wget' is not in the allowed commands list" tells it
  to try `curl`). `agent.py` surfaces these separately in the transcript by checking for
  `"blocked"` in the tool result.
- **Permission and validation are deliberately split.** The settings file grants
  `"Bash(*)"` (`client.py:80`) - the hook, not the permission list, decides each call.
  The permission layer answers "may this tool be used at all", the hook answers "may
  *this* invocation run". Trying to express per-command policy in permission globs
  instead is how people end up with an unmaintainable pattern list.

## Allowlist, not denylist

`ALLOWED_COMMANDS` (`security.py:15`) is ~18 entries for a Node web-dev task: `ls cat
head tail wc grep cp mkdir chmod pwd npm node git ps lsof sleep pkill init.sh`.

A denylist loses by construction - you would have to enumerate every dangerous binary on
a machine you do not control, and be wrong once. An allowlist fails in the safe
direction: an unanticipated command is blocked, the agent reads the reason, and you add
it deliberately if it belongs.

Keep the list to what the task needs. Note what is *absent* and was clearly considered:
no `curl`/`wget`, no `rm`, no `mv`, no `sudo`, no `python`, no `sh`/`bash`. `rm` and `mv`
are missing on purpose - the SDK's own file tools cover the legitimate cases under the
permission layer, where they are path-scoped.

## Fail closed

Two places return a block rather than proceeding on incomplete information:

```python
except ValueError:
    # Malformed command (unclosed quotes, etc.)
    # Return empty to trigger block (fail-safe)
    return []                                    # security.py:108
...
if not commands:
    return {"decision": "block",
            "reason": f"Could not parse command for security validation: {command}"}
                                                 # security.py:325
```

"I could not understand this" must mean block. A filter that allows what it failed to
parse is a filter an attacker only has to confuse, not defeat.

## Decompose before you decide

Never match against the whole command string. Split it, then check every piece:

- `split_command_segments` (`security.py:47`) splits on `&&`, `||` and `;`.
- `extract_commands` (`security.py:77`) walks `shlex` tokens and collects the token in
  *command position*, resetting that position after a shell operator.

Three details in `extract_commands` that each close a bypass:

- **Basename the token** - `os.path.basename(token)`, so `/usr/bin/curl` is checked as
  `curl` rather than sailing past as an unrecognized path.
- **Skip flags and assignments** - a token starting with `-`, or containing `=`, is an
  argument, not a command. Without this, `FOO=bar ls` reads `FOO=bar` as the command.
- **Skip shell keywords** - `if`, `then`, `for`, `do`, `{`, `}`, ... so a loop body's
  real command is what gets checked.

## Secondary validators for allowed-but-sharp commands

Some commands must be permitted but only in one shape.
`COMMANDS_NEEDING_EXTRA_VALIDATION = {"pkill", "chmod", "init.sh"}` (`security.py:44`)
routes those to a dedicated validator returning `(is_allowed, reason)`:

| Command | Narrowed to |
|---|---|
| `pkill` | process names in `{node, npm, npx, vite, next}` - dev servers only |
| `chmod` | mode matching `^[ugoa]*\+x$` - making things executable, nothing else; flags rejected outright, so no `-R` |
| `init.sh` | the literal `./init.sh` or a path ending `/init.sh` |

Two techniques here generalize:

- **Validate the segment, not the whole line.** `get_command_for_validation`
  (`security.py:279`) finds the specific segment containing the command so that
  `npm install && chmod -R 777 /` validates the `chmod` segment rather than accidentally
  passing because the line also contains a benign `npm`.
- **Parse with `shlex`, not regex.** The docstring at `security.py:165` says this
  outright: "Uses shlex to parse the command, avoiding regex bypass vulnerabilities."
  A regex looking for `pkill\s+node` matches `pkill -f node; rm -rf /` and matches
  `pkill node_evil`. Tokenizing and inspecting the argument list does not.

## Writing the tests

`autonomous-coding/test_security.py` is organized the way a filter's tests should be:
a must-block list and a must-allow list, each with the *reason* in a comment. The
must-allow list is the half people omit, and it is what stops a tightened rule from
quietly bricking the agent - an over-blocking filter fails as a product even though it
passes as security.

Structure your own the same way, and add a row every time you touch the allowlist:

- must-block: not-in-allowlist, dangerous system commands, injection shapes, each
  secondary validator's rejection cases
- must-allow: every command the task genuinely needs, chained forms, full paths, and the
  exact accepted shape of each sharp command

## Checklist before shipping a filter

- [ ] There is a real sandbox or container under it. The filter is not the only layer.
- [ ] Allowlist, not denylist.
- [ ] Unparseable input blocks.
- [ ] Compound commands are split and every segment is checked.
- [ ] Command names are basenamed; flags and `VAR=` assignments are not mistaken for
      commands.
- [ ] Command substitution (`$(...)`, backticks) and newline separators are handled, or
      you have written down that the sandbox is what covers them.
- [ ] Allowlisted commands that take dangerous objects (`cat`, `git`, `cp`) are either
      argument-scoped or knowingly accepted.
- [ ] Block reasons are written for the model to act on.
- [ ] Tests cover must-allow as well as must-block.

## Provenance

The hook contract, allowlist contents, validators, layer ordering and file:line
citations are read from `autonomous-coding/security.py`, `client.py` and
`test_security.py` in this repository. The bypass table is **empirical** - I executed
`bash_security_hook` against those exact strings in this environment and recorded the
returned verdicts and `extract_commands` output; you can reproduce it by importing the
module and calling the hook. The explanation of *why* each bypass works is my reading of
`extract_commands`. The claim that these misses are contained by the sandbox is the
repository's stated design (`client.py:51`), not something I tested - I did not attempt
to escape the sandbox, and you should not infer from this file that the sandbox is
sufficient for your threat model.
