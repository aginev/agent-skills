---
name: critically-evaluate
description: >-
  Get an adversarial second opinion on a design doc, ADR, RFC, spec, or plan by
  running OpenAI's Codex CLI against it and then having Claude critically review
  Codex's feedback point by point. Use this whenever the user runs
  `/critically-evaluate`, or asks to "critically evaluate", "poke holes in",
  "stress-test", "pull apart", "get a second opinion on", "red-team", or
  "sanity-check" a document, decision record, architecture doc, or plan — even
  if they don't name Codex explicitly. The deliverable is a list of the
  assumptions/findings Codex raised, each with Claude's own verdict and
  reasoning. Trigger this rather than reviewing the document yourself, because
  the whole point is to combine an independent external model's critique with
  Claude's grounded review.
---

# Critically evaluate

This skill orchestrates a two-model critique of a single document (typically an
ADR, RFC, design doc, spec, or implementation plan):

1. **Codex** (OpenAI's CLI, running in the current project) reads the document
   *and the surrounding codebase* and produces an adversarial critique.
2. **Claude** then critically reviews Codex's critique — agreeing, disagreeing,
   or qualifying each point with its own reasoning.

The reason for two models is that each catches what the other misses. Codex
brings a genuinely independent perspective and can ground claims in the actual
code; Claude then filters that critique so the user isn't handed a wall of
objections to sort through alone. The final artifact is a point-by-point list:
every assumption/finding Codex raised, paired with Claude's verdict.

The flow is strictly sequential and each stage feeds the next:

```
resolve file → Codex critiques it → Claude reviews Codex's critique → list of assumptions + Claude's verdicts → offer to apply edits
```

## Step 1 — Resolve the target file

The user invokes this as `/critically-evaluate @some/path/to/doc.md` or bare
`/critically-evaluate`.

- If a path was provided (with or without a leading `@`), strip the `@` and use
  it. Confirm the file exists relative to the project root before continuing.
- If no path was provided, **ask the user which file to evaluate** and wait for
  an answer. Don't guess or pick a file yourself — evaluating the wrong document
  wastes an expensive Codex run.

Keep the path as given (usually relative to the project root); you'll substitute
it into the Codex prompt as `{{ FILE }}`.

## Step 2 — Run Codex against the file

Codex runs non-interactively via `codex exec`. A few things matter here:

- **Read-only is the default and is what we want.** `codex exec` runs in a
  read-only sandbox unless told otherwise, so it can inspect the file and the
  codebase to verify claims without changing anything. Do **not** pass
  `--sandbox workspace-write` — this stage only reads.
- **Model and reasoning effort** are set with flags. The model defaults to
  `gpt-5.5` but is configurable (see below). Reasoning effort is `xhigh`.
- **Run from the project root** so Codex sees the whole codebase, and **capture
  the final message to a file** with `-o` so the feedback is easy to hand to the
  next stage. `codex exec` streams progress to stderr and prints only the final
  message to stdout, but `-o` gives you a clean file regardless.
- Codex requires a git repo. If the project isn't one, add `--skip-git-repo-check`.

Because the Codex prompt is long and multi-line, write it to a temp file and
pipe it in via `codex exec -` (the `-` makes Codex read the prompt from stdin).
This avoids shell-quoting problems.

**Model override:** default to `gpt-5.5`. If the user's environment expects a
different name (for example a Codex-tuned variant) or the user asks for another
model, use that instead. You can let the user set it once via a `CODEX_MODEL`
environment variable and fall back to `gpt-5.5`.

Concretely:

```bash
MODEL="${CODEX_MODEL:-gpt-5.5}"
PROMPT_FILE="$(mktemp)"
FEEDBACK_FILE="$(mktemp)"

# Write the substituted Codex prompt (see template below) to "$PROMPT_FILE",
# with {{ FILE }} replaced by the resolved path.

codex exec \
  -m "$MODEL" \
  -c model_reasoning_effort="xhigh" \
  -o "$FEEDBACK_FILE" \
  - < "$PROMPT_FILE"

# Codex's critique is now in "$FEEDBACK_FILE".
```

If `codex` isn't found, errors, or exits non-zero, stop and tell the user
plainly (e.g. "Codex CLI isn't installed / not authenticated / the model name
was rejected") rather than fabricating feedback. The value of this skill is a
real external critique; inventing one would defeat the purpose.

### Codex prompt template

Substitute `{{ FILE }}` with the resolved path, verbatim otherwise:

```
Critically evaluate `{{ FILE }}`:
- Look for inconsistencies, weak assumptions, overlooked details, or underestimated complexity.
- Identify gaps or areas that could be refined, simplified, or clarified.
- Ensure all claims and statements are supported by the current codebase.
- If a third-party library or service is referenced, verify that it supports the claimed functionality and determine whether there is a more current or recommended approach.
- Recommend alternative approaches where appropriate, and explain where you disagree with a decision or statement.
- Backward compatibility is not a concern. API changes are allowed without migration paths.
- Stay focused on the scope of this document. Avoid scope creep and unrelated improvements.
- Avoid over-engineering and premature optimization.
- Challenge decisions that appear insufficiently justified.
- Be skeptical and clearly explain your objections.
- Qualify each finding using one of the following severity levels:
  - **Critical**
  - **High**
  - **Medium**
  - **Low**
```

## Step 3 — Have Claude critically review Codex's feedback

Read `$FEEDBACK_FILE`. Then critically review it using the prompt below, with
`{{ FEEDBACK }}` replaced by the full text Codex produced. This is Claude's own
turn to think — don't rubber-stamp Codex. Ground your verdicts in the actual
document and codebase: where Codex is right, say so; where it's overreaching,
mistaken, or has misread the code, push back and explain why.

### Claude review prompt

```
Critically review the following third-party feedback:

____

{{ FEEDBACK }}
____

- Determine which points you agree with and update the document accordingly. Answer any open questions raised in the feedback.
- Avoid over-engineering and premature optimization. Keep the plan focused.
- Backward compatibility is not a concern. API changes are allowed, and deprecation flags are unnecessary.
- If multiple valid approaches exist, or if a decision depends on user preference or missing information, use `AskUserQuestion` to gather clarification. Always provide your recommended approach, including its pros and cons.
- Do not hesitate to push back where appropriate. You do not have to accept every suggestion. Clearly explain when and why you disagree.
- For each feedback item, provide the rationale behind your decision. Do not use tables.
```

When the feedback raises a genuine fork — multiple valid approaches, or a call
that hinges on the user's preference or on information not in the document — use
`AskUserQuestion` to gather that input before finalizing your verdict, and
always include your own recommendation.

## Step 4 — Present the assumptions + review list

This is the deliverable the user is after. Present it **in chat** (no file).

Treat each discrete point Codex raised — every objection, flagged weak
assumption, or challenged decision — as one entry. Preserve Codex's severity
label on each. For every entry, give Claude's verdict and the reasoning behind
it. Do not use tables (they render poorly and flatten the reasoning).

**Color coding.** The user wants severity and the two voices colour-coded.
Markdown has no native text colour, and raw HTML `<span style>` colour does not
render in most chat/terminal Markdown viewers — so use coloured emoji markers,
which render reliably everywhere. Use a red→green severity gradient and distinct
coloured squares for the two voices (squares vs circles keeps the source labels
from visually colliding with the severity markers):

- Severity: 🔴 Critical · 🟠 High · 🟡 Medium · 🟢 Low
- Voices: 🟥 **Codex:** (the external critique) · 🟩 **Claude:** (your review)

Put a one-line legend under the summary block so the mapping is explicit. Prefix
every finding heading and every summary tally line with its severity emoji.

Use this structure. Lead with the severity breakdown as its own block — the
user wants to see the shape of the results at a glance before reading any prose,
so put the count and the per-severity tally up top, then your overall take on a
separate line:

```
## Critical evaluation of `<file>`

**N findings in total**
- 🔴 Critical — X
- 🟠 High — Y
- 🟡 Medium — Z
- 🟢 Low — W

_Severity:_ 🔴 Critical · 🟠 High · 🟡 Medium · 🟢 Low

<one or two sentences with your overall take>

### 1. 🟠 [High] <short title of the assumption/finding>
🟥 **Codex:** <concise restatement of the point Codex made>
🟩 **Claude:** <Agree / Partially agree / Disagree> — <your reasoning, grounded in the doc and code; note where Codex misread something or where the point stands>

### 2. 🔴 [Critical] ...
```

Order entries by severity (Critical first). Keep each restatement faithful to
Codex — don't soften or inflate it — and keep your verdict honest. Omit any
severity level with a count of zero (and drop it from the legend too).

If, while reviewing, you spot a material problem Codex missed and can ground it
in the document or code, you may add it after the Codex-derived entries under a
clearly labelled heading (e.g.
`### 🟠 [<Severity>] Not raised by Codex — <title>`, using the same severity
emoji and the 🟩 **Claude:** voice) so it's obvious this is Claude's own finding
rather than part of Codex's critique. Keep these to genuine, grounded issues;
the main job is still to review what Codex raised.

## Step 5 — Offer to apply edits

After presenting the list, offer to apply the changes you agreed with — but let
the user pick. **Present the proposed edits as a numbered list, one edit per
line**, so they can choose by number. A run-on paragraph is hard to scan and
gives the user no way to accept some edits but not others. Each item should name
the specific change and which finding it addresses, and **carry the same
severity emoji as the finding it fixes** (🔴/🟠/🟡/🟢) so the user can see at a
glance which edits are the urgent ones. Then tell them they can reply with
specific numbers or apply everything. For example:

```
**Offer — edits I can apply to `<file>`:**
1. 🔴 Correct the "<false claim>" statement (finding 1).
2. 🟠 Add a <mitigation> section (finding 2).
3. 🟡 Replace/justify <choice> (finding 3).

Reply with the numbers you want (e.g. `1, 3`), or say `all` to apply
everything. I'll only touch what you pick and will wait for your go-ahead.
```

If a single edit addresses more than one finding, use the emoji of the
highest-severity finding involved.

Only list edits tied to points you actually endorsed. Wait for the user's
selection before editing anything, then apply exactly the items they chose
(`all` means every listed item; a subset means only those numbers) and briefly
summarize what changed. If the user declines, stop — the review list stands on
its own.
