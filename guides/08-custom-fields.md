---
slug: custom-fields
sortOrder: 8
seo:
  title: Working with Custom Fields
  description: Define and manage custom fields on Contacts and Organizations to store structured data across types including text, numbers, dates, booleans, list selections, and Karbon user references, then read and update values per entity via the CustomFieldValues endpoint.
---

# Working with Custom Fields

Custom fields let you store additional structured data against Contacts and Organizations. They support several data types including free text, numbers, dates, booleans, lists, and references to Karbon users (Colleague fields).

> **Terminology note:** In the Karbon API, `Contact` refers to what Karbon calls a **Person** in the UI. `Organization` means the same in both. Throughout this guide, "entity" means either a Contact (Person) or an Organization.

## Field Types

| Type                 | Description                                     |
| -------------------- | ----------------------------------------------- |
| `Text`               | Free text string                                |
| `Number`             | Decimal number                                  |
| `Date`               | ISO 8601 date                                   |
| `Boolean`            | `true` or `false`                               |
| `Colleague`          | Reference to a Karbon user (stores a `UserKey`) |
| `ListSingleSelect`   | One value from a predefined list                |
| `ListMultipleSelect` | Multiple values from a predefined list          |

## Listing Custom Field Definitions

```http
GET https://api.karbonhq.com/v3/CustomFields
Authorization: Bearer {token}
AccessKey: {key}
```

This returns all field definitions, including their `Key` values you'll need to set values.

## Creating a Custom Field Definition

```http
POST https://api.karbonhq.com/v3/CustomFields
Authorization: Bearer {token}
AccessKey: {key}
Content-Type: application/json

{
  "Name": "Industry",
  "Type": "ListSingleSelect",
  "Options": ["Technology", "Healthcare", "Finance", "Legal", "Other"]
}
```

For a `Colleague` (user reference) field:

```json
{
  "Name": "Client Partner",
  "Type": "Colleague"
}
```

## Reading Custom Field Values for an Entity

```http
GET https://api.karbonhq.com/v3/CustomFieldValues/3Nb8jKpL5tQR
Authorization: Bearer {token}
AccessKey: {key}
```

Where `{EntityKey}` is the `ContactKey` (a Person in Karbon) or `OrganizationKey`.

Response:

```json
{
  "EntityKey": "3Nb8jKpL5tQR",
  "CustomFieldValues": [
    {
      "Key": "4rRS8QdFP5q7",
      "Name": "Industry",
      "Type": "ListSingleSelect",
      "Value": ["Technology"]
    },
    {
      "Key": "2hdBRYV62ZFp",
      "Name": "Client Partner",
      "Type": "Colleague",
      "Value": ["3v9YJmt55hLY"]
    }
  ]
}
```

## Setting Custom Field Values

PUT the entire set of custom field values for an entity. Omitting a field clears it:

```http
PUT https://api.karbonhq.com/v3/CustomFieldValues/3Nb8jKpL5tQR
Authorization: Bearer {token}
AccessKey: {key}
Content-Type: application/json

{
  "EntityKey": "3Nb8jKpL5tQR",
  "CustomFieldValues": [
    {
      "Key": "4rRS8QdFP5q7",
      "Name": "Industry",
      "Type": "ListSingleSelect",
      "Value": ["Healthcare"]
    },
    {
      "Key": "2hdBRYV62ZFp",
      "Name": "Client Partner",
      "Type": "Colleague",
      "Value": ["3v9YJmt55hLY"]
    }
  ]
}
```

## Resolving Colleague Field Values

Colleague fields store a `UserKey`, not a name. To resolve the user's name:

```http
GET https://api.karbonhq.com/v3/Users/3v9YJmt55hLY
Authorization: Bearer {token}
AccessKey: {key}
```

To find the right `UserKey` to assign, list users and filter:

```http
GET /v3/Users?$filter=EmailAddress eq 'marcus.okonkwo@example.com'
```

## Deleting a Custom Field Definition

Deleting a field definition removes the field and all its values from all entities:

```http
DELETE https://api.karbonhq.com/v3/CustomFields/{CustomFieldKey}
Authorization: Bearer {token}
AccessKey: {key}
```

This is irreversible — confirm before deleting.
