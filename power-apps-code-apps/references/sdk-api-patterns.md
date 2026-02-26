# Power Apps SDK API Patterns

Package: `@microsoft/power-apps` (v1.0.4+)

**Important**: The `initialize()` function was removed in v1.0. Do NOT call `initialize()`. Data calls, context retrieval, and platform interactions work directly without initialization.

## Table of Contents

- [App Context](#app-context)
- [Dataverse CRUD](#dataverse-crud)
- [Query Options (IGetAllOptions)](#query-options)
- [Nontabular Connectors](#nontabular-connectors)
- [SharePoint Operations](#sharepoint-operations)
- [SQL Stored Procedures](#sql-stored-procedures)
- [Dataverse Metadata](#dataverse-metadata)
- [Telemetry / App Insights](#telemetry)

---

## App Context

```typescript
import { getContext } from "@microsoft/power-apps/app";

const ctx = await getContext();

// App context
ctx.app.appId;          // string
ctx.app.environmentId;  // string
ctx.app.queryParams;    // Record<string, string> - URL query params

// User context
ctx.user.fullName;              // string
ctx.user.objectId;              // string - Entra object ID
ctx.user.tenantId;              // string
ctx.user.userPrincipalName;     // string

// Host context
ctx.host.sessionId;  // string - changes each app open
```

---

## Dataverse CRUD

All generated services follow the same pattern. After running `npx power-apps add-data-source -a dataverse -t accounts`, use:

```typescript
import { AccountsService } from "./generated/services/AccountsService";
import type { Accounts } from "./generated/models/AccountsModel";

// CREATE
await AccountsService.create(newAccount as Omit<Accounts, "accountid">);

// READ single record
const record = await AccountsService.get(accountId);

// READ multiple with query options
const result = await AccountsService.getAll({
  select: ["name", "accountnumber"],
  filter: "address1_country eq 'USA'",
  orderBy: ["name asc"],
  top: 50,
  maxPageSize: 100,
});

// UPDATE (partial - only changed fields)
await AccountsService.update(accountId, { name: "New Name" });

// DELETE
await AccountsService.delete(accountId);
```

### Mapper Rule: Match Generated Model Casing Exactly

When writing or modifying any `mapToDataverse` function that returns an object sent to a generated Dataverse service (`create` / `update`):

- Read the corresponding generated model interface in `generated/models/` before defining property names.
- Use exact property casing from the generated `*Base` interface.
- For navigation bindings, keep the generated PascalCase shape for `@odata.bind` keys (for example: `"ppcoe_MadeBy@odata.bind"`, not `"ppcoe_madeby@odata.bind"`).
- Never assume Dataverse logical-name casing (often lowercase) matches generated TypeScript model casing; the SDK generator may capitalize segments.
- If the mapper returns `Record<string, unknown>` instead of a generated model type, add a comment naming the target generated interface, for example: `// Target: Ppcoe_decisionsBase`.

---

## Query Options

The `getAll()` method accepts `IGetAllOptions`:

```typescript
interface IGetAllOptions {
  maxPageSize?: number;    // Page size for results
  select?: string[];       // Columns to return
  filter?: string;         // OData filter expression
  orderBy?: string[];      // Sort order, e.g. ["name asc"]
  top?: number;            // Limit number of results
  skip?: number;           // Skip N results
  skipToken?: string;      // Pagination continuation token
}
```

---

## Nontabular Connectors

Example: Office 365 Users connector.

```typescript
import { Office365UsersService } from "./generated/services/Office365UsersService";

// Get current user profile
const profile = (
  await Office365UsersService.MyProfile_V2(
    "id,displayName,jobTitle,userPrincipalName"
  )
).data;

// Get user photo
const photo = (await Office365UsersService.UserPhoto_V2(profile.id)).data;
```

Method signatures are auto-generated based on the connector's API. Check the generated service file for available methods and parameters.

---

## SharePoint Operations

```typescript
import { MyListService } from "./generated/services/MyListService";

// Read all items
const result = await MyListService.getAll();

// Read single item
const record = await MyListService.get(id);

// Create item (omit ID — auto-generated)
await MyListService.create(payload as Omit<MyList, "ID">);

// Update item
await MyListService.update(recordId, updatePayload);

// Delete item
await MyListService.delete(recordId);

// Get choice/lookup column values
const choices = await MyListService.getReferencedEntity("", "ColumnName");
```

---

## SQL Stored Procedures

```typescript
import { GetAllProjectsService } from "./generated/services/GetAllProjectsService";

const result = await GetAllProjectsService.GetAllProjects();
if (result.success && result.data?.ResultSets?.Table1) {
  const data = result.data.ResultSets.Table1 as ProjectItem[];
}
```

---

## Dataverse Metadata

```typescript
const { data } = await AccountsService.getMetadata({
  metadata: ["Privileges", "DisplayName"],
  schema: {
    columns: "all", // or specific: ["name", "telephone1"]
    oneToMany: true,
    manyToOne: true,
    manyToMany: true,
  },
});

// Column display names
data.Attributes?.forEach((attr) => {
  const label = attr.DisplayName?.UserLocalizedLabel?.Label;
});

// Required fields
const required = data.Attributes?.filter((a) => a.IsRequiredForForm);

// Lookup relationships
data.ManyToOneRelationships?.map((rel) => ({
  lookupField: rel.ReferencingAttribute,
  relatedTable: rel.ReferencedEntity,
}));
```

---

## Telemetry

### Azure Application Insights Integration

```typescript
import { ApplicationInsights } from "@microsoft/applicationinsights-web";
import { setConfig } from "@microsoft/power-apps/app";

const appInsights = new ApplicationInsights({
  config: {
    connectionString:
      "InstrumentationKey=<KEY>;IngestionEndpoint=<ENDPOINT>",
  },
});
appInsights.loadAppInsights();
appInsights.trackPageView();

setConfig({
  logger: {
    logMetric: (value: Metric) => {
      appInsights.trackEvent({ name: value.type }, value.data);
    },
  },
});
```

### Built-in Metric Types

- `SessionLoadSummary` — `successfulAppLaunch`, `appLoadResult`, `timeToAppInteractive`
- `NetworkRequest` — `url`, `method`, `duration`, `statusCode`, `responseSize`
