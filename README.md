# Evan-Lausier_Claude-Skills-Library

A curated library of Claude AI skill definitions — reusable prompt modules that extend Claude's capabilities for specific task categories. Skills are designed to be activated by trigger phrases in conversation and drive higher-quality, more consistent outputs.

## What Is a Skill?

A skill is a structured prompt module that instructs Claude to follow a specific methodology, use particular tools, or produce output in a defined format. Skills are referenced in a Claude project's system prompt or instructions and loaded on demand. Each skill lives in its own folder with a `SKILL.md` containing the full instructions and a `README.md` with a human-readable summary.

## Structure

```
Evan-Lausier_Claude-Skills-Library/
├── README.md                     ← this file
├── Document_Creation/            ← Word, PDF, PowerPoint, Excel
├── Data_Visualization/           ← Charts, diagrams, interactive widgets
├── File_Processing/              ← Reading and extracting from uploaded files
├── Web_Artifacts/                ← React, HTML, frontend UI components
├── Creative/                     ← Canvas art, algorithmic visuals, Slack GIFs
├── Communication/                ← Internal comms, doc co-authoring
└── Meta/                         ← Skill creation, MCP building, theming
```

## Skill Index

| Skill | Category | Trigger Phrases | Description |
|-------|----------|-----------------|-------------|
| [Word (docx)](./Document_Creation) | Document Creation | "Word doc", "docx", "report", "memo", "letter" | High-quality .docx generation with formatting, headers, and tables |
| [PDF](./Document_Creation) | Document Creation | "PDF", "create a PDF", "fill a form", "merge PDFs" | PDF creation, extraction, merging, and form-filling |
| [PowerPoint (pptx)](./Document_Creation) | Document Creation | "deck", "slides", "presentation", ".pptx" | Professional slide deck generation |
| [Excel (xlsx)](./Document_Creation) | Document Creation | "spreadsheet", ".xlsx", "Excel" | Structured spreadsheet creation and data formatting |
| [Chart](./Data_Visualization) | Data Visualization | "chart", "graph", "bar chart", "line chart" | Inline SVG/HTML charts from structured data |
| [Diagram](./Data_Visualization) | Data Visualization | "diagram", "flowchart", "architecture", "process map" | Flow diagrams, system architecture, process maps |
| [Interactive](./Data_Visualization) | Data Visualization | "interactive", "calculator", "simulator", "widget" | Interactive HTML/React tools and dashboards |
| [General File Reading](./File_Processing) | File Processing | Uploaded file path present | Routes to correct reading strategy by file type |
| [PDF Reading](./File_Processing) | File Processing | Uploaded .pdf file | Text extraction, OCR, table extraction from PDFs |
| [Frontend Design](./Web_Artifacts) | Web Artifacts | "build a page", "landing page", "UI", "component" | Production-grade React/HTML interfaces |
| [Web Artifacts Builder](./Web_Artifacts) | Web Artifacts | "multi-page app", "complex artifact", "shadcn" | Multi-component Claude.ai artifacts with routing |
| [Canvas Design](./Creative) | Creative | "poster", "art", "design", "illustration" | Original visual designs as .png or .pdf |
| [Algorithmic Art](./Creative) | Creative | "generative art", "flow field", "p5.js" | p5.js algorithmic art with seeded randomness |
| [Slack GIF](./Creative) | Creative | "GIF for Slack", "make me a GIF" | Optimized animated GIFs for Slack |
| [Internal Comms](./Communication) | Communication | "status report", "leadership update", "newsletter" | Structured internal communications |
| [Doc Co-authoring](./Communication) | Communication | "help me write a doc", "technical spec", "proposal" | Structured workflow for co-authoring documentation |
| [Skill Creator](./Meta) | Meta | "create a skill", "edit a skill", "benchmark a skill" | Create, modify, and evaluate skill definitions |
| [MCP Builder](./Meta) | Meta | "build an MCP server", "MCP integration" | Guide for creating high-quality MCP servers |
| [Theme Factory](./Meta) | Meta | "apply a theme", "style this artifact" | Apply preset or custom visual themes to artifacts |
| [Brand Guidelines](./Meta) | Meta | "Anthropic branding", "brand colors" | Apply Anthropic's official brand and typography |

## How to Use Skills

Reference the relevant SKILL.md path in your Claude project instructions:

```
Available skills:
- For Word documents: [Document_Creation/SKILL.md path]
- For uploaded files: [File_Processing/SKILL.md path]
- For diagrams: [Data_Visualization/SKILL.md path]
```

Claude will read the SKILL.md before responding to any matching request.

## Author

Evan Lausier — [Evan-Lausier-AI-Labs](https://github.com/Evan-Lausier-AI-Labs)
