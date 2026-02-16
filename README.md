# openexperience

**Professional expertise for AI agents.**

openexperience is an open specification for packaging professional expertise -- the judgment, processes, and domain knowledge that professionals use daily -- into a format any AI agent framework can consume.

## The Problem

LLMs have knowledge, but not experience. They can write an email, but they don't know when to send it, who to CC, whether to use a case study or a pricing sheet, or that this prospect went quiet after the demo and needs a breakup sequence. That judgment comes from years of professional practice.

openexperience bridges that gap. It defines a standard way to capture professional expertise as portable, framework-agnostic packages that give AI agents the judgment of a seasoned professional.

## What's in an Experience Package?

An experience package is a directory of markdown and YAML files:

| Component | Format | Purpose |
|---|---|---|
| **Manifest** | `experience.yaml` | Package metadata, dependencies, component listing |
| **Functions** | `functions/*.md` | Individual capabilities (markdown instructions + YAML frontmatter) |
| **Processes** | `processes/*.yaml` | Multi-step workflows that chain functions together |
| **Actions** | `actions/*.yaml` | What the agent can DO (the output vocabulary) |
| **Tools** | `tools/*.yaml` | Abstract interfaces to external systems |
| **Schemas** | `schemas/*.yaml` | Data contracts the host must provide |
| **Knowledge** | `knowledge/*.md` | Reference material (methodologies, frameworks, guidelines) |
| **Persona** | `persona/` | Agent identity and behavioral rules |

Only `experience.yaml`, `functions/`, `persona/`, and `README.md` are required. Everything else is opt-in based on what the expertise demands.

## How It Works

1. **Browse** experience packages for your domain
2. **Install** by cloning or downloading the package into your project
3. **Wire up** the abstract tool interfaces to your concrete implementations (your CRM, your ticketing system, your codebase)
4. **Your agent** now has the judgment of a seasoned professional

The format is framework-agnostic. Any agent framework can consume it:
- Read the markdown directly as system prompts
- Map functions to agent definitions
- Map processes to workflow chains
- Map tools to your integration layer

## Specification

The full specification is in [spec.md](spec.md). It covers every component type, their YAML schemas, and required vs. optional fields.

## Example Packages

- [radiant-sales-expert](https://github.com/openexperience/radiant-sales-expert) -- B2B sales expertise: deal intelligence, next-best-action, content composition, MEDDPICC framework

## Contributing

openexperience is open source. To create your own experience package:

1. Read the [specification](spec.md)
2. Create a new repo following the directory structure
3. Package your professional expertise as functions, processes, and knowledge
4. Open a PR to add your package to the directory above

## License

MIT
