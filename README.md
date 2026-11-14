# Agent Skills

Reusable instructions for coding agents. These skills follow the directory + `SKILL.md` convention, with YAML frontmatter and optional `references/` files.

## Available skills

### [cinematic-web-frontend](cinematic-web-frontend/SKILL.md)

Build distinctive, polished, animation-rich frontend experiences **without generic AI-generated layouts**. Includes practical guidance for design systems, navigation choreography, WebGL2, Three.js, canvas lifecycle, mobile responsiveness, accessibility, and performance tuning.

Based on a technical review of a ToolMC frontend snapshot, with the unneeded marketing text and site-specific branding removed. Its patterns are generalized; original third-party website assets and shader implementations are **not** redistributed.

References:
- [Rendering and performance](cinematic-web-frontend/references/rendering-and-performance.md)
- [Motion and interaction](cinematic-web-frontend/references/motion-and-interaction.md)

### Usage

Place the `cinematic-web-frontend/` folder inside your coding agent's skills directory (for example, `.agents/skills/` for agents that support it) or provide its `SKILL.md` as agent instructions. Keep the `references/` subfolder alongside it for on-demand context.

Example requests:
- "Build this landing page with cinematic transitions, minimal text, and no rounded feature-card grid. Use cinematic-web-frontend."
- "Review my canvas/Three.js site for stalled transitions, offscreen RAF loops, excessive GPU cost, and reduced-motion support."
- "Create a unique dark website with strong typography and responsive interactive visuals, not a template."

## License

Repository: Apache-2.0. This skill is an original set of instructions inspired by publicly visible implementation patterns. It does not grant any rights to ToolMC's original source code, branding, artwork, or other third-party content.
