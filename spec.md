# openexperience Specification v0.1

## Overview

An **experience package** is a directory of markdown and YAML files that encodes professional expertise in a framework-agnostic format. Any AI agent framework can consume it -- by reading the markdown directly as prompts, mapping functions to agent definitions, or mapping processes to workflow chains.

This document defines the structure, file formats, and schemas that all experience packages must follow.

## Design Principles

- **Framework-agnostic**: No code. Markdown instructions, YAML definitions. Any framework can consume it.
- **Human-readable**: A person can read the package and understand the expertise without any tooling.
- **Ports and adapters**: The experience declares what it needs (tools, data shapes). The host project provides concrete implementations.
- **Progressive complexity**: Only a few components are required. Everything else is opt-in.

## Package Structure

```
<package-name>/
  experience.yaml          # REQUIRED -- package manifest
  README.md                # REQUIRED -- human overview and installation guide
  persona/                 # REQUIRED -- agent identity and behavior
    identity.md            #   Who the agent is
    instructions.md        #   Behavioral rules
  functions/               # REQUIRED -- individual capabilities
    <name>.md              #   One file per function
  processes/               # OPTIONAL -- multi-step workflows
    <name>.yaml            #   One file per process
  actions/                 # OPTIONAL -- output vocabulary
    <name>.yaml            #   One file per action type
  tools/                   # OPTIONAL -- external integration interfaces
    <name>.yaml            #   One file per tool
  schemas/                 # OPTIONAL -- data contracts
    <name>.yaml            #   One file per schema
  knowledge/               # OPTIONAL -- reference material
    <name>.md              #   One file per topic
  integrations/            # OPTIONAL -- framework-specific install guides
    <framework>.md         #   One file per framework
```

---

## 1. Manifest (`experience.yaml`)

**Required.** The root manifest declares package metadata and what the package contains.

### Schema

```yaml
# -- Required fields --
name: string                    # Package identifier (kebab-case, e.g. "radiant-sales-expert")
version: string                 # Semantic version (e.g. "0.1.0")
description: string             # One-line description of what this experience provides
author: string                  # Author name or organization

# -- Optional fields --
license: string                 # License identifier (e.g. "MIT", "Apache-2.0")
category: string                # Domain category (e.g. "sales", "support", "engineering", "marketing", "legal")
tags: string[]                  # Searchable tags (e.g. ["b2b", "crm", "meddpicc"])
repository: string              # URL to the package repository
homepage: string                # URL to the package homepage

# -- Dependency declarations --
requires:
  tools: string[]               # Abstract tool names this package needs (e.g. ["crm", "email", "calendar"])
  schemas: string[]             # Data shapes the host must provide (e.g. ["contact", "opportunity", "activity"])

# -- Component listing --
components:
  functions: string[]           # List of function names (matching filenames without .md)
  processes: string[]           # List of process names (matching filenames without .yaml)
  actions: string[]             # List of action type names (matching filenames without .yaml)
  tools: string[]               # List of tool interface names (matching filenames without .yaml)
  schemas: string[]             # List of schema names (matching filenames without .yaml)
  knowledge: string[]           # List of knowledge file names (matching filenames without .md)
```

### Example

```yaml
name: radiant-sales-expert
version: 0.1.0
description: B2B sales expertise -- deal intelligence, next-best-action, content composition
author: Radiant AI
license: MIT
category: sales
tags: [b2b, crm, meddpicc, sales-intelligence]
repository: https://github.com/openexperience/radiant-sales-expert

requires:
  tools: [crm, email, calendar, playbooks]
  schemas: [contact, opportunity, activity, pipeline-stage]

components:
  functions:
    - summarise-activity
    - contact-intelligence
    - deal-intelligence
    - next-best-action
    - evaluate-actions
    - compose-content
    - playbook-selection
  processes:
    - activity-pipeline
    - next-best-action
    - action-reevaluation
    - content-composition
  actions:
    - email
    - call
    - meeting
    - task
    - linkedin-message
    - lookup
    - add-contact
    - update-pipeline-stage
    - no-action
  tools:
    - crm
    - email
    - calendar
    - playbooks
  schemas:
    - contact
    - opportunity
    - activity
    - pipeline-stage
  knowledge:
    - sales-methodologies
    - objection-handling
    - sales-process
    - email-composition
```

---

## 2. Persona (`persona/`)

**Required.** Defines the agent's identity and behavioral rules. Contains two files.

### `persona/identity.md`

Free-form markdown describing who the agent is. This typically becomes the opening section of a system prompt.

Should include:
- Professional background and expertise level
- Areas of specialization
- Communication voice and tone
- Referenced methodologies or frameworks

### `persona/instructions.md`

Free-form markdown defining behavioral rules. This typically becomes the rules/constraints section of a system prompt.

Should include:
- What the agent should always do
- What the agent should never do
- Decision-making principles
- Quality standards and output expectations

### Example

**`persona/identity.md`:**
```markdown
You are a senior B2B sales executive with over two decades of experience closing 
enterprise deals. You are a master of the MEDDPICC framework and draw on methodologies 
from Founding Sales, The Challenger Sale, and The Sales Acceleration Formula.

You communicate with confidence but without arrogance. You are direct, data-driven, 
and always focused on advancing the deal.
```

**`persona/instructions.md`:**
```markdown
## Core Rules

- Always base recommendations on observable evidence from activities and communications
- Never fabricate information about contacts, companies, or deal history
- When uncertain, say so explicitly rather than guessing
- Prioritize actions that advance the deal over actions that maintain the status quo

## Output Standards

- Return empty arrays when no data exists -- never fill with placeholders
- Keep reasoning concise (10-500 characters) but specific
- Always cite the source activity that informed a recommendation
```

---

## 3. Functions (`functions/`)

**Required.** Each function is a single capability the agent can perform, defined as a markdown file with YAML frontmatter.

### Frontmatter Schema

```yaml
---
# -- Required fields --
name: string                    # Function identifier (matches filename without .md)
description: string             # What this function does (one line)

# -- Optional fields --
model: string                   # Recommended model (e.g. "gpt-5", "claude-4")
reasoning: string               # Recommended reasoning effort (e.g. "high", "medium", "low")
input: string                   # Free-text description of expected input
output: string                  # Free-text description of expected output
tools: string[]                 # Abstract tool names this function uses (subset of manifest requires.tools)
requires:
  schemas: string[]             # Data shapes this function expects (subset of manifest requires.schemas)
  knowledge: string[]           # Knowledge files this function references (by name, without .md)
  functions: string[]           # Other functions this function may call (by name)
---
```

### Body

The markdown body contains the actual instructions -- the expertise. This is what becomes the system prompt or agent instructions when consumed by a framework.

The body should:
- Be written as direct instructions to the agent (second person: "You should...", "Analyze the...")
- Be cleaned of framework-specific references (no imports, no TypeScript, no Zod)
- Replace product-specific references with generic placeholders (e.g., "your product/service")
- Preserve the full depth of the original expertise -- do not summarize

### Example

```markdown
---
name: summarise-activity
description: Summarize CRM activities into structured, actionable intelligence
model: gpt-5
reasoning: high
input: A raw CRM activity (email, call transcript, meeting notes)
output: Structured summary with relevance score, key insights, MEDDPICC extractions, and action items
tools: [crm]
requires:
  schemas: [activity, contact]
  knowledge: [sales-methodologies]
---

# Summarise Activity

You are analyzing a CRM activity to extract actionable sales intelligence.

## Actor Attribution Rules

Clearly distinguish between:
- **Seller offered**: "We offered to send an executive summary"
- **Prospect requested**: "They requested an executive summary as a prerequisite"  
- **Prospect accepted**: "They accepted our offer to review the materials"

Do NOT conflate these. If the seller offered something and the prospect said "yes", 
that is ACCEPTANCE, not a REQUIREMENT.

## Relevance Filtering

For EVERY mentioned tool, product, or company, determine:
1. Is this about a problem your product/service solves?
2. Is this a requirement for purchasing your product?
3. Is this a competitor or alternative?
4. Or is this just casual conversation?

...
```

---

## 4. Processes (`processes/`)

**Optional.** YAML files defining multi-step workflows that chain functions together.

### Schema

```yaml
# -- Required fields --
name: string                    # Process identifier (matches filename without .yaml)
description: string             # What this process does

# -- Optional fields --
trigger: string                 # Event that starts this process (e.g. "activity.created", "ticket.opened")

# -- Steps (at least one required) --
steps:
  - id: string                  # Unique step identifier
    function: string            # Function to call (by name)
    input: object               # Input mapping -- values can reference:
                                #   $trigger.<field>     -- data from the trigger event
                                #   $context.<field>     -- data from the host context
                                #   $steps.<id>.output   -- output from a previous step
    output: string              # Named output key for downstream steps (optional)
    condition: string           # When to execute (simple boolean expression, optional)
    approval: string            # "required" or "optional" (optional)

# -- Branching (optional) --
branches:
  - condition: string           # Expression to evaluate
    steps: []                   # Steps to execute if condition is true (same schema as above)

# -- Parallelism (optional) --
parallel:
  - group: string               # Group identifier
    steps: []                   # Steps that can run concurrently (same schema as above)
```

### Input Mapping

Steps receive input through a mapping object. Values can reference:

| Reference | Meaning |
|---|---|
| `$trigger.<field>` | Data from the event that started the process |
| `$context.<field>` | Data from the host project's context (e.g., current user, organization) |
| `$steps.<id>.output` | Output from a previously completed step |
| Literal values | Static strings, numbers, booleans |

### Example

```yaml
name: activity-pipeline
description: Process new CRM activities into intelligence and actionable next steps
trigger: activity.created

steps:
  - id: summarize
    function: summarise-activity
    input:
      activity: $trigger.activity

  - id: deal-intel
    function: deal-intelligence
    input:
      opportunity: $context.opportunity
      contacts: $context.contacts
      activities: $context.recentActivities

parallel:
  - group: contact-intelligence
    steps:
      - id: contact-intel
        function: contact-intelligence
        input:
          contact: $trigger.contact
          activities: $context.recentActivities

branches:
  - condition: $context.existingActions.length > 0
    steps:
      - id: reevaluate
        function: evaluate-actions
        input:
          context: $steps.deal-intel.output
          existingActions: $context.existingActions
        approval: required

  - condition: $context.existingActions.length == 0
    steps:
      - id: decide
        function: next-best-action
        input:
          context: $steps.deal-intel.output
        approval: required
```

---

## 5. Actions (`actions/`)

**Optional.** YAML files defining what the agent can recommend or do. This is the **output vocabulary** -- the set of concrete things the agent can propose.

Not all experiences need actions. An analytics expert might only produce insights. A sales expert might have 9 action types. The spec supports both.

### Schema

```yaml
# -- Required fields --
name: string                    # Action type identifier (matches filename without .yaml)
description: string             # When this action should be used

# -- Required: detail schema --
schema:
  fields:
    - name: string              # Field name
      type: string              # Type: string, number, boolean, date, datetime, email, enum, array, object
      required: boolean         # Whether this field must be provided
      description: string       # What this field is for
      constraints:              # Optional validation constraints
        min: number             #   Minimum value or length
        max: number             #   Maximum value or length
        pattern: string         #   Regex pattern
        values: string[]        #   Enum values (for type: enum)
        items: object           #   Item schema (for type: array)

# -- Optional: content composition --
requires_composition: boolean   # Whether this action needs AI-generated content (default: false)
content_strategy: string        # Markdown instructions for how to compose this action's content
                                # Only relevant when requires_composition is true

composed_fields:                # Fields that are AI-generated (subset of schema.fields)
  - name: string                # Field name (must match a field in schema.fields)
    description: string         # What the composed content should contain
    constraints:                # Output constraints
      min_length: number
      max_length: number
      format: string            # e.g. "html", "markdown", "plain"
```

### Example

```yaml
name: email
description: Send an email to one or more contacts. Use for outreach, follow-ups, replies, and scheduled communications.

schema:
  fields:
    - name: to
      type: array
      required: true
      description: Recipient email addresses
      constraints:
        min: 1
        items: { type: email }
    - name: cc
      type: array
      required: false
      description: CC email addresses
      constraints:
        items: { type: email }
    - name: subject
      type: string
      required: false
      description: Email subject line (composed by content agent)
      constraints:
        max: 200
    - name: body
      type: string
      required: false
      description: Email body (composed by content agent)
      constraints:
        max: 5000
        format: html
    - name: scheduled_for
      type: datetime
      required: true
      description: When to send the email (ISO 8601)
    - name: reply_to_message_id
      type: string
      required: false
      description: Message ID to reply to (for threading)
    - name: priority
      type: enum
      required: false
      description: Email priority
      constraints:
        values: [low, normal, high]

requires_composition: true
content_strategy: |
  ## Email Composition Guidelines

  Write as a B2B sales professional. Tone: friendly but professional, 
  low-friction replies, avoid sales jargon.

  ### Rules
  - Single CTA per email
  - No signature block (sign-off only, e.g., "Best, [Name]")
  - If replying to a thread, do not re-state context already discussed
  - If new outreach, include a personalization anchor in the first 1-2 sentences

  ### Personalization Anchor
  The opening must connect something specific about the prospect's situation 
  to the reason for the email. Do not just name-drop a factoid.

composed_fields:
  - name: subject
    description: Concise, relevant subject line
    constraints:
      min_length: 1
      max_length: 200
  - name: body
    description: Professional email body in HTML
    constraints:
      min_length: 10
      max_length: 5000
      format: html
```

---

## 6. Tools (`tools/`)

**Optional.** YAML files declaring abstract integration interfaces. The experience says "I need a tool that can do X" -- the host project provides the concrete implementation.

This is the ports-and-adapters pattern: the experience knows *what* to do and *when*. The host knows *how* and *where*.

### Schema

```yaml
# -- Required fields --
name: string                    # Tool identifier (matches filename without .yaml)
description: string             # What this tool provides access to

# -- Required: operations --
operations:
  - name: string                # Operation identifier (e.g. "fetch_contacts")
    description: string         # What this operation does
    input:                      # Named parameters
      <param_name>:
        type: string            # Type: string, number, boolean, object, array
        required: boolean
        description: string
    output:                     # Return shape
      <field_name>:
        type: string
        description: string
```

### Example

```yaml
name: crm
description: Access to CRM data -- contacts, deals/opportunities, and activities

operations:
  - name: fetch_contacts
    description: Get contacts associated with a deal
    input:
      opportunity_id:
        type: string
        required: true
        description: The deal/opportunity identifier
    output:
      contacts:
        type: array
        description: List of contacts with name, email, role, company, and last activity date

  - name: fetch_opportunity
    description: Get deal/opportunity details
    input:
      opportunity_id:
        type: string
        required: true
        description: The deal/opportunity identifier
    output:
      opportunity:
        type: object
        description: Deal details including name, stage, amount, close date, and custom fields

  - name: fetch_activities
    description: Get recent activities for a contact or deal
    input:
      contact_id:
        type: string
        required: false
        description: Filter by contact
      opportunity_id:
        type: string
        required: false
        description: Filter by deal
      limit:
        type: number
        required: false
        description: Maximum number of activities to return
    output:
      activities:
        type: array
        description: List of activities with type, subject, body, date, participants, and direction
```

---

## 7. Schemas (`schemas/`)

**Optional.** YAML files defining the data shapes the experience expects the host project to provide. This is the integration boundary -- it decouples the experience from any specific data source.

### Schema

```yaml
# -- Required fields --
name: string                    # Schema identifier (matches filename without .yaml)
description: string             # What this data represents

# -- Required: fields --
fields:
  - name: string                # Field name
    type: string                # Type: string, number, boolean, date, datetime, object, array
    required: boolean           # Whether the host must provide this field
    description: string         # What this field represents
    fields: []                  # Nested fields (for type: object)
    items: object               # Item schema (for type: array)
```

### Example

```yaml
name: contact
description: A person involved in a deal or interaction

fields:
  - name: id
    type: string
    required: true
    description: Unique identifier for the contact
  - name: name
    type: string
    required: true
    description: Full name
  - name: email
    type: string
    required: true
    description: Primary email address
  - name: role
    type: string
    required: false
    description: Job title or role in the organization
  - name: company
    type: string
    required: false
    description: Company or organization name
  - name: last_activity
    type: datetime
    required: false
    description: Timestamp of most recent interaction
```

---

## 8. Knowledge (`knowledge/`)

**Optional.** Pure markdown reference documents containing domain expertise, frameworks, methodologies, and guidelines that functions reference.

### Conventions

- One file per topic
- Filename should be descriptive and kebab-case (e.g., `sales-methodologies.md`, `escalation-policies.md`)
- No YAML frontmatter required (but may be added for metadata if desired)
- Content should be well-structured with headers, lists, and examples
- Functions reference knowledge files by name (without `.md`) in their frontmatter `requires.knowledge` field

### Example

A `sales-methodologies.md` file might cover MEDDPICC, Challenger Sale, and SPIN selling frameworks with detailed explanations of when and how to apply each.

---

## 9. Integrations (`integrations/`)

**Optional.** Markdown files with framework-specific installation and configuration guides. One file per supported framework.

These are not part of the formal spec -- they're documentation that helps users install the experience into their specific setup.

### Conventions

- Filename matches the framework name (e.g., `openclaw.md`, `mastra.md`, `langchain.md`)
- Should include:
  - How to map functions to the framework's agent/skill definitions
  - How to map processes to the framework's workflow system
  - How to wire up abstract tool interfaces to concrete implementations
  - Any framework-specific configuration

---

## Validation Rules

A valid experience package must satisfy:

1. **`experience.yaml` exists** and contains all required fields (`name`, `version`, `description`, `author`)
2. **`README.md` exists** and is non-empty
3. **`persona/identity.md` exists** and is non-empty
4. **`persona/instructions.md` exists** and is non-empty
5. **`functions/` contains at least one `.md` file**, and each file has valid frontmatter with `name` and `description`
6. **Every function, process, action, tool, schema, and knowledge file** listed in `experience.yaml` `components` has a corresponding file in the correct directory
7. **Every tool referenced** in function frontmatter `tools` fields is listed in `experience.yaml` `requires.tools`
8. **Every schema referenced** in function frontmatter `requires.schemas` fields is listed in `experience.yaml` `requires.schemas`
9. **Every knowledge file referenced** in function frontmatter `requires.knowledge` fields exists in `knowledge/`
10. **Every function referenced** in process step `function` fields exists in `functions/`

---

## File Naming Conventions

- All filenames use **kebab-case** (e.g., `next-best-action.md`, `activity-pipeline.yaml`)
- Function files use `.md` extension
- Process, action, tool, and schema files use `.yaml` extension
- Knowledge files use `.md` extension
- Integration files use `.md` extension

---

## Type Reference

The following types are available for use in schema fields, action fields, and tool parameters:

| Type | Description |
|---|---|
| `string` | Text value |
| `number` | Numeric value (integer or float) |
| `boolean` | True or false |
| `date` | Date without time (YYYY-MM-DD) |
| `datetime` | Date with time (ISO 8601) |
| `email` | Email address |
| `enum` | One of a fixed set of values (specify in `constraints.values`) |
| `array` | Ordered list (specify item type in `constraints.items` or `items`) |
| `object` | Nested structure (specify fields in `fields`) |

---

## Versioning

Experience packages follow [Semantic Versioning](https://semver.org/):

- **MAJOR**: Breaking changes to schemas, tool interfaces, or function contracts
- **MINOR**: New functions, processes, actions, or knowledge files (backward compatible)
- **PATCH**: Improvements to instructions, knowledge content, or bug fixes

The spec itself is versioned separately. This document is **v0.1**.
