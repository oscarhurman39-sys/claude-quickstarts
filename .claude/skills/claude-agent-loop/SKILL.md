---
name: claude-agent-loop
description: Build or debug a hand-written tool-use agent loop against the Claude Messages API - loop shape, tool schemas, tool_result pairing, prompt-cache breakpoint placement, token accounting, context truncation, and MCP tool bridging. Use when writing an agentic loop, adding or changing tools, or diagnosing 400s on tool results, parallel tool calls that quietly stop happening, or a cache hit rate stuck at zero.
---

# Claude agent loop

How to write the loop, taken from the reference implementation in `agents/` and the
sampling loop in `computer-use-demo/computer_use_demo/loop.py`.

Reach for this when you are building the loop **yourself**. If the user just wants an
agent and does not care who owns the loop, say so first: the SDK's tool runner
(`client.beta.messages.tool_runner`) removes most of this file, and Managed Agents
removes the hosting too. Hand-writing the loop is the right call when you need control
flow the runner's per-turn hooks do not fit.

## The loop

```python
from anthropic import Anthropic

client = Anthropic()

async def agent_loop(user_input, tools, tool_dict, system_blocks):
    messages = [{"role": "user", "content": user_input}]

    while True:
        response = client.messages.create(
            model="claude-opus-5",
            max_tokens=16000,
            system=system_blocks,
            tools=[tool.to_dict() for tool in tools],
            messages=messages,
        )

        # Append the whole content list, never an extracted string.
        messages.append({"role": "assistant", "content": response.content})

        tool_calls = [b for b in response.content if b.type == "tool_use"]
        if not tool_calls:
            return response

        results = await execute_tools(tool_calls, tool_dict)   # concurrently
        messages.append({"role": "user", "content": results})  # one message
```

`agents/agent.py:96` is this loop with logging and MCP setup around it.

**Append `response.content` wholesale.** Pulling the text out and appending
`{"role": "assistant", "content": text}` silently drops thinking blocks and compaction
blocks. Compaction in particular is stateful: the server hands back a compaction block
that replaces the compacted history on the next request, so dropping it loses the
summarization and the context comes back in full.

## The five invariants

Break one of these and you get either a 400 or - worse - a silent behavioural
regression that looks like the model got dumber.

1. **Every `tool_use` gets a `tool_result` with the same `tool_use_id`**, in the very
   next user message. No exceptions, no dropping the ones you did not like.
2. **All results for one assistant turn go in ONE user message.** Splitting them across
   several user messages trains the model to stop making parallel calls. Nothing errors;
   the model just gets more serial over the conversation and you never find out why.
   `agents/utils/tool_util.py:33` gathers them concurrently and returns a single list
   precisely so the caller appends one message.
3. **A tool failure is a `tool_result` with `is_error: True`, not a raised exception.**
   `agents/utils/tool_util.py:7` catches `KeyError` (unknown tool) and `Exception`
   (anything the tool threw) and turns both into error results. An exception that
   escapes the executor kills the turn and leaves a `tool_use` with no result, so the
   next request 400s on the orphan.
4. **Never orphan a `tool_result` when you trim history.** See below - this is the trap.
5. **Parse tool inputs as JSON; never string-match the serialized input.** Current
   models vary their JSON escaping (Unicode, forward slashes) in `tool_use.input`.

## The truncation trap

`agents/utils/history_util.py:69` drops history in *pairs* and then overwrites the new
first message with a placeholder:

```python
def remove_message_pair():
    self.messages.pop(0)
    self.messages.pop(0)
...
remove_message_pair()
self.messages[0] = TRUNCATION_MESSAGE   # "[Earlier history has been truncated.]"
```

The overwrite looks cosmetic. It is load-bearing. Trace a tool conversation:

```
[u0, a1(tool_use), u2(tool_result), a3(tool_use), u4(tool_result), a5]
pop two   ->  [u2(tool_result), a3, u4, a5]      # u2's tool_use is gone: orphaned
overwrite ->  [u_trunc,         a3, u4, a5]      # orphan destroyed, history valid
```

Two consequences worth knowing before you copy this:

- It is only safe because the history **strictly alternates user/assistant**, so index 0
  is always a user message. The API permits consecutive same-role messages and combines
  them into one turn; the moment you append two user messages in a row, pair-wise
  popping desynchronizes and this starts orphaning results for real.
- It destroys information rather than summarizing it. Before reaching for it, check
  whether the two server-side options fit: **context editing** (beta
  `context-management-2025-06-27`, `clear_tool_uses_20250919`) clears old tool results,
  and **compaction** (beta `compact-2026-01-12`) summarizes. Hand-rolled truncation is
  the fallback, not the default.

## Prompt caching

Caching is a **prefix match** over `tools` -> `system` -> `messages`. One byte changes
anywhere in the prefix and everything after it is invalidated. Cache reads bill at
roughly 0.1x input; cache writes at roughly 1.25x.

Two placements appear in this repo, and they are not equivalent:

**Single trailing breakpoint** (`agents/utils/history_util.py:113`) puts
`cache_control` on every block of the last message. Simple, but the breakpoint moves
every turn, so each turn writes a fresh cache entry.

**Rolling breakpoints** (`computer-use-demo/computer_use_demo/loop.py:287`) is the better
pattern for a long loop:

```python
breakpoints_remaining = 3
for message in reversed(messages):
    if message["role"] == "user" and isinstance(content := message["content"], list):
        if breakpoints_remaining:
            breakpoints_remaining -= 1
            content[-1]["cache_control"] = {"type": "ephemeral"}
        else:
            # delete the breakpoint that has rolled off
            if isinstance(content[-1], dict) and "cache_control" in content[-1]:
                del content[-1]["cache_control"]
            break
```

Three breakpoints on the three most recent user turns, **one deliberately left for
`tools`/`system`** so that prefix stays cached across sessions. The request cap is 4
breakpoints, so the reservation is the whole point of stopping at 3. Deleting the
rolled-off breakpoint matters: a stale `cache_control` left in place is a byte change in
the prefix on the next turn.

Rules that follow from prefix-matching:

- Nothing volatile before the last breakpoint. A `datetime.now()` in the system prompt,
  an unsorted `json.dumps`, or a tool list whose order varies per request will pin your
  hit rate at zero.
- The minimum cacheable prefix is model-dependent (roughly 512-4096 tokens). Below it,
  caching silently does nothing - not an error, just no hit.
- **Verify, do not assume.** If `usage.cache_read_input_tokens` is zero across repeated
  requests with the same prefix, you have a silent invalidator.

## Token accounting

The three usage counters are disjoint, and this is the single easiest thing to get
wrong:

| Field | What it counts | Relative price |
|---|---|---|
| `usage.input_tokens` | uncached input only | 1x |
| `usage.cache_creation_input_tokens` | tokens written to cache | ~1.25x |
| `usage.cache_read_input_tokens` | tokens served from cache | ~0.1x |

`input_tokens` is **not** the total. To get true context size you must sum all three,
which is what `agents/utils/history_util.py:60` does:

```python
total_input = (
    usage.input_tokens
    + getattr(usage, "cache_read_input_tokens", 0)
    + getattr(usage, "cache_creation_input_tokens", 0)
)
```

Reading `input_tokens` alone on a well-cached loop makes context look like it is barely
growing, right up until the request 400s for exceeding the window.

For counting ahead of a request, use `client.messages.count_tokens`. Never use
`tiktoken` - it is a different tokenizer and the answer will be wrong.
`history_util.py:31` falls back to `len(system) / 4` when the count call fails; that is
a crash-guard heuristic, not a measurement, and it should not be load-bearing.

## Tools

The contract is three fields (`agents/tools/base.py`):

```python
@dataclass
class Tool:
    name: str
    description: str
    input_schema: dict[str, Any]

    def to_dict(self) -> dict[str, Any]:
        return {"name": self.name, "description": self.description,
                "input_schema": self.input_schema}

    async def execute(self, **kwargs) -> str: ...
```

Notes that save time:

- `description` is prompt, not documentation. It is the only thing telling the model
  *when* to reach for the tool. Say what it is for and when not to use it.
- Anthropic-defined tools (`bash_20250124`, `text_editor_20250728`, the computer tools)
  are declared by `type` and take **no** `input_schema`. Defining your own tool named
  `bash` with your own schema is a different tool and does not get the server-side
  behaviour.
- Keep the tool list order deterministic. `tools` is the first thing in the cache prefix.
- Add `strict: true` (with `additionalProperties: false` and `required`) when you want
  the input guaranteed to validate against your schema.

## Beta headers

`agents/agent.py:110` merges a default beta header with any caller-supplied
`extra_headers` rather than letting one clobber the other:

```python
default_headers = {"anthropic-beta": "code-execution-2025-05-22"}
if "extra_headers" in params:
    custom_headers = params.pop("extra_headers")
    merged_headers = {**default_headers, **custom_headers}
```

Worth copying the shape, but note the merge is **key-wise**: a caller passing its own
`anthropic-beta` replaces the default rather than adding to it. If you need several
betas at once, join them into one comma-separated value, or use the SDK's `betas=[...]`
argument on `client.beta.messages.*`, which handles the list properly.

## MCP tools

`agents/utils/connections.py:117` bridges MCP servers into the same `Tool` interface:
connect, `list_tools()`, wrap each as a `Tool` whose `input_schema` is the server's
`inputSchema`, and let the loop treat them like any other tool. Two details:

- Connections live in an `AsyncExitStack` and are torn down when the run ends
  (`agents/agent.py:157`). MCP tools are per-run, so the code snapshots the original
  tool list and restores it in a `finally`. If you skip that, tools accumulate across
  runs and your cache prefix changes every time.
- One server failing to start should not take out the run. `connections.py` catches per
  server and continues with whatever connected.

Note this is an MCP **client** in your process. That is different from the API's MCP
connector (`mcp_servers=[...]` plus a matching `{type: "mcp_toolset"}` entry in `tools`,
beta `mcp-client-2025-11-20`), where the API talks to the server for you.

## Before you copy model IDs out of this repo

The quickstarts pin different models per project and some are well behind the current
API. `agents/agent.py:27` still defaults to `claude-sonnet-4-20250514`. Do not carry
that forward. Current default is `claude-opus-5`, and current model IDs take **no date
suffix**.

The related drift, since an old loop usually carries all of it:

- **Thinking.** `thinking={"type": "enabled", "budget_tokens": N}` is removed on current
  models and returns a 400. Use `thinking={"type": "adaptive"}` and steer depth with
  `output_config={"effort": "low"|"medium"|"high"|"xhigh"|"max"}`.
  `computer-use-demo/computer_use_demo/loop.py:141` shows both branches side by side.
- **Sampling.** `temperature` / `top_p` / `top_k` are removed on current models. This is
  the one that bites on a model swap: `agents/agent.py` sends `temperature: 1.0`
  unconditionally, which is fine against the old model it pins and a 400 the moment you
  repoint it at a current one. Drop the field, do not just change the model ID.
- **Prefill.** Prefilling the last assistant turn returns a 400. Use structured outputs
  (`output_config.format`) instead.
- **Streaming.** Default to streaming for long input/output or a high `max_tokens`;
  non-streaming requests near the 128K output ceiling hit HTTP timeouts.

## Debugging

| Symptom | Cause |
|---|---|
| 400 on the next request after a tool turn | A `tool_use` with no matching `tool_result`, or a `tool_result` whose `tool_use` was trimmed away |
| Model stops making parallel tool calls over a long session | Results were split across multiple user messages instead of one |
| `cache_read_input_tokens` always 0 | Volatile content before the last breakpoint, non-deterministic tool order, or a prefix under the model's minimum |
| Cache hit rate drops the turn after a trim | A stale `cache_control` left on a rolled-off message |
| Context looks small, then the request 400s for length | Reading `input_tokens` alone instead of summing all three counters |
| Turn dies with a stack trace from inside a tool | Tool exception escaping the executor instead of becoming an `is_error` result |
| A tool the model was using fine stops being called | Tool list order or description changed - check it is not also killing your cache |

## Provenance

Loop shape, tool contract, executor, history and MCP bridging are read from `agents/`
and `computer-use-demo/computer_use_demo/loop.py` in this repository; file:line
references above point at the exact source. The truncation trace in "The truncation
trap" is an analysis of `history_util.py`, derived by stepping the code, not a quote.
API semantics (prefix caching and breakpoint cap, the three usage counters and their
relative prices, single-user-message tool_result pairing, current model IDs, adaptive
thinking, removed `budget_tokens` / sampling / prefill) were checked against the bundled
`claude-api` reference rather than recalled. Version-dated values - beta header strings,
tool type versions, per-model behaviour - drift; re-check them against
https://docs.claude.com before relying on them in new code.
