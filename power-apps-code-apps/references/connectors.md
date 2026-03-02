# Connector-Specific Patterns

Guidance for working with specific Power Platform connectors in Code Apps.

---

## Office 365 — People Picker

When implementing a people picker backed by the Office 365 connector, the `User.Id` returned is the **AAD Object ID** — not the Dataverse `systemuserid`.

Before calling `onChange`, resolve the correct `systemuserid` by querying the `systemuser` table:

```
filter=azureactivedirectoryobjectid eq '<aadObjectId>'&select=systemuserid&top=1
```

Store only the resolved `systemuserid` in `PersonRef.id` so OData bind expressions (`/systemusers(<id>)`) are valid on save.
