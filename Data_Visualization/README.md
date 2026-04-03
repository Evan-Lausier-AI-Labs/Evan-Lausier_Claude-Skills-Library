# Data Visualization Skills

Skills for creating inline charts, diagrams, and interactive widgets that render in the Claude chat interface via the visualizer tool.

## Skills in This Category

| Skill | Output | Trigger Phrases |
|-------|--------|-----------------|
| [Chart](#chart) | SVG/HTML inline | "chart", "graph", "bar chart", "line chart", "pie chart", "plot" |
| [Diagram](#diagram) | SVG inline | "diagram", "flowchart", "architecture", "process map", "show me how" |
| [Interactive](#interactive) | HTML widget | "interactive", "calculator", "simulator", "dashboard", "widget", "tool" |

---

## Chart

Generates inline SVG or HTML charts directly in the conversation. Supports bar, line, area, pie, scatter, and combo charts.

**Chart type selection guide:**

| Data Shape | Chart Type |
|------------|------------|
| Comparison across categories | Bar / column |
| Trend over time | Line / area |
| Part-to-whole | Pie / donut |
| Correlation | Scatter |
| Distribution | Histogram |

**Key behaviors:**
- Loads the `chart` visualizer module before generating
- Uses CSS variables for theming — never hardcoded colors
- Keeps background transparent so it renders correctly in both light and dark mode
- Labels axes, includes a legend when multiple series are present
- Handles empty data, single data points, and very large values gracefully

---

## Diagram

Generates inline SVG diagrams for system architecture, process flows, entity relationships, decision trees, and sequence diagrams.

**Key behaviors:**
- Loads the `diagram` visualizer module before generating
- Uses consistent box sizing, arrow weights, and connector styles
- Applies semantic color coding: green = success/start, red = error/stop, blue = process, gray = external
- Adds a legend when color coding is non-obvious
- Defaults to left-to-right layout; switches to top-to-bottom for hierarchies

**Common diagram types:**
- Flowchart / decision tree
- System architecture (services, APIs, databases)
- Entity-relationship diagram
- Sequence / interaction diagram
- Network topology

---

## Interactive

Generates interactive HTML widgets that render inline in Claude chat. Used for calculators, configurators, simulators, games, data explorers, and dashboards.

**Key behaviors:**
- Loads the `interactive` visualizer module before generating
- Uses vanilla JS or React — no external dependencies beyond approved CDN libs
- Manages state cleanly without localStorage (not supported in Claude artifacts)
- Includes input validation and sensible default values
- Shows loading indicators for any async operations

**Approved CDN libraries:** React, Tailwind, recharts, d3, Plotly, Three.js (r128), mathjs, lodash, Tone.js
