# Power Apps Code Apps Skill

An agent skill for building, developing, and deploying [Power Apps Code Apps](https://learn.microsoft.com/en-us/power-apps/maker/code-apps/) — code-first web applications (React + TypeScript + Vite) that run inside the Power Platform.

## Install

```bash
npx skills add chumleesockson/power-apps-code-apps-skill
```

## What's Included

- **SKILL.md** — Core workflow: scaffolding, data sources, SDK usage, build & deploy
- **references/cli-commands.md** — Complete `pac code` CLI reference (auth, scaffolding, data sources, deployment)
- **references/project-structure.md** — Project layout, technology stack, and Vite configuration
- **references/sdk-api-patterns.md** — API patterns for Dataverse, connectors, SharePoint, SQL, and telemetry

## When It Activates

This skill triggers when working with:

- `power.config.json` files
- `@microsoft/power-apps` imports
- `pac code` CLI commands
- Power Apps Code Apps projects
