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

---

## 7. Workflows

### Workflow 1 — Onboard a new client

**1. Create the Organization**
```
POST /v3/Organizations
{ "FullName": "Acme Corp" }
```
Save the returned `OrganizationKey`.

**2. Create the Contact**
```
POST /v3/Contacts
{
  "FirstName": "Jane",
  "LastName": "Smith",
  "BusinessCards": [{
    "IsPrimaryCard": true,
    "OrganizationKey": "{OrganizationKey}",
    "RoleOrTitle": "CFO",
    "EmailAddresses": ["jane@acme.com"]
  }]
}
```
Save the returned `ContactKey`.

**3. Create a Work Item for the client**
```
POST /v3/WorkItems
{
  "Title": "2025 Tax Return",
  "AssigneeEmailAddress": "advisor@firm.com",
  "ClientKey": "{ContactKey}",
  "ClientType": "Contact",
  "StartDate": "2025-07-01T00:00:00Z",
  "PrimaryStatus": "ReadyToStart"
}
```

---

### Workflow 2 — Pull timesheet data for active work

**1. Collect WorkItem keys for all in-progress statuses**

Run one paginated request per status — `PrimaryStatus` only supports `eq`:
```
GET /v3/WorkItems?$filter=PrimaryStatus eq 'ReadyToStart'&$top=100
GET /v3/WorkItems?$filter=PrimaryStatus eq 'InProgress'&$top=100
GET /v3/WorkItems?$filter=PrimaryStatus eq 'Waiting'&$top=100
```
Page each using `@odata.nextLink`. Collect all `WorkItemKey` values.

**2. Fetch timesheets for those WorkItems (batched)**
```
GET /v3/Timesheets?$filter=WorkItemKeys/any(x: x in ('{key1}', '{key2}', ...))&$expand=TimeEntries&$top=100
```
Page using `@odata.nextLink`. If key count is very large, chunk into multiple requests.

---

### Workflow 3 — Create and assign a Colleague custom field

**1. Create the field definition**
```
POST /v3/CustomFields
{
  "Name": "Relationship Manager",
  "Type": "Colleague",
  "IsVisibleToContacts": true
}
```
Save the returned `Key` as `{CustomFieldKey}`.

**2. Assign a value to a Contact**
```
PUT /v3/CustomFieldValues/{ContactKey}
{
  "EntityKey": "{ContactKey}",
  "CustomFieldValues": [{
    "Key": "{CustomFieldKey}",
    "Name": "Relationship Manager",
    "Type": "Colleague",
    "Value": ["{UserKey}"]
  }]
}
```

**3. Read back and resolve the user's name**
```
GET /v3/CustomFieldValues/{ContactKey}
```
Extract the `Value[0]` (a UserKey), then:
```
GET /v3/Users/{UserKey}
```

---

### Workflow 4 — Create a Work Item from a Work Template

**1. Find the template key**
```
GET /v3/WorkTemplates?$filter=Title eq 'Annual Tax Return'
```
Save the returned `WorkTemplateKey`.

**2. Create the Work Item referencing the template**
```
POST /v3/WorkItems
{
  "Title": "2025 Annual Tax Return — Acme Corp",
  "AssigneeEmailAddress": "advisor@firm.com",
  "ClientKey": "{ContactKey}",
  "ClientType": "Contact",
  "StartDate": "2025-07-01T00:00:00Z",
  "WorkTemplateKey": "{WorkTemplateKey}"
}
```
The template pre-populates tasks and structure. Required fields (`AssigneeEmailAddress`, `Title`, `ClientKey`, `ClientType`, `StartDate`) must still be provided.

---

### Workflow 5 — Add a monthly repeat schedule to a Work Item

**1. Create the Work Schedule**
```
POST /v3/WorkSchedules
{
  "CreatedFromWorkItemKey": "{WorkItemKey}",
  "RecurrenceFrequency": "Month",
  "CustomFrequencyMultiple": 1,
  "ScheduleStartDate": "2025-08-01T00:00:00Z",
  "ScheduleEndDate": null,
  "ScheduleDueDateMethod": "DaysFromStartDate",
  "ScheduleDueDateDays": 30,
  "PreventStartEndOnWeekend": true,
  "InitializeTasksBeforeStartDateUnits": "Days",
  "InitializeTasksBeforeStartDateMultiple": 7,
  "WorkItemTitleDefinition": "[{\"Text\":\"Monthly Bookkeeping \",\"Variable\":null,\"Format\":null,\"Offset\":0},{\"Text\":null,\"Variable\":\"RepeatPeriod\",\"Format\":\"DD MMM, YYYY - DD MMM, YYYY\",\"Offset\":0}]"
}
```
Save the returned `WorkScheduleKey`.

**2. Link the schedule back to the Work Item**
```
PUT /v3/WorkItems/{WorkItemKey}
{
  "AssigneeEmailAddress": "advisor@firm.com",
  "Title": "Monthly Bookkeeping — Acme Corp",
  "ClientKey": "{ContactKey}",
  "ClientType": "Contact",
  "StartDate": "2025-08-01T00:00:00Z",
  "WorkScheduleKey": "{WorkScheduleKey}"
}
```

> **`WorkItemTitleDefinition`** is a JSON array serialized as a string. Each element is a segment with `Text` (literal string) or `Variable` (dynamic value), plus `Format` (date format) and `Offset`. The example above produces titles like `"Monthly Bookkeeping 01 Jul, 2025 - 31 Jul, 2025"`.

---

### Workflow 6 — Subscribe to Invoice webhook

**1. Create the subscription**
```
POST /v3/WebhookSubscriptions
{
  "TargetUrl": "https://yourapp.example.com/webhooks/karbon",
  "WebhookType": "Invoice",
  "SigningKey": "your-signing-key-min-16-chars"
}
```
Only one subscription per `WebhookType` is allowed. If one already exists, delete it first.

**2. Handle incoming payloads**
```json
{
  "ResourcePermaKey": "{InvoiceKey}",
  "ResourceType": "Invoice",
  "ActionType": "Updated",
  "TimeStamp": "2025-07-01T10:00:00Z"
}
```
Respond with HTTP 2xx within the timeout — 10 failed deliveries auto-cancels the subscription.

**3. Check or remove the subscription**
```
GET    /v3/WebhookSubscriptions/Invoice
DELETE /v3/WebhookSubscriptions/Invoice
```

**4. Re-subscribe if auto-cancelled**

Poll `GET /v3/WebhookSubscriptions/Invoice` — a 404 means the subscription was cancelled. Re-POST to reinstate.

---

### Workflow 7 — Update contact information (email, phone, address)

Contact details (email, phone, address) are stored on the Contact's **BusinessCard**, not directly on the Contact record.

**1. Get the BusinessCardKey**
```
GET /v3/Contacts/{ContactKey}?$expand=BusinessCards
```
Find the relevant BusinessCard and save its `BusinessCardKey`.

**2. Update the BusinessCard**
```
PUT /v3/BusinessCards/{BusinessCardKey}
{
  "EntityType": "Contact",
  "EntityKey": "{ContactKey}",
  "IsPrimaryCard": true,
  "OrganizationKey": "{OrganizationKey}",
  "EmailAddresses": ["jane.smith@acme.com"],
  "PhoneNumbers": [
    { "Number": "+14155550100", "CountryCode": "US", "Label": "Work" }
  ],
  "Addresses": [
    {
      "AddressLines": "123 Main St",
      "City": "San Francisco",
      "StateProvinceCounty": "CA",
      "ZipCode": "94105",
      "CountryCode": "US",
      "Label": "Physical"
    }
  ]
}
```

> PUT replaces the entire BusinessCard — include all existing fields you want to keep, not just the changed ones. Valid `PhoneNumbers.Label` values: `Work`, `Mobile`, `Office`, `Fax`, `Home`, `Other`. Valid `Addresses.Label` values: `Physical`, `Mailing`, `Legal`, `Home`. Country codes are ISO 3166-1 alpha-2 (e.g. `US`, `AU`, `GB`).
