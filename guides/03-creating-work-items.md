# Creating a New Work Item

Work items represent the tasks and engagements you manage for clients. You can create them from scratch or from a work template that pre-populates tasks and settings.

## Required Fields

Every work item requires these fields:

| Field | Description |
|---|---|
| `Title` | Name of the work item |
| `AssigneeEmailAddress` | Email of the Karbon user responsible |
| `ClientKey` | Key of the associated Contact, Organization, or ClientGroup |
| `ClientType` | `Contact`, `Organization`, or `ClientGroup` |
| `StartDate` | ISO 8601 datetime string (e.g., `2026-04-01T00:00:00Z`) |

## Creating a Basic Work Item

```http
POST https://api.karbonhq.com/v3/WorkItems
Authorization: Bearer {token}
AccessKey: {key}
Content-Type: application/json

{
  "Title": "FY2026 Income Tax Return",
  "AssigneeEmailAddress": "marcus.okonkwo@example.com",
  "ClientKey": "7wPqXnT4mBjK",
  "ClientType": "Organization",
  "StartDate": "2026-04-01T00:00:00Z",
  "DeadlineDate": "2026-06-30T00:00:00Z"
}
```

A successful response returns the created work item including its generated `WorkItemKey`.

## Work Types and Statuses

Work types are tenant-specific. Fetch yours from `TenantSettings` before creating work:

```http
GET https://api.karbonhq.com/v3/TenantSettings
```

The response includes available `WorkTypes` and `WorkStatuses`. Use the exact string values when setting these fields on a work item.

The built-in `PrimaryStatus` values are:

| Value | Meaning |
|---|---|
| `Planned` | Not yet started |
| `Ready To Start` | Ready to begin |
| `In Progress` | Actively being worked on |
| `Waiting` | Blocked or awaiting external input |
| `Completed` | Done |

## Setting Fees

Record the way the work item should be billed using the `FeeSettings` object:

```json
{
  "FeeSettings": {
    "FeeType": "FixedFee",
    "FeeValue": 1500.00
  }
}
```

```json
{
  "FeeSettings": {
    "FeeType": "TimeAndMaterials",
    "FeeValue": null
  }
}
```

```json
{
  "FeeSettings": {
    "FeeType": "NonBillable",
    "FeeValue": null
  }
}
```

Note: Hourly rates and time estimates are read-only and cannot be set via the API.

## Creating from a Work Template

Work templates pre-populate tasks, assignments, and settings — great for recurring work like tax returns or audits.

### Step 1: Find the template

```http
GET https://api.karbonhq.com/v3/WorkTemplates
Authorization: Bearer {token}
AccessKey: {key}
```

You can filter by title:

```http
GET /v3/WorkTemplates?$filter=Title eq 'FY Income Tax Return'
```

This returns a list of templates with their `WorkTemplateKey` values. You can also sort templates to find the most frequently used:

```http
GET /v3/WorkTemplates?$orderby=NumberOfWorkItemsCreated desc
```

### Step 2: Create the work item using the template key

Add `WorkTemplateKey` to your request body. The template will pre-populate tasks and other settings:

```http
POST https://api.karbonhq.com/v3/WorkItems
Authorization: Bearer {token}
AccessKey: {key}
Content-Type: application/json

{
  "Title": "FY2026 Income Tax Return",
  "AssigneeEmailAddress": "marcus.okonkwo@example.com",
  "ClientKey": "7wPqXnT4mBjK",
  "ClientType": "Organization",
  "StartDate": "2026-04-01T00:00:00Z",
  "DeadlineDate": "2026-06-30T00:00:00Z",
  "WorkTemplateKey": "2vBsCfGk9hJD"
}
```

## Filtering Existing Work Items

Before creating work, you may want to check if similar work already exists for a client:

```http
GET /v3/WorkItems?$filter=ClientKey eq '7wPqXnT4mBjK' and WorkType eq 'Tax'
```

Other useful filters:

```http
# Work assigned to a specific person
GET /v3/WorkItems?$filter=AssigneeEmailAddress eq 'marcus.okonkwo@example.com'

# In-progress work only
GET /v3/WorkItems?$filter=PrimaryStatus eq 'In Progress'

# Work starting after a date
GET /v3/WorkItems?$filter=StartDate ge 2026-01-01
```

**Note:** `ClientKey`, `PrimaryStatus`, and `WorkScheduleKey` do not support `contains` — use `eq` only.

## Updating a Work Item

Use PUT to update a work item. PUT replaces the entire record — any fields you omit will be cleared. **Always GET the record first, apply your changes, then PUT it back** to avoid accidentally losing data:

```http
PUT https://api.karbonhq.com/v3/WorkItems/{WorkItemKey}
Authorization: Bearer {token}
AccessKey: {key}
Content-Type: application/json

{
  "Title": "FY2026 Income Tax Return",
  "AssigneeEmailAddress": "marcus.okonkwo@example.com",
  "ClientKey": "7wPqXnT4mBjK",
  "ClientType": "Organization",
  "StartDate": "2026-04-01T00:00:00Z",
  "DeadlineDate": "2026-06-30T00:00:00Z",
  "PrimaryStatus": "In Progress"
}
```

For partial updates, PATCH is supported but only for `Description` and `DeadlineDate`.
