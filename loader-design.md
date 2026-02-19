# OpenExperts → OpenClaw Loader Design

Version: `draft-1`

## Overview

The loader bridges an openexperts package into an OpenClaw sub-agent session. It reads the package, assembles a system prompt, wires up tools, and provides a persistent session the main agent can route tasks to.

## How It Works

```
expert-package/          loader           OpenClaw
─────────────────    ─────────────    ─────────────────
expert.yaml        →  parse manifest
orchestrator.md    →  ┐
persona/*.md       →  ├─ assemble   →  system prompt
                      ┘
functions/*.md     →  skill index   →  on-demand skills
processes/*.md     →  skill index   →  on-demand skills
tools/*.yaml       →  tool binding  →  MCP / tool config
knowledge/*.md     →  context plan  →  preloaded / on-demand
state/*.md         →  state init    →  workspace files
```

## 1. Package Resolution

Expert packages live in a known location:

```
~/.openclaw/experts/
  radiant-sales-expert/
    expert.yaml
    orchestrator.md
    ...
  customer-success-expert/
    expert.yaml
    ...
```

Install is just cloning or copying the directory:

```bash
cd ~/.openclaw/experts
git clone https://github.com/openexperts/radiant-sales-expert
```

The loader discovers packages by scanning `~/.openclaw/experts/*/expert.yaml`.

## 2. System Prompt Assembly

The loader reads the package and assembles a system prompt for the sub-agent session:

```
┌─────────────────────────────────────────────┐
│ SYSTEM PROMPT                               │
│                                             │
│ ## Identity                                 │
│ {persona/identity.md contents}              │
│                                             │
│ ## Rules                                    │
│ {persona/rules.md contents}                 │
│                                             │
│ ## How to Operate                           │
│ {orchestrator.md contents}                  │
│                                             │
│ ## Available Functions                      │
│ (index only — name + description)           │
│ - classify-email-intent: Determine the...   │
│ - determine-next-action: Given a...         │
│ - compose-response: Draft a...              │
│                                             │
│ ## Available Processes                      │
│ (index only — name + description + trigger) │
│ - inbound-email-triage: End-to-end...       │
│                                             │
│ ## Knowledge Available                      │
│ (index only — name + description)           │
│ - meddpicc: MEDDPICC sales methodology     │
│ - competitive-battle-cards: Competitor...   │
│                                             │
│ ## State Files                              │
│ - state/pipeline.md (persistent)            │
│ - state/session-notes.md (session)          │
│                                             │
│ ## Instructions                             │
│ When you need a function, read it from:     │
│   {workspace}/functions/{name}.md           │
│ When you need knowledge, read it from:      │
│   {workspace}/knowledge/{name}.md           │
│ State files are at:                         │
│   {workspace}/state/{name}.md               │
│ Read and write them as instructed by        │
│ functions and processes.                    │
└─────────────────────────────────────────────┘
```

### Why Index-Only for Functions/Knowledge

Loading every function and knowledge file into the system prompt would blow the context window. Instead:

- **System prompt** gets an index (name + description) so the agent knows what's available
- **Full content** is loaded on demand when the agent reads the file during execution
- **Persona + orchestrator** are always fully loaded (they're the agent's identity and routing brain)

This mirrors how OpenClaw skills work — SKILL.md is read when needed, not preloaded.

## 3. Tool Binding

The package declares abstract tools. The user binds them to concrete implementations.

### Binding Config

```yaml
# ~/.openclaw/experts/radiant-sales-expert/bindings.yaml
# (user-created, not part of the package)

tools:
  crm:
    type: mcp
    server: hubspot-mcp
    # maps to an MCP server configured in OpenClaw
  email:
    type: mcp
    server: nylas-mcp
  calendar:
    type: mcp
    server: google-calendar-mcp
```

### Operation Mapping

The abstract tool spec declares operations like `crm.get_contact`. The binding config can optionally map these to specific MCP tool names if they don't match 1:1:

```yaml
tools:
  crm:
    type: mcp
    server: hubspot-mcp
    operations:
      get_contact: hubspot_get_contact_by_email
      get_deal: hubspot_get_deals_for_contact
```

If no operation mapping is provided, the loader assumes the MCP server exposes tools matching the operation names.

### Validation

On load, the loader checks:
- Every tool declared in `requires.tools` has a binding
- Every bound MCP server is configured and reachable
- Warns (not errors) if operation names don't match

## 4. State Management

State files need to survive across sub-agent sessions.

### Directory Layout

```
~/.openclaw/experts/radiant-sales-expert/
  state/                    ← package templates (read-only reference)
    pipeline.md
    session-notes.md

~/.openclaw/workspace/expert-state/radiant-sales-expert/
  state/                    ← runtime state (agent reads/writes here)
    pipeline.md
    session-notes.md
```

### Lifecycle

1. **First run:** Loader copies template state files from the package to the runtime location.
2. **Each session spawn:**
   - `scope: persistent` files are left as-is (carry forward)
   - `scope: session` files are reset to template contents
3. **Agent reads/writes** state files at the runtime location during execution.

### Scratchpad

Process scratchpad files (`./scratch/triage-{id}.md`) are created in the runtime workspace. Old scratchpad files can be cleaned up periodically or by the agent.

## 5. Sub-Agent Session Config

The loader generates a session configuration:

```yaml
# generated — not a real OpenClaw config file, 
# but represents what the loader assembles

session:
  label: "expert:radiant-sales-expert"
  agentId: "radiant-sales-expert"
  
  systemPrompt: |
    {assembled from step 2}
  
  workspace: ~/.openclaw/workspace/expert-state/radiant-sales-expert/
  
  tools:
    - mcp: hubspot-mcp
    - mcp: nylas-mcp
    - read   # for loading functions/knowledge/state on demand
    - write  # for updating state files
    - edit   # for surgical state updates
  
  model: "anthropic/claude-sonnet-4-20250514"  # configurable per expert
  
  persistent: true  # keep session alive between tasks
```

## 6. Main Agent Integration

### Expert Registry

The main agent (me) needs to know what experts are available. The loader writes a registry file:

```markdown
<!-- ~/.openclaw/workspace/EXPERTS.md (auto-generated) -->

# Available Experts

## radiant-sales-expert
- **Description:** B2B sales expertise for inbound deal triage and next-best-action
- **Session:** expert:radiant-sales-expert
- **Triggers:** new_email (inbound prospect emails)
- **Capabilities:** email classification, next-action determination, response composition
- **Tools bound:** crm → hubspot-mcp, email → nylas-mcp

## customer-success-expert
- ...
```

This gets loaded into my context so I know what I can delegate.

### Routing

When I receive a task, I check EXPERTS.md and route:

```
User: "Handle that email from the Acme lead"
  ↓
Main agent reads EXPERTS.md
  ↓
Matches: radiant-sales-expert (email classification, response composition)
  ↓
sessions_send(label="expert:radiant-sales-expert", message="...")
  ↓
Expert sub-agent runs inbound-email-triage process
  ↓
Returns result to main agent
  ↓
Main agent relays to user
```

### Trigger Automation

For automatic triggers (not just user-initiated), cron jobs handle it:

```yaml
# Example: check for new prospect emails every 15 minutes
name: sales-ae-inbox-check
schedule:
  kind: cron
  expr: "*/15 * * * *"
  tz: Australia/Sydney
payload:
  kind: agentTurn
  message: >
    Check for new inbound emails using the email tool.
    For each new email from a prospect or lead, run the
    inbound-email-triage process. Report results.
sessionTarget: isolated
delivery:
  mode: announce
```

## 7. CLI Commands (Proposed)

```bash
# Install a package
openclaw expert install https://github.com/openexperts/radiant-sales-expert

# List installed experts
openclaw expert list

# Configure tool bindings
openclaw expert bind radiant-sales-expert crm hubspot-mcp

# Validate a package (check bindings, tool availability)
openclaw expert validate radiant-sales-expert

# Remove a package
openclaw expert remove radiant-sales-expert

# Spawn an expert session manually
openclaw expert run radiant-sales-expert "Triage this email: ..."
```

## 8. Full Flow Example

### Setup (one-time)

```bash
# Install the expert package
openclaw expert install https://github.com/openexperts/radiant-sales-expert

# Bind tools (assumes MCP servers already configured)
openclaw expert bind radiant-sales-expert crm hubspot-mcp
openclaw expert bind radiant-sales-expert email nylas-mcp

# Validate
openclaw expert validate radiant-sales-expert
# ✓ Package valid
# ✓ All required tools bound
# ✓ MCP servers reachable
# ✓ State files initialized
```

### Runtime (automatic)

```
[Cron: every 15 min] → Sales AE sub-agent checks inbox
  → New email from sarah@acme.com
  → Reads functions/classify-email-intent.md
  → Calls crm.get_contact(email="sarah@acme.com")
  → Calls crm.get_deal(contact_id="...")
  → Classifies: buying_signal, high urgency
  → Reads functions/determine-next-action.md
  → Recommends: schedule demo, multi-thread with champion
  → Reads functions/compose-response.md
  → Drafts response
  → Updates state/pipeline.md with deal movement
  → Returns to main agent:
    "🔥 Buying signal from Sarah at Acme (Stage 3, $45k).
     She's asking about enterprise pricing — looks ready to move.
     Draft reply attached. Recommend scheduling demo this week
     and looping in their VP Engineering (your champion)."
  → Main agent sends to Dylan via chat
```

### Runtime (user-initiated)

```
Dylan: "What's my pipeline looking like?"
  → Main agent routes to Sales AE
  → Sales AE reads state/pipeline.md
  → Returns summary
  → Main agent relays to Dylan
```

## 9. Open Questions

1. **Persistent sessions vs. re-spawn:** Should expert sub-agents stay alive between tasks (lower latency, maintains conversation context) or spin up fresh each time (cleaner, cheaper)? Probably configurable per expert.

2. **Multi-expert coordination:** If a task spans two experts (e.g., closing a deal involves Sales AE + Legal review), who orchestrates? The main agent seems right, but we might need a convention for it.

3. **Expert-to-expert communication:** Can the Sales AE sub-agent directly call the CS expert? Or must everything go through the main agent? Starting with hub-and-spoke (main agent as router) is simpler.

4. **Context from main agent:** When the main agent routes a task, how much context does it pass? Just the task, or also relevant user context (timezone, preferences, recent conversation)? Probably a slim context envelope.

5. **State conflicts:** If a cron-triggered run and a user-initiated run both hit the same state file simultaneously, what happens? File locking? Queue tasks per expert?

6. **Package updates:** When the package author pushes a new version, how does that flow? `openclaw expert update`? Does it preserve state files and bindings?

7. **Secrets/credentials:** Tool bindings may need auth. This should defer to OpenClaw's existing MCP server config rather than putting credentials in the expert package or bindings file.
