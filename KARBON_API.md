# Karbon API — Agent Quick Reference

> **For agents and developers.** Concise patterns for making correct Karbon API calls.
> Full spec: [KarbonAPI.json](./KarbonAPI.json)

---

## 1. Overview

- **Base URL:** `https://api.karbonhq.com`
- **Version:** v3
- **Standard:** OData (v2 URI conventions)

---

## 2. Authentication

Two headers required on every request:

```
Authorization: Bearer {token}
AccessKey: {key}
```

- **`Authorization` token** — GUID format: `550e8400-e29b-41d4-a716-446655440000`. Obtained by registering at the Karbon Developer Center.
- **`AccessKey`** — JWT format: three Base64URL parts separated by dots, starts with `eyJ...`. Found in Karbon under **Settings → Connected Apps → {Your API Application}**.

---

## 3. Common Patterns

### Pagination

All list endpoints support pagination. Max 100 items per request.

| Parameter | Purpose |
|-----------|---------|
| `$top` | Items per page (max: 100) |
| `$skip` | Items to skip |

Response fields:

| Field | Description |
|-------|-------------|
| `@odata.count` | Total matching records |
| `@odata.nextLink` | URL for the next page — use this directly, don't increment `$skip` manually |

**Response envelope:**
```json
{
  "@odata.context": "...",
  "@odata.count": 323,
  "@odata.nextLink": "https://api.karbonhq.com/v3/Contacts?$skip=100",
  "value": [ ... ]
}
```

---

### OData Filtering (`$filter`)

Supported operators vary by endpoint and field — not all operators work with all fields.

| Operator | Meaning |
|----------|---------|
| `eq` | Exact match |
| `contains(field, 'val')` | Partial string match |
| `and` | Combine conditions |
| `ge` / `le` / `gt` / `lt` | ≥ / ≤ / > / < (dates/numbers) |
| `in` | Set membership |

**Filterable fields by endpoint:**

| Endpoint | Filterable fields | Supported operators |
|----------|-------------------|---------------------|
| `GET /v3/Contacts` | `FullName`, `EmailAddress`, `PhoneNumber`, `ContactType` | `eq`, `contains`, `and` |
| `GET /v3/Organizations` | `FullName`, `EmailAddress`, `ContactType` | `eq`, `contains`, `and` |
| `GET /v3/ClientGroups` | `FullName` | `eq` only |
| `GET /v3/WorkItems` | `AssigneeEmailAddress`, `ClientKey`†, `PrimaryStatus`†, `Title`, `WorkScheduleKey`†, `WorkStatus`, `WorkType` | `eq`, `contains`†, `and` |
| `GET /v3/WorkItems` | `StartDate` | `ge`, `le`, `and` |
| `GET /v3/Timesheets` | `StartDate` (gt), `EndDate` (lt), `Status` (eq), `UserKey` (in), `WorkItemKeys` (any/in) | mixed — see example below |
| `GET /v3/Users` | `Name`, `EmailAddress` | `eq` |
| `GET /v3/WorkTemplates` | `Title`, `WorkTypeKey`, `PublishedDate`, `DateModified`, `DateLastWorkItemCreated`, `NumberOfWorkItemsCreated`, `HasScheduledClientTaskGroups`, `DraftHasChanges` | `eq` |

† `contains` is not supported for `ClientKey`, `PrimaryStatus`, or `WorkScheduleKey`.

**Timesheets `WorkItemKeys` filter example** (batch multiple keys):
```
GET /v3/Timesheets?$filter=WorkItemKeys/any(x: x in ('2m6pSFxRzcF2', '2y7H6dhQL7mD'))
```

---

### Ordering (`$orderby`)

Append ` desc` to any sortable field for descending order.

| Endpoint | Sortable fields |
|----------|----------------|
| `GET /v3/Contacts` | `FullName`, `LastModifiedDateTime` |
| `GET /v3/Organizations` | `FullName`, `LastModifiedDateTime` |
| `GET /v3/ClientGroups` | `FullName` (default: `ClientGroupKey`) |
| `GET /v3/WorkItems` | `StartDate`, `DeadlineDate` |
| `GET /v3/Timesheets` | `StartDate`, `EndDate` |
| `GET /v3/Invoices` | `InvoiceDate`, `CreatedAt`, `UpdatedAt`, `InvoiceNumber` |
| `GET /v3/WorkTemplates` | `WorkTypeKey`, `PublishedDate`, `NumberOfWorkItemsCreated`, `DateLastWorkItemCreated`, `DateModified` |

---

### Expanding related data (`$expand`)

Pass a comma-separated list where multiple values are supported. Only available on the endpoints listed below.

| Endpoint | `$expand` options |
|----------|-------------------|
| `GET /v3/Contacts/{key}` | `BusinessCards`, `ClientTeam`, `ClientAccess` |
| `GET /v3/Contacts/GetContactByUserDefinedIdentifier(...)` | `BusinessCards` |
| `GET /v3/ClientGroups/GetClientGroupByUserDefinedIdentifier(...)` | `BusinessCard` |
| `GET /v3/Organizations/{key}` | `BusinessCards` |
| `GET /v3/Organizations/GetOrganizationByUserDefinedIdentifier(...)` | `BusinessCards` |
| `GET /v3/Timesheets` | `TimeEntries` |
| `GET /v3/Timesheets/{key}` | `TimeEntries` |

---

### Rate Limiting

- **HTTP 429:** `{ "statusCode": "429", "message": "Rate limit is exceeded. Try again in 10 seconds." }`
- Retry after the indicated delay.

---

### UserDefinedIdentifier Lookups

Contacts, Organizations, and ClientGroups support lookup by a custom identifier:

```
GET /v3/Contacts/GetContactByUserDefinedIdentifier(UserDefinedIdentifier='{id}')
GET /v3/Organizations/GetOrganizationByUserDefinedIdentifier(UserDefinedIdentifier='{id}')
GET /v3/ClientGroups/GetClientGroupByUserDefinedIdentifier(UserDefinedIdentifier='{id}')
```

---

## 4. Resource Quick Reference

| Tag | Key Endpoints | Notes |
|-----|---------------|-------|
| **Billing** | `GET /v3/Invoices`, `GET /v3/Invoices/{key}`, `GET /v3/Payments`, `POST /v3/ManualPayments`, `DELETE /v3/ManualPayments/{key}`, `POST /v3/ReverseManualPayment` | Read-only for invoices/payments |
| **Business Cards** | `GET /v3/BusinessCards/{key}`, `PUT /v3/BusinessCards/{key}` | Attached to Contact or Organization. Set `OrganizationKey` on a Contact's BusinessCard to associate the Contact with an Organization. Always `null` on Organization-side cards. |
| **Client Groups** | `GET`, `POST /v3/ClientGroups`, `GET/PUT/PATCH /v3/ClientGroups/{key}` | Check `CreateClientGroup` schema for required fields |
| **Comments** | `GET /v3/Comments('{key}')` | Read-only; OData-style key syntax |
| **Contacts** | `GET`, `POST /v3/Contacts`, `GET/PUT/PATCH /v3/Contacts/{key}` | Required: `FirstName`, `LastName`. Associate with an Organization via `OrganizationKey` on a BusinessCard (see Business Cards). |
| **Custom Fields** | `GET/PUT /v3/CustomFieldValues/{EntityKey}`, `GET/POST /v3/CustomFields`, `DELETE /v3/CustomFields/{key}` | EntityKey is the key of the Contact/Org. Field types: `Text`, `Number`, `Date`, `Boolean`, `Colleague`, `ListSingleSelect`, `ListMultipleSelect`. `Colleague` fields store/return a UserKey (also referred to as UserID elsewhere in the API). |
| **Estimate Summaries** | `GET /v3/EstimateSummaries/{WorkItemKey}` | Read-only. Returns per-user `HourlyRate`, `EstimateMinutes`, `ActualMinutes`. Not writable via API. |
| **Files** | `GET /v3/FileList/{EntityType}`, `GET /v3/Files`, `POST /v3/Files` | EntityType in path for listing |
| **Integrated Workflows** | `GET /v3/IntegrationTaskDefinitions`, `GET/PUT /v3/IntegrationTasks/{key}` | Restricted to approved integration partners |
| **Notes** | `POST /v3/Notes`, `GET /v3/Notes/{id}` | Required: `Subject`, `Body` (HTML supported), `AuthorEmailAddress` |
| **Organizations** | `GET`, `POST /v3/Organizations`, `GET/PUT/PATCH /v3/Organizations/{key}` | Required: `FullName` |
| **Tags** | _(no paths in spec — beta, not enabled for all users)_ | — |
| **Tenant Settings** | `GET /v3/TenantSettings` | Returns valid `ContactTypes`, `WorkTypes`, `WorkStatuses`, `TenantKey`, `ClientAccessActivated` |
| **Timesheets** | `GET /v3/Timesheets`, `GET /v3/Timesheets/{key}` | Expand `TimeEntries` for detail |
| **Users** | `GET /v3/Users`, `POST /v3/Users`, `GET /v3/Users/{id}` | — |
| **Webhook Subscriptions** | `POST/DELETE /v3/WebhookSubscriptions`, `GET/DELETE /v3/WebhookSubscriptions/{type}` | One subscription per entity type; 10 retries then auto-cancelled |
| **Work Items** | `GET`, `POST /v3/WorkItems`, `GET/PUT/PATCH /v3/WorkItems/{key}` | Most filterable resource |
| **Work Schedules** | `POST /v3/WorkSchedules`, `GET/PUT /v3/WorkSchedules/{key}` | For repeating work |
| **Work Templates** | `GET /v3/WorkTemplates`, `GET /v3/WorkTemplates/{key}` | Read-only |

---

## 5. Key Concepts

### ClientType values

Used when creating or referencing Work Items:

`Contact` | `Organization` | `ClientGroup`

---

### WorkItem required fields (POST and PUT)

`AssigneeEmailAddress`, `Title`, `ClientKey`, `ClientType`, `StartDate`

PATCH only supports `Description` and `DeadlineDate`.

### WorkItem fee settings

Set billing behaviour via `FeeSettings` in POST/PUT body:

| `FeeType` | `FeeValue` |
|-----------|-----------|
| `FixedFee` | The fee amount (decimal) |
| `TimeAndMaterials` | Must be `null` |
| `NonBillable` | Must be `null` |

Hourly rates and time estimates (from `EstimateSummaries`) are **read-only** — not writable via the API.

---

### PrimaryStatus vs WorkStatus

Both exist on WorkItems. **Prefer `PrimaryStatus` and `SecondaryStatus`** — these are the recommended fields for tracking work progress. `WorkStatus` is a legacy field.

**PrimaryStatus** values are fixed (not tenant-specific):

| Value | Meaning |
|-------|---------|
| `Planned` | Not yet started |
| `ReadyToStart` | Ready to begin |
| `InProgress` | Actively being worked on |
| `Waiting` | Blocked or waiting |
| `Completed` | Done |

**SecondaryStatus** values are tenant-customizable. Valid values come from `GET /v3/TenantSettings`.

---

### Webhook payload format

```json
{
  "ResourcePermaKey": "{EntityKey}",
  "ResourceType": "{EntityType}",
  "ActionType": "{ActionType}",
  "TimeStamp": "YYYY-MM-DDTHH:mm:ssZ"
}
```

Webhook entity types: `Contact` (also covers ClientGroups, Organizations), `Work`, `Note`, `User`, `IntegrationTask`, `Invoice`, `EstimateSummary`, `CustomField`

---

### ClientAccessActivated

Flag in TenantSettings indicating if Karbon for Clients (K4C) is enabled. The `ClientAccess` expand on Contacts is only meaningful when this is `true`.

---

## 6. Tips for Agents

- **Always fetch TenantSettings first** if you need WorkTypes, ContactTypes, or SecondaryStatus values — these are tenant-specific. PrimaryStatus values are fixed.
- **Use `UserDefinedIdentifier`** if you control entity creation; it enables reliable lookups without storing Karbon-generated keys.
- **Page with `@odata.nextLink`** — use the URL from the response directly, don't manually increment `$skip`.
- **`$expand` adds latency** — only request it when you need the nested data.
- **Notes body supports HTML** — plain text is fine but HTML formatting is available.
- **Tags are beta** — no API paths are currently exposed in the spec.
- **Colleague custom field values** store a UserKey (same as UserID). To resolve to a name, call `GET /v3/Users/{UserKey}` as a second step — the custom field response does not expand user details.
