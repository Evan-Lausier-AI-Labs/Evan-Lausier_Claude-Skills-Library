# Web Artifacts Skills

Skills for building frontend interfaces that render in the Claude chat interface or as standalone files. These skills drive design quality above the generic Claude default.

## Skills in This Category

| Skill | Output | Trigger Phrases |
|-------|--------|-----------------|
| [Frontend Design](#frontend-design) | JSX / HTML artifact | "build a page", "landing page", "UI", "component", "website", "dashboard" |
| [Web Artifacts Builder](#web-artifacts-builder) | Multi-view React app | "multi-page app", "complex artifact", "shadcn", "routing between views" |

---

## Frontend Design

Creates distinctive, production-grade frontend interfaces with intentional design quality. Reads the `frontend-design` SKILL.md before generating any interface to avoid generic AI aesthetics.

**Design principles applied:**
- Thoughtful typography hierarchy (not just Tailwind defaults)
- Intentional color palette and whitespace
- Micro-interactions and hover states
- Mobile-responsive layouts
- Avoids: cookie-cutter card grids, default blue-500 buttons, unstyled Times New Roman headings

**Tech stack:** React + Tailwind core utilities (pre-built stylesheet only, no compiler), or plain HTML/CSS/JS for simpler cases.

**Available CDN libraries:** React, Tailwind, recharts, lucide-react, d3, Three.js (r128), mathjs, lodash, Tone.js, shadcn/ui, Chart.js, Plotly, Papaparse, SheetJS, mammoth, tensorflow

**Hard constraints:**
- Never use `localStorage` or `sessionStorage` — not supported in Claude artifacts
- Never include `<html>`, `<head>`, or `<body>` wrapper tags in HTML artifacts
- Default export required for React components, no required props
- THREE.CapsuleGeometry not available in r128 — use CylinderGeometry or SphereGeometry instead

---

## Web Artifacts Builder

For elaborate multi-component Claude.ai artifacts requiring navigation between views, complex state management, or shadcn/ui components. Reads the `web-artifacts-builder` SKILL.md for the correct multi-component patterns.

**When to use (all conditions should apply):**
- App needs navigation between multiple distinct screens
- Requires complex shared state across components
- Explicitly requests shadcn/ui
- A single JSX file would meaningfully exceed 400 lines

**When NOT to use:** Single-screen tools, calculators, data displays — use Frontend Design instead.
