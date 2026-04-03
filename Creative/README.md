# Creative Skills

Skills for original visual and creative outputs — static artwork, animated GIFs, and algorithmic generative art. These skills ensure Claude creates original work rather than reproducing copyrighted IP or copying existing artists.

## Skills in This Category

| Skill | Output | Trigger Phrases |
|-------|--------|-----------------|
| [Canvas Design](#canvas-design) | .png / .pdf | "poster", "art", "design", "illustration", "visual", "make me something" |
| [Algorithmic Art](#algorithmic-art) | p5.js sketch | "generative art", "flow field", "particle system", "p5.js", "procedural" |
| [Slack GIF Creator](#slack-gif-creator) | Animated .gif | "GIF for Slack", "make me a GIF", "animated GIF" |

---

## Canvas Design

Creates original visual designs as `.png` or `.pdf` files using design philosophy — intentional color, typography, and composition. Never reproduces existing artists' work or copyrighted characters.

**When to use:** User asks for a poster, piece of art, design, illustration, or any static visual deliverable intended for download.

**Key behaviors:**
- Reads the `canvas-design` SKILL.md before starting
- Generates original concepts — does not mimic specific artists' signatures
- Uses Pillow for raster, SVG for scalable, or Cairo for print-quality output
- Delivers to `/mnt/user-data/outputs/`

**Never generates:** Disney/Marvel/Nintendo/sports league characters or branding, celebrity likenesses, reproductions of existing paintings or photographs, sexual or violent content.

---

## Algorithmic Art

Creates generative art using p5.js with seeded randomness, interactive parameter controls, and deterministic outputs. Renders as an interactive sketch in the Claude chat interface.

**Styles supported:** flow fields, Perlin noise landscapes, particle systems, recursive geometry, L-systems, cellular automata, wave interference patterns, Voronoi diagrams, fractal trees.

**Key behaviors:**
- Always uses a random seed so outputs are reproducible
- Includes sliders or buttons for key parameters (seed, density, speed, color)
- Reads the `algorithmic-art` SKILL.md before starting
- Avoids directly copying specific artists' signature styles (e.g. "Tyler Hobbs style" → inspired by, not copied)

---

## Slack GIF Creator

Creates animated GIFs optimized for Slack — correct file size, dimensions, frame count, and smooth looping.

**Slack GIF hard constraints:**
- Max file size: 2MB
- Recommended dimensions: 128×128 to 512×512 px
- Frame count: 10–30 frames for smooth loops
- Color palette: 256 colors max (GIF format limitation)
- Always infinite loop

**Key behaviors:**
- Reads the `slack-gif-creator` SKILL.md for constraints and validation tools
- Validates file size after generation; re-compresses if over limit
- Uses Pillow's `ImageSequence` for frame assembly
- Delivers to `/mnt/user-data/outputs/`
