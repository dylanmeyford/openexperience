# openexperts Specification

Version: `1.0`

## 1. Overview

openexperts is an open format for packaging professional expertise into files that AI frameworks can consume.

An expert package captures:

- Role judgment (how to think)
- Decision heuristics (how to decide)
- Workflows (how to execute)
- Domain references (what to reference)

### Design Principles

- **No code in the package**: packages are markdown and YAML only.
- **Framework agnostic**: any agent framework can consume the files.
- **Agentic-loop native**: processes are playbooks for an agent loop, not an executable DSL.
- **Portable and readable**: experts can author and review files directly.

## 2. Package Structure

An expert package is a directory with the following structure:

```text
expert-package/
  README.md
  expert.yaml
  orchestrator.md
  persona/
    identity.md
    rules.md
  functions/
    *.md
  processes/            # optional
    *.md
  tools/                # optional
    *.yaml
  knowledge/            # optional
    *.md
  state/                # optional
    *.md
  scratch/              # runtime (auto-created, not committed)
    *.md
```

### Required Components

- `README.md`
- `expert.yaml`
- `orchestrator.md`
- `persona/` (at least one persona file)
- `functions/` (at least one function file)

### Optional Components

- `processes/`
- `tools/`
- `knowledge/`
- `state/`
- `scratch/` (runtime-generated)

### Naming Conventions

- Use kebab-case file names where practical (for example `classify-email-intent.md`).
- Function and process files use markdown with YAML frontmatter.
- Tool files use YAML.

## 3. Manifest (`expert.yaml`)

The manifest is the package index and metadata source.

### Required Fields

- `spec` (string): spec version for this package format, for example `"1.0"`.
- `name` (string): package name.
- `version` (string): package version (SemVer recommended).
- `description` (string): short package summary.
- `components` (object): lists package component files.

### Optional Fields

- `author` (string)
- `license` (string)
- `requires` (object): dependencies such as abstract tools.
- `triggers` (array): automation entry points that invoke processes on events.
- `concurrency` (object): package-level concurrency defaults inherited by all triggers.
- `execution` (object): package-level process execution defaults inherited by all processes.
- `delivery` (object): package-level output delivery defaults inherited by all processes.
- `policy` (object): action governance rules declaring which tool operations require human approval.
- `outputs` (array of strings): human-readable list of primary deliverables this expert produces. Each entry is a short label (for example `"email drafts"`, `"pipeline summaries"`). These are descriptive metadata for indexing and documentation — they are not typed contracts. For typed output declarations, see process-level `outputs`.

### `requires`

`requires` currently defines one key: `tools`. Future spec versions may add additional dependency types (for example `knowledge_sources`, `credentials`). Frameworks should ignore unrecognized keys under `requires` and should not error on them.

### `requires.tools`

`requires.tools` is an array of abstract tool names the package expects at runtime.

Example:

```yaml
requires:
  tools:
    - crm
    - email
    - calendar
```

### `concurrency`

`concurrency` sets the default execution model for all triggers in the package. Individual triggers inherit these defaults and may override them.

#### Concurrency Fields

- `default` (string): default concurrency mode. One of `parallel`, `serial`, or `serial_per_key`. Defaults to `parallel` if omitted.
- `key` (string): default `concurrency_key` field path applied when `default` is `serial_per_key`. Individual triggers may override this with their own `concurrency_key`.

#### When to set a package-level default

Set a package-level default when most or all triggers share the same concurrency model. This avoids repeating the same setting on every trigger and prevents silent bugs from a missed per-trigger declaration.

| Expert type | Recommended default |
|---|---|
| Sales agent | `serial_per_key`, key: `contact_id` |
| Support agent | `serial_per_key`, key: `ticket_id` |
| Legal agent | `serial_per_key`, key: `matter_id` |
| Marketing agent | `parallel` |
| Report generator | `serial` |

#### Example

```yaml
# All triggers process in order per contact by default.
# Any trigger can override this locally if it needs different behavior.
concurrency:
  default: serial_per_key
  key: contact_id
```

### `execution`

`execution` sets the default runtime guarantees for all processes in the package. Individual processes inherit these defaults and may override them in their own frontmatter.

#### Execution Fields

- `timeout` (string): maximum wall-clock time a process may run before the framework considers it failed. Use human-readable durations such as `5m`, `30m`, `2h`. Defaults to no timeout if omitted.
- `idempotent` (boolean): declares whether it is safe to re-run the full process from step 1 on failure. When `true`, the framework may retry without risk of duplicate side-effects. When `false`, the framework should use execution-log-guided resumption rather than a full restart. Defaults to `false`.
- `retry` (object): retry policy applied when a process fails.
  - `max_attempts` (integer): maximum number of attempts including the first. Defaults to `1` (no retry).
  - `backoff` (string): `fixed` or `exponential`. Defaults to `exponential`.
  - `delay` (string): initial delay between attempts. Defaults to `30s`.
- `on_failure` (string): action taken when all retry attempts are exhausted. One of:
  - `escalate` (default): notify the main agent and user that the process failed and requires attention.
  - `abandon`: log the failure silently and drop the task. Use only for non-critical background tasks.
  - `dead_letter`: queue the failed invocation for manual review and reprocessing.
- `resume_from_execution_log` (boolean): when `true`, the framework maintains an opaque execution log tracking which steps have completed, retry count, and failure details. On retry, the framework injects this context into the agent's prompt so the agent knows where it left off. Requires `idempotent: false`. Defaults to `false`.

  The execution log and the process `scratchpad` serve different purposes and work together:
  - **Execution log** (framework-owned): tracks step completion status, retry count, and failure metadata. The framework writes it; the agent never reads or writes it directly. The framework uses it to tell the agent "you completed steps 1–4, step 5 failed."
  - **Scratchpad** (agent-owned): stores working data — intermediate results, CRM lookups, classification outputs — that the agent needs to resume meaningfully. The agent reads and writes it directly.

  A process can use either mechanism independently, or both together. When both are present, the execution log tells the agent *where* to resume and the scratchpad provides the *working state* needed to continue.

#### Why these defaults matter

Without explicit execution policy, a failed process silently disappears. For a sales agent, a failed email triage is a missed deal signal. Package authors should always declare a policy — even just `on_failure: escalate` — so failures are visible.

#### Example

```yaml
# Processes get 10 minutes, retry up to 3 times with exponential backoff,
# resume using execution logs (and scratchpad context), and escalate if still failing.
execution:
  timeout: 10m
  idempotent: false
  retry:
    max_attempts: 3
    backoff: exponential
    delay: 30s
  on_failure: escalate
  resume_from_execution_log: true
```

### `delivery`

`delivery` sets default output delivery behavior for all processes in the package. Individual processes inherit these defaults and may override them in process frontmatter.

#### Delivery Fields

- `format` (string): `narrative`, `structured`, or `both`. Defaults to `both`.
- `channel` (string): output destination. `main` delivers to the primary agent session and chat channel. Defaults to `main`. Future spec versions or framework extensions may define additional channel types (for example `slack`, `email`). Frameworks should treat unrecognized channel values as an error at load time.
- `sla_breach` (string): action when expected delivery SLA is exceeded. One of `warn` or `escalate`. Defaults to `warn`.

#### Example

```yaml
delivery:
  format: both
  channel: main
  sla_breach: warn
```

### `policy`

`policy` declares the governance rules for agent actions in this package. It defines which tool operations require human approval before executing, and what the default approval tier is for operations not explicitly listed in `overrides`.

#### Approval Tiers

Every tool operation has an approval tier. There are three tiers:

- `auto`: the agent executes the operation immediately with no human involvement. Use for reads, internal state updates, and any action with no external side-effects.
- `confirm`: the agent pauses before executing, presents the proposed action to the user via the main chat channel, and waits for explicit approval. If approved, execution continues. If rejected or timed out, the process follows the `on_failure` policy. Use for external-facing writes such as updating a deal stage, scheduling a meeting, or creating a formal record.
- `manual`: the agent prepares and drafts the action but never executes it. The completed draft is delivered to the human for them to execute themselves. The process considers this step done once the draft is delivered. Use for actions that should never be automated, such as sending an external email, posting to social media, or committing to pricing.

#### Policy Fields

- `approval` (object):
  - `default` (string): approval tier applied to all tool operations not listed in `overrides`. One of `auto`, `confirm`, `manual`. Defaults to `confirm` if omitted — requiring human approval is the safe default for any unknown operation.
  - `overrides` (object): per-operation tier overrides. Keys are `tool_name.operation_name`; values are `auto`, `confirm`, or `manual`. Use to tighten or relax the default for specific operations.
  - `timeout` (string): how long to wait for human approval on a `confirm` operation before timeout handling is applied (for example `24h`). Defaults to no timeout if omitted.
  - `on_timeout` (string): action when an approval times out. One of `reject` or `escalate`. Defaults to `reject`. When `reject`, the step is treated as a failed operation and the process follows its `execution.on_failure` policy (which may itself escalate, abandon, or dead-letter). When `escalate`, the framework immediately notifies the escalation channel without treating the step as a failure, giving the human a chance to approve late.

#### Approval Tier Resolution Order

When the framework needs the effective approval tier for a tool operation, it resolves in this order:

1. **`policy.approval.overrides`** — if the operation has an explicit entry (for example `crm.create_note: auto`), use it.
2. **`policy.approval.default`** — if no override exists, use the package-level default.
3. **`confirm`** — if no `policy` block exists at all.

The `approval` field on a tool operation declaration (in `tools/*.yaml`) is **documentation only**. It declares the natural risk level of the operation for human readers and tooling, but it does not participate in runtime resolution. The package `policy` block is the sole source of truth for runtime approval behavior. This avoids ambiguity when the same tool definition is shared across packages with different governance requirements.

#### Why `confirm` is the safe default

An expert package that omits a `policy` block, or omits an operation from `overrides`, defaults to `confirm`. This means unknown or undeclared operations always pause for human review. Package authors opt *down* to `auto` for safe operations and opt *up* to `manual` for irreversible ones — they never accidentally grant silent write access.

#### `policy.escalation`

`escalation` defines how the expert hands off to a human when it cannot or should not proceed autonomously.

Fields:

- `channel` (string): where escalation messages are delivered. Defaults to `main` (the primary agent session and chat channel).
- `on_low_confidence` (boolean): when `true`, processes should escalate rather than act when the agent judges its own output confidence as low. The agent is responsible for flagging low confidence in its reasoning; this setting tells it what to do when it does. Defaults to `true`.
- `on_manual_ready` (boolean): when `true`, the framework notifies the escalation channel whenever a `manual`-tier action draft is ready for human execution. Defaults to `true`.

Escalation messages must include:
- Which process escalated and why
- The relevant context (contact, deal, message) so the human can act immediately
- The agent's recommended action, even if it cannot execute it autonomously

#### Example

```yaml
policy:
  approval:
    default: confirm
    overrides:
      crm.get_contact: auto
      crm.get_deal: auto
      crm.create_note: auto
      email.get_email: auto
      crm.update_deal_stage: confirm
      email.send: manual
      calendar.schedule_meeting: manual
  escalation:
    channel: main
    on_low_confidence: true
    on_manual_ready: true
```

### `triggers`

`triggers` declares the automation entry points for this package. Each trigger defines when a process runs, how it is invoked, and what runtime behavior the framework should apply.

Triggers are declarative. The framework (or loader) is responsible for wiring each trigger to the underlying runtime mechanism (webhook, cron schedule, channel event, etc.). Package authors declare intent; they do not configure infrastructure.

#### Trigger Fields

Required:

- `name` (string): unique trigger name within the package. Must match the `trigger` field on the target process.
- `type` (string): one of `webhook`, `cron`, or `channel`.
- `process` (string): name of the process to invoke when this trigger fires.

Optional:

- `preset` (string): maps to a built-in runtime preset (for example `gmail` for the OpenClaw Gmail hook). When set, the framework uses its built-in handling for this event source. Presets are framework-defined — this spec does not maintain a registry of valid preset names. Framework documentation should list supported presets, describe the payload shape each preset provides (including any enrichment fields), and specify which `dedupe_key` and `concurrency_key` paths are available.
- `requires_tool` (string): abstract tool name that provides this trigger when no preset covers it. The framework must resolve this tool before the trigger can be active.
- `expr` (string): cron expression (5-field standard or 6-field with seconds). Required when `type` is `cron`.
- `tz` (string): IANA timezone for the cron expression. Defaults to UTC.
- `dedupe_key` (string): field path in the incoming payload used to identify duplicate events. The framework should skip processing if an event with the same key was already handled within a reasonable window.
- `session` (string): `isolated` (default, recommended) or `main`. `isolated` creates a new, independent session for each trigger invocation — it has its own context window and ends when the process completes. `main` enqueues the event into the primary long-running agent session, which persists across interactions. For the purpose of state file `scope`, a "session" maps to a single isolated session or a single continuous main session. State files with `scope: session` reset at the start of each isolated session and when the main session is restarted.
- `concurrency` (string): overrides the package-level `concurrency.default` for this trigger. One of:
  - `parallel`: each invocation runs immediately in its own session, with no coordination. Use for independent workloads such as processing multiple articles or documents at the same time.
  - `serial`: all invocations of this trigger queue globally and run one at a time. Use when overlap would cause duplicate work or corrupted output, such as a nightly report that must not run twice.
  - `serial_per_key`: invocations queue per a grouping key derived from the trigger payload. Invocations with the same key run in order; invocations with different keys run in parallel. Use for entity-scoped workflows such as a sales agent that must process emails for the same deal sequentially, while handling different deals concurrently.
- `concurrency_key` (string): overrides the package-level `concurrency.key` for this trigger. Required when this trigger's effective concurrency mode is `serial_per_key` and no package-level key is set. A dot-notation path into the trigger payload identifying the grouping field (for example `contact_id`, `deal_id`, `thread_id`).

  **Key resolution**: the `concurrency_key` path is resolved against the trigger payload *after* any enrichment the framework or preset performs. For example, a `gmail` preset may enrich the raw webhook payload with a `contact_id` derived from sender email before the concurrency key is evaluated. If the key path cannot be resolved for a given invocation, the framework should fall back to `serial` (global queue) for that invocation and emit a warning.
- `payload_mapping` (object): maps trigger payload fields to process input names. Keys are process `inputs` names; values are dot-notation paths in the incoming payload.
- `description` (string): human-readable explanation of when this trigger fires.

#### Trigger Types

**`webhook`** — fires when an inbound HTTP event is received. The framework maps the trigger to a webhook endpoint or preset handler. Use this for email (Gmail PubSub), external app notifications, and any event-driven push source.

**`cron`** — fires on a schedule defined by `expr` and `tz`. Use this for proactive tasks: scanning for opportunities, generating daily summaries, sending follow-up reminders.

**`channel`** — fires when a message arrives on a connected messaging channel (for example iMessage, WhatsApp, Telegram, Slack). The framework routes incoming channel messages to the process when the message matches the trigger's channel source. The framework is responsible for normalizing channel payloads into a consistent shape. At minimum, channel payloads should include `sender_id`, `message_text`, and `channel_name` fields so that `payload_mapping` and `concurrency_key` can reference them portably.

#### Example Triggers Block

```yaml
triggers:
  # serial_per_key: emails for the same deal are processed in order,
  # but emails for different deals run in parallel.
  - name: new_email
    type: webhook
    preset: gmail
    dedupe_key: message_id
    session: isolated
    concurrency: serial_per_key
    concurrency_key: contact_id
    payload_mapping:
      message_id: messages[0].id
    process: inbound-email-triage
    description: Fires when a new email arrives in the monitored inbox.

  # serial: the morning scan must not overlap with itself.
  - name: opportunity_scan
    type: cron
    expr: "0 8 * * 1-5"
    tz: Australia/Sydney
    session: isolated
    concurrency: serial
    process: scan-for-opportunities
    description: Runs every weekday morning to scan for new signals on active accounts.

  # parallel: each article is independent; process them all at the same time.
  - name: new_article
    type: webhook
    requires_tool: content_feed
    dedupe_key: article_id
    session: isolated
    concurrency: parallel
    process: process-article
    description: Fires when a new article is published to the monitored feed.

  # serial_per_key: LinkedIn DMs serialized per sender.
  - name: linkedin_dm
    type: webhook
    requires_tool: linkedin
    dedupe_key: message_id
    session: isolated
    concurrency: serial_per_key
    concurrency_key: sender_id
    payload_mapping:
      sender_id: messages[0].from
    process: handle-linkedin-dm
    description: Fires when a new LinkedIn DM is received. Requires a LinkedIn tool binding.
```

### `components`

`components` lists files by component type so frameworks can discover them without scanning every file.

Valid keys:

- `orchestrator` (string): path to `orchestrator.md`.
- `persona` (array of strings): persona file paths.
- `functions` (array of strings): function file paths.
- `processes` (array of strings): process file paths.
- `tools` (array of strings): tool declaration file paths.
- `knowledge` (array of strings): knowledge file paths.
- `state` (array of strings): state template file paths.

### Complete Example Manifest

```yaml
spec: "1.0"
name: radiant-sales-expert
version: "0.1.0"
description: B2B sales expertise for inbound deal triage and next-best-action
author: Openexperts Community
license: MIT

requires:
  tools:
    - crm
    - email
    - calendar

# All triggers default to serial_per_key on contact_id.
# The opportunity scan overrides to serial (global queue, no per-key needed).
concurrency:
  default: serial_per_key
  key: contact_id

# Reads and internal notes are auto. Writes need confirmation. External sends are manual.
# Escalate to main chat when confidence is low or a manual draft is ready.
policy:
  approval:
    default: confirm
    timeout: 24h
    on_timeout: escalate
    overrides:
      crm.get_contact: auto
      crm.get_deal: auto
      crm.create_note: auto
      email.get_email: auto
      crm.update_deal_stage: confirm
      email.send: manual
      calendar.schedule_meeting: manual
  escalation:
    channel: main
    on_low_confidence: true
    on_manual_ready: true

# All processes get 10 minutes, retry 3 times, resume from scratchpad, escalate on failure.
execution:
  timeout: 10m
  idempotent: false
  retry:
    max_attempts: 3
    backoff: exponential
    delay: 30s
  on_failure: escalate
  resume_from_execution_log: true

# Process output defaults (can be overridden per process).
delivery:
  format: both
  channel: main
  sla_breach: warn

# Primary deliverables produced by this expert.
outputs:
  - email drafts
  - deal triage reports
  - pipeline summaries

triggers:
  - name: new_email
    type: webhook
    preset: gmail
    dedupe_key: message_id
    session: isolated
    payload_mapping:
      message_id: messages[0].id
    process: inbound-email-triage
    description: Fires when a new email arrives in the monitored inbox.

  - name: opportunity_scan
    type: cron
    expr: "0 8 * * 1-5"
    tz: Australia/Sydney
    session: isolated
    concurrency: serial        # override: global queue, not per-contact
    process: scan-for-opportunities
    description: Runs every weekday morning to scan for new signals on active accounts.

components:
  orchestrator: orchestrator.md
  persona:
    - persona/identity.md
    - persona/rules.md
  functions:
    - functions/classify-email-intent.md
    - functions/determine-next-action.md
    - functions/compose-response.md
  processes:
    - processes/inbound-email-triage.md
    - processes/scan-for-opportunities.md
  tools:
    - tools/crm.yaml
    - tools/email.yaml
    - tools/calendar.yaml
  knowledge:
    - knowledge/meddpicc.md
    - knowledge/competitive-battle-cards.md
  state:
    - state/pipeline.md
    - state/session-notes.md
```

## 4. Orchestrator (`orchestrator.md`)

The orchestrator is the package entry point. It tells the agent:

- When to use this expert package
- Which functions or processes to reach for
- How to route between scenarios

### Format

- Plain markdown
- No frontmatter required

### Runtime Consumption

Frameworks should load orchestrator content into persistent context (for example, system prompt/bootstrap context) so the agent can route correctly at runtime.

## 5. Persona (`persona/`)

Persona files define identity and behavior.

Recommended files:

- `persona/identity.md`: role, voice, and style
- `persona/rules.md`: behavioral constraints, priorities, and guardrails

Additional persona files are allowed.

### Runtime Consumption

Frameworks should load persona content into persistent context with the orchestrator.

## 6. Functions (`functions/*.md`)

A function is a self-contained capability. Functions contain domain instructions, heuristics, and output expectations that can be used on demand inside an agent loop.

### Required Format

Each function file must include:

1. YAML frontmatter
2. Markdown instruction body

### Function Frontmatter Fields

Required:

- `name` (string)
- `description` (string)

Optional:

- `inputs` (array of objects with `name`, `type`, optional `description`)
- `outputs` (array of objects with `name`, `type`, optional `description`, optional `enum`)
- `tools` (array of abstract tool names)
- `knowledge` (array of knowledge file references)
- `tags` (array of strings): categorization for indexing and routing
- `session` (string): function execution mode. One of:
  - `inline` (default): execute in the current agent session.
  - `isolated`: execute in a fresh child sub-agent session and return only declared outputs to the caller.

Functions that drive escalation behavior should include a `confidence` output field with:

- `type: string`
- `enum: [high, medium, low]`

Frameworks can use this convention to enforce `policy.escalation.on_low_confidence`.

### Function Body Requirements

The markdown body should include:

- Clear instructions
- Decision rules / evaluation heuristics
- Output formatting guidance

### Example Function

```markdown
---
name: classify-email-intent
description: Determine the intent and urgency of an inbound email
tags: [analysis, classification, email]
inputs:
  - name: email_body
    type: string
  - name: sender_context
    type: object
    description: CRM context for sender, deal stage, and history
outputs:
  - name: intent
    type: string
    enum: [objection, question, buying_signal, churn_risk, scheduling, other]
  - name: urgency
    type: string
    enum: [high, medium, low]
  - name: reasoning
    type: string
  - name: confidence
    type: string
    enum: [high, medium, low]
tools:
  - crm
knowledge:
  - knowledge/competitive-battle-cards.md
---

## Classify Email Intent

Given email body and sender CRM context, classify intent and urgency.

### Decision Rules

- If the sender is in a late deal stage and mentions a competitor, classify as `objection` and `high`.
- If there has been >14 days of inactivity and the reply is short/hesitant, classify as `churn_risk`.
- If the email asks product or technical details, classify as `question`.

### Output

Return:
- Intent
- Urgency
- Reasoning (1-2 sentences)
- Confidence
```

### Runtime Consumption

Frameworks should expose functions as readable/invokable capabilities (for example, skills) and load them on demand instead of always preloading all function content.

When `session: isolated` is set on a function:

- The framework should execute that function in a new child session.
- The child session input should be the function's declared `inputs` values.
- The caller should receive only the function's declared `outputs` values plus any framework metadata.
- Child session working context is not merged back into the caller session.

Frameworks may also provide runtime overrides (for example, force `isolated` for specific functions), but package-level function frontmatter remains the canonical default.

## 7. Processes (`processes/*.md`)

A process is a multi-step workflow that coordinates tool usage and function usage.

Processes are authored as markdown playbooks, not executable workflow code.

### Process Frontmatter Fields

Required:

- `name` (string)
- `description` (string)

Optional:

- `trigger` (string): event label that commonly initiates the process.
  This field must match a `triggers[].name` entry in `expert.yaml`. The manifest trigger defines invocation mechanics; this process field declares which trigger this process responds to.
- `inputs` (array): starting data requirements
- `outputs` (array): expected final outputs
- `functions` (array of function names): discovery and indexing support
- `tools` (array of abstract tool names): discovery and indexing support
- `scratchpad` (string): recommended working file path pattern
- `tags` (array of strings): categorization for indexing and routing
- `context` (array): file paths to preload before step execution (for example state or knowledge files)
- `execution` (object): overrides the package-level execution defaults for this process. Supports the same fields as the package-level `execution` block (`timeout`, `idempotent`, `retry`, `on_failure`, `resume_from_execution_log`). Only specified fields are overridden; unspecified fields inherit from the package default.
- `delivery` (object): declares how the process output is delivered when it completes.
  - `format` (string): `narrative` (a human-readable chat summary), `structured` (the typed `outputs` object), or `both`. Defaults to `both`.
  - `channel` (string): where the output is delivered. `main` delivers to the primary agent session and chat. Defaults to `main`.
  - `sla` (string): expected wall-clock time from trigger to delivery (for example `5m`, `30m`). Used by the framework to detect slow runs and warn the user. Not a hard timeout — use `execution.timeout` for that.
  - `sla_breach` (string): action taken if `sla` is exceeded before the process completes. One of `warn` (notify the user that the process is running slow but let it continue) or `escalate` (notify and flag for human attention). Defaults to `warn`.

### Process Body Requirements

A process body should include:

- Context preamble: when and why this process runs
- Ordered steps section
- Completion section with expected output

### Checklist Pattern (Required for resumable processes)

Use markdown checkboxes for step tracking:

- `- [ ] Step ...`

This gives the agent explicit progress cues in agentic loops and is the mechanism by which the agent knows which steps are complete when resuming from a scratchpad.

### Scratchpad Pattern

Processes that declare a `scratchpad` path must instruct the agent to write intermediate step results to that file as they complete. This enables:

- **Resumption**: on retry, the agent reads the scratchpad to recover working data (CRM lookups, classification results, draft content) and continues from the last completed step, preventing duplicate side-effects such as double CRM writes or duplicate email sends. This works independently of the framework execution log — the scratchpad provides domain-level working state, while the execution log (if enabled) provides step-completion metadata.
- **Auditability**: every process run leaves a record of what happened and what was decided at each step.
- **Context recovery**: if context is compacted or the session is interrupted, the scratchpad preserves the working state.

The first step of any resumable process must always be: create (or read if it already exists) the scratchpad file.

### Dry Run Mode

Frameworks may invoke any process in dry-run mode for testing and safety checks.

In dry-run mode:

- The agent narrates intended actions step-by-step.
- Tool operations at `auto` tier execute normally. By definition, `auto` operations have no external side-effects (reads, internal state updates), so they are safe to run.
- Tool operations at `confirm` and `manual` tiers are **not executed**. The agent describes what it would do, including the operation name, inputs, and expected outcome.
- The process returns a planned execution report instead of producing side-effecting changes.

If a package author has incorrectly assigned `auto` to an operation with external side-effects, dry-run mode will not catch it. Frameworks may optionally cross-reference tool operation descriptions or add a `side_effects` flag to operations in future spec versions to improve dry-run safety.

### Example Process

```markdown
---
name: inbound-email-triage
description: End-to-end handling of an inbound sales email
trigger: new_email
inputs:
  - name: message_id
    type: string
outputs:
  - name: draft_response
    type: object
  - name: classification
    type: object
  - name: recommended_action
    type: object
functions:
  - classify-email-intent
  - determine-next-action
  - compose-response
tags: [inbound, triage, sales]
context:
  - state/pipeline.md
  - knowledge/meddpicc.md
tools:
  - email
  - crm
scratchpad: ./scratch/triage-{message_id}.md
execution:
  timeout: 10m
  # inherits retry and on_failure from package-level execution defaults
delivery:
  format: both
  channel: main
  sla: 5m
  sla_breach: escalate
---

## Inbound Email Triage

Run this process whenever a new prospect email arrives.
Follow every step in order and check it off after completion.
If resuming after a failure, read the scratchpad first and skip any already-completed steps.

### Steps

- [ ] Open (or create) the scratchpad at `./scratch/triage-{message_id}.md`. If it exists, read it and resume from the first unchecked step.
- [ ] Fetch the inbound email using the `email` tool and record sender, subject, body.
- [ ] Fetch contact details from `crm` using sender email and record account context.
- [ ] Fetch active deal details from `crm` and record stage, amount, competitors.
- [ ] Read and apply `classify-email-intent` using email + CRM context; record output.
- [ ] Read and apply `determine-next-action`; record output.
- [ ] Read and apply `compose-response`; record draft response.

### Completion

Return:
- Draft response
- Classification
- Recommended action
```

## 8. Tools (`tools/*.yaml`)

Tool files define abstract interfaces to external systems.

They define what operations are needed, not how integrations are implemented.

### Tool Fields

- `name` (string): abstract tool name referenced by functions/processes
- `description` (string)
- `operations` (array)

Each operation includes:

- `name` (string)
- `description` (string)
- `approval` (string): the natural approval tier for this operation. One of `auto`, `confirm`, or `manual`. This field is **documentation only** — it makes the operation's risk level explicit and self-documenting for human readers and tooling. It does not affect runtime behavior. The effective approval tier at runtime is determined solely by the package `policy` block (see *Approval Tier Resolution Order* in section 3).
- `input` (object shape using simple type declarations)
- `output` (object shape using simple type declarations)

### Type System

Use readable simple types:

- `string`
- `number`
- `boolean`
- `object`
- `array`

Complex fields can define nested `properties` using the same simple types.

### Example Tool

```yaml
name: crm
description: Customer relationship management interface
operations:
  - name: get_contact
    description: Retrieve a contact by email address
    approval: auto
    input:
      type: object
      properties:
        email:
          type: string
    output:
      type: object
      properties:
        contact_id:
          type: string
        name:
          type: string
        company:
          type: string
        deal_stage:
          type: string
        last_activity:
          type: string
        notes:
          type: array
          items:
            type: string

  - name: get_deal
    description: Retrieve active deal details for a contact
    approval: auto
    input:
      type: object
      properties:
        contact_id:
          type: string
    output:
      type: object
      properties:
        deal_name:
          type: string
        amount:
          type: number
        stage:
          type: string
        competitors:
          type: array
          items:
            type: string
        close_date:
          type: string

  - name: update_deal_stage
    description: Update the stage of an active deal
    approval: confirm
    input:
      type: object
      properties:
        deal_id:
          type: string
        stage:
          type: string
        reason:
          type: string
    output:
      type: object
      properties:
        success:
          type: boolean

  - name: create_note
    description: Create a note on a contact record
    approval: auto
    input:
      type: object
      properties:
        contact_id:
          type: string
        body:
          type: string
    output:
      type: object
      properties:
        note_id:
          type: string
```

### Runtime Binding

Frameworks bind tool names declared in expert package files to concrete implementations (for example MCP servers, API clients, or internal services).

## 9. Knowledge (`knowledge/*.md`)

Knowledge files contain reference material the agent can use while executing functions and processes.

Examples:

- Methodologies
- Frameworks
- Battle cards
- Templates
- Policy references

### Format

- Plain markdown
- Optional frontmatter fields such as `name`, `description`, `tags`, `type`

`type` values:

- `static`: stable references such as methodologies, templates, and playbooks.
- `dynamic`: frequently updated material such as market updates or competitor research.
- `private`: sensitive internal information. Frameworks must not include `private` knowledge in package indexes, public-facing summaries, debug logs, or API responses that leave the runtime boundary. `private` knowledge may still be loaded into agent context for use during function and process execution — the restriction applies to exposure outside the agent session, not to the agent itself.

If omitted, `type` defaults to `static`.

### Runtime Consumption

Frameworks may:

- Load selected knowledge into persistent context
- Load knowledge on demand via read/skill mechanisms

The strategy should consider context budget and file size.

## 10. State (`state/*.md`)

State files are builder-defined markdown templates for local read/write storage. The expert builder decides what state the expert tracks, what structure it uses, and how the agent should interact with it. The agent reads from and writes to these files at runtime.

This is intentionally opinionated: the builder designs the files, not the framework. State files are not generic scratch space — they represent deliberate, named storage that the builder has decided this expert needs.

### Format

- Plain markdown
- Optional YAML frontmatter

### State Frontmatter Fields

Optional:

- `name` (string)
- `description` (string)
- `scope` (string): `session` for state that resets each run, `persistent` for state retained across runs. Defaults to `persistent`.

### Example State File

```markdown
---
name: pipeline
description: Running tracker of active deals and risks
scope: persistent
---

## Active Deals

<!-- Agent writes active deal summaries here -->

## Deal Risks

<!-- Agent writes flagged risks, stalled deals, or churn signals here -->

## Recently Closed

<!-- Agent writes recently closed deals here -->
```

### Referencing State Files

Functions and processes that need to read or write state should reference the file path explicitly in their body or frontmatter. The builder decides which files each function or process uses.

Example in a function body:

```markdown
Before evaluating next action, read `./state/pipeline.md` to load current deal context.
After determining next action, update `./state/pipeline.md` with any changes to deal status or risk flags.
```

### Runtime Consumption

Frameworks should:

- Make state files readable and writable by the agent at runtime
- Preserve state file contents across agent calls when `scope` is `persistent`
- Reset state files to their template contents between sessions when `scope` is `session`

The framework should not impose a structure on state files. The builder's template is the contract.

## 11. Scratch (`scratch/`)

The `scratch/` directory holds runtime-generated working files created by process scratchpads. It is not committed to version control and is not listed in `components`.

### Lifecycle

- The `scratch/` directory is auto-created by the framework (or agent) at runtime when a process writes its first scratchpad file.
- Scratch files are ephemeral by default. Frameworks should clean up scratch files after the associated process completes successfully.
- On process failure with retry, scratch files are preserved so the agent can resume from them.
- Frameworks may retain scratch files for a configurable retention period for debugging and auditing.

### Format

- Plain markdown.
- No required frontmatter. The process that creates the file defines its structure.
- File names should follow the pattern declared in the process `scratchpad` field (for example `triage-{message_id}.md`).

### Relationship to State

Scratch files are distinct from state files. State files are builder-designed, named, persistent storage with defined templates. Scratch files are ephemeral, process-scoped working notes that exist only for the duration of a single process invocation and its retries.

## 12. Consumption Model (Framework Authors)

This spec does not require a workflow engine. It defines portable artifacts that frameworks can map into their own agent runtime model.

### Recommended Runtime Flow

1. Parse `expert.yaml`, validate required fields, and load component indexes.
2. Load `orchestrator.md` and persona files into persistent agent context.
3. Register functions and processes as on-demand readable capabilities.
4. Bind abstract tools to concrete integrations and verify `requires.tools` are satisfied.
5. Initialize state templates at runtime locations; reset `scope: session` files between sessions.
6. Apply package-level defaults (`concurrency`, `execution`, `delivery`, and `policy`) to all triggers and processes.
7. Wire manifest triggers by type (`webhook`, `cron`, `channel`) and register their handlers.
8. On trigger invocation, apply effective concurrency policy (`parallel`, `serial`, `serial_per_key`) and queue or start execution.
9. Resolve process inputs using `payload_mapping` (or raw payload fallback), preload any declared `context` files, and run the process checklist.
10. Enforce approval policy for each tool operation (`auto`, `confirm`, `manual`) including approval timeout behavior.
11. On completion, deliver outputs using effective process `delivery` settings and emit escalation notifications when configured.
12. On failure or low confidence, apply `execution.retry`, `resume_from_execution_log`, `on_failure`, and `policy.escalation` rules.

### Portability Rules

- A consumer framework may adapt runtime mechanics.
- A consumer framework should preserve file semantics and intent.
- Packages should remain valid without framework-specific code.

## 13. Validation

Frameworks should validate packages at load time and surface clear errors. The following rules define the minimum validation requirements.

### Required Structure

- `expert.yaml` must exist and contain all required fields (`spec`, `name`, `version`, `description`, `components`).
- `orchestrator.md` must exist at the path declared in `components.orchestrator`.
- At least one persona file must exist at a path listed in `components.persona`.
- At least one function file must exist at a path listed in `components.functions`.

### Cross-Reference Integrity

- Every `triggers[].process` value must match the `name` field of a process listed in `components.processes`. Error if unresolved.
- Every process `trigger` field (if present) must match a `triggers[].name` entry in the manifest. Warn if unresolved.
- Every process `functions[]` entry should match the `name` field of a function listed in `components.functions`. Warn if unresolved.
- Every function or process `tools[]` entry must appear in `requires.tools`. Error if a tool is referenced but not declared as a dependency.
- Every function `knowledge[]` path should match a path listed in `components.knowledge`. Warn if unresolved.
- Every `policy.approval.overrides` key should follow the format `tool_name.operation_name` and reference a tool in `requires.tools` and an operation declared in that tool's YAML file. Warn if unresolved.

### File Existence

- Every path listed in `components` must point to a file that exists in the package directory. Error if missing.

### Severity Levels

- **Error**: the package cannot be loaded. The framework must halt and report the issue.
- **Warn**: the package can be loaded but may have issues at runtime. The framework should log the warning and continue.

Frameworks may implement additional validation beyond these minimums.

## 14. Versioning

### Spec Version

Packages should include a `spec` field in `expert.yaml`.

Current spec version: `1.0`.

### Package Versioning

Package authors should use SemVer for `version`.

Recommended interpretation:

- `MAJOR`: breaking changes to package structure or function/process contracts
- `MINOR`: backward-compatible capability additions
- `PATCH`: backward-compatible fixes and clarifications

## Non-Goals

### Multi-Expert Coordination

Coordination between multiple expert packages (for example, sales handing off to legal review) is not defined by this spec. The recommended pattern is hub-and-spoke routing through the main agent session, which dispatches tasks to expert sessions and synthesizes outputs. Direct expert-to-expert communication is framework-specific and out of scope.

To keep openexperts portable and simple, this specification intentionally does not define:

- Template interpolation syntax (for example `{{step.output}}`)
- A typed executable workflow DSL
- A mandatory execution engine
- Full JSON Schema requirements for tool declarations
