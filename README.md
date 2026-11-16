# Agent skills

Skills that teach AI coding agents how to do specific jobs well. They use the open [Agent Skills](https://agentskills.io) format, so they work in Claude Code, claude.ai and GitHub Copilot.

| Skill | What it does |
| --- | --- |
| [progressive-enhancement](skills/progressive-enhancement/SKILL.md) | Builds and reviews web UI so it works from the server HTML alone and JavaScript only makes it better. Read the article: [Progressive enhancement](docs/progressive-enhancement.md) |

## Install

### Claude Code

```bash
claude plugin marketplace add <your-github-user>/agent-skills
claude plugin install progressive-enhancement@saoir-skills
```

### GitHub Copilot

With the GitHub CLI:

```bash
gh skill install <your-github-user>/agent-skills
```

Or copy the skill folder yourself:

- For one project: `skills/progressive-enhancement` → `.github/skills/progressive-enhancement` in that repo
- For all your projects: `skills/progressive-enhancement` → `~/.copilot/skills/progressive-enhancement`

Skills work in Copilot agent mode in VS Code, the Copilot CLI and the Copilot cloud agent.

### claude.ai

Download `progressive-enhancement-skill.zip` from the latest release, then go to **Customize → Skills → + → Create skill → Upload a skill**. Code execution must be turned on in your settings.

### Anything else

Most agents that support Agent Skills read a folder containing `SKILL.md`. Copy `skills/progressive-enhancement` to wherever your agent looks for skills.

## progressive-enhancement

It covers:

- The approach: baseline, enhancement and the switch between them
- When it's not worth it: canvas editors, real-time apps, internal tools and prototypes, and the cheap parts worth keeping anyway
- Patterns for forms and mutations, validation, dependent selects (WCAG 3.2.2 On Input), search and filters, pagination and infinite scroll, client-side routing, accordions, tabs, dialogs, menus, like/follow buttons, clipboard and share, inputs, images and carousels
- CSS fallbacks, `@supports`, no-JS styling hooks and motion
- Recipes for React Router, Next.js and framework-free code
- Hydration traps, a PR review checklist and testing steps

Ask your agent to build or review something like "a second dropdown that depends on the first", "make this form work without JavaScript", "infinite scroll for the job list" or "review this modal for accessibility". The skill loads on its own. In Claude Code you can also run `/progressive-enhancement:progressive-enhancement`.

## Layout

```
.claude-plugin/marketplace.json      Claude Code marketplace
skills/
  progressive-enhancement/
    SKILL.md
    references/
      patterns.md       component-by-component baseline → enhancement → pitfalls
      on-input.md       controls that load data on change (WCAG 3.2.2)
      css.md            fallbacks, @supports, no-JS hooks, motion
      react-router.md   React Router v7 / Remix recipes
      nextjs.md         Next.js App Router recipes
      vanilla.md        framework-free recipes
docs/
  progressive-enhancement.md         the article, with its diagram
```

## Adding a skill

1. Add a folder under `skills/` with a `SKILL.md`.
2. Add an entry for it in `.claude-plugin/marketplace.json`, copying the progressive-enhancement entry and changing the name, source and description.
3. Add a row to the table at the top of this README.
