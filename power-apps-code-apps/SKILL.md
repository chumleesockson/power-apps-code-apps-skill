---
name: power-apps-code-apps
description: "Build, develop, and deploy Power Apps Code Apps — code-first web applications that run inside the Power Platform. Use this skill when working on Power Apps Code Apps projects: scaffolding new apps, writing React components that use the @microsoft/power-apps SDK, adding data sources and connectors (Dataverse, SQL, SharePoint, Office 365), using generated CRUD services and typed models, running npx power-apps commands (init, push, run, add-data-source, list-codeapps, list-datasets, list-tables, list-connection-references, list-environment-variables, delete-data-source), configuring Vite builds with the power-apps-vite plugin, deploying to Power Platform environments or diagnosing missing functionality caused by absent data sources or connectors. Note: listing connections, solutions, and SQL stored procedures requires the PAC CLI. Triggers on: power.config.json files, @microsoft/power-apps imports, npx power-apps commands, Power Apps Code Apps projects."
license: MIT
metadata:
  version: 1.1.0
---

# Power Apps Code Apps

Code Apps are code-first web applications (React + TypeScript + Vite) that run inside Power Platform with managed authentication, 1500+ connectors, governance, and ALM.

## Architecture

```
Your Code (React SPA)
    ↓
Power Apps SDK (@microsoft/power-apps)
    ↓
Power Apps Host (Entra auth, app loading, error handling)
```

Authentication is managed by the host — never implement auth flows in app code. End users need a Power Apps Premium license.

## Mandatory Rule

**ALWAYS read this skill before recommending or running any `npx power-apps` CLI command.** Do not rely on other context sources (e.g., copilot-instructions.md) for CLI syntax — this skill is the authoritative reference.

**SDK v1.0.4+**: Most CLI commands use the npm-based CLI (`npx power-apps`). Some discovery commands (listing connections, solutions, SQL stored procedures) still require the PAC CLI. See [references/cli-commands.md](references/cli-commands.md) for the full command reference.

## SDK Version Check

Before doing any other work in a Code Apps project, check the installed SDK version in `package.json` (the `@microsoft/power-apps` dependency). If the version is below **1.0.4**, stop and upgrade first:

```bash
npm install @microsoft/power-apps@latest
```

Breaking changes by version:
- **< 1.0.0**: `initialize()` existed and was required — removed in v1.0. All code calling `initialize()` must be updated.
- **< 1.0.4**: The PAC CLI (`pac code`) was used for all commands — mostly replaced by `npx power-apps` in v1.0.4 (PAC CLI still needed for listing connections, solutions, and SQL stored procedures).

After upgrading, verify the project still builds (`npm run build`) before continuing with the user's request.

## Core Workflow

### 1. Scaffold a New Project

Use the starter template (React 19, Tailwind CSS 4, shadcn/ui, React Router, TanStack Query, TanStack Table, Zustand, Lucide icons). See [references/cli-commands.md](references/cli-commands.md) for scaffolding and all other CLI commands.

### 2. Add Data Sources

See [references/cli-commands.md](references/cli-commands.md) for the full data source workflow. Key points:

- **Dataverse** tables are added directly: `npx power-apps add-data-source -a dataverse -t {tableName}`
- **All other connectors** require a connection and connection reference created in the Power Apps UI first, then use `npx power-apps add-data-source -a {apiName} -cr {connectionRefLogicalName} -s {solutionId}`
- **Do NOT guess API names** — discover them via `pac connection list` (requires PAC CLI)
- Adding a data source auto-generates typed models and services under `generated/`. Never hand-edit these files.

### 3. Use Generated Services in Code

```typescript
import { AccountsService } from "./generated/services/AccountsService"
import type { Accounts } from "./generated/models/AccountsModel"

// Read with query options
const { data } = await AccountsService.getAll({
	select: ["name", "accountnumber"],
	filter: "address1_country eq 'USA'",
	orderBy: ["name asc"],
	top: 50,
})

// Create
await AccountsService.create({ name: "Contoso" } as Omit<Accounts, "accountid">)

// Update (partial)
await AccountsService.update(id, { name: "New Name" })

// Delete
await AccountsService.delete(id)
```

For complete API patterns (context, connectors, SharePoint, SQL, metadata, telemetry), see [references/sdk-api-patterns.md](references/sdk-api-patterns.md).

### 4. Access App/User Context

```typescript
import { getContext } from "@microsoft/power-apps/app"

const ctx = await getContext()
ctx.user.fullName // Current user's name
ctx.user.objectId // Entra object ID
ctx.app.environmentId // Environment ID
ctx.app.queryParams // URL query parameters
```

### 5. Build and Deploy

Always deploy to a specific solution — never to the default solution.

```bash
npm run build
npx power-apps push
```

The `push` command publishes a new version to the Power Platform environment configured during `init`. Use Power Platform Pipelines to promote solutions across stages (Dev → Test → Prod).

## Key Rules

- **Do NOT call `initialize()`** — removed in SDK v1.0. All APIs work directly.
- **Do NOT edit `power.config.json`** — generated by the CLI for internal use.
- **Use `npx power-apps` for most CLI commands** — as of SDK v1.0.4, the npm CLI handles core workflows (init, run, push, add/delete data sources, list datasets/tables/connection references/environment variables/code apps). Listing connections, solutions, and SQL stored procedures still requires the **PAC CLI** (`pac connection list`, `pac solution list`, `pac code list-sql-stored-procedures`).
- **Do NOT store secrets in app code** — apps are hosted on public endpoints. Use authenticated data sources instead.
- **Check SDK version first** — before starting work on any existing Code Apps project, verify `@microsoft/power-apps` in `package.json` is **1.0.4** or later. If not, upgrade with `npm install @microsoft/power-apps@latest` before proceeding.
- **Do NOT hand-edit `generated/`** — re-run `npx power-apps add-data-source` to regenerate.
- **Use the `@` import alias** — maps to `./src` via Vite config (e.g., `import { Button } from "@/components/ui/button"`).
- **Use `@microsoft/power-apps-vite`** — required Vite plugin, already configured in the starter template.

## Project Structure

For the complete project layout and technology stack, see [references/project-structure.md](references/project-structure.md).

## TanStack Query Pattern

Wrap data fetching in TanStack Query for caching and state management:

```typescript
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query"
import { AccountsService } from "./generated/services/AccountsService"

function useAccounts() {
	return useQuery({
		queryKey: ["accounts"],
		queryFn: () => AccountsService.getAll({ top: 100 }),
	})
}

function useCreateAccount() {
	const queryClient = useQueryClient()
	return useMutation({
		mutationFn: (account: Omit<Accounts, "accountid">) =>
			AccountsService.create(account),
		onSuccess: () =>
			queryClient.invalidateQueries({ queryKey: ["accounts"] }),
	})
}
```

## Limitations

- No Power Apps mobile/Windows app support
- No Power Platform Git integration
- No environment variables for secrets
- No FetchXML, polymorphic lookups, or Dataverse actions/functions
- Schema changes require delete + re-add of data source
- Connections must be created in the Power Apps UI, not via CLI
