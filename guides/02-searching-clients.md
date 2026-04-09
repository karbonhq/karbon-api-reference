---
slug: searching-clients
sortOrder: 2
seo:
  title: Searching for and Retrieving Client Details
  description: Search and filter across Karbon's three client entity types—Contacts, Organizations, and ClientGroups—using OData-style queries, retrieve full contact details including emails and phone numbers via $expand=BusinessCards, and update client records or look them up by your own identifiers.
---

# Searching for and Retrieving Client Details

Karbon has three client entity types: **Contacts** (individuals), **Organizations** (businesses), and **ClientGroups** (collections of contacts/orgs). This guide covers searching across all three and fetching full client details including contact information.

> **Terminology note:** In the Karbon API, `Contact` refers to what Karbon calls a **Person** in the UI. `Organization` and `ClientGroup` mean the same in both.

## Listing Clients

### Contacts

```http
GET https://api.karbonhq.com/v3/Contacts
Authorization: Bearer {token}
AccessKey: {key}
```

### Organizations

```http
GET https://api.karbonhq.com/v3/Organizations
Authorization: Bearer {token}
AccessKey: {key}
```

### Client Groups

```http
GET https://api.karbonhq.com/v3/ClientGroups
Authorization: Bearer {token}
AccessKey: {key}
```

## Filtering Results

Use `$filter` to search within results. The API uses OData-style filter syntax.

### Search by name (exact match)

```http
GET /v3/Contacts?$filter=FullName eq 'Priya Sharma'
```

### Search by name (partial match)

```http
GET /v3/Contacts?$filter=contains(FullName, 'Sharma')
```

### Search by email

```http
GET /v3/Contacts?$filter=contains(EmailAddress, 'priya.sharma')
```

### Combine conditions

```http
GET /v3/Contacts?$filter=contains(FullName, 'Sharma') and ContactType eq 'Individual'
```

### Filter Organizations

```http
GET /v3/Organizations?$filter=contains(FullName, 'Meridian')
```

**Note:** the `ClientGroups` endpoint only supports `eq` on `FullName` — `contains` is not available.

## Supported Filter Fields

| Endpoint            | Fields                                                   | Operators               |
| ------------------- | -------------------------------------------------------- | ----------------------- |
| `/v3/Contacts`      | `FullName`, `EmailAddress`, `PhoneNumber`, `ContactType` | `eq`, `contains`, `and` |
| `/v3/Organizations` | `FullName`, `EmailAddress`, `ContactType`                | `eq`, `contains`, `and` |
| `/v3/ClientGroups`  | `FullName`                                               | `eq` only               |

**Note:** Filters on `FullName` and `EmailAddress` are case insensitive for both Contacts and Organizations.

## Sorting Results

Append `$orderby` to control sort order:

```http
GET /v3/Contacts?$orderby=FullName
GET /v3/Contacts?$orderby=LastModifiedDateTime desc
```

Sorting by `LastModifiedDateTime desc` is useful for incremental sync — it puts the most recently changed records first. You can compare the `LastModifiedDateTime` on each record against what you last stored to skip unnecessary updates, and check the `LastModifiedDateTime` of the last record on a page to decide whether older records (on subsequent pages) are worth fetching at all.

## Retrieving a Single Client

Once you have a `ContactKey` (e.g., from a list response or a webhook payload), fetch the full record:

```http
GET https://api.karbonhq.com/v3/Contacts/{ContactKey}
Authorization: Bearer {token}
AccessKey: {key}
```

## Fetching Contact Details with `$expand=BusinessCards`

Contact details like email addresses, phone numbers, and physical addresses are not stored directly on the Contact record — they live on associated **BusinessCard** entities.

To retrieve them in a single request, use `$expand`:

```http
GET https://api.karbonhq.com/v3/Contacts/3Nb8jKpL5tQR?$expand=BusinessCards
Authorization: Bearer {token}
AccessKey: {key}
```

The response includes a `BusinessCards` array with full contact detail:

```json
{
  "ContactKey": "3Nb8jKpL5tQR",
  "FirstName": "Priya",
  "LastName": "Sharma",
  "BusinessCards": [
    {
      "BusinessCardKey": "4mWxYqN7rZPK",
      "EntityType": "Contact",
      "EntityKey": "3Nb8jKpL5tQR",
      "IsPrimaryCard": true,
      "OrganizationKey": "7wPqXnT4mBjK",
      "RoleOrTitle": "CFO",
      "EmailAddresses": [
        "priya.sharma@example.com"
      ],
      "PhoneNumbers": [
        { "Number": "2025550174", "CountryCode": "US", "Label": "Mobile" }
      ],
      "Addresses": [
        {
          "AddressLines": "14 Westbourne Terrace",
          "City": "London",
          "StateProvinceCounty": "England",
          "ZipCode": "W2 3UA",
          "CountryCode": "GB",
          "Label": "Physical"
        }
      ]
    }
  ]
}
```

The same `$expand=BusinessCards` pattern works on Organizations:

```http
GET /v3/Organizations/7wPqXnT4mBjK?$expand=BusinessCards
```

To retrieve all Contacts associated with an Organization, use `$expand=Contacts`:

```http
GET /v3/Organizations/7wPqXnT4mBjK?$expand=Contacts
```

## Retrieving Client Team Members

Client team membership is separate from custom fields and contact details. For Organizations, use `$expand=ClientTeam` to retrieve the assigned team members and their roles. For ClientGroups, the `ClientTeam` array is included inline in the response.

```http
GET /v3/Organizations/7wPqXnT4mBjK?$expand=ClientTeam
```

The `ClientTeam` array returns each member's `MemberKey` (a `UserKey`), `MemberType`, and `RoleType` (excerpt):

```json
{
  "ClientTeam": [
    {
      "MemberKey": "JTphCpQqQYg",
      "MemberType": "User",
      "RoleType": "ClientOwner"
    },
    {
      "MemberKey": "nRML2ngs7WJ",
      "MemberType": "User",
      "RoleType": "UserDefinedRole1"
    }
  ]
}
```

`RoleType` values include `ClientOwner`, `ClientManager`, `UserDefinedRole1`, and `UserDefinedRole2`.

## Updating Contact Details

To update a phone number, email, or address, you PUT the entire BusinessCard. First get the `BusinessCardKey` from the expanded contact, then:

PUT replaces the entire BusinessCard — any fields you omit will be cleared. **Always GET the BusinessCard first, apply your changes, then PUT it back** to avoid accidentally losing contact details:

```http
PUT https://api.karbonhq.com/v3/BusinessCards/4mWxYqN7rZPK
Authorization: Bearer {token}
AccessKey: {key}
Content-Type: application/json

{
  "EntityType": "Contact",
  "EntityKey": "3Nb8jKpL5tQR",
  "IsPrimaryCard": true,
  "OrganizationKey": "7wPqXnT4mBjK",
  "EmailAddresses": [
    "priya.sharma@example.com"
  ],
  "PhoneNumbers": [
    { "Number": "2025550174", "CountryCode": "US", "Label": "Mobile" }
  ],
  "Addresses": [
    {
      "AddressLines": "22 Canary Wharf",
      "City": "London",
      "StateProvinceCounty": "England",
      "ZipCode": "E14 5AB",
      "CountryCode": "GB",
      "Label": "Physical"
    }
  ]
}
```

**`EmailAddresses`:** Array of plain email address strings (no label)
**Supported phone labels:** Work, Mobile, Office, Fax, Home, Other
**Supported address labels:** Physical, Mailing, Legal, Home
**Country codes:** ISO 3166-1 alpha-2 (e.g., `US`, `AU`, `GB`)

## Looking Up Clients by Your Own Identifier

If you store your own identifiers against Karbon clients, you can look them up without needing to store Karbon's keys:

```http
GET /v3/Contacts/GetContactByUserDefinedIdentifier(UserDefinedIdentifier='MAG-C-0042')
GET /v3/Organizations/GetOrganizationByUserDefinedIdentifier(UserDefinedIdentifier='MAG-ORG-001')
GET /v3/ClientGroups/GetClientGroupByUserDefinedIdentifier(UserDefinedIdentifier='MAG-GRP-007')
```

The Contacts and Organizations endpoints support `$expand=BusinessCards` (plural). ClientGroups uses `$expand=BusinessCard` (singular):

```http
GET /v3/ClientGroups/GetClientGroupByUserDefinedIdentifier(UserDefinedIdentifier='MAG-GRP-007')?$expand=BusinessCard
```
