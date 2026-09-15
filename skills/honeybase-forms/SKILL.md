---
name: honeybase-forms
description: Author SurveyJS form definitions for Honeybase tasks — the 15 allowed question types and why the rest are refused, the nine blocked keys, question naming (a question `name` becomes `task.form.data.<name>` downstream), the attach/snapshot lifecycle, conditional-logic idioms, and worked examples that are CI-verified to validate and render. Honeybase forms are a deliberately subsetted SurveyJS and nothing about that subset is in your training data — ALWAYS load this before writing or editing a Honeybase form schema (`validate_form_schema`, `create_form`, `update_form`, or a Create/Update Task node's `formAttachment`).
version: 2026-09-13
---

# Honeybase forms

A Honeybase form is a **SurveyJS survey definition (JSON)** stored in an organization's form library and attached to human tasks. The SPA renders it with `survey-core` **3.0.4**.

Two things make writing one different from writing plain SurveyJS, and both are why this skill exists:

1. **The server rejects a subset of SurveyJS** — the form renders un-sandboxed inside the SPA's own JS context, next to the user's session, so anything that fetches, redirects or injects markup is refused at save.
2. **The server's validator only catches *unsafe*, never *wrong*.** An invented property name, a `visibleIf` with the wrong syntax, `choices` nested in the wrong place, `required` instead of `isRequired` — all of these save cleanly and produce a broken form nobody notices until an assignee opens the task. Copy the worked examples below rather than writing from memory, and check property names against `references/surveyjs-subset.json`.

## Lifecycle: library form → attached → snapshot → submission

```
Forms library (/forms)        one row per reusable form, has a version
  │  attach
  ▼
Create Task node   formAttachment: {formId, required, autoComplete, presentation}
Update Task node   formAction: "Set" | "Remove" | "NoChange"  +  formAttachment
  │  at run time the runner SNAPSHOTS the schema + version onto the task
  ▼
Task              the assignee fills it at the task; the snapshot never changes
  │  submit
  ▼
Task Update trigger / JS nodes read task.form.data.<question name>
```

- **Snapshot-on-attach is the important part.** The runner freezes the schema and its version onto the task when the task is created/updated. Editing the library form afterwards bumps the library version and changes nothing on tasks already in flight. To change what an in-flight task shows, you must re-attach.
- `formAttachment.required` (default `true`) — the task cannot be completed until the form is submitted.
- `formAttachment.autoComplete` (default `true`) — submitting the form completes the task.
- `formAttachment.presentation` (default `"Full"`) — how the assignee sees the form. See [Presentations](#presentations) below.
- `formAttachment.formId` is **the form's public id (UUID) over MCP** and **the numeric id in the workflow editor / Barry**. The MCP boundary translates it in both directions; passing a number over MCP is rejected with `'<NodeName>'.data.formAttachment.formId must be the form's public id (UUID); see list_forms.`

### Presentations

`formAttachment.presentation` is one of four strings:

| Value | What the assignee sees | Needs `formId`? |
|---|---|---|
| `Full` (default) | The form in the task's full view (its own tab); the side panel shows an indicator with an "Open full view" button | yes |
| `Inline` | The form rendered compactly right under the task's badges, in the side panel too. **At most 3 questions** — the save is rejected naming the form when it has more (panels are free, hidden questions count) | yes |
| `Approval` | An **Approve / Reject** button pair under the badges. A built-in template: no library form | no |
| `ApprovalWithNotes` | Approve / Reject plus an optional note | no |

- The two approval templates are real frozen forms: `task.form.data.decision` is `"approved"` or `"rejected"` (and `task.form.data.notes` the note, or `null`). Branch on it from a **Task Update** trigger — a Form Submission trigger never fires for a template (there is no library form to name).
- Templates force `required` and `autoComplete` to `true` and ignore `formDataCode`: the decision IS the task's resolution.
- `Full`/`Inline` with no `formId` is rejected at save time (`Node 'X' shows a form as Full but no form is picked — pick a library form, or choose the Approval presentation, which needs none`). A `formId` on a template is inert.
- `task.form.formId` is `null` for a template; `task.form.presentation` carries the value (always present, `"Full"` for forms attached before the field existed).

### `formAttachment.required` is NOT SurveyJS `isRequired`

Two different things with confusable names:

| | Where | Meaning |
|---|---|---|
| `isRequired: true` | on a **question** in the schema | SurveyJS will not let the user submit the form with that answer empty |
| `required: true` | on the node's **`formAttachment`** | Honeybase will not let the **task** be completed until the form is submitted |

There is **no `required` property on a SurveyJS question**. Writing `"required": true` on a question is silently ignored by survey-core (it is not in the serializer) and validates on the server — a classic invisible bug. Always `isRequired`.

## Question names are the downstream contract

A question's `name` is the key everything downstream addresses:

```js
// in a JS node, or a Task Update trigger's filter expression
task.form.data.invoice_number
```

So **name questions like database fields, never `question1`.** Rules:

- `snake_case`, starting with a letter: `invoice_number`, `passed`, `stop_address`.
- **No dots** — SurveyJS treats `a.b` as a nested path and it breaks `task.form.data.<key>` addressing.
- No spaces, no `{`/`}` (expression syntax), no leading digits.
- Unique across the whole form, including inside `paneldynamic` templates and `matrixdynamic` columns.
- **Renaming a question is a breaking change**: the old key vanishes from new submissions and every downstream reference must be updated. Tasks already holding a snapshot keep the old key forever.

### The `task.form` object

`task.form` is `null` when no form is attached. Otherwise, exactly:

```json
{
  "formId": 12,
  "name": "Supplier invoice intake",
  "version": 3,
  "required": true,
  "autoComplete": true,
  "submitted": false,
  "submittedAt": null,
  "submittedBy": null,
  "data": {
    "supplier_name": null,
    "invoice_number": null,
    "invoice_total": null,
    "cost_centre": null,
    "notes": null
  },
  "presentation": "Full"
}
```

- `data` **always carries every question key of the snapshot**, `null` until answered — so `task.form.data.x` is a stable path before any submission exists. Build downstream nodes against it immediately.
- Branch on **`submitted`**, not on whether `data` is empty.
- After submission the answers overlay the null placeholders; unanswered optional questions stay `null`.
- `task.form.formId` here is the **numeric** id (unlike `formAttachment.formId` over MCP).
- `task.form.presentation` is always a string; `task.form.formId` is `null` for the `Approval` / `ApprovalWithNotes` templates.
- `data` is scrubbed and capped at 64 KB. Do not design a form whose answers approach that.

## The allowlist: 15 question types

`text` · `comment` · `radiogroup` · `checkbox` · `dropdown` · `tagbox` · `boolean` · `rating` · `ranking` · `matrix` · `matrixdropdown` · `matrixdynamic` · `panel` · `paneldynamic` · `multipletext`

| Type | Use it for | Answer shape in `data` |
|---|---|---|
| `text` | one-line input; `inputType` ∈ `text number date time datetime-local email tel url password month week range color` | string (or number) |
| `comment` | multi-line free text | string |
| `radiogroup` | pick one, radio buttons | the chosen `value` |
| `dropdown` | pick one, long list | the chosen `value` |
| `checkbox` | pick many, checkboxes | array of `value`s |
| `tagbox` | pick many, compact multi-select | array of `value`s |
| `boolean` | yes/no | `true`/`false` |
| `rating` | numeric scale, stars, smileys | number |
| `ranking` | drag choices into order | ordered array of `value`s |
| `matrix` | grid, one radio choice per row | `{rowValue: columnValue}` |
| `matrixdropdown` | grid of editable cells, **fixed** rows | `{rowValue: {colName: v}}` |
| `matrixdynamic` | table the user adds rows to | array of `{colName: v}` |
| `multipletext` | several short inputs under one title | `{itemName: v}` |
| `panel` | **layout only** — groups questions, is not an answer | — (no key) |
| `paneldynamic` | repeating group of questions | array of `{questionName: v}` |

Plus **`file`**, which is allowed **only when the deployment has object storage configured**. The skill is hosted once for every deployment and cannot know whether yours does — run `validate_form_schema` and it will tell you (`Question type 'file' is not allowed` means it is not configured). When it is allowed, two extra rules apply: `storeDataAsText: true` is refused (it would base64-inline bytes into the submission), and `maxSize` may not exceed the deployment's per-file cap (default 25 MB).

### What is refused, and why

Everything else — notably `html`, `image`, `imagepicker`, `imagemap`, `signaturepad`, `slider`, `buttongroup`, `expression` — is rejected with `Question type '<t>' is not allowed`. The reason is one sentence: **the form renders un-sandboxed in the SPA's own JS context**, so raw markup, remote images and unvetted custom widgets are all injection surface. There is no flag, no escape hatch and no per-org override; do not try to route around it.

Two consequences worth planning for:

- **No `expression` question.** There is no computed/display-only field. Compute in a JS node downstream, or use survey-level `calculatedValues` (allowed — they carry `name`/`expression`/`includeIntoResult` and no `type`), remembering that a calculated value's key is **not** in `questionKeys`, so it is *absent* from `task.form.data` before submission rather than `null`.
- **No images or rich text.** Put context in `title` / `description` on the survey, page, panel or question. They are plain text.

### The nine blocked keys

Rejected wherever they appear in the tree — including nested inside `validators` and `triggers`, where the type allowlist does not apply but this check still does.

| Key | Rejection message |
|---|---|
| `choicesByUrl` | Loading choices from an external URL (choicesByUrl) is not allowed |
| `surveyId` | Loading the form definition from SurveyJS's hosted service (surveyId) is not allowed; put the definition in the schema |
| `surveyPostId` | Posting results to SurveyJS's hosted service (surveyPostId) is not allowed; submissions are stored on the task |
| `navigateToUrl` | Redirecting on completion (navigateToUrl) is not allowed; use the default thank-you page |
| `navigateToUrlOnCondition` | Conditional completion redirects (navigateToUrlOnCondition) are not allowed |
| `completedHtml` | Custom completion HTML (completedHtml) is not allowed |
| `completedHtmlOnCondition` | Conditional completion HTML (completedHtmlOnCondition) is not allowed |
| `completedBeforeHtml` | Custom already-completed HTML (completedBeforeHtml) is not allowed |
| `loadingHtml` | Custom loading HTML (loadingHtml) is not allowed |

In short: **no external fetches, no redirects, no raw HTML at any point in the form's lifecycle.** For a fixed choice list, inline it in `choices`. For a list that must come from a system of record, generate the schema from that system and `update_form`.

Also refused before any of that: a schema over **256 KB**, or one that is not valid JSON.

## SurveyJS 2.x, not 1.x

`survey-core` 2.x renamed a pile of properties. The old names are deprecated aliases that still resolve at runtime but are **absent from `references/surveyjs-subset.json`** and from SurveyJS's current docs — if you write one from memory you are writing against a schema nobody is checking. Use the 2.x name:

| Wrong (1.x / invented) | Right (3.0.4) |
|---|---|
| `required` | `isRequired` |
| `hasOther` / `hasNone` / `hasSelectAll` | `showOtherItem` / `showNoneItem` / `showSelectAllItem` |
| `optionsCaption` / `placeHolder` | `placeholder` |
| `showClearButton` | `allowClear` |
| `isAllRowRequired` / `isAllRowUnique` | `eachRowRequired` / `eachRowUnique` |
| `goNextPageAutomatic` | `autoAdvanceEnabled` (survey level) |
| `requiredText` | `requiredMark` (survey level) |
| `panelAddText` / `panelRemoveText` | `addPanelText` / `removePanelText` |

Other shape mistakes that validate but render wrong:

- `choices` belongs on the **question**, not on the page or inside `elements`.
- Choice items are `{"value": "...", "text": "..."}` — or a bare string, which becomes both value and text. `data` stores the **`value`**, so pick machine-friendly values (`"ops"`, not `"Operations"`).
- `matrixdropdown`/`matrixdynamic` cells are declared in `columns`, each `{"name", "title", "cellType", ...}`. **`cellType`, not `type`.** Column names do **not** become top-level `data` keys.
- `paneldynamic`'s children go in `templateElements`, not `elements`.
- A `panel` is a container: give it `elements`, and remember its `name` produces no answer key.

## Conditional logic

Branching is done with **properties, not types**, so all of it passes the allowlist.

| Property | On | Effect |
|---|---|---|
| `visibleIf` | question, panel, page | show only when the expression is true |
| `enableIf` | question, panel | render but disable until true |
| `requiredIf` | question | make required only when true |
| `resetValueIf` / `setValueIf` + `setValueExpression` | question | clear/derive an answer reactively |
| `choicesVisibleIf` / `choicesEnableIf` | select questions | filter the choice list |
| `templateVisibleIf` | `paneldynamic` | show only matching panels |
| `defaultValueExpression` | question | computed initial value |

Expression syntax:

- Reference an answer with braces: `{passed} = false`, `{invoice_total} > 1000`.
- Inside a `paneldynamic` template, reference a sibling in the **same panel** with `{panel.<name>}`; inside a `matrixdynamic` row, `{row.<colName>}`.
- Operators: `= <> > >= < <=`, `and` `or` `not`, `empty` / `notempty`, `contains` / `notcontains` (arrays), `anyof` / `allof` (arrays), `*` `+` `-` `/`.
- String literals take single quotes: `{failure_reasons} contains 'safety'`.
- Functions (all verified present in 3.0.4): `iif(cond, a, b)`, `age()`, `today()`, `currentDate()`, `dateDiff()`, `dateAdd()`, `round()`, `sum()`, `avg()`, `min()`, `max()`, `sumInArray()`, `countInArray()`, `displayValue()`.
- Watch out: a question that has never been answered is `empty`, and `{x} = false` is **false** for an unanswered boolean. Guard with `{x} notempty and ...` when that matters.

Survey-level settings that interact with branching:

- `clearInvisibleValues`: `"onComplete"` (default), `"onHidden"`, `"onHiddenContainer"`, `"none"`. This decides whether an answer the user gave and then hid survives into `task.form.data`. Choose deliberately — `"onHidden"` is usually what you want for a branchy form.
- `triggers` (`complete`, `setvalue`, `copyvalue`, `skip`, `runexpression`) are allowed; their `type` values are not question types and the validator knows that.
- **"Can the form branch?" is nearly always answered by `visibleIf`, `paneldynamic` or `matrixdynamic`** — not by needing a type we refuse.

## The iteration loop

Over MCP:

1. **`list_forms`** — always, before creating anything. Prefer updating an existing form to adding a near-duplicate.
2. **`validate_form_schema`** — free: no write, no permission, no audit entry. It runs the *same* validator `create_form` does and returns `{"valid":…, "message":…, "questionKeys":[…]}` in both outcomes. **Iterate here.** Check `questionKeys` is exactly the list you intend downstream nodes to use — that is the cheapest way to catch a name you nested in the wrong place.
3. **`create_form`** (or `update_form`, which is a *full* replace — `get_form` first and send every field back) — requires the manage-forms permission. Both return `publicId`, `version`, `questionKeys` and the app link.
4. **Attach** by setting `formAttachment.formId` to that **`publicId`** on a Create/Update Task node, then `save_workflow_draft`.
5. **Open `/forms/{publicId}`** (the `Link:` in the tool result) and eyeball the rendered form, or ask the user to. Validation cannot tell you a form reads badly.

`validate_form_schema` is cheap enough that there is never a reason to skip it. `create_form`/`update_form` are audited, permission-gated writes; do not use them as a spell-checker.

## Worked examples

Every example below is verified in CI: it passes the real server validator and produces exactly the `questionKeys` listed under it.

### 1. Short intake form

<!-- example: intake -->
```json
{
  "title": "Supplier invoice intake",
  "showQuestionNumbers": "off",
  "pages": [
    {
      "name": "invoice",
      "elements": [
        { "type": "text", "name": "supplier_name", "title": "Supplier", "isRequired": true },
        {
          "type": "text",
          "name": "invoice_number",
          "title": "Invoice number",
          "isRequired": true,
          "placeholder": "INV-0000"
        },
        {
          "type": "text",
          "name": "invoice_total",
          "title": "Total (USD)",
          "inputType": "number",
          "isRequired": true,
          "validators": [{ "type": "numeric", "minValue": 0 }]
        },
        {
          "type": "dropdown",
          "name": "cost_centre",
          "title": "Cost centre",
          "isRequired": true,
          "choices": [
            { "value": "ops", "text": "Operations" },
            { "value": "eng", "text": "Engineering" },
            { "value": "sales", "text": "Sales" }
          ]
        },
        { "type": "comment", "name": "notes", "title": "Anything the approver should know?" }
      ]
    }
  ]
}
```

`questionKeys`: `supplier_name`, `invoice_number`, `invoice_total`, `cost_centre`, `notes`

Validators are `{"type": "<kind>", ...}` inside `validators` — `numeric` (`minValue`/`maxValue`), `text` (`minLength`/`maxLength`), `email`, `regex` (`regex`, `caseInsensitive`), `answercount` (`minCount`/`maxCount`), `expression` (`expression`). Their `type` is a validator kind, not a question type, and the allowlist correctly ignores it.

### 2. Branching with `visibleIf` / `requiredIf`

<!-- example: branching -->
```json
{
  "title": "Site inspection outcome",
  "clearInvisibleValues": "onHidden",
  "pages": [
    {
      "name": "outcome",
      "elements": [
        {
          "type": "boolean",
          "name": "passed",
          "title": "Did the site pass inspection?",
          "isRequired": true
        },
        {
          "type": "checkbox",
          "name": "failure_reasons",
          "title": "What failed?",
          "visibleIf": "{passed} = false",
          "requiredIf": "{passed} = false",
          "showOtherItem": true,
          "otherText": "Something else",
          "choices": [
            { "value": "safety", "text": "Safety" },
            { "value": "cleanliness", "text": "Cleanliness" },
            { "value": "paperwork", "text": "Paperwork" }
          ]
        },
        {
          "type": "comment",
          "name": "safety_detail",
          "title": "Describe the safety issue",
          "visibleIf": "{failure_reasons} contains 'safety'",
          "isRequired": true
        },
        {
          "type": "text",
          "name": "reinspection_date",
          "title": "Re-inspection date",
          "inputType": "date",
          "visibleIf": "{passed} = false"
        },
        {
          "type": "rating",
          "name": "site_score",
          "title": "Overall score",
          "rateMin": 1,
          "rateMax": 5,
          "visibleIf": "{passed} = true"
        }
      ]
    }
  ]
}
```

`questionKeys`: `passed`, `failure_reasons`, `safety_detail`, `reinspection_date`, `site_score`

Note that `isRequired` on a hidden question is not enforced — SurveyJS only validates visible questions — so `visibleIf` + `isRequired` and `requiredIf` are both fine; use `requiredIf` when the question stays visible. With `showOtherItem`, the "other" free text lands in `data` as `failure_reasons-Comment` alongside the `"other"` entry in the array; if a downstream node needs that text, prefer an explicit `visibleIf` comment question instead.

### 3. Repeating sections with `paneldynamic`

<!-- example: repeating -->
```json
{
  "title": "Delivery report",
  "pages": [
    {
      "name": "delivery",
      "elements": [
        { "type": "text", "name": "driver_name", "title": "Driver", "isRequired": true },
        {
          "type": "paneldynamic",
          "name": "stops",
          "title": "Stops made",
          "templateTitle": "Stop {panelIndex}",
          "addPanelText": "Add a stop",
          "removePanelText": "Remove this stop",
          "panelCount": 1,
          "minPanelCount": 1,
          "maxPanelCount": 20,
          "templateElements": [
            { "type": "text", "name": "stop_address", "title": "Address", "isRequired": true },
            {
              "type": "text",
              "name": "stop_arrived_at",
              "title": "Arrival time",
              "inputType": "time"
            },
            { "type": "boolean", "name": "stop_signed_for", "title": "Signed for?" },
            {
              "type": "comment",
              "name": "stop_exception",
              "title": "What went wrong?",
              "visibleIf": "{panel.stop_signed_for} = false"
            }
          ]
        }
      ]
    }
  ]
}
```

`questionKeys`: `driver_name`, `stops`, `stop_address`, `stop_arrived_at`, `stop_signed_for`, `stop_exception`

**Read that key list carefully — it is the one place the examples surprise people.** The answers live under the panel's own key as an array:

```js
task.form.data.stops // => [{stop_address: "...", stop_arrived_at: "09:15", stop_signed_for: true}, ...]
task.form.data.stops.length
task.form.data.stop_address // => always null. Not a real key.
```

The template question names are reported as keys too, and appear in `data` as permanent `null` placeholders. Ignore them; iterate the array. (Because they occupy the same namespace, keep template names distinct from top-level ones — hence the `stop_` prefix.)

### 4. Tables with `matrixdynamic`

<!-- example: table -->
```json
{
  "title": "Stock count",
  "pages": [
    {
      "name": "count",
      "elements": [
        { "type": "text", "name": "counted_by", "title": "Counted by", "isRequired": true },
        {
          "type": "matrixdynamic",
          "name": "stock_lines",
          "title": "Counted lines",
          "rowCount": 1,
          "minRowCount": 1,
          "maxRowCount": 50,
          "addRowText": "Add a line",
          "columns": [
            {
              "name": "sku",
              "title": "SKU",
              "cellType": "text",
              "isRequired": true,
              "isUnique": true
            },
            { "name": "counted_qty", "title": "Counted", "cellType": "text", "isRequired": true },
            {
              "name": "condition",
              "title": "Condition",
              "cellType": "dropdown",
              "choices": [
                { "value": "sellable", "text": "Sellable" },
                { "value": "damaged", "text": "Damaged" }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

`questionKeys`: `counted_by`, `stock_lines`

Unlike `paneldynamic`, **column names produce no extra keys**: `task.form.data.stock_lines` is an array of `{sku, counted_qty, condition}`. Use `matrixdynamic` for a few short fields per row and `paneldynamic` when a row needs its own branching or long text. `matrixdropdown` is the same thing with a fixed `rows` list instead of user-added rows.

## Reference

- **`references/surveyjs-subset.json`** — the authoritative property list: SurveyJS's own JSON Schema for survey definitions, generated from the installed `survey-core` 3.0.4 and subsetted to exactly what Honeybase accepts. Its `honeybase` block carries `allowedQuestionTypes` and `conditionalQuestionTypes`. Look a property up here before using it; if it is not in the definition for that type, do not write it.
  - Recipe: `jq '.definitions.paneldynamic' references/surveyjs-subset.json` for a type's own properties; each type inherits through its `allOf: [{"$ref": "question"}, …]` chain, so check the referenced bases too. `jq -r '.honeybase' …` for the allowlist.
  - `survey-core` is MIT © Devsoft Baltic OÜ; that file is a derived subset and keeps the attribution.
- **Honeybase's own docs** at `https://docs.honeybase.ai` complement this bundle with prose and context: `https://docs.honeybase.ai/concepts/tasks-and-forms/` for how forms attach to tasks and flow downstream, and the wider [Concepts](https://docs.honeybase.ai/concepts/) pages. Keep checking property names against `references/surveyjs-subset.json` — it is the authoritative allowlist.
- **SurveyJS's own docs** (`https://surveyjs.io/form-library/documentation`) for anything deeper — expression functions, masks, localization. Everything there about `choicesByUrl`, `completedHtml`, custom widgets, `html`/`image` questions and the Creator's built-in themes does **not** apply here.

## Sharing this skill

`.claude/skills/honeybase-forms/` is self-contained — copy it into any project's `.claude/skills/` (or `~/.claude/skills/`). It is also hosted at `https://honeybase.ai/skills/honeybase-forms/` (SKILL.md, references/surveyjs-subset.json, version.json — compare `version.json` with the `version:` line above to know when to re-download); the MCP `get_forms_skill` tool hands out those URLs and the `curl` commands. Pair it with `honeybase-graphql-api` if the integration also reads or writes tasks directly.

## Changelog

- 2026-09-13 — `survey-core` 2.x → 3.0.4; `references/surveyjs-subset.json` regenerated (property list only, no new guidance yet).
