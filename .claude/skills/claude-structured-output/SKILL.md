---
name: claude-structured-output
description: Get reliable machine-readable output out of Claude and render it - structured outputs vs tool-use-as-contract vs the legacy prompt-plus-prefill JSON trick, plus the generative-UI pattern where a tool is never executed and its input IS the render payload. Use when a response must parse, when building chat UI that emits charts or cards, when choosing between output_config.format and a tool, or when migrating JSON-mode code that now returns 400.
---

# Structured output and generative UI

Both Next.js quickstarts in this repo need machine-readable output and solve it
differently: `financial-data-analyst/app/api/finance/route.ts` uses a tool as a render
contract; `customer-support-agent/app/api/chat/route.ts` uses prompt-instructed JSON
with an assistant prefill. Reading them side by side is the fastest way to see the
tradeoff - and one of them no longer runs on current models.

## Pick the mechanism first

| | Use when | Mechanism |
|---|---|---|
| **A. Structured outputs** | The whole response is one known shape. "Return this JSON." | `output_config: {format: {...}}` on `messages.create()` |
| **B. Tool as contract** | The model must *choose*: prose or structure, or between several shapes. | A tool definition; `tool_choice: {type: "auto"}` |
| **C. Prompt + prefill + parse** | Legacy only. | Instructions in the system prompt, assistant prefill, `JSON.parse` |

Default to **A** when the shape is fixed - it constrains the response rather than asking
nicely. Reach for **B** when "answer in text" is a legitimate outcome, which is exactly
the generative-UI case below. **C** is what you migrate off; the deprecated
`output_format` parameter is also gone, superseded by `output_config.format`.

## B in practice: the tool is a render contract

The finance route never *executes* `generate_graph_data`. There is no handler, no
`tool_result`, no second API call. The tool exists so the model has a typed channel for
"emit a chart", and `tool_use.input` is handed straight to the React renderer.

```ts
const response = await client.messages.create({
  model, max_tokens: 4096,
  tools,                                // finance/route.ts:37
  tool_choice: { type: "auto" },        // finance/route.ts:220
  messages: anthropicMessages,
  system: `...`,
});

const toolUseContent = response.content.find((c) => c.type === "tool_use");
const textContent    = response.content.find((c) => c.type === "text");

return new Response(JSON.stringify({
  content:    textContent?.text || "",
  hasToolUse: response.content.some((c) => c.type === "tool_use"),
  chartData:  processToolResponse(toolUseContent),
}));
```

Four things this gets right, all worth copying:

**`tool_choice: auto`, and a response with two lanes.** "What was Q3 revenue?" should
get a sentence; "chart it" should get a chart; "chart it and explain" should get both.
Forcing the tool would make every answer a chart. So the payload carries `content` *and*
`chartData`, and the client renders whichever are present. If you force a tool, you have
decided the model never gets to just talk.

**Normalize at the boundary.** Pie charts need `{segment, value}`, but the model emits
whatever keys the data had, so the route reshapes them, coalescing across the plausible
names the model might have used (`finance/route.ts:376`):

```ts
segment: item[segmentKey] || item.segment || item.category || item.name,
value:   item[valueKey]   || item.value,
```

Do this once, server-side. The alternative is every component defending against every
shape the model might produce.

**Never let the model choose presentation.** The tool schema has no color field. Colors
are assigned after the fact, positionally, from design tokens (`finance/route.ts:392`):

```ts
color: `hsl(var(--chart-${index + 1}))`
```

The model picks *semantics* - which series exist, what they are called, which chart type
fits. The design system picks *pixels*. Let the model emit hex codes and your charts stop
matching your app, break in dark mode, and fail contrast. This generalizes past charts:
no model-chosen colors, fonts, spacing, or icon names.

**Validate before rendering** (`finance/route.ts:358`):

```ts
if (!chartData.chartType || !chartData.data || !Array.isArray(chartData.data)) {
  throw new Error("Invalid chart data structure");
}
```

A tool schema is a strong hint, not a guarantee, unless you set `strict: true` (which
also requires `additionalProperties: false` and `required` on the schema). This route
deliberately uses `additionalProperties: true` on data rows so arbitrary series can come
through - a reasonable trade that makes the runtime check mandatory rather than optional.

Design notes for the schema itself: enumerate `chartType` rather than leaving it a free
string, so the model cannot invent a renderer you do not have; and keep the system prompt
telling the model *when* each variant is appropriate. The system prompt here is mostly a
catalogue of which chart suits which question, which is the part that decides output
quality once the plumbing works.

## C in practice, and why it is now broken

The support route asks for JSON in the system prompt, then prefills an opening brace to
stop the model prefacing it with prose (`chat/route.ts:221`):

```ts
anthropicMessages.push({ role: "assistant", content: "{" });
const response = await client.messages.create({ ... temperature: 0.3 });
const textContent = "{" + response.content.filter(...).map(b => b.text).join(" ");
const validatedResponse = responseSchema.parse(sanitizeAndParseJSON(textContent));
```

It was a good technique. It does not run on current models - **two separate 400s**:

- **Assistant prefill is removed** on Fable 5/5.1, Opus 5, Sonnet 5 and the 4.6/4.7/4.8
  family. A trailing assistant turn returns 400. The whole `"{"` + re-concatenate trick
  depends on it.
- **`temperature` is removed** on the same models (`chat/route.ts:231`,
  `finance/route.ts:218` at 0.7).

Migrating this shape: drop the prefill and the temperature, move the Zod schema's shape
into `output_config.format`, and keep the Zod `.parse()` as a runtime assertion at the
boundary. The keep-worthy parts of the original are the schema-validate-then-use
discipline and the typed error fallback that returns a well-formed object on failure, so
the client never receives something it cannot render.

Note the surrounding design is still good: retrieval failure is non-fatal (the prompt
says "No information found for this query" and the model carries on), and diagnostics
travel in headers rather than polluting the typed body.

## One bug worth not copying

`finance/route.ts:433` catches `Anthropic.APIError`, then `:444` catches
`Anthropic.AuthenticationError`.

[Inference] In the Anthropic SDKs the specific status errors derive from the base API
error, so the first branch swallows the second and the auth branch is unreachable - an
auth failure reports as a generic API error. I did not verify the TypeScript class
hierarchy directly (`node_modules` is not installed here), so confirm against the SDK
before treating this as settled. Either way the rule holds: **order catch clauses
most-specific first**, and distinguish retryable (429, 5xx, connection) from
non-retryable (400, 401, 404) rather than collapsing them into one branch.

## Debugging

| Symptom | Cause |
|---|---|
| 400 on a request that used to work | Assistant prefill or `temperature` on a current model |
| `JSON.parse` fails on the model's text | Prose wrapped around the JSON - the problem structured outputs exists to solve |
| Every answer comes back as a chart/card | `tool_choice` forced when it should be `auto` |
| Model narrates "I'll use the tool now" | Say so in the system prompt - this route does, at `finance/route.ts:327` |
| Renderer crashes on a field that is usually there | Schema hint treated as a guarantee; add `strict: true` or validate at the boundary |
| Chart colors clash or vanish in dark mode | Model-chosen colors instead of design tokens |
| Auth failures surface as generic API errors | Catch order: base class before subclass |

## Provenance

Code shapes, schema design, normalization, color injection, validation and every
file:line citation are read from `financial-data-analyst/app/api/finance/route.ts` and
`customer-support-agent/app/api/chat/route.ts` in this repository. The API facts -
prefill removed and `temperature`/`top_p`/`top_k` removed on current models (400),
`output_config.format` superseding the deprecated `output_format`, and `strict: true`
requiring `additionalProperties: false` plus `required` - were checked against the
bundled `claude-api` reference rather than recalled. The SDK error-hierarchy claim is
explicitly marked as inference and unverified here. I did not run either route (no
`node_modules`, no API key), so the 400s are predicted from the documented model
behaviour, not observed.
