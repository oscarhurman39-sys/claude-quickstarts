---
name: claude-computer-use
description: Build or debug a computer-use or browser-use agent with Claude - screenshot sizing and the two coordinate spaces, cache-safe screenshot pruning, hosted vs explicit tool declarations and which one gets the safety classifiers, tool-version/beta-flag pairing, provider request-size caps, batching, and sandboxing. Use when clicks land in the wrong place, screenshot history is dominating cost or context, or when wiring up computer/browser/bash tools.
---

# Claude computer use

Drawn from `computer-use-best-practices/` (macOS reference implementation),
`computer-use-demo/` (Docker + X11) and `browser-use-demo/` (Playwright) in this
repository. The first of those pairs with Anthropic's
[computer-use best-practices guide](https://claude.com/blog/best-practices-for-computer-and-browser-use-with-claude).

## Safety first, because this one is not boilerplate

A computer-use agent has full control of a mouse, keyboard and screen, and it reads
untrusted pixels. `computer-use-demo/README.md` states the risk plainly: Claude will
sometimes follow commands found in content even when they conflict with the user's
instructions, so instructions embedded in a webpage or an image can override yours.
Every screenshot of a web page is a prompt-injection surface.

The precautions both caution blocks list (`computer-use-demo/README.md`,
`computer-use-best-practices/README.md`):

- **Run it in a disposable VM or a minimally-privileged container**, not on a machine
  with real data. The `computer-use-demo` exists to make the container route easy;
  `computer-use-best-practices` runs natively and therefore leads with a caution to use
  a macOS VM.
- **Keep sensitive data away from it** - credentials in particular, since a screenshot
  of them goes to the API and there is no safeguard against that.
- **Allowlist network access** rather than leaving egress open, to bound exposure to
  hostile content.
- **Put a human in front of consequential actions** - purchases, accepting terms,
  anything with real-world effect.

Then the implementation-level guards:

- Sandbox anything shell-shaped. `computer-use-best-practices/sandbox/default.sb` is a
  `sandbox-exec` profile with no network and write access only to a scratch directory.
- Cap tool output. `max_shell_output_bytes = 64 * 1024` (`constants.py:115`) exists
  because `yes` fills RAM in seconds, well before a 30s timeout fires, and because
  uncapped output goes straight into context.

If you want built-in safeguards and human-in-the-loop controls rather than a teaching
scaffold, `computer-use-best-practices/README.md:19` points at Cowork instead of this
architecture.

## Coordinates: the thing that actually breaks

The model emits click coordinates in the pixel space of **the image it was shown**. If
that is not the same space as your screen or viewport, every click is wrong, and it is
wrong in a way that looks like the model being bad at seeing rather than a units bug.

The API resizes any image whose long edge exceeds `max_edge_px` **or** whose tile count
exceeds the token budget (`constants.py:100-105`: `px_per_token = 28`,
`max_edge_px = 1568`, `max_tokens = 1568`). So there are three ways to be correct and
one common way to be wrong:

**Option A - send an image that is already within budget.** The server's early-return
fires, no resize happens, and the model's coordinates are already in your space. Nothing
to scale. `browser_viewport = (1456, 819)` (`constants.py:108`) is chosen for exactly
this reason: a 16:9 viewport that arrives pre-sized, so the browser tool never scales
anything.

**Option B - resize yourself, then scale back.** Resize with the same rule the API uses
so it does not resize again, take the model's `(x, y)` in the resized space, and map it
home:

```
screen_x = model_x * original_width / sent_width
screen_y = model_y * original_height / sent_height
```

**Option C - let the server resize and scale from the documented target size.** This is
what `browser-use-demo/browser_use_demo/tools/coordinate_scaling.py:22` does. Resized
dimensions are documented per aspect ratio:

| Aspect ratio | Resized to | Aspect ratio | Resized to |
|---|---|---|---|
| 1:1 | 1092x1092 | 3:2 | 1344x896 |
| 3:4 | 951x1268 | 9:16 | 819x1456 |
| 4:3 | 1268x951 | 16:9 | **1456x819** |
| 2:3 | 896x1344 | 1:2 | 784x1568 |
| 2:1 | 1568x784 | | |

`scale_x = viewport_width / 1456`, `scale_y = viewport_height / 819` for a 16:9
viewport. Note this is the most fragile option - it hardcodes server behaviour that can
change. Prefer A.

**The wrong way** is to resize for the token budget and then treat the returned
coordinates as screen pixels. Symptom: clicks land consistently up and to the left (or
down and right), with error proportional to distance from the origin. A constant offset
instead points at something else - a title bar, a Retina/HiDPI scale factor, or a
viewport that is not the screen.

Two more coordinate rules, from the system prompt at `constants.py:301`:

- "Coordinates you emit must refer to the most recent screenshot you were shown." Put
  that in your system prompt. Without it the model will happily reuse coordinates from
  three screenshots ago.
- Inside a batch, **every** coordinate refers to the screenshot taken *before* the
  batch. A batch that scrolls and then clicks using a post-scroll coordinate misses.

## Screenshot cost and cache-safe pruning

A screenshot is roughly 1,500 input tokens, and a trajectory produces close to one per
turn. Thirty turns is ~45k tokens of image history that you resend and pay for on
*every* subsequent call.

The instinct is to keep only the last N images. **That instinct is wrong twice over.**

**First, it fights the cache.** Caching is a prefix match. "Keep the last N" changes
*which* image gets replaced on every single turn once you exceed N, so the prefix
changes every turn and the cache misses every turn. You trade ~0.1x cache reads for 1x
uncached input on the entire conversation.

Two fixes appear in this repo:

- **Interval pruning** (`computer-use-best-practices`, default). Let the kept-image
  count *step* from `image_prune_min` up to
  `image_prune_min + image_prune_interval - 1`, then drop back. For
  `image_prune_interval` consecutive turns the same old images map to the same
  placeholder, the prefix is byte-identical, and the cache keeps hitting. You pay one
  cache write per interval instead of one per turn. Defaults: `image_prune_min = 3`,
  `image_prune_interval = 40` (`constants.py:134`).
- **Chunked removal** (`computer-use-demo`). Same idea, cruder:
  `images_to_remove -= images_to_remove % min_removal_threshold`
  (`loop.py:250`) rounds removals down to a chunk so the prefix only changes once per
  chunk instead of once per turn.

**Second, it often is not worth doing at all.** Cache reads are ~0.1x input, so
carrying an already-cached screenshot one more turn costs almost nothing, while dropping
it forces a re-cache of everything after it. The saving only pays back the invalidation
after that image would have been read on the order of ten more times.
`computer-use-demo/computer_use_demo/loop.py:128` takes the strong version of this
position and disables image truncation outright whenever caching is on:

```python
betas.append(PROMPT_CACHING_BETA_FLAG)
_inject_prompt_caching(messages)
# Because cached reads are 10% of the price, we don't think it's
# ever sensible to break the cache by truncating images
only_n_most_recent_images = 0
```

### Choosing: prune, compact, or neither

`enable_autocompaction` turns on server-side summarization - once input tokens cross
`autocompaction_trigger_tokens` (default 150,000; the API floor is 50,000 and lower
values are clamped, `constants.py:50`), the server condenses the conversation into a
summary block and the client drops everything before it.

| | Image pruning | Server-side compaction |
|---|---|---|
| Cost of the event | Re-process the changed suffix; no output tokens | A full extra API turn, whole conversation in, paragraphs out |
| Cache after the event | Prefix intact until the next rollover | Cold - the prefix is brand new |
| Latency | Fast | Slow; those summary output tokens are generated serially |
| What is lost | Old screenshots, permanently | Nothing textual - it is summarized |
| What it cannot bound | User/assistant text, which accumulates forever | - |

The decision rule the reference implementation lands on: **pruning is only worth it if
it actually prevents a compaction.**

- Short, screenshot-light tasks that never approach the trigger: prune nothing
  (`image_prune_strategy = "none"`) and let compaction sit there as a rare safety net.
- Long, screenshot-heavy tasks where images alone would cross the trigger: prune, and
  keep `image_prune_min` high enough that the model still has visual context after a
  rollover.

The shipped defaults are tuned so screenshot-heavy runs prune before they ever reach the
compaction trigger, and screenshot-light runs do neither.

## Hosted vs explicit tool declarations

You can declare the computer tool two ways, and the choice is a **safety** choice, not a
style one:

```python
# Hosted: the server supplies description and schema.
{"type": "computer_20250124", "name": "computer", "display_width_px": w, ...}

# Explicit: you supply name, description and JSON schema yourself.
{"name": "computer", "description": "...", "input_schema": {...}}
```

Execution is client-side either way - only the *declaration* changes. But the API runs
computer-use-specific safety classifiers, **including prompt-injection detection on
screenshot content, only when the request declares the hosted tool type.** An explicit
schema puts the request on the generic safety path, and prompt-injection mitigation
becomes entirely yours: system prompt instructions, VM isolation, your own monitoring.

Reference implementations use explicit schemas so you can read exactly what the model
sees. That is a pedagogical tradeoff. For anything touching real data, prefer the hosted
declaration - and note you can mix, declaring `computer` as hosted while keeping your
batch/browser/bash tools explicit (`cfg.use_hosted_computer_tool`).

## Tool versions travel with beta flags

The computer tool is versioned, and each version travels with a specific beta flag. The
repo keeps the two in a single `ToolGroup` table so they cannot drift apart in the call
site - treat the pair as one unit rather than picking a version and a flag separately.
From `computer-use-demo/computer_use_demo/tools/groups.py`:

| Tool version | Beta flag |
|---|---|
| `computer_use_20241022` | `computer-use-2024-10-22` |
| `computer_use_20250124` | `computer-use-2025-01-24` |
| `computer_use_20250429` | `computer-use-2025-01-24` |
| `computer_use_20251124` | `computer-use-2025-11-24` |

Note `20250429` reuses the January flag - the mapping is not mechanical, so look it up
rather than deriving it from the date. The group also pins which tool *classes* go
together (`ComputerTool20251124` ships with `BashTool20250124` and `EditTool20250728`);
keeping that table in one place is why adding a new version is a single entry rather
than a scatter of conditionals.

This table reflects what this repo pins today. Check the current docs for newer
revisions before starting something new.

## Provider caps and first-party-only features

Vertex and Bedrock impose a request-body size cap that the first-party API does not
(`constants.py:55`):

```python
PROVIDER_MAX_MESSAGE_MB = {"anthropic": None, "vertex": 18, "bedrock": 11}
```

Those are deliberately conservative - the comment records Vertex tripping empirically
around 21 MB and Bedrock around 13 MB, with headroom left for text content. A
screenshot-heavy trajectory hits this surprisingly fast. The implementation enforces it
inside the image pruner: if the serialized message list exceeds the cap, force-prune to
`image_prune_min` and reset the interval cycle so following turns start small.

Server-side compaction and the advisor tool are **first-party only**. The config raises
a `ValueError` at startup naming the conflicting setting rather than failing deep inside
the SDK on the first call - worth copying that shape.

## Batching

`computer_batch` / `browser_batch` chain several predictable actions into one turn.
Round-tripping a screenshot per click is the dominant latency and cost in a trajectory,
so batching is the main lever when the model can confidently plan ahead.

The steering is a nudge, not a rule: `BATCH_REMINDER` (`constants.py:380`) is appended
after a lone single-action call, and the actions that trigger it are enumerated in
`BATCH_REMINDER_ACTIONS`. Two prompt rules go with it (`constants.py:301`): end a batch
with a screenshot whenever the batch likely changed something you need to verify, and
remember every coordinate in the batch refers to the pre-batch screenshot.

## Image encoding

- `jpeg_quality = 75` (`constants.py:94`) - matches what the desktop app's encoder uses
  and is a good size/fidelity trade. Raising it costs request bytes, not tokens; token
  cost is a function of dimensions.
- `min_screenshot_bytes = 1024` (`constants.py:98`) - reject screenshots that encode
  below this. It catches degenerate all-black captures, which on macOS usually mean
  missing Screen Recording permission. A permissions preflight is the primary guard;
  this is the backstop.

## Debugging

| Symptom | Cause |
|---|---|
| Clicks off by an amount that grows with distance from origin | Coordinate space mismatch - the model's pixels are not your pixels. Check whether the server resized |
| Clicks off by a constant offset | Title bar, HiDPI/Retina scale factor, or viewport vs screen origin |
| Clicks were fine, then drifted after a UI change | Screenshot dimensions changed and crossed the resize threshold |
| Cache hit rate collapses after ~N turns | "Keep last N images" pruning invalidating the prefix every turn |
| Cost climbing steadily on a long run | Screenshot history being resent uncached - check `cache_read_input_tokens` |
| Works on the first-party API, 400s on Vertex/Bedrock | Request body over the provider cap - force-prune images |
| Screenshots arrive black | Screen Recording permission (macOS), or no display attached |
| `ValueError` naming a setting at startup on Vertex/Bedrock | Compaction or advisor enabled on a non-first-party provider |
| Batched clicks after the first one miss | Coordinates computed against a post-action state instead of the pre-batch screenshot |

## Provenance

Configuration values, pruning strategies, provider caps, tool-version/beta pairings,
prompt text and the safety-classifier tradeoff are read from
`computer-use-best-practices/constants.py` and `README.md`,
`computer-use-demo/computer_use_demo/{loop.py,tools/groups.py}`, and
`browser-use-demo/browser_use_demo/tools/coordinate_scaling.py` in this repository;
citations above point at the exact source. The documented resize table originates in
Anthropic's vision docs as cited in `coordinate_scaling.py:22`. The per-screenshot token
figure (~1,500) and the empirical Vertex/Bedrock trip points are this repository's
measurements, reported as such rather than independently confirmed here. The Debugging
table and the coordinate failure signatures are reasoning from the mechanisms described
above, not observations from a recorded run - use them as hypotheses to check, not
diagnoses. Prefix-match
cache semantics and cache-read pricing (~0.1x) were checked against the bundled
`claude-api` reference. Dated values - tool type versions, beta flag strings, the resize
table - drift; re-check against https://docs.claude.com before relying on them.
