# CLI Commands for Code Apps

> **SDK v1.0.4+**: Most Code Apps CLI commands use the npm-based CLI (`npx power-apps`). Authentication is handled automatically — sign in with your Power Platform account when prompted on first use.
>
> **PAC CLI still required for**: listing connections (`pac connection list`), listing solutions (`pac solution list`), and listing SQL stored procedures (`pac code list-sql-stored-procedures`). These cannot be done with `npx power-apps`.

## Project Scaffolding

```bash
# Starter template (React + Tailwind + shadcn/ui + TanStack + Zustand)
npx degit microsoft/PowerAppsCodeApps/templates/starter#main my-app

# Minimal Vite template
npx degit microsoft/PowerAppsCodeApps/templates/vite#main my-app

# After scaffolding
cd my-app
npm install
npm install @microsoft/power-apps

# Initialize the code app (interactive mode — prompts for display name and environment)
npx power-apps init

# Or pass options directly
npx power-apps init --displayName "My App Name" --environmentId {environmentId}
```

`npx power-apps init` generates `power.config.json` with environment metadata and authenticates automatically. App logic should never interact with this file directly.

## Local Development

```bash
npx power-apps run
```

This starts a local development server. Open the URL labeled **Local Play** in the same browser profile as your Power Platform tenant.

> **Note**: Since December 2025, Chrome and Edge block requests from public origins to local endpoints by default. You may need to grant browser permission for local network access during development.

## Build and Deploy

Always deploy to a specific solution — never to the default solution. This ensures healthy ALM and enables pipeline promotion across environments.

```bash
# Build the app
npm run build

# Push to Power Platform environment
npx power-apps push
```

`npx power-apps push` publishes a new version of the code app to the environment configured during `init`. When the command finishes, it returns a Power Apps URL to run the app.

Use **Power Platform Pipelines** to promote solutions across stages (Dev → Test → Prod). Pipelines include preflight checks for dependencies and connection references.

## Data Source Management

### Prerequisites for Non-Dataverse Connectors

Non-Dataverse data sources require two things created in the Power Apps UI first:

1. **Connection**: [make.powerapps.com](https://make.powerapps.com) > **Connections** > **+ New connection** — create for the desired connector (SQL, SharePoint, Office 365, etc.)
2. **Connection reference**: [make.powerapps.com](https://make.powerapps.com) > **Solutions** > your solution > **+ New** > **More** > **Connection Reference** — create a reference that points to the connection above

Neither can be created via CLI — only through the UI.

**Note:** Excel Online (Business) and Excel Online (OneDrive) connectors are not supported.

### Adding Data Sources

#### Dataverse (direct — no connection needed)

```bash
npx power-apps add-data-source -a dataverse -t {tableName}
# Example: npx power-apps add-data-source -a dataverse -t accounts
```

#### Non-Dataverse with Connection References (recommended)

Connection references make solutions portable across environments (Dev/Test/Prod). Use this approach for production apps.

**Important: Do NOT guess API names** (the `-a` value). Always discover them from the environment.

**Step 1: Discover the API name** (requires PAC CLI)

```bash
pac connection list
```

Output includes the **API name** (e.g., `shared_sql`, `shared_azureblobstorage`, `shared_office365users`) and connection ID for each connection. Use the API name as the `-a` value.

**Step 2: Get solution ID** (requires PAC CLI)

```bash
pac solution list
```

**Step 3: List connection references in the solution**

```bash
npx power-apps list-connection-references -env {environmentURL} -s {solutionID}
```

Output includes display name, **logical name**, and connector for each reference. Match the connector column to the API name from Step 1. Use the **logical name** as the `-cr` value.

**If multiple references exist for the same connector** (e.g., two SQL references pointing to different databases), present all matching options with their display name, logical name, and description to the user and ask which one to use.

**Step 4: Add the data source**

```bash
npx power-apps add-data-source -a {apiName} -cr {connectionReferenceLogicalName} -s {solutionID}
```

#### Non-Dataverse with Direct Connection (not portable)

Direct connections bind to a specific user's connection ID. Avoid for production — use connection references instead.

```bash
# Nontabular connector
npx power-apps add-data-source -a "shared_office365users" -c "{connectionId}"

# SQL table
npx power-apps add-data-source -a "shared_sql" -c "{connectionId}" -t "[dbo].[TableName]" -d "server.database.windows.net,dbname"

# SQL stored procedure
npx power-apps add-data-source -a "shared_sql" -c "{connectionId}" -d "server,db" -sp "[dbo].[ProcName]"
```

### Auto-Generated Files

Adding a data source auto-generates typed models and services under `generated/`:

```
generated/
  models/
    {TableName}Model.ts     # TypeScript type definitions
  services/
    {TableName}Service.ts   # CRUD service methods
```

### Discovery Commands

```bash
# List all code apps in the environment
npx power-apps list-codeapps

# Discover datasets for a connector
npx power-apps list-datasets -a {apiId} -c {connectionId}

# Discover tables within a dataset
npx power-apps list-tables -a {apiId} -c {connectionId} -d {datasetName}

# List connection references in a solution
npx power-apps list-connection-references -env {environmentURL} -s {solutionID}

# List environment variables in the environment
npx power-apps list-environment-variables
```

#### Commands requiring PAC CLI

These discovery commands are **not** available via `npx power-apps` — use the PAC CLI instead:

```bash
# List available connections (needed to discover API names and connection IDs)
pac connection list

# List solutions (needed to get solution IDs)
pac solution list

# List SQL stored procedures for a dataset
pac code list-sql-stored-procedures -c {connectionId} -d {datasetName}
```

### Other Commands

```bash
# Log out the current user
npx power-apps logout

# Manage telemetry settings
npx power-apps telemetry
```

### Deleting Data Sources

```bash
npx power-apps delete-data-source -a {apiName} -ds {dataSourceName}
```

**Important**: There is no refresh command. When a schema changes, delete and re-add the data source.
