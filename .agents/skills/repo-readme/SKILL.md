---
name: repo-readme
description: Use when creating, rewriting, refreshing, or improving a repository README that needs a concise, evidence-based GitHub presentation.
---

# Repository README

Create or update the root `README.md` as a concise, visual-first project entry point. Make the project recognizable, explain how to start it, and link to deeper documentation. Use only claims supported by repository evidence.

## Inspect before writing

Inspect the current README, manifests and lock files, source tree, tests, CI and deployment configuration, Docker files, `.env.example`, API specifications, documentation, assets, license, and Git metadata when available.

Never invent features, versions, deployment details, CI status, commands, integrations, coverage, release status, architecture, or URLs. Preserve useful existing information unless evidence shows it is obsolete, duplicated, or incorrect. State test or CI status only when current evidence supports it. Verify every command, badge, icon, link, and referenced asset against the repository or its authoritative source.

## Visual identity

Start with a visual hero. Keep the header focused on one dominant idea, a strong and deliberate composition, and generous negative space. Use a small, coherent palette and considered typography; monospace is optional, not the default. The hero communicates project identity, while technical architecture belongs in its own later section.

Keep the logo and README hero as separate assets:

```text
docs/assets/
├── logo.svg
└── readme-hero.svg
```

`logo.svg` is the reusable project identity and must work outside the README at small sizes. It is a mark, not an architecture diagram. `readme-hero.svg` is the wide README composition and may incorporate the logo, project name, and a short tagline. When assets may be created, use this convention; do not make the hero the only usable logo.

Prefer a distinctive abstract, geometric, or editorial symbol with a clear silhouette. Avoid boxes with arrows, pipeline stages, terminal windows, database cylinders, robots, code brackets, and generic AI branding. Do not diagram the system in its logo. Keep SVGs lightweight and maintainable, and derive colors from an existing identity when available.

Before accepting a visual, check that it expresses one idea, reads at small sizes, looks like a reusable identity rather than a technical diagram, and remains recognizable without explanatory copy.

## Header and content order

Use this order, omitting any section or element that lacks repository evidence or a useful purpose:

1. Hero image
2. Project name
3. One-line description
4. Compact badges
5. Technology icons
6. Optional visual preview
7. Quick Start
8. Architecture
9. Development
10. Testing
11. API / Interfaces
12. Deployment
13. Project Structure
14. Contributing
15. License

Keep explanatory text to a minimum. The one-line description should state what the project is, without generic marketing copy. Keep Quick Start close to the top, immediately after the visual header. Prefer images, concise tables, GitHub-compatible Mermaid diagrams, commands, and links to long paragraphs. Put detailed material in `docs/` and link to it from the relevant section.

### Badges and technology icons

Use a compact, consistent badge row for verifiable project status, such as CI, license, release, coverage, or publication. Include only badges with authoritative targets and values supported by evidence. Do not use badges as a second inventory of the stack.

Show a separate row of icons for a few primary technologies confirmed by repository evidence. Keep icons visually distinct from badges; omit unsupported technologies or icons rather than implying usage. Do not turn dependency lists into icon walls.

### Optional preview

Add a representative screenshot or output only when it helps explain an application, CLI, hardware result, or generated artifact. Prefer one useful preview; verify that its local file exists. Do not use the architecture diagram as decorative header art.

## Sections

### Quick Start

Lead with the shortest verified path to run the project. Use the repository's actual package manager, scripts, and tools; do not substitute generic commands. Include only steps confirmed by manifests, scripts, documentation, or configuration. Document required environment setup only when evidenced, and never expose secrets.

### Architecture

Explain structure after Quick Start. Prefer a small Mermaid diagram or concise component table when repository evidence supports the relationships. Omit diagrams for simple projects. Keep architecture, feature lists, and technical labels out of the hero.

### Development and Testing

Use a short command table or code block for verified development, lint, build, and test commands. Do not claim tests pass unless they were run successfully or current authoritative CI evidence supports the claim. Link to detailed contributor guidance when it exists.

### API / Interfaces and Deployment

Document public interfaces such as REST, OpenAPI, MCP, CLI, package API, or deep links only when present. Link to canonical specifications or deeper docs rather than duplicating them. Describe deployment only when repository configuration or authoritative documentation supports it; do not expose sensitive infrastructure details.

### Project Structure, Contributing, and License

Show only the important paths, usually four to ten entries, with brief descriptions. Include contribution instructions and license information only when evidenced by repository files or authoritative project documentation. Never infer a license from common practice.

## Writing and navigation

Write short sentences and paragraphs. Use headings, commands, images, tables, and diagrams to make information scannable. Avoid repeated explanations, marketing filler, excessive HTML or emoji, giant tables, and a full repository dump. A table of contents is optional and belongs only in a README long enough to benefit from it.

Treat the README as the entry point: help readers identify the project, reach Quick Start, and find deeper material. Link to existing documentation under `docs/` rather than copying lengthy explanations into the README. Omit empty or unsupported sections.

## Final review

Before finishing, verify that:

- every factual claim is supported by repository evidence;
- commands match project configuration and documentation;
- badges, technology icons, links, and image paths are valid and truthful;
- the logo is reusable and separate from the README-specific hero;
- the README follows the prescribed order and Quick Start is near the top;
- unsupported and empty sections, redundant prose, and invented claims are removed;
- the result reads as a concise GitHub landing page, with detailed material linked from `docs/`;
- the final diff contains only intended README changes.
