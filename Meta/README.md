# Meta Skills

Skills about skills — tools for creating new skill definitions, building MCP servers, applying visual themes, and configuring Claude's brand presentation.

## Skills in This Category

| Skill | Trigger Phrases |
|-------|-----------------|
| [Skill Creator](#skill-creator) | "create a skill", "edit a skill", "test a skill", "benchmark a skill", "optimize a skill trigger" |
| [MCP Builder](#mcp-builder) | "build an MCP server", "MCP integration", "create an MCP tool", "FastMCP" |
| [Theme Factory](#theme-factory) | "apply a theme", "style this artifact", "make it look like X", "dark mode", "brand it" |
| [Brand Guidelines](#brand-guidelines) | "Anthropic branding", "brand colors", "official style", "Anthropic design" |

---

## Skill Creator

Guides the full lifecycle of a Claude skill: creation from scratch, modification of existing skills, evaluation against test cases, benchmarking output quality variance, and optimizing trigger descriptions for accurate activation.

**Workflow:**
1. Define the skill purpose and the problem it solves
2. Identify trigger conditions (true positives) and non-trigger conditions (true negatives)
3. Write the `SKILL.md` with instructions, constraints, examples, and edge cases
4. Write a concise trigger description for the skills registry (used to decide when to activate)
5. Run evals — verify the skill fires on true positives and stays quiet on negatives
6. Measure output quality variance across multiple runs
7. Iterate until the skill is stable and consistent

**Key insight:** The trigger description is as important as the SKILL.md itself. A poorly written trigger causes both false positives (skill fires when it shouldn't) and false negatives (skill doesn't fire when it should).

---

## MCP Builder

A guide for creating high-quality MCP (Model Context Protocol) servers that expose external APIs and services as Claude tools. Covers both Python (FastMCP) and Node/TypeScript (MCP SDK).

**Tool design principles:**
- Names should be action-oriented verbs: `get_customer`, `create_invoice`, `list_transactions`
- Descriptions must be complete — Claude reads them to decide whether to call the tool
- Always validate and sanitize inputs before making API calls
- Return structured JSON that Claude can reason about
- Handle rate limits, auth errors, and timeouts gracefully with informative messages
- Never expose credentials in tool responses

**Python (FastMCP) vs TypeScript (MCP SDK):**
- FastMCP: preferred for Python-native APIs, data processing, and rapid prototyping
- TypeScript SDK: preferred for Node.js ecosystems, browser-adjacent tools, and npm packages

---

## Theme Factory

Applies one of 10 preset visual themes to any artifact (slides, docs, reports, HTML pages), or generates a custom theme on the fly based on a description.

**Preset themes:** Corporate, Minimal, Dark Mode, Warm, High Contrast, Pastel, Tech, Editorial, Retro, Nature.

**Custom theme generation:** Describe a mood, brand, or reference ("like Notion", "Wall Street Journal editorial", "cyberpunk terminal") and the skill generates a matching color palette and typography pairing.

**When to use:** After creating an artifact when the user wants a specific look, or when starting a visual project where consistency across multiple outputs matters.

---

## Brand Guidelines

Applies Anthropic's official brand colors, typography, and visual style to any artifact. Use when producing content that represents Anthropic or needs to match Anthropic's design system.

**Anthropic brand tokens:**

| Token | Value |
|-------|-------|
| Primary (Coral) | `#D4603A` |
| Dark | `#1A1A1A` |
| Light | `#F5F0E8` |
| Accent | `#C8A882` |
| Heading typeface | Styrene A |
| Body typeface | Söhne |
| Fallback stack | `system-ui, -apple-system, sans-serif` |

**Do NOT use for:** Personal projects, client deliverables, or anything not explicitly representing Anthropic.
